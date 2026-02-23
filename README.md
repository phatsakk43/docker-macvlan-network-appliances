# Running Network‑Appliance Containers on Docker Using MACVLAN  
### A reproducible pattern with Omada Controller + AdGuard Home as examples

For about a year, I ran AdGuard Home in Docker using the standard bridge network. It worked flawlessly: stable, isolated, and simple. At the time, I was using a dedicated Omada hardware controller, so there were no conflicts. Everything changed when I replaced the hardware controller with the software version.

The Omada software controller uses host networking by default when deployed using the popular `mbentley/omada-controller` image. This is the setup most people follow, because it’s the first and most widely referenced container build. As soon as I deployed it, the controller effectively *became the server* from the LAN’s perspective. It bound to the host’s ports, took over the host’s network identity, and AdGuard instantly became unreachable. Bridge networking couldn’t fix it. Host networking made things worse.

I’ll also admit that this was the first time I truly understood the practical difference between L2 and L3 networking. Docker abstracts so much that you can run containers for years without thinking about ARP, broadcast domains, or how host networking collapses identity. This problem forced me to learn how these layers interact — and once I understood that, the solution became clear.

That was the moment I realized something important:

> **Network appliances don’t behave well when forced to share the host’s identity.**

This repo documents the pattern I eventually built to solve that problem: using Docker MACVLAN networks, a systemd‑managed shim interface, and deterministic MAC/IP identity to run network‑appliance containers as first‑class LAN devices—without VMs, without dedicated hardware, and without sacrificing the benefits of Docker. 

For a brief overview of Docker networking modes (bridge, host, macvlan, ipvlan, etc.), see  
[Docker Networking Overview](docs/docker-networking-overview.md).

---

## What counts as a “network appliance”?

Some containers behave like *applications*: Plex, Transmission, Sonarr, Radarr, Nextcloud, etc. These only need a port or two, don’t care about MAC/IP identity, and work perfectly on the default Docker bridge network.

Other containers behave like *network appliances*. These expect to be real devices on the LAN:

- AdGuard Home  
- Pi‑hole  
- Omada Controller  
- UniFi Controller  
- Home Assistant  
- Syncthing  
- Anything that relies on broadcast/multicast discovery  
- Anything that needs a stable MAC/IP for DHCP reservations  
- Anything that binds privileged ports or expects L2 adjacency  

These services break in subtle ways when forced into bridge networking (NAT, no broadcast) or host networking (identity collapse, port conflicts).

MACVLAN is the middle ground that gives them what they need.

---

## Why MACVLAN solves the problem

MACVLAN allows each container to behave like a bare‑metal device on your LAN:

- **Unique MAC address**  
- **Unique IP address**  
- **Full LAN participation** (ARP, broadcast, multicast)  
- **No NAT**  
- **No port collisions**  
- **No need for a VM or dedicated hardware**  

This is exactly what network appliances want.

But MACVLAN has sharp edges:

- The host cannot talk to MACVLAN containers by default  
- Docker must start *after* the shim interface exists  
- Compose caches network IDs  
- Ghost networks can break container startup  
- Identity must be deterministic  

This repo documents the fixes for all of these.

---

## The pattern: MACVLAN + systemd shim + deterministic identity

This repo includes:

- A working `docker-compose.yml` for MACVLAN‑based appliances  
- A `macvlan-shim.service` systemd unit that restores host ↔ container communication  
- A clean boot‑ordering model (shim → Docker → Compose)  
- A reproducible MAC/IP identity scheme  
- Troubleshooting steps for ghost networks and stale metadata  
- Example implementations for Omada Controller and AdGuard Home  

The result is a stable, reboot‑proof pattern that behaves like dedicated hardware without the overhead.

---

## When to use this pattern

Use this pattern when:

- You want a container to behave like a real LAN device  
- You need stable MAC/IP identity  
- You want to avoid port collisions  
- You want to avoid host networking  
- You want to avoid running a full VM  
- You want clean separation between appliances and applications  

Do **not** use this pattern for simple apps that only need a port or two. Bridge networking is simpler and more appropriate for those.

---

## Repository structure

```text
.
├── README.md
├── docker-compose.yml
├── systemd/
│   └── macvlan-shim.service
└── docs/
    ├── docker-networking-overview.md
    ├── what-is-a-network-appliance.md
    ├── why-macvlan.md
    ├── the-shim-and-host-reachability.md
    ├── boot-order-and-systemd.md
    ├── pitfalls-and-recovery.md
    ├── omada-example.md
    └── adguard-example.md
```

---

## Example: Omada Controller + AdGuard Home

These two services make a perfect demonstration of the pattern:

- **Omada Controller** wants to be a controller on the LAN.  
- **AdGuard Home** wants to be a DNS server with a stable identity.  
- Host networking makes Omada take over the host.  
- Bridge networking hides AdGuard behind NAT.  
- MACVLAN gives both their own MAC/IP and avoids all conflicts.

The included Compose file shows how to run both cleanly on the same host.

---

## Goals of this repo

- Document a reproducible MACVLAN pattern for network appliances  
- Explain the pitfalls and how to avoid them  
- Provide working examples for Omada and AdGuard  
- Help others avoid the “host networking takeover” problem  
- Provide a clean alternative to VMs or dedicated hardware  
