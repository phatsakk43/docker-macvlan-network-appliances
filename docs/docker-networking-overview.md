# Docker Networking Overview

Network‑appliance containers behave differently from normal application containers, and understanding why requires a basic grasp of Docker’s networking modes. Docker supports several network drivers, each with different behavior at Layer 2 (L2) and Layer 3 (L3). This document gives a concise overview of the modes most relevant to running network appliances.

---

## Bridge Networking

Bridge is Docker’s default network mode. Containers receive an internal IP on a private subnet, and Docker performs NAT when they talk to the outside world.

**Characteristics**

- Containers get private IPs (e.g., 172.x.x.x)
- Outbound traffic is NATed through the host
- Inbound traffic requires port‑mapping (`-p 80:80`)
- No broadcast or multicast from the LAN reaches the container
- Containers cannot advertise their own MAC address

**Good for**

- Applications (Plex, Transmission, Sonarr, Radarr, Nextcloud)
- Anything that only needs port‑mapping
- Services that don’t care about L2 identity

**Not good for**

- DNS servers, controllers, discovery‑based services  
- Anything that needs to appear as a real device on the LAN

---

## Host Networking

Host networking places the container directly inside the host’s network namespace.

**Characteristics**

- Container shares the host’s IP address
- No port‑mapping — container binds directly to host ports
- Full access to broadcast/multicast
- No isolation between host and container

**Good for**

- High‑performance workloads that need raw network access
- Some controllers that rely heavily on broadcast

**Not good for**

- Running multiple appliances on the same host  
- Avoiding port conflicts  
- Maintaining isolation  
- Anything where the container should have its own identity

**Important note**

This is why the `mbentley/omada-controller` image causes issues by default — it uses host networking, so it *becomes* the server.

---

## MACVLAN Networking

MACVLAN creates virtual interfaces that behave like separate physical NICs. Each container gets its own MAC address and IP on the LAN.

**Characteristics**

- Each container has a unique MAC and IP
- Full L2 participation (ARP, broadcast, multicast)
- No NAT
- No port conflicts
- Containers appear as real devices on the LAN
- Host cannot reach MACVLAN containers without a shim

**Good for**

- DNS servers (AdGuard, Pi‑hole)
- Network controllers (Omada, UniFi)
- Home Assistant
- Syncthing
- Anything that needs stable identity and L2 adjacency

**Not good for**

- Situations where the host must reach the container without a shim
- Networks where the switch blocks multiple MACs per port

---

## IPVLAN Networking

IPVLAN is similar to MACVLAN but does not create new MAC addresses. Instead, all containers share the parent interface’s MAC and get unique IPs.

**Characteristics**

- All containers share the host’s MAC
- Containers get unique IPs
- Works better on networks that restrict MAC spoofing
- L2 mode and L3 mode behave differently

**Good for**

- Environments where MACVLAN is blocked by the switch
- Situations where you want L3 separation without extra MACs

**Not good for**

- Appliances that rely on unique MAC identity
- DHCP servers or services tied to MAC reservations

---

## None Networking

The `none` driver disables networking entirely.

**Characteristics**

- No network interface
- No connectivity

**Good for**

- Highly isolated workloads
- Containers that communicate only via IPC or mounted volumes

---

## Overlay Networking (Swarm)

Overlay networks span multiple Docker hosts and are used primarily in Swarm mode.

**Characteristics**

- Multi‑host networking
- VXLAN encapsulation
- Not relevant for single‑host appliance setups

**Good for**

- Distributed applications
- Multi‑node clusters

**Not good for**

- Local network appliances
- Simple home‑lab setups

---

## Summary Table

| Driver     | L2 Identity | LAN Visibility | Port Conflicts | Host ↔ Container | Best For |
|------------|-------------|----------------|----------------|------------------|----------|
| Bridge     | No          | No             | Possible       | Yes              | Apps |
| Host       | Host’s MAC  | Yes            | Yes            | N/A              | Performance workloads |
| MACVLAN    | Yes         | Yes            | No             | No (needs shim)  | Network appliances |
| IPVLAN     | Shared MAC  | Yes            | No             | No (needs shim)  | MAC‑restricted networks |
| None       | No          | No             | No             | No               | Isolation |
| Overlay    | Virtual     | Virtual only   | No             | Yes              | Swarm clusters |

---

## Why This Matters for This Project

Network appliances rely on L2 behavior — ARP, broadcast, multicast, and stable MAC/IP identity. Bridge networking hides them. Host networking collapses them into the host. MACVLAN gives them the identity they expect, and the rest of this repo documents how to make that setup stable and reproducible.
