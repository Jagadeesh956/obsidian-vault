# OSI Layer 7 — Application Layer Protocols and Technologies

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

This is the layer users and developers interact with directly — the actual protocols that define *what* is being communicated, not just how bytes get there. It's also the layer where most of Session/Presentation's real-world logic ended up living (as covered in the index note), so expect TLS and session-state references throughout.

## HTTP — and the deep-dive on HTTP/1.0 → HTTP/2 you asked for

### HTTP/0.9 → HTTP/1.0 — the problem HTTP itself first solved
Tim Berners-Lee's original HTTP/0.9 (1991) was almost comically minimal: one line (`GET /page.html`), plain HTML response, connection closes. **HTTP/1.0** (1996) added headers, status codes, and support for other content types (via MIME, see Layer 6) — but every single request opened a **brand new TCP connection**, did a full three-way handshake, then closed it (`Connection: close` was effectively the only behavior). For a page with 20 embedded images, that's 20 separate TCP handshakes — brutally slow, especially before broadband.

### HTTP/1.1 (1997) — solved connection reuse, but not the real bottleneck
- **What it fixed:** `Connection: keep-alive` let one TCP connection serve multiple sequential requests, avoiding repeated handshakes. Added chunked transfer encoding, host-based virtual hosting (`Host` header — critical since it's what lets one IP serve many domains), and pipelining (sending multiple requests without waiting for each response).
- **What it still got wrong:** Pipelining was rarely used in practice because of **head-of-line blocking at the application level** — responses had to come back in the exact order requests were sent, so one slow response stalled everything behind it even on the same connection. Browsers worked around this the crude way: opening **multiple parallel TCP connections per domain** (browsers capped this around 6-8) — which just moves the problem to "handshake overhead × 6" and adds real complexity (each connection has its own TCP slow-start ramp-up, wasting available bandwidth at the start of every one of those parallel connections).

### HTTP/2 (2015, based on Google's SPDY) — how it actually solves this
This is the mechanism you specifically asked about:
- **Binary framing layer:** HTTP/2 replaces HTTP/1.1's human-readable text format with a binary protocol underneath — the semantics (methods, headers, status codes) are unchanged, but everything is now split into small binary **frames** in a **single TCP connection** per origin.
- **True multiplexing:** Multiple requests/responses (called "streams," each with a stream ID) are interleaved over that one connection simultaneously — frame 1 of stream A, frame 1 of stream B, frame 2 of stream A can all be sent back to back, and the receiver reassembles them by stream ID. This directly fixes HTTP/1.1's application-level head-of-line blocking, since a slow response on stream A no longer blocks stream B's frames from arriving.
- **Header compression (HPACK):** HTTP headers are extremely repetitive across requests (cookies, user-agent, etc.) — HPACK maintains a shared compression table between client/server so repeated header fields are sent as small references instead of full text every time, a meaningful bandwidth win at scale.
- **Server push (mostly deprecated since):** Let a server proactively send resources it knows the client will need (e.g., push `style.css` alongside `index.html` without waiting for a separate request) — in practice this under-delivered on its promise (browsers had poor cache-awareness of pushed resources) and Chrome removed support in 2022; most of the *real* gain came from multiplexing + HPACK, not push.
- **The one problem HTTP/2 could NOT fix:** Because it still rides on **one single TCP connection**, TCP's own Layer-4 head-of-line blocking (a lost packet stalls *the whole connection*, all multiplexed streams included) remains — this is precisely why HTTP/3 exists.
- **Owner:** IETF, RFC 7540 (originated from Google's SPDY experiment).

### HTTP/3 (2022) — solving HTTP/2's remaining problem
- **What it fixes:** Replaces TCP entirely with **QUIC** (see [[OSI Layer 4 - Transport Layer Protocols and Technologies]]) — QUIC's independent per-stream reliability means a lost packet only stalls the one stream it belonged to, not the whole connection. Also folds the TLS 1.3 handshake into QUIC's own handshake, cutting connection setup latency further.
- **Owner:** IETF, RFC 9114.
- **Practical adoption:** Cloudflare, Google, Facebook serve enormous traffic over HTTP/3 already; most browsers support it; still often falls back to HTTP/2 on restrictive networks that throttle/block UDP.

## SFTP — a genuinely different problem than FTP, as you asked

This is worth being precise about, because "SFTP" is commonly (and wrongly) assumed to just be "FTP but encrypted" — it's actually a different protocol family solving a related but distinct problem.

- **FTP (1971, predates TCP/IP itself — later adapted onto it):** **Problem solved:** standardized file transfer between systems. **How it works:** Uses **two separate connections** — a control connection (port 21, commands/responses) and a *separate* data connection (port 20, or a dynamically negotiated port in passive mode) for the actual file bytes. **Disadvantages:** Completely unencrypted (credentials and file contents both sent in plaintext — a serious problem by modern standards); the dual-connection model is a nightmare for NAT/firewall traversal (this is exactly why FTP is notoriously painful behind NAT — the firewall has to track and dynamically open the negotiated data port).
- **FTPS:** **What it is:** FTP with TLS bolted on (either "implicit," TLS from the start, or "explicit," upgrading via `AUTH TLS` mid-session) — still the same two-connection architecture and its NAT-traversal headaches, just now encrypted.
- **SFTP — SSH File Transfer Protocol:** **This is the key distinction: SFTP is not FTP-plus-encryption at all — it's a completely different protocol, built as a subsystem of SSH (Secure Shell) from the start.** It uses a **single connection** (SSH's normal port 22), inside which SFTP runs as a binary, packet-based subprotocol for file operations (open, read, write, list directory, permissions). Because it rides entirely inside one already-authenticated, already-encrypted SSH session, it inherits SSH's key-based authentication (the same `-i /Users/jagadeesh/.ssh/cka_gcp` key you've been using to SSH into your GCP VMs works identically for SFTP) and has none of FTP's dual-connection NAT problems.
- **Why this matters practically:** If your firewall/security group only opens port 22 (as yours almost certainly does for those `cp-1`/`wk-1`/`wk-2` instances), SFTP works with zero extra configuration, while FTP/FTPS would need additional ports opened and passive-mode firewall rules. This is exactly why SFTP, not FTPS, became the default for secure file transfer in modern Linux/cloud environments.
- **Owner:** FTP is IETF RFC 959 (1985); SFTP isn't a standalone IETF RFC at all — it's defined as part of the SSH protocol suite by the IETF `secsh` working group drafts, implemented ubiquitously by OpenSSH.

## SMTP / IMAP / POP3 — email

- **Problem solved:** SMTP (1982) standardized mail *transmission* between mail servers. It deliberately does nothing about mail *storage/retrieval* for end users — that's a separate concern.
- **POP3:** Downloads mail to one device, typically deleting it from the server — designed for an era of single-device, offline mail reading.
- **IMAP:** Keeps mail on the server, syncing state (read/unread, folders) across multiple devices — solved POP3's exact limitation as people started checking email from more than one device.
- **Owner:** IETF — SMTP RFC 5321, IMAP RFC 3501, POP3 RFC 1939.

## DNS — Domain Name System

- **Problem solved:** Humans can't remember IP addresses; also, IPs change (server migrations, load balancing) while names should stay stable.
- **How it works:** Hierarchical, distributed database — root servers → TLD servers (`.com`, `.io`) → authoritative nameservers for a specific domain, with heavy caching (TTLs) at every resolver in the chain. Directly relevant to your K8s work: **CoreDNS** (covered in your Kubernetes doc) implements this exact hierarchical-lookup model, just scoped to `cluster.local` instead of the public internet.
- **Owner:** IETF, RFC 1034/1035; root zone governed by ICANN/IANA.

## SSH — Secure Shell

- **Problem solved:** Telnet and rlogin sent everything — including passwords — in plaintext, trivially sniffable on any shared network segment.
- **How it works:** Encrypted, authenticated remote shell access, supporting both password and public-key authentication (asymmetric crypto — exactly the key pair you use for every `ssh -i ~/.ssh/cka_gcp` command). Also provides generic secure tunneling (port forwarding, and SFTP as covered above, ride on top of the same transport).
- **Owner:** IETF (SSH-2 protocol, RFC 4251-4254); OpenSSH (OpenBSD project) is the dominant implementation on virtually every Linux server, including your GCP VMs.

## DHCP — Dynamic Host Configuration Protocol

- **Problem solved:** Manually configuring IP address, subnet mask, gateway, and DNS server on every device doesn't scale.
- **How it works:** DORA sequence — Discover (broadcast), Offer, Request, Acknowledge — a client broadcasts for an IP, a DHCP server offers a lease from its pool, client confirms, server acknowledges and tracks the lease (with an expiry/renewal cycle).
- **Owner:** IETF, RFC 2131.

## SNMP — Simple Network Management Protocol

- **Problem solved:** Standardized, vendor-neutral monitoring/management of network devices (routers, switches) at scale, rather than every vendor having a proprietary management interface.
- **How it works:** Manager polls agents (or agents send unsolicited "traps" on events) for values from a standardized **MIB** (Management Information Base) tree.
- **Owner:** IETF, currently SNMPv3, RFC 3411-3418.

## WebSocket

- **Problem solved:** HTTP's request/response model is fundamentally wrong for real-time, bidirectional communication (chat, live dashboards) — before WebSocket, apps faked it with wasteful long-polling (repeatedly opening HTTP requests and holding them open).
- **How it works:** Starts as a normal HTTP request with an `Upgrade: websocket` header; once the server agrees, the same TCP connection is repurposed for a full-duplex, low-overhead framed protocol — no more request/response pairing needed, either side can send anytime.
- **Owner:** IETF, RFC 6455.

## gRPC

- **Problem solved:** Traditional REST/JSON APIs have real overhead (text parsing, no strict schema, one request per HTTP/1.1 round-trip) for high-performance service-to-service communication, especially at the scale/latency-sensitivity of microservices.
- **How it works:** Built on HTTP/2 (gets multiplexing "for free"), uses Protocol Buffers (compact binary serialization with a strict, code-generated schema) instead of JSON, and supports streaming (client, server, or bidirectional) natively via HTTP/2 streams.
- **Owner:** Google (open-sourced 2015), now a CNCF project — notably, this is the same governance home as Kubernetes itself, and gRPC is what the Kubernetes API server, etcd, and most cloud-native control planes actually use internally.

---
Related concepts: [[TLS]] · [[QUIC]] (see [[OSI Layer 4 - Transport Layer Protocols and Technologies]]) · [[CoreDNS]] · [[CNI (Container Network Interface)]]
