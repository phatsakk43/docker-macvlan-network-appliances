# What Is a Network Appliance?

Most Docker containers behave like applications: they listen on a port, respond to requests, and don’t care about how they appear on the LAN. These containers work perfectly on Docker’s default bridge network because they don’t need to participate in broadcast domains, advertise their own MAC address, or maintain a stable identity.

But some containers behave very differently. They expect to function as devices on the network, not just applications running behind NAT. These are what this project calls network appliances.

Understanding the difference between an application and a network appliance is the key to understanding why MACVLAN is the right tool for certain containers — and why bridge and host networking both fail in predictable ways.

---

## Characteristics of a network appliance

A container is acting like a network appliance when it expects one or more of the following:

- stable MAC/IP identity that other devices recognize  
- direct participation in the LAN’s broadcast domain  
- Layer‑2 adjacency (ARP, mDNS, SSDP, DHCP, discovery protocols)  
- privileged or well‑known ports that conflict with other services  
- DHCP reservations tied to a specific MAC address  
- the ability to be reached as a device, not a port‑mapped service  

These expectations come from the physical world. Hardware controllers, DNS servers, and discovery‑based services were designed long before containerization existed. They assume they are running on bare metal.

When you put them in Docker, they still behave like bare‑metal devices — and that’s where the trouble starts.

---

## Examples of network appliances

These containers commonly behave like network appliances:

- AdGuard Home — DNS, DHCP, and LAN‑visible identity  
- Pi‑hole — same reasons as AdGuard  
- Omada Controller — discovery, provisioning, and L2 communication  
- UniFi Controller — same pattern as Omada  
- Home Assistant — heavy use of multicast and discovery protocols  
- Syncthing — peer discovery and direct LAN communication  
- Anything that expects to be a first‑class LAN device  

These containers break in subtle ways when forced into bridge networking (NAT, no broadcast) or host networking (identity collapse, port conflicts).

---

## Why bridge networking fails for appliances

Bridge networking hides containers behind the host using NAT. This is perfect for applications, but it breaks appliances because:

- they cannot receive broadcast or multicast traffic  
- they cannot advertise their own MAC address  
- they cannot be given DHCP reservations  
- they cannot be discovered by other devices  
- they appear as services on the host, not devices on the LAN  

This is why AdGuard worked fine until another appliance (Omada) entered the picture — the moment two appliances need identity, bridge networking collapses.

---

## Why host networking fails for appliances

Host networking gives a container the host’s network namespace. This solves the broadcast/multicast problem, but creates a new one:

- the container becomes the host  
- it binds to the host’s ports  
- it takes over the host’s identity  
- it conflicts with other appliances  
- it breaks isolation entirely  

This is exactly what happened with the mbentley/omada-controller image: it used host networking by default, took over the server’s identity, and AdGuard disappeared.

Host networking is not a neutral choice — it is a takeover.

---

## Why MACVLAN works

MACVLAN gives each appliance:

- its own MAC address  
- its own IP address  
- full participation in the LAN  
- no NAT  
- no port collisions  
- no identity conflicts  
- no need for a VM or dedicated hardware  

It is the only Docker networking mode that allows a container to behave like a real device on the LAN while still keeping the benefits of containerization.

---

## Summary

A network appliance is a container that expects to behave like a real device on the LAN. These containers break under bridge networking and collide under host networking. MACVLAN is the clean, stable solution that gives them the identity and adjacency they need.

The rest of the docs build on this definition and show how to implement the pattern reliably.
