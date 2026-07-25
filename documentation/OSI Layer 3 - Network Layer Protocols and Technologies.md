# OSI Layer 3 — Network Layer Protocols and Technologies

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

Layer 3 is where **routing** — moving a packet across *multiple* networks, not just one local segment — becomes possible. This is the layer that introduces logical, hierarchical addressing (IP) independent of physical hardware, which is what lets the internet scale to billions of devices.

## IP — Internet Protocol (IPv4 and IPv6)

- **Problem solved:** Layer 2 addressing (MAC) is flat and non-hierarchical — there's no way to summarize "all these million devices are reachable via this one path," which makes it useless at internet scale. IP introduces hierarchical, *routable* addressing.
- **How it works (IPv4):** 32-bit addresses, divided into network/host portions via a subnet mask (or CIDR notation like `/24`). Routers forward packets hop-by-hop based on **longest-prefix match** against their routing table (exactly the tables you inspected via `ip route show` in the Linux networking doc). Includes a TTL field specifically to prevent infinite routing loops (each hop decrements it; hits zero → dropped, ICMP Time Exceeded sent back).
- **Why IPv6 exists:** IPv4's 32-bit space (~4.3 billion addresses) was exhausted faster than expected once NAT workarounds stopped being enough for an internet-of-everything world. IPv6 uses 128-bit addresses (effectively unlimited), and also simplified the header (removed header checksum — relies on Layer 2/4 checksums instead — and made fragmentation the sender's job, not routers', for performance), and built in mandatory support for IPsec.
- **Disadvantages of IPv6:** Adoption has been slow for decades due to NAT extending IPv4's life far longer than expected, and the cost/complexity of dual-stack migration.
- **Owner:** IETF — IPv4 is RFC 791 (1981), IPv6 is RFC 8200.
- **Implementations:** Every OS network stack, every router's forwarding ASIC; this is exactly the layer where your `10.0.0.0/24` GCP subnet and `10.244.0.0/24` pod CIDR live.

## ICMP — Internet Control Message Protocol

- **Problem solved:** IP itself is a "best-effort, fire and forget" protocol with no mechanism to report problems (unreachable host, TTL expired, fragmentation needed but forbidden). ICMP is IP's error-reporting/diagnostic companion.
- **How it works:** `ping` uses ICMP Echo Request/Reply; `traceroute` abuses the TTL-expiry mechanism (sends packets with incrementing TTLs, and reads the "Time Exceeded" ICMP replies from each hop along the way to map the path).
- **Disadvantages:** Frequently rate-limited or blocked by firewalls (since it's also used for reconnaissance/DoS in some attack forms), which is why "ping doesn't work but the service is actually up" is such a common false alarm.
- **Owner:** IETF, RFC 792.

## Routing protocols

Routing protocols exist because static routes don't scale — someone/something has to *automatically* discover and distribute "how do I reach network X" across potentially thousands of routers, and adapt when links fail.

### OSPF (Open Shortest Path First)
- **Problem solved:** RIP's (see below) massive inefficiency at scale.
- **How it works:** Link-state protocol — every router builds a complete map of the network topology (via flooding "link-state advertisements") and independently computes shortest paths using Dijkstra's algorithm.
- **Advantages:** Fast convergence, scales well within a single administrative domain (an "Autonomous System").
- **Owner:** IETF, RFC 2328. "Open" in the name specifically because it was designed as a non-proprietary alternative to Cisco's proprietary IGRP.

### BGP (Border Gateway Protocol)
- **Problem solved:** OSPF-style link-state flooding doesn't scale to the *entire internet* (hundreds of thousands of routes across organizations that don't trust each other with internal topology details) — the internet needed a protocol for routing *between* autonomous systems (ASes), based on policy, not just shortest-path math.
- **How it works:** Path-vector protocol — routers exchange reachability info plus the AS-path a route traverses, and apply policy (business relationships, not just distance) to choose routes. This is literally what makes the global internet a "network of networks" work — your ISP peers with other ISPs via BGP.
- **Disadvantages:** No inherent security — BGP hijacking (an AS falsely announcing routes it doesn't own) is a real, still-unsolved-at-scale internet vulnerability; famous incidents have rerouted major traffic through wrong countries.
- **Owner:** IETF, RFC 4271. **Directly relevant to your Calico BGP-mode CNI plugin** from the K8s doc — Calico literally runs a BGP daemon (`bird`) on each node to distribute pod-subnet routes, reusing this exact internet-scale protocol at cluster scale.

### RIP (Routing Information Protocol) — the "problem" OSPF solved
- **How it worked:** Distance-vector, hop-count based, max 15 hops (16 = "infinity"/unreachable) — an artificial ceiling that made it useless for large networks.
- **Why it was superseded:** Slow convergence (the "count-to-infinity" problem, where routers can loop-feed each other stale info after a failure) and the 15-hop ceiling.
- **Owner:** IETF, RFC 2453 (RIPv2).

## IGMP — Internet Group Management Protocol

- **Problem solved:** Efficiently delivering the same stream (e.g., IPTV multicast) to many subscribers without sending a separate copy to each.
- **How it works:** Hosts signal routers which multicast groups they want to join/leave; routers use this to prune multicast delivery trees so traffic only flows where there are actual listeners.
- **Owner:** IETF, RFC 3376.

## IPsec

- **Problem solved:** IP itself has zero built-in confidentiality or integrity — anyone on the path can read or tamper with a packet.
- **How it works:** Operates at the network layer (unlike TLS, which is per-application/per-TCP-connection) — encrypts/authenticates entire IP packets, in either Transport mode (payload only, used host-to-host) or Tunnel mode (entire original packet wrapped inside a new one, used for site-to-site VPNs).
- **Owner:** IETF, RFC 4301 series.
- **Implementations:** Most enterprise VPN appliances (Cisco ASA, strongSwan, WireGuard is a modern alternative with a different, simpler design philosophy).

---
Related concepts: [[Routing Table]] · [[Policy Routing]] · [[VXLAN]] · [[CNI (Container Network Interface)]]
