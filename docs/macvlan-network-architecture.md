# MACVLAN Network Architecture

MACVLAN is a Linux kernel feature that allows multiple virtual network interfaces to exist on top of a single physical NIC. Each virtual interface behaves like an independent device on the LAN, complete with its own MAC address and IP. This makes MACVLAN ideal for network‑appliance containers that need stable identity and full participation in the broadcast domain.

Understanding how MACVLAN works at Layer 2 (L2) explains both its strengths and its limitations, including why the host cannot reach MACVLAN containers by default and why boot ordering matters.

---

## How MACVLAN Works at Layer 2

MACVLAN creates virtual interfaces that attach directly to a parent interface (e.g., `eth0`). Each virtual interface:

- has its own MAC address,
- participates in ARP, broadcast, and multicast,
- appears to the LAN as a separate device,
- bypasses Docker’s NAT and bridge layers entirely.

From the LAN’s perspective, each MACVLAN container looks like a physical device plugged into the same switch port as the host.

### Key architectural behaviors

- The parent interface forwards frames based on MAC address.
- The kernel filters traffic so the host does not receive frames destined for MACVLAN interfaces.
- Containers communicate directly with the LAN at L2 and L3.
- Containers do **not** communicate with the host unless a host‑reachability shim is added.

This architecture is what makes MACVLAN powerful for appliances and restrictive for everything else.

---

## Why the Host Cannot Reach MACVLAN Containers

MACVLAN isolates the host from its own MACVLAN interfaces. This is not a bug — it is a design choice.

### The kernel enforces L2 separation

When a MACVLAN interface is created:

- the host stops receiving frames destined for the MACVLAN MACs,
- the MACVLAN interfaces stop receiving frames destined for the host MAC,
- the parent interface becomes a forwarding device, not a shared L2 domain.

This prevents MAC spoofing loops and ensures clean separation between virtual interfaces.

### Resulting behavior

- Containers can reach the LAN.
- LAN devices can reach containers.
- Containers cannot reach the host.
- The host cannot reach containers.

This is why a host‑reachability shim exists — but only for deployments where the host must act as a client.

---

## Why MACVLAN Requires a Stable Parent Interface

MACVLAN interfaces attach to a specific parent interface (e.g., `eth0`, `enp3s0`, `eno1`). If the parent interface:

- changes names,
- comes up late in the boot process,
- is temporarily missing,
- is replaced by a different NIC,

then Docker may:

- create a ghost MACVLAN network,
- attach containers to a stale network ID,
- assign incorrect MAC/IP identities,
- break DHCP reservations.

This is why the boot‑ordering shim exists: it ensures the parent interface is created before Docker starts.

---

## Why MACVLAN Is Ideal for Network Appliances

Network appliances expect:

- stable MAC identity,
- stable IP identity,
- full broadcast/multicast participation,
- direct L2 adjacency,
- DHCP reservations tied to MAC,
- no NAT,
- no port conflicts.

MACVLAN provides all of these without requiring:

- a VM,
- a dedicated NIC,
- host networking,
- port‑mapping,
- or complex routing.

This makes MACVLAN the cleanest way to run appliance‑style containers on a single host.

---

## When MACVLAN Is the Wrong Tool

MACVLAN is not appropriate when:

- the host must reach the container (unless a shim is added — see *macvlan-shims-and-when-to-use-them.md*)
- the switch restricts multiple MACs per port,
- the platform wraps Docker inside Kubernetes (TrueNAS SCALE),
- the platform has known MACVLAN issues (Unraid kernel panics),
- the host is a workstation that needs to access appliance UIs,
- the environment uses NIC bonding modes incompatible with MACVLAN.

In these cases, IPVLAN or a VM may be more appropriate.

---

## Summary

MACVLAN provides virtual interfaces that behave like independent physical devices on the LAN. This architecture gives containers their own MAC and IP, full L2 participation, and clean identity — all of which are essential for network appliances. The same architecture also isolates the host from MACVLAN containers and requires stable boot ordering to avoid ghost networks.

Understanding these behaviors is key to deploying MACVLAN reliably and choosing the right shim pattern
