# Networking Protocols — Master Index (OSI 7-Layer Model)

> Companion series: one note per OSI layer, each covering the popular protocols/standards at that layer — why they were created (what came before and what problem it solved), how they work, advantages/disadvantages, the standards body or organization that owns/maintains them, and the hardware/software that implements them.

This index note ties the series together and gives the OSI ↔ TCP/IP mapping, since most real-world discussion (including your K8s/Terraform/SSH work) happens in TCP/IP terms even though OSI is the cleaner teaching model.

## The series

- [[OSI Layer 1 - Physical Layer Protocols and Technologies]]
- [[OSI Layer 2 - Data Link Layer Protocols and Technologies]]
- [[OSI Layer 3 - Network Layer Protocols and Technologies]]
- [[OSI Layer 4 - Transport Layer Protocols and Technologies]]
- [[OSI Layer 5 - Session Layer Protocols and Technologies]]
- [[OSI Layer 6 - Presentation Layer Protocols and Technologies]]
- [[OSI Layer 7 - Application Layer Protocols and Technologies]]

## OSI vs TCP/IP — why two models, and why OSI still wins for teaching

The **OSI model** (ISO/IEC 7498, 1984) was designed *before* most of its own protocols existed — it's a theoretical, vendor-neutral reference model built by committee (ISO) to standardize networking in an era when IBM SNA, DECnet, Xerox XNS, and early TCP/IP were all incompatible competitors. It never won as an actual protocol suite (the OSI protocols themselves — X.25, CLNP, TP4 — mostly lost to TCP/IP), but its **layering vocabulary won completely** — everyone still says "Layer 3" or "Layer 7" today.

The **TCP/IP model** (4 layers: Link, Internet, Transport, Application) is what the real Internet actually runs — it grew organically from ARPANET/DARPA-funded research (Vint Cerf and Bob Kahn's 1974 TCP paper) rather than top-down design, which is why it's coarser: Session/Presentation/Application collapse into one "Application" layer because in practice those concerns (encryption, encoding, session state) ended up implemented *inside* application protocols (TLS inside HTTP, MIME inside SMTP) rather than as distinct universal layers.

| OSI Layer | TCP/IP Layer | Real-world home of the logic |
|---|---|---|
| 7 Application | Application | HTTP, DNS, SSH, SMTP logic itself |
| 6 Presentation | Application | TLS, MIME, compression — usually a library the app calls |
| 5 Session | Application | Mostly folded into TCP connection state + app-level session tokens |
| 4 Transport | Transport | TCP, UDP, QUIC |
| 3 Network | Internet | IP, ICMP, routing protocols |
| 2 Data Link | Link | Ethernet, Wi-Fi, VLANs, ARP |
| 1 Physical | Link | Cabling, radio, PHY chipsets |

This is exactly why Layer 5 and 6 will feel "thinner" than the others in this series — they're conceptually real but there are few protocols that live *purely* there; most of their job got absorbed into Layer 7 protocols and libraries over the decades. I've kept them as separate notes anyway, as requested, but flagged this explicitly in each.

## How to read each layer note

Each note follows the same structure per protocol:
- **The problem it solved** — what existed before, and why it broke down
- **How it works** — mechanism, not just definition
- **Advantages / disadvantages**
- **Standards body / owner** — who actually maintains the spec (IETF RFC, IEEE working group, W3C, or a single vendor)
- **Implementations** — real hardware/software that speaks it, so you can go inspect it yourself

Two specific deep-dives you asked for — **HTTP/1.0 → HTTP/2's actual mechanism** and **SFTP vs FTP/FTPS (different problem entirely)** — live in [[OSI Layer 7 - Application Layer Protocols and Technologies]], under their own headers.
