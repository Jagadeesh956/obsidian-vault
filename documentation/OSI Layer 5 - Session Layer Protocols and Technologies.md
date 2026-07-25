# OSI Layer 5 — Session Layer Protocols and Technologies

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

**Honest framing first:** Layer 5 is the thinnest layer in practice, and this note will be shorter than the others for a real reason, not laziness — as covered in the index note, most "session management" concerns (keeping track of who's logged in, resuming a dropped connection, correlating requests to a conversation) ended up implemented *inside* Layer 7 application protocols and libraries (cookies, JWTs, TLS session tickets) rather than as a distinct universal wire protocol. Still, a few protocols/mechanisms live squarely here.

## What "session" means at this layer

The job: establish, maintain, synchronize, and tear down a **logical conversation** between two applications — potentially spanning multiple underlying transport connections, and providing checkpointing/recovery so a conversation can resume after an interruption, which is a level above what TCP's single-connection reliability gives you.

## NetBIOS

- **Problem solved:** Early 1980s IBM PC-networks needed a way for applications to establish named sessions between machines on a LAN, before TCP/IP dominance — an application-naming and session layer for early Windows/DOS networking (file sharing, printer sharing).
- **How it works:** Provides name resolution (NetBIOS names for machines), datagram service, and true session service (reliable, connection-oriented, guaranteed delivery — session establishment separate from and above the transport).
- **Relevance today:** Legacy — still lurking in Windows environments (`nbtstat`, SMB historically ran over NetBIOS over TCP/IP, port 139), though modern SMB (445) runs directly over TCP, bypassing NetBIOS entirely.
- **Owner:** Originally IBM (1983), later adapted for TCP/IP by IETF (RFC 1001/1002).

## RPC — Remote Procedure Call (and session semantics within it)

- **Problem solved:** Letting a program call a function that actually executes on a different machine, as if it were local — the session layer piece is tracking the multi-step "call → execute → return" conversation as one logical unit, including matching a response back to the right pending call.
- **How it works:** Client stub serializes arguments, sends over the network, server stub deserializes and executes, response flows back — the "session" is the call's lifetime. Modern implementations like gRPC (built on HTTP/2) fold this concern back up into Layer 7, which is the general pattern across this whole layer.
- **Owner:** Sun Microsystems (original ONC RPC, RFC 1057), later many implementations (DCE/RPC, gRPC, Thrift).

## PPTP — Point-to-Point Tunneling Protocol

- **Problem solved:** Establishing a persistent, authenticated tunnel session between a client and a VPN server, layered over PPP.
- **How it works:** Uses a TCP control connection (port 1723) to manage the session/tunnel state, with a separate GRE-encapsulated data channel for the actual traffic.
- **Disadvantages:** Its encryption (MS-CHAPv2 based) has known serious cryptographic weaknesses — effectively deprecated in favor of IPsec/WireGuard/OpenVPN for anything security-sensitive.
- **Owner:** A vendor consortium including Microsoft (1999), later IETF-documented (RFC 2637, informational).

## TLS session resumption — the modern, practical example

Rather than a standalone protocol, this is the clearest real example of session-layer *behavior* living inside a Layer 7/6 mechanism: a fresh TLS handshake is expensive (asymmetric crypto, extra round trips). **Session tickets/session IDs** let a client and server skip re-negotiating everything on a reconnect — the server issues an encrypted ticket capturing session state, the client presents it on the next connection, and the server can resume without redoing the full handshake. This is precisely "session layer" functionality (maintain/resume a logical conversation across separate transport connections), just implemented as a TLS extension rather than a separate OSI-Layer-5 protocol — a good concrete illustration of why this note is shorter than its neighbors.

## Sockets API — the practical session abstraction developers actually touch

Not a wire protocol at all, but worth naming: the **Berkeley sockets API** (`socket()`, `bind()`, `connect()`, `accept()`) is the *programming* abstraction that represents "a session" to application code, regardless of which underlying protocols are really in play. Every example command in your earlier docs (`ssh`, `kubectl`, `curl`) ultimately goes through this API to open what the OS considers a "connection," which is the practical, code-level stand-in for Layer 5 in almost all real software.

---
Related concepts: [[TLS]] · [[gRPC]] · [[SMB]]
