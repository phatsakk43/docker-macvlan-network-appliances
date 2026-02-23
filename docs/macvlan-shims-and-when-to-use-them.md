# MACVLAN Shims and When to Use Them

MACVLAN is a powerful way to give containers their own MAC address and IP on the LAN. This is ideal for network appliances such as DNS servers, controllers, and discovery‑based services. But MACVLAN also introduces strict Layer‑2 behavior that affects how the host and containers interact. To manage this, two different “shim” patterns exist in the MACVLAN ecosystem. They solve different problems, and only one of them is used in this deployment.

This document explains both patterns, when each is appropriate, and why this project implements only the boot‑ordering shim.

---

## Two different shim patterns

The MACVLAN community uses the word “shim” to describe two unrelated mechanisms. They are often conflated, which leads to confusion.

### Boot‑ordering shim
Ensures the MACVLAN parent interface exists before Docker starts. Prevents Docker from creating ghost networks, stabilizes MAC/IP identity, and keeps DHCP reservations consistent. Does not enable host‑to‑container communication.

### Host‑reachability shim
Creates a host‑side MACVLAN interface with its own IP so the host can reach MACVLAN containers. Useful when the host is a client of the containers. Not implemented in this deployment.

Understanding the difference between these two patterns is essential for choosing the right one.

---

## When the boot‑ordering shim is the correct choice

The boot‑ordering shim is appropriate when the host is an infrastructure node rather than a client. In this role, the host:

- runs Docker and systemd
- owns the MACVLAN parent interface
- must remain reachable even if containers fail
- must not depend on containers for DNS or control
- is not where you access appliance UIs
- should avoid circular dependencies

In this scenario, the host does not need to reach MACVLAN containers. The containers communicate with the rest of the LAN at Layer 3, and the host remains independent of their health or availability.

This deployment uses the boot‑ordering shim exclusively. It ensures deterministic MAC/IP identity and prevents Docker from creating stale or inconsistent MACVLAN networks during boot.

---

## When the host‑reachability shim is appropriate

Some deployments require the host to act as a client of the containers. This is common when:

- the host is a workstation or desktop
- the host is the only management device on the network
- the host needs to access the appliance’s web UI
- the host runs monitoring or health‑check scripts
- the host relies on the appliance for DNS
- the host participates in controller discovery

In these cases, the host must have Layer‑2 adjacency with the MACVLAN network. A host‑reachability shim provides this by giving the host its own MACVLAN interface and IP address.

This pattern is well documented elsewhere, but it is not used or tested in this project.

---

## Why this deployment does not use host‑reachability

This project intentionally avoids host‑to‑container communication for several reasons:

- The host is dedicated hardware, not a workstation.
- All management occurs from other LAN devices.
- The host must remain independent of container health.
- Avoiding unnecessary L2 adjacency reduces complexity.
- Circular dependencies between host and containers are undesirable.
- The value of the traffic does not justify the cost of opening the L2 “door.”

The containers communicate with the LAN at Layer 3, and the host does not need to participate in that path. Keeping the host isolated at L2 is the simpler and more robust design.

---

## A simple decision rule

Choosing the correct shim depends entirely on the role of the host.

### Use the boot‑ordering shim when:
- the host is infrastructure, not a client
- the host must remain independent of container health
- management happens from other LAN devices
- you want the simplest, most stable MACVLAN setup

### Use the host‑reachability shim when:
- the host is a workstation or VM
- the host needs to access container UIs
- the host runs scripts or monitoring tools
- the host relies on the containers for DNS or control

This project falls into the first category.

---

## Summary

MACVLAN introduces strict Layer‑2 behavior that requires careful handling. Two shim patterns exist:

- The boot‑ordering shim ensures the MACVLAN interface exists before Docker starts and stabilizes container identity. This is the shim used in this deployment.
- The host‑reachability shim enables the host to reach MACVLAN containers, but is only necessary when the host acts as a client. It is not implemented here.

Understanding the role of the host is the key to choosing the correct pattern. Only open communication pathways when they are explicitly required; unnecessary L2 adjacency adds complexity without benefit.
