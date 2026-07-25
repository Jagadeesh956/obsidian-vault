# OSI Layer 2 — Data Link Layer Protocols and Technologies

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

Layer 2 is where **frames** and **addressing within a local segment** (MAC addresses) first appear. Its job: move a frame reliably between two directly-connected nodes (or nodes on the same shared/switched segment), and arbitrate who gets to transmit when multiple devices share a medium .

## Ethernet (IEEE 802.3 MAC layer)

- **Problem solved:** Layer 1 gives you bits on a wire; something has to decide *whose* bits they are and detect collisions. Original Ethernet (1980, Xerox PARC — Bob Metcalfe) ran on a shared coax bus where every device saw every frame.
- **How it works:** Frames carry source/destination **MAC addresses** (48-bit, burned into the NIC, globally unique via IEEE-assigned OUI prefixes), an EtherType field, payload, and a CRC checksum (FCS) for error detection. Early Ethernet used **CSMA/CD** (Carrier Sense Multiple Access with Collision Detection) — listen before transmitting, and if a collision is detected mid-transmission, back off and retry after a random delay .
- **Advantages:** Simple, cheap, scaled from 10 Mbps to 400 Gbps without changing the frame format.
- **Disadvantages:** CSMA/CD degrades badly under load (collisions increase quadratically with more active senders) — this problem is why switches (not hubs) became universal: a switch gives every port its own **collision domain**, making CSMA/CD irrelevant on modern full-duplex switched Ethernet.
- **Owner:** IEEE 802.3.
- **Implementations:** Every NIC, switch, and the `eth0`/`ens3`-style interfaces you saw in the CNI discussion earlier — this is the exact frame format `veth` pairs and Linux bridges operate on.

## Wi-Fi MAC (802.11, CSMA/CA)

- **Problem solved:** Radio can't reliably *detect* collisions the way wired Ethernet can (a transmitting radio can't hear over its own transmission) — so collision *detection* doesn't work over the air.
- **How it works:** **CSMA/CA** (Collision *Avoidance*) — listen, wait a random backoff, then transmit, and use **ACK frames** from the receiver to confirm delivery (no ACK = assume collision/loss, retransmit). This is fundamentally more conservative/wasteful than wired CSMA/CD, which is part of why Wi-Fi throughput is always noticeably below its "rated" speed under real load.
- **Owner:** IEEE 802.11.

## ARP (Address Resolution Protocol)

- **Problem solved:** Layer 3 (IP) addresses are logical/routable; Layer 2 (MAC) needs the actual hardware address to put a frame on the wire. Something has to map "I know the IP, what's the MAC?"
- **How it works:** Broadcasts "who has 10.0.0.1? tell 10.0.0.5" to the entire local segment; the owner replies directly with its MAC. Result gets cached in the local **ARP table** (`ip neigh` on Linux) to avoid re-asking for every packet.
- **Disadvantages:** No authentication at all — this is why **ARP spoofing/poisoning** attacks exist (a malicious host just replies to ARP requests claiming to own an IP it doesn't, redirecting traffic through itself — classic MITM technique). Modern switches mitigate via Dynamic ARP Inspection.
- **Owner:** IETF, RFC 826 (1982).
- **Relevance to your K8s work:** every time a pod's veth is attached to `cni0` (see your Linux networking doc), ARP is exactly how other pods on the same bridge learn its MAC to actually deliver frames to it.

## VLAN tagging (IEEE 802.1Q)

- **Problem solved:** Before VLANs, isolating traffic (e.g., separating an org's finance dept from engineering) required physically separate switches/cabling per group — expensive and inflexible.
- **How it works:** Inserts a 4-byte tag into the Ethernet frame carrying a 12-bit VLAN ID (4094 usable VLANs), letting one physical switch fabric behave as many logically isolated broadcast domains. Trunk ports carry multiple VLANs' tagged traffic between switches; access ports serve one VLAN, untagged, to end devices.
- **Advantages:** Logical segmentation without new cabling; also used for traffic prioritization (802.1p bits live in the same tag).
- **Disadvantages:** VLAN hopping attacks exist if trunk ports are misconfigured; 4094 VLAN ceiling became a real limitation in large multi-tenant cloud/datacenter environments — directly motivating **VXLAN** (Layer 3 overlay with a 24-bit segment ID, 16 million+ segments) which you already encountered in the K8s networking doc.
- **Owner:** IEEE 802.1Q.

## Spanning Tree Protocol — STP (IEEE 802.1D) and successors (RSTP, MSTP)

- **Problem solved:** Redundant physical links between switches (added for resilience) create Layer 2 **loops** — since Ethernet frames have no TTL, a broadcast frame in a loop replicates forever, a "broadcast storm" that melts the network in seconds.
- **How it works:** Switches exchange BPDUs (Bridge Protocol Data Units) to elect a "root bridge" and computationally block redundant links until needed, forming a loop-free logical tree while keeping the physical redundancy for failover.
- **Disadvantages:** Original STP's convergence time (30-50 seconds after a link failure) was too slow for modern needs — **RSTP** (802.1w) cut this to sub-second by rethinking the state machine.
- **Owner:** IEEE 802.1D / 802.1w.

## PPP (Point-to-Point Protocol)

- **Problem solved:** Needed a standard way to frame IP traffic over serial/dial-up links (modems) where there's no shared-medium addressing concern at all — just two endpoints.
- **How it works:** Framing plus built-in authentication (PAP/CHAP) and link negotiation (LCP). Still underlies most DSL connections today (PPPoE — PPP over Ethernet) and VPN tunneling.
- **Owner:** IETF, RFC 1661.

## LACP — Link Aggregation Control Protocol (IEEE 802.3ad)

- **Problem solved:** A single physical link is a single point of failure and a throughput ceiling.
- **How it works:** Bundles multiple physical links into one logical channel, negotiated dynamically between two switches/hosts, providing both redundancy and combined bandwidth.
- **Owner:** IEEE 802.3ad.

---
Related concepts: [[veth pair]] · [[Linux Bridge]] · [[VXLAN]] · [[Network Namespace]] · [[VLAN]] · [[Spanning Tree Protocol]]
