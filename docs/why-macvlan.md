# Why MACVLAN Works for Network Appliances

Network‑appliance containers behave like real devices on the LAN. They expect to send and receive broadcast traffic, advertise their own MAC address, participate in ARP, and maintain a stable identity that other devices can rely on. Docker’s default networking modes were not designed for this behavior, which is why appliances break in subtle and frustrating ways when run on bridge or host networking.

MACVLAN is the only Docker networking mode that gives containers the L2 identity and adjacency they expect, without collapsing them into the host or hiding them behind NAT. This document explains why.

---

## The Limitations of Bridge Networking

Bridge networking is Docker’s default mode. It works perfectly for applications because it provides:

- NAT isolation  
- predictable port mapping  
- simple firewall behavior  
- no need for a unique MAC/IP  

But these same features break appliances:

- NAT hides the container behind the host  
- broadcast and multicast traffic do not reach the container  
- the container cannot advertise its own MAC address  
- DHCP reservations cannot be tied to the container  
- discovery protocols fail silently  
- the container appears as “a service on the host,” not a device  

This is why AdGuard worked fine until another appliance entered the picture. Bridge networking is fundamentally incompatible with services that expect L2 presence.

---

## The Limitations of Host Networking

Host networking solves the broadcast/multicast problem by giving the container the host’s network namespace. But this creates a new set of problems:

- the container **becomes the host**  
- it binds to the host’s ports  
- it takes over the host’s identity  
- it conflicts with other appliances  
- it breaks isolation entirely  

This is exactly what happened with the `mbentley/omada-controller` image. It uses host networking by default, and as soon as it started, it took over the server’s identity. AdGuard didn’t “break” — it was simply overshadowed.

Host networking is not a neutral choice. It is a takeover.

---

## What MACVLAN Provides

MACVLAN creates a virtual network interface on the host that behaves like a physical NIC. Each container attached to the MACVLAN network receives:

- **its own MAC address**  
- **its own IP address**  
- **full participation in the LAN’s broadcast domain**  
- **no NAT**  
- **no port collisions**  
- **no identity conflicts**  
- **no need for a VM or dedicated hardware**  

From the LAN’s perspective, each container is a separate device — exactly what network appliances expect.

---

## Why MACVLAN Behaves Like Bare Metal

MACVLAN operates at Layer 2. This means:

- ARP works normally  
- broadcast and multicast traffic reach the container  
- the container can advertise its own MAC  
- DHCP reservations work as expected  
- discovery protocols function correctly  
- the container is reachable directly from the LAN  

This is the behavior that bridge and host networking cannot provide simultaneously.

---

## The Sharp Edges of MACVLAN

MACVLAN is powerful, but it has quirks:

- the host cannot reach MACVLAN containers by default  
- Docker must start *after* the MACVLAN interface exists  
- Compose caches network IDs, which can cause ghost networks  
- stale MACVLAN networks can break container startup  
- identity must be deterministic (MAC/IP must not change)  

This repo documents the fixes for all of these issues, including the systemd shim that restores host ↔ container communication.

---

## Why MACVLAN Is the Right Tool for Appliances

Network appliances expect to behave like real devices. MACVLAN is the only Docker networking mode that:

- preserves container isolation  
- avoids port conflicts  
- avoids identity collapse  
- avoids NAT  
- provides full L2 adjacency  
- allows multiple appliances to coexist cleanly  
- avoids the overhead of running full VMs  

It is the closest you can get to bare metal while staying inside Docker.

---

## Summary

MACVLAN works for network appliances because it gives containers their own MAC, their own IP, and full participation in the LAN’s broadcast domain. Bridge networking hides appliances. Host networking collapses them into the host. MACVLAN is the middle ground that preserves identity, isolation, and functionality.

The next document explains how the systemd shim restores host ↔ container communication, which is the final piece of the pattern.
