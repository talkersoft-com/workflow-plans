Master Requirements — Networking (KVM / libvirt Management UI)

1. Purpose & Scope

Define all networking capabilities the system must provide to manage KVM/libvirt virtual machines in production. The system is a full UI integrated with Ansible: the UI presents and validates network configuration, and Ansible (plus libvirt) applies it. Networking entities are stored, reusable, and validated — consistent with the rest of the platform's reuse-first philosophy.

Two distinct layers must be respected throughout:


Host networking (physical NICs, host bridges) — configured via netplan / systemd-networkd / NetworkManager on the host OS.
libvirt virtual networking (NAT, routed, open, isolated) — configured through libvirt's own definitions.


The system must never blur these layers in the UI; the boundary between them drives most of the validation and footgun-prevention requirements below.


2. Supported Network Modes

The system must support, per-VM and per-NIC, the following modes, with the tradeoff (LAN visibility vs. isolation) surfaced inline in the UI:


NAT — libvirt default. VMs share the host's outbound IP, sit on a private subnet, libvirt provides a virtual router + DHCP/DNS. Good isolation; no inbound from LAN without explicit forwarding.
Routed — VMs get LAN-routable addresses, traffic routes through the host (no NAT). LAN can reach VMs; host stays in the path.
Bridged — VM attaches directly to a physical NIC via a host bridge; appears as a first-class device on the LAN; addressed by the LAN's own DHCP. This is the "entire network can see them" case.
Isolated — private virtual network; VMs talk to each other but have no route off-host. The "fully isolated" case.
Open — libvirt creates and owns the bridge (like routed) but does not insert firewall/iptables rules; the operator supplies their own filtering (or none). This is "managed bridge, unmanaged filtering" — not "no networking."


Requirements:


Mode is selectable per NIC, not just per VM (a VM may have one bridged NIC + one isolated NIC).
The UI explains each mode's visibility/isolation tradeoff at the point of selection.
Switching modes triggers re-validation of dependent settings (DHCP, addressing, firewall).



3. Host ↔ libvirt Layer Boundary


Host bridges are a host-OS concern. A bridged VM requires a pre-existing host bridge over a physical NIC (e.g. br0), created by netplan / systemd-networkd / NetworkManager. libvirt's bridged network only references that bridge.
v1 assumption: pre-existing host bridges. The system references and validates host bridges (exists, is up, backed by a real NIC) rather than creating them. Document the expected setup ("create br0 in netplan, point the system at it").
Host bridge management is an opt-in, later capability. If/when the system creates or edits host bridges, it does so by generating host network config (e.g. netplan YAML) via Ansible — a clearly separated, higher-privilege, riskier surface that must be explicitly enabled and audited.
Clear UI separation. Host-network objects and libvirt-network objects are visually and conceptually distinct; the UI never implies one "attaches to" the other except at the legitimate bridged-mode reference point.



4. Virtual Networks as Managed, Reusable Objects


Named virtual networks as first-class DB entities (mirroring libvirt <network> definitions), reusable across VMs — referenced, never copy-pasted.
Per-network attributes: forwarding mode, bridge name, subnet/CIDR, gateway, DHCP range, static reservations, DNS, domain name, autostart.
Versioning & history on network definitions, consistent with the rest of the platform.
"Used-by" reverse index — list every VM/NIC attached to a network before allowing edit or delete; block or warn on deletion of an in-use network (prevent the stale-reference / dangling-config failure class).
Autostart management per network.



5. DHCP & Addressing

DHCP responsibility forks by mode; the system must handle both paths and prevent the classic two-DHCP-servers footgun.

libvirt-managed networks (NAT, routed, isolated):


libvirt provides DHCP/DNS via a per-network dnsmasq instance (auto-launched and lifecycle-managed by libvirt — the system does not run dnsmasq directly).
The system generates the network definition's <dhcp> block: a dynamic <range> plus <host> static reservations (MAC → IP).
Each network has an independent DHCP scope; isolated networks do not cross-talk.
The system reads the lease table (/var/lib/libvirt/dnsmasq/<network>.leases) to display each VM's current IP.


Bridged networks (VM on the physical LAN):


libvirt is not in the addressing path; the VM is served by the existing LAN DHCP (router / pfSense / OPNsense / Windows DHCP, etc.).
The system must not offer or write a <dhcp> block here — the UI hides/grays DHCP config and states "addressing handled by your LAN DHCP." Two DHCP servers on one L2 segment is an error condition to prevent.
Static reservations for bridged VMs live on the upstream DHCP server (outside the system's reach); the system can only display the leased IP, not assign it.


Cross-cutting addressing:


Static reservation management (MAC → IP) for libvirt-managed networks via <host> entries.
Local DNS resolution for VM hostnames (dnsmasq integration) so VMs are reachable by name; per-network domain.
Live IP display for every VM regardless of mode — from the lease file where available, and from qemu-guest-agent when there's no lease file to read (e.g. bridged, or isolated with agent).
Validation: subnet/CIDR sanity, range-within-subnet, no overlap with other managed networks, gateway inside subnet.



6. Bridging Specifics


Distinguish bridge types: Linux bridge, macvtap, Open vSwitch (OVS) bridge — and surface their behavioral differences (notably: macvtap cannot do host↔guest traffic by default).
Show and validate the physical NIC backing each bridge; given strict NIC requirements, validate the NIC exists and is up before allowing a VM to attach.
VLAN tagging (802.1Q) per interface/network — place a VM on a tagged VLAN without dedicated physical NICs.
Bonded / teamed NICs as bridge uplinks for redundancy.
Pre-attach validation catches the common bridged-mode failures: missing bridge, down uplink, wrong NIC, nonexistent VLAN.



7. Per-VM / Per-NIC Interface Management


Multiple NICs per VM, each independently configured (mode, network, MAC, model, QoS).
MAC address assignment and locking (stable MACs for DHCP reservations and licensing); duplicate-MAC detection across the managed estate.
NIC model selection: virtio (performance) vs. emulated e1000/rtl8139 (compatibility).
Hotplug / unplug NICs on running VMs.
Link state up/down toggle per NIC.
Bandwidth / QoS limits — inbound/outbound rate caps (libvirt-supported).



8. Isolation, Segmentation & Security


Per-network firewall / filtering via libvirt nwfilter — anti-spoof, MAC/IP locking, custom rule sets, reusable filter definitions.
Inter-VM isolation toggles even within a shared network.
Port forwarding / DNAT for NAT networks (e.g. expose VM:80 as host:8080), managed as part of the network definition.
Network zones / segments — enforce that, e.g., test machines cannot reach prod networks.
Open mode explicitly requires the operator to supply filtering; the UI flags that no libvirt-managed firewall rules are in place.
Least-privilege handling for any host-level network changes; all mutating network actions audited.



9. Ansible Integration


Network configuration authored/validated in the UI is applied via Ansible (and libvirt), keeping the apply path idempotent and auditable.
Network definitions, nwfilters, and host-bridge configs are reusable Ansible-managed entities with idempotency guards.
Dry-run / check-mode support before applying network changes; show a diff of what will change.
Secrets touched by networking (e.g. credentials for upstream services) follow the platform's secret-handling rules — referenced via Vault, never stored plaintext.
Long-running apply operations use the platform's async job_id pattern with streamed output and status polling.



10. Validation & Footgun Prevention

The system must validate before apply and refuse or warn on:


Subnet/CIDR overlaps across managed networks.
DHCP range outside its subnet; gateway outside subnet.
DHCP configured on a bridged network (two-DHCP-server condition).
Duplicate MAC addresses across VMs.
Missing or down uplink NIC / nonexistent host bridge for bridged mode.
Nonexistent VLAN or invalid VLAN tag.
Deleting a virtual network, bridge, or nwfilter still in use (reverse-index check).
Open mode with no operator-supplied filtering (warn, since it's intentional but risky).



11. Observability & Operations


Live per-NIC stats (rx/tx), link state, current IP, attached network.
Lease table view per libvirt network; guest-agent IP fallback for bridged/agent cases.
Topology view — which VMs sit on which networks, with mode and reachability indicated.
Connectivity / reachability tests between VMs and to/from the LAN.
Audit trail for every network change: who, what, which VM/network, before/after, applied via which Ansible run (secrets redacted).
Health surfacing: down uplinks, dnsmasq failures, missing bridges, lease exhaustion.



12. Cross-Cutting Requirements


Reuse-first. Networks, nwfilters, and (later) host-bridge configs are reusable, versioned, reverse-indexed entities — same model as roles elsewhere in the platform.
Layer integrity. Host vs. libvirt networking never conflated in UI or data model.
Safety by default. v1 references existing host bridges and validates rather than mutating host network config; host-bridge management is explicit opt-in.
Consistency with platform. Async execution (job_id), Vault-based secret references, idempotent Ansible apply, full audit, and round-trip-safe config generation all apply here as elsewhere.

Share



5a. Network Definition Lifecycle & Ansible Apply

This section defines how network definitions are stored, rendered, and applied to hosts. The app is the source of truth, libvirt holds the live definition, and Ansible is the apply mechanism. Network definitions are never written directly into libvirt's on-disk files.

Canonical store, not file-editing.


The app stores each network definition as structured data (mode, subnet/CIDR, gateway, DHCP range, reservations, DNS, domain, autostart), not as raw XML.
libvirt XML is rendered from that structured data at apply time (templated — leveraging the platform's reusable-entity model).
The system must never hand-edit files under /etc/libvirt/qemu/networks/. libvirt caches definitions in memory and overwrites out-of-band file changes. All changes flow through libvirt via its proper commands.


Apply path (app → render → Ansible → libvirt).


Apply is performed with the native community.libvirt.virt_net Ansible module, not by writing files.
Standard flow per host: define (from rendered XML) → state: active → autostart: yes.
The module is idempotent on presence/state, so re-applying a definition is safe.
Long-running applies use the platform's async job_id pattern with streamed output and status polling.


Template vs. per-host instance (required scoping model).


Each hypervisor host runs its own libvirt daemon with its own network definitions; a network named test-net on host A is a distinct object from test-net on host B.
The system models networks in two tiers:

Reusable network templates — the abstract definition (mode, subnet, DHCP range, reservations), defined once.
Per-host instantiations — that template realized on a specific host, with host-specific bindings resolved (backing bridge/NIC, actual virbr name, per-host autostart).



State is tracked per (host, network-definition) pair; each instance has its own live status.


Live-edit disruption rules.


In-place modification of a running network is constrained in libvirt and must be treated as potentially disruptive.
Use virsh net-update (via the module/command) for changes it supports live — notably adding/removing DHCP <host> reservations without bouncing the network.
Treat structural changes (subnet, mode, bridge/NIC) as disruptive: they typically require net-destroy → net-undefine → net-define → net-start, which drops connectivity for all VMs on that network during the cycle.
Before applying a disruptive change, the system must: (1) surface the attached-VM list via the reverse index, (2) warn that those VMs will lose connectivity, and (3) gate the change behind explicit confirmation.


DHCP reservation edits (preferred live path).


Adding/removing static <host> MAC→IP reservations on a libvirt-managed network is done live via net-update, without a destroy/redefine cycle — this is the common, low-risk edit and should be a first-class, non-disruptive operation in the UI.


example edit this directly from the ui..
<network>
  <name>test-net</name>
  <bridge name='virbr1'/>
  <forward mode='nat'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.50' end='192.168.100.200'/>
      <host mac='52:54:00:ab:cd:01' name='vm1' ip='192.168.100.10'/>
    </dhcp>
  </ip>
</network>