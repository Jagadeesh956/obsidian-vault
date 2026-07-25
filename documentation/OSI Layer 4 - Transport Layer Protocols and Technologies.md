# OSI Layer 4 — Transport Layer Protocols and Technologies

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

Layer 4 is where **end-to-end** semantics live — not "how do I get this packet to the right host" (Layer 3's job) but "how do I get this *data stream* to the right *application* on that host, reliably or not, as needed." This is also where **ports** first appear, letting one IP address serve many simultaneous conversations.

## TCP — Transmission Control Protocol

- **Problem solved:** IP alone gives no guarantee of delivery, ordering, or flow control — packets can arrive out of order, duplicated, or not at all. Applications like file transfer or a database connection need a reliable byte stream, not "best effort."
- **How it works:**
  - **Three-way handshake** (SYN → SYN-ACK → ACK) establishes a connection before any data flows.
  - **Sequence numbers** on every byte let the receiver reorder out-of-sequence segments and detect gaps.
  - **ACKs + retransmission timers** guarantee delivery — unacknowledged data gets resent.
  - **Sliding window flow control** prevents a fast sender from overwhelming a slow receiver.
  - **Congestion control** (slow start, congestion avoidance, algorithms like Cubic or BBR) prevents the sender from overwhelming the *network*, backing off when packet loss suggests congestion.
- **Advantages:** Reliable, ordered, congestion-aware — the safe default for anything where correctness matters more than latency (HTTP, SSH, database connections — including your `ssh -i ... ubuntu@<ip>` from earlier, and the `kubectl` → API server connection, both TCP).
- **Disadvantages:** All that reliability machinery costs latency — the handshake alone is a full round-trip before any data moves, and **head-of-line blocking** (if one segment is lost, everything behind it in the stream waits) hurts badly on lossy/high-latency links. This exact problem is a core reason HTTP/2's multiplexing still stalls under packet loss, and part of why HTTP/3 abandoned TCP entirely (see [[OSI Layer 7 - Application Layer Protocols and Technologies]]).
- **Owner:** IETF, RFC 793 (1981), heavily amended since (RFC 9293 is the current consolidated spec).
- **Implementations:** Every OS TCP/IP stack (Linux's is in-kernel, tunable via the very `sysctl` mechanism from your Linux networking doc).

## UDP — User Datagram Protocol

- **Problem solved:** TCP's reliability guarantees are sometimes actively unwanted overhead — for DNS lookups (one query, one answer, just retry if it's lost — cheaper than a full handshake), video calls (a late frame is useless, better to drop it than delay everything waiting for retransmission), or DHCP (broadcast-based, connectionless by nature).
- **How it works:** Just source port + destination port + length + checksum, then payload — no handshake, no ordering, no retransmission, no congestion control. "Fire and forget."
- **Advantages:** Minimal overhead, low latency, supports broadcast/multicast (TCP can't).
- **Disadvantages:** Any reliability, ordering, or congestion-awareness needed has to be built by the *application* itself if it needs it at all (this is exactly what QUIC does — see below).
- **Owner:** IETF, RFC 768 (1980) — remarkably, essentially unchanged in over 40 years, because its whole point is to be minimal.

## SCTP — Stream Control Transmission Protocol

- **Problem solved:** Telecom signaling (originally) needed something in between TCP and UDP — reliable and ordered *within* independent streams, but without one lost packet blocking unrelated streams (TCP's head-of-line blocking problem again), plus built-in multi-homing (a connection can survive a path failing over to a backup network path).
- **How it works:** "Multi-streaming" — multiple independent, ordered sub-streams within one association, so loss in stream A doesn't stall stream B.
- **Disadvantages:** Middlebox support (firewalls/NATs) is spotty since it never got TCP/UDP's ubiquity, which badly limited real-world adoption outside telecom.
- **Owner:** IETF, RFC 4960. Used heavily in telecom SS7/Diameter signaling and WebRTC's data channel (layered underneath it).

## QUIC

- **Problem solved:** TCP's head-of-line blocking (above) becomes a serious problem once you multiplex many logical streams over one connection (exactly what HTTP/2 does) — one lost TCP segment stalls *every* HTTP/2 stream sharing that connection, even ones with no relation to the lost data. TCP also bakes its handshake+congestion state into kernel implementations that are slow to evolve. QUIC's answer: rebuild transport-layer reliability *on top of UDP*, in user-space, where Google (and later the IETF) could iterate fast without waiting on OS kernel upgrades across the entire internet.
- **How it works:** Runs over UDP but reimplements TCP's reliability/congestion control itself, with true independent stream multiplexing (loss in one stream doesn't block others — solving TCP+HTTP/2's head-of-line problem at the root) and **mandatory encryption** (TLS 1.3 is built into the handshake itself, not layered after, cutting connection setup to often 1 round-trip instead of TCP+TLS's 2-3).
- **Disadvantages:** CPU-heavier than kernel TCP (userspace processing), and some restrictive corporate/carrier networks block or throttle UDP traffic, occasionally forcing fallback to HTTP/2-over-TCP.
- **Owner:** Originally a Google protocol (2012), standardized by the IETF as RFC 9000 (2021).
- **Implementations:** HTTP/3 (built on QUIC), used by Google/YouTube, Cloudflare, Facebook at massive scale already.

## Ports — the piece that makes multiplexing possible

Both TCP and UDP share the same **16-bit port** concept (0-65535): well-known ports (0-1023, e.g. 22=SSH, 443=HTTPS, 6443=Kubernetes API — exactly the port in your `cp_public_ip:6443` from earlier), registered ports (1024-49151), and dynamic/ephemeral ports (49152-65535, used as the *source* port for outbound client connections). This is literally what lets kube-proxy's DNAT rules (from the K8s networking doc) rewrite `10.96.5.20:80` → `10.244.2.14:8080` — the port is part of the routing decision, not just the IP.

---
Related concepts: [[Routing Table]] · [[iptables]] · [[IPVS]] · [[CoreDNS]]
