# Plan: Feature Flag Health Endpoint — Close the Probe Gap

## Objective

Wire up the missing health-write path so that vorch-service's ansible-runner health probe can actually persist its `healthy` + `last_probed_at` result back to the API. Today the probe fires every 5 minutes and silently fails on every tick because the endpoint it calls does not exist. The end state is a fully operational three-state feature-flag model: disabled / enabled-but-unhealthy / enabled-and-healthy.

## Background

A gap analysis of the ansible plans vs. the live codebase (2026-05-28) found one blocking issue.

**What vorch-service does** (`vorch-service/healthprobe/probe.go`):
```
PATCH /api/v1/admin/feature-flags/playbooks
Body: { "healthy": true, "last_probed_at": "2026-05-28T..." }
```

**What the API actually has** (`FeatureFlagsController`):
```
PATCH /api/v1/featureflags/{key}
Body: { "Enabled": bool }   ← only accepts enabled, not healthy/lastProbedAt
```

Two mismatches:
1. **URL path**: probe calls `/api/v1/admin/feature-flags/...`, controller is at `/api/v1/featureflags/...`
2. **Body shape**: probe sends `{healthy, last_probed_at}`, controller only reads `{Enabled}`

The entity and DTO models are correct — `FeatureFlag` already has `Healthy bool` and `LastProbedAt DateTime?` fields, and `IsEnabledAsync` already checks both `Enabled` and `Healthy`. The gap is purely in the write path.

**Everything else in the ansible bundle is fully implemented:**
- All 12 DB tables deployed, migration `AddAnsibleConstraintsAndTargetTimestamps` applied
- All API endpoints (roles, role files, revisions, playbook-local roles, materialization, multi-VM run, cancel)
- All service-layer hooks (auto-revisioning, ArgumentSpecsTranslator)
- vorch-service consumers, per-target flow, Vault two-token strategy, cancel handling
- vorch-lib shared types
- cloud-manager-web: all pages, components, Redux slices

## Design

Three changes, all in `cloud-manager-api`:

### 1. Expand `UpdateFlagRequest` to accept optional health fields

```csharp
// FeatureFlagsController.cs
public record UpdateFlagRequest(
    bool? Enabled,
    bool? Healthy,
    DateTime? LastProbedAt
);
```

Make `Enabled` nullable so health-only PATCH (from the probe) doesn't accidentally flip the enabled state.

### 2. Add `UpdateHealthAsync` to `IFeatureFlagService`

```csharp
// IFeatureFlagService.cs
Task<FeatureFlag> UpdateHealthAsync(string key, bool healthy, DateTime? lastProbedAt);
```

### 3. Route: add `/api/v1/admin/feature-flags/{key}` alias

The probe calls the `admin/` prefix path; the web UI toggle calls the existing `/api/v1/featureflags/{key}`. Two options:

**Option A (recommended):** Add a second route attribute on the existing PATCH action:
```csharp
[HttpPatch("{key}")]
[Route("api/v1/admin/feature-flags/{key}")]  // alias for probe
public async Task<IActionResult> UpdateFlag(string key, [FromBody] UpdateFlagRequest request)
```

**Option B:** Add a dedicated `AdminFeatureFlagsController` at the `admin/` prefix. More explicit but unnecessary layering for one endpoint.

Recommendation: Option A — single action, two route attributes, no controller duplication.

### Controller logic after changes

```csharp
[HttpPatch("{key}")]
[Route("api/v1/admin/feature-flags/{key}")]
public async Task<IActionResult> UpdateFlag(string key, [FromBody] UpdateFlagRequest request)
{
    if (request.Enabled.HasValue)
        updated = await _service.UpdateEnabledAsync(key, request.Enabled.Value);
    if (request.Healthy.HasValue)
        updated = await _service.UpdateHealthAsync(key, request.Healthy.Value, request.LastProbedAt);
    return Ok(updated);
}
```

Both fields optional — either can be sent alone or together.

### `FeatureFlagService.UpdateHealthAsync` implementation

```csharp
public async Task<DtoModel.FeatureFlag> UpdateHealthAsync(string key, bool healthy, DateTime? lastProbedAt)
{
    var flag = await _context.FeatureFlags.FirstOrDefaultAsync(f => f.Key == key)
        ?? throw new KeyNotFoundException($"Feature flag '{key}' not found");
    flag.Healthy = healthy;
    flag.LastProbedAt = lastProbedAt ?? DateTime.UtcNow;
    await _context.SaveChangesAsync();
    _logger.LogInformation("Feature flag {Key} health updated: Healthy={Healthy}", key, healthy);
    return _mapper.Map<DtoModel.FeatureFlag>(flag);
}
```

No migration needed — `Healthy` and `LastProbedAt` columns already exist on the `feature_flags` table.

## Implementation phases

| Phase | Title | Description |
|-------|-------|-------------|
| 0000 | Setup | hv_status + hv_init/hv_next |
| 0001 | Service layer | Add `UpdateHealthAsync` to `IFeatureFlagService` + `FeatureFlagService`; make `UpdateEnabledAsync` safe when called alongside health update |
| 0002 | Controller | Expand `UpdateFlagRequest` to nullable fields; add `/api/v1/admin/feature-flags/{key}` route alias; update action to dispatch to correct service methods |
| 0003 | Tests + smoke | Unit tests for new service method; integration test that confirms probe body `{healthy, last_probed_at}` round-trips correctly; manually trigger health probe and verify `GET /api/v1/featureflags` reflects the update |

## Open questions

- **Auth on the admin route**: The probe hits `/api/v1/admin/feature-flags/...` with no auth token today (vorch-service has no API key concept). If the API grows auth middleware on the `admin/` prefix, the probe will need a service token. Recommend adding a `CLOUD_MANAGER_API_KEY` env var to vorch-service and a bearer-token check on the admin route as a Phase 0004 follow-on once auth lands elsewhere.
