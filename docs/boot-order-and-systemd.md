# Boot Ordering and systemd

MACVLAN networking is sensitive to timing. Docker must see the correct network interfaces at startup, and Compose must attach containers to a stable, known MACVLAN network. If the MACVLAN parent interface does not exist when Docker starts, Docker may create a ghost network, cache stale network IDs, or assign incorrect MAC/IP identities to containers. These failures are subtle, persistent, and difficult to diagnose after the fact.

To prevent this, the MACVLAN parent interface must be created before Docker starts. This is the purpose of the boot‑ordering shim and the systemd unit that manages it.

This pattern assumes a standard Docker installation where the Docker service lifecycle is managed by systemd. Platforms that wrap Docker inside other orchestration layers (such as Kubernetes‑based systems) may not support this approach.

---

## Why boot ordering matters

Docker inspects the host’s network interfaces at startup. If the MACVLAN parent interface is missing or not yet configured, Docker may:

- create a new MACVLAN network with a different internal ID  
- attach containers to a stale or incorrect network  
- assign containers the wrong MAC or IP address  
- break DHCP reservations tied to specific MACs  
- produce inconsistent behavior across reboots  

These issues often appear as:

- containers receiving unexpected IP addresses  
- DHCP servers handing out new leases instead of reserved ones  
- containers failing to start due to missing networks  
- containers appearing on the LAN with the wrong identity  
- MACVLAN networks that cannot be removed or recreated cleanly  

Once Docker caches a bad network state, it tends to persist until manually purged.

---

## What the boot‑ordering shim does

The boot‑ordering shim ensures that the MACVLAN parent interface exists before Docker starts. It does this by:

- creating the MACVLAN parent interface early in the boot process  
- assigning it the correct configuration  
- ensuring it is up and stable before Docker initializes  
- preventing Docker from creating ghost networks  
- ensuring Compose attaches containers to the correct network  

This shim does not enable host‑to‑container communication. Its purpose is stability, not reachability.

---

## Why systemd is the right tool

systemd is the native service manager on modern Linux systems. It provides:

- precise control over startup order  
- dependency management  
- automatic restarts  
- logging and status reporting  
- guaranteed execution before Docker  

By placing the shim in a systemd unit and declaring Docker as a dependent service, the system ensures:

- the MACVLAN interface is created first  
- Docker starts only after the interface exists  
- Compose sees a consistent network environment  
- container identity remains stable across reboots  

This eliminates the race conditions that cause ghost networks and identity drift.

---

## What happens without systemd ordering

If the MACVLAN interface is created manually or by a script that runs after Docker, the following problems may occur:

- Docker starts before the interface exists  
- Docker creates a placeholder MACVLAN network  
- Compose attaches containers to the wrong network  
- containers receive incorrect MAC/IP assignments  
- DHCP reservations break  
- the network becomes unrecoverable without manual cleanup  

These failures are often intermittent, depending on boot timing, making them difficult to reproduce or diagnose.

---

## How the systemd unit fits into the overall architecture

The systemd unit is part of a three‑stage boot sequence:

1. **systemd creates the MACVLAN parent interface**  
   Ensures the correct network exists before Docker starts.

2. **Docker starts with the correct network state**  
   Prevents ghost networks and stale network IDs.

3. **Compose starts containers with stable MAC/IP identity**  
   Ensures deterministic behavior across reboots.

This sequence guarantees that containers always attach to the correct MACVLAN network with the correct identity.

---

## Summary

MACVLAN networking requires deterministic boot ordering. If Docker starts before the MACVLAN parent interface exists, it may create ghost networks, assign incorrect MAC/IP identities, or break DHCP reservations. The boot‑ordering shim, managed by systemd, ensures that the MACVLAN interface is created early in the boot process so Docker and Compose always see a consistent network environment.

This deployment uses the boot‑ordering shim exclusively. It does not implement host‑reachability, and the systemd unit is focused solely on ensuring stable MACVLAN behavior and predictable container identity.
