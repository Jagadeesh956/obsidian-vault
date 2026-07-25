# Application Connection Lifecycle — Mapping Protocols to What Your Code Actually Does

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

This note answers the question every developer eventually hits: *"My code calls `httpClient.get(url)` or `new Socket(host, port)` — what is actually happening underneath, at which layer, and which parts am I actually responsible for driving vs. which parts the framework/OS handles for me?"*

Worked scenario used throughout: a **Java process on Linux** (client) calling a **Python/Node service on a different Linux or Windows host** (server) — two different OSes, two different language runtimes, connected over the internet. Every step below names which OSI layer it belongs to, linking back to the layer notes.

## Part 1 — The full timeline of one connection, start to finish

| # | Event | OSI Layer | Who initiates | Triggering application call |
|---|---|---|---|---|
| 1 | DNS resolution of hostname → IP | [[OSI Layer 7 - Application Layer Protocols and Technologies|L7]] (DNS is itself an app-layer protocol) | Client | `InetAddress.getByName()` (Java), implicit inside `connect()`/`http.get()` in most languages |
| 2 | TCP three-way handshake (SYN, SYN-ACK, ACK) | [[OSI Layer 4 - Transport Layer Protocols and Technologies|L4]] | Client → Server | `socket.connect()` / `new Socket(host, port)` — **this call blocks until the handshake completes** |
| 3 | Server's `accept()` returns a new connected socket | L4 | Server | Server framework's accept loop (Tomcat thread pool, Netty event loop `bind()`/`accept()`) |
| 4 | TLS handshake (if HTTPS/gRPC-TLS/etc.) | [[OSI Layer 6 - Presentation Layer Protocols and Technologies|L6]] | Client → Server | Happens automatically inside `HttpsURLConnection`/`HttpClient` once you use `https://`; explicit if you're doing raw `SSLSocket` |
| 5 | Application request sent (HTTP request line + headers + body, or a gRPC frame, etc.) | [[OSI Layer 7 - Application Layer Protocols and Technologies|L7]] | Client | `httpClient.send(request)`, `outputStream.write(...)` |
| 6 | Server processes and writes response | L7 | Server | Your controller/handler method returning a response object |
| 7 | Connection **kept alive** for reuse, OR closed | L4 (mechanism) driven by L7 policy (HTTP `Connection` header, HTTP/2 stream multiplexing) | Either side | Nothing explicit if using a pooled client — the pool decides |
| 8 | TCP connection torn down (FIN/ACK exchange, or abrupt RST) | L4 | Whoever decides to close | `socket.close()`, `connection.close()`, or an idle-timeout firing inside the framework |
| 9 | `TIME_WAIT` state on the side that sent the first FIN | L4 (OS kernel bookkeeping) | OS, not your code | Nothing — pure kernel housekeeping to guarantee delayed duplicate packets don't corrupt a future connection reusing the same port pair |

The critical thing to internalize: **steps 2, 4, 8 are protocol-mandated exchanges that happen on the wire regardless of language/OS** — a Java client and a Go server agree on this dance purely because they both implement the same RFCs (TCP's RFC 9293, TLS's RFC 8446, HTTP's RFC 7540/9114). Nothing about "Java talking to Python" requires any special handling — **that's the entire point of standardized protocols**: your tech stack is irrelevant to the wire format, which is exactly why polyglot microservices architectures work at all.

## Part 2 — When does application code actually "open" a connection?

This is more nuanced than "when you call connect()" because most real code goes through a client library, not raw sockets. Three patterns:

### Pattern A — Eager/explicit (raw sockets, some RPC clients)
```java
Socket socket = new Socket("api.example.com", 443); // BLOCKS here until TCP handshake completes
```
The connection opens at the exact line of code that constructs the socket. This is rare in modern app code — almost everyone uses a higher-level client.

### Pattern B — Lazy, on first use (most HTTP clients: Java `HttpClient`, Python `requests`, Node `axios`)
```java
HttpClient client = HttpClient.newHttpClient(); // no connection yet — just builder/config
HttpResponse<String> response = client.send(request, BodyHandlers.ofString()); // connection opens HERE
```
Constructing the client does *nothing* on the wire. The connection (TCP handshake, TLS handshake) actually happens the moment you issue the first request — this trips people up when they assume "I created the client, so it must have already connected," then get confused why their first real request has extra latency (that's the handshake cost, paid on that call).

### Pattern C — Pre-warmed/pooled (connection pools: HikariCP for databases, Apache HttpClient pooling, `http.Agent` in Node)
```java
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(10);
config.setMinimumIdle(2); // pool eagerly opens 2 connections at startup, before any query runs
HikariDataSource ds = new HikariDataSource(config);
```
Here, connections open **before you ask for one** — the pool proactively establishes and holds a set of already-handshaked connections so your actual request-time code just borrows one (paying zero handshake latency) instead of paying for a fresh TCP+TLS handshake on every single call. This is the entire performance argument for connection pooling, and it's *why* database drivers and high-throughput HTTP clients are built this way rather than opening fresh per call.

## Part 3 — When does it actually close, and who decides?

This is where most developer confusion lives, because "close" isn't one event — there are at least four different things people mean by it:

### 3.1 Application-level close (your code calls `.close()`)
```java
try (Socket socket = new Socket(host, port)) {
    // use it
} // socket.close() called automatically here — try-with-resources
```
Calling `close()` on a raw socket sends a TCP **FIN** — a clean, orderly shutdown telling the other side "I have no more data to send." The peer can still finish sending its own pending data before it also sends FIN (this asymmetry is called a "half-close").

### 3.2 Pooled-client "close" often does NOT close the TCP connection at all
This is the single biggest source of confusion for developers using Tomcat/Netty/OkHttp/etc:
```java
response.close(); // Apache HttpClient: releases the connection back to the POOL — doesn't send FIN
```
When you're using a pooled HTTP client, calling `.close()` on a *response* or *request context* almost always just returns the underlying TCP connection to the pool for reuse — the actual socket stays open on the wire, waiting for the *next* request to reuse it (this is HTTP `Connection: keep-alive` in action, from the HTTP/1.1 note in [[OSI Layer 7 - Application Layer Protocols and Technologies]]). The real TCP FIN only gets sent later, when:
- The pool's idle-connection eviction timer decides the connection has been unused too long, or
- The pool is shut down entirely (`client.close()` on the client object itself, not a response), or
- **The server** decides to close it first (its own idle timeout, e.g. Tomcat's default `keepAliveTimeout`), which the client discovers on its *next* attempted use (a `Connection reset by peer` or a clean-looking write that then fails)

### 3.3 Protocol-driven close (the peer decided, not you)
- HTTP/1.0: closes after every single response, always (no keep-alive at all — see the HTTP evolution deep-dive in the Layer 7 note).
- HTTP/1.1 & HTTP/2: stays open until an idle timeout on *either* side fires, or a `Connection: close` header is explicitly sent, or the process exits.
- HTTP/2: additionally, individual **streams** (logical requests) close independently via `END_STREAM` flags, while the underlying single TCP connection stays open and gets reused for the next request — this is why in HTTP/2 you genuinely cannot reason about "a connection" and "a request" as the same lifecycle the way you could in HTTP/1.0.

### 3.4 OS/kernel-forced close (neither side's app code decided)
- **Idle OS-level timeouts** on the connection (rare for direct connections, common for anything behind a **load balancer or NAT gateway** — cloud LBs frequently silently drop idle connections after 60-350 seconds depending on provider, which is a very common "works locally, fails in prod through the LB" bug).
- **RST** instead of a clean FIN — happens when a process crashes, a firewall/security-group rule blocks the connection mid-flight, or code writes to an already-closed socket. This is why your GCP security-group rules earlier in this vault matter operationally, not just at connection-open time — a mid-connection SG change can RST live connections.

## Part 4 — "But my framework abstracts all of this... so why do I still have to manage it?"

This is the core of your second question, and the honest answer is: **frameworks abstract the mechanics (syscalls, buffer management, event-loop plumbing, protocol-compliant byte framing) — they do not, and structurally cannot, abstract the *policy* decisions**, because policy is inherently about your application's specific tradeoffs (latency vs resource usage, correctness vs throughput) that no generic library can decide for you. Concretely, here's what's abstracted vs what you still drive:

| Abstracted by the framework (you never touch this) | Still your responsibility (policy, not mechanics) |
|---|---|
| Raw `socket()`/`bind()`/`accept()` syscalls | Server thread-pool size / event-loop worker count |
| TCP handshake bytes, retransmission on loss | Connect timeout — how long to wait before giving up |
| TLS handshake, cipher negotiation, cert validation logic | Which TLS versions/ciphers to *allow* (a config policy) |
| HTTP request/response line parsing, chunked encoding | Read timeout — how long to wait for a slow/stuck response |
| HTTP/2 frame multiplexing over one TCP connection | Max connections per pool, max requests per connection |
| Buffer allocation for incoming bytes | What to do when the buffer/queue is full (backpressure policy) |
| FIN/ACK sequencing on a clean shutdown | *When* to decide a connection is stale and should be evicted |

This is precisely why every serious HTTP client/server exposes configuration for **timeouts, pool sizes, and keep-alive duration** — not because the abstraction is incomplete, but because those numbers encode business/operational tradeoffs unique to your service (a payments API should time out fast and retry; a long-running report-generation call shouldn't).

### Concrete example: Tomcat (thread-per-connection) vs Netty (event loop) — same problem, different mechanics, same policy burden on you

**Tomcat's classic model:** one OS thread per connection (or per request, depending on connector). The framework abstracts the accept loop and thread handoff completely — you never see a raw socket. But *you* still configure `maxThreads`, `connectionTimeout`, `keepAliveTimeout` in `server.xml`/`application.properties`, because the framework has no way to know how much concurrency your hardware/downstream dependencies can actually sustain. Set `maxThreads` too low, and legitimate requests queue and time out under load your server could otherwise handle — that's not a framework bug, it's an unset policy decision.

**Netty's model:** a small pool of event-loop threads multiplexes potentially thousands of connections via non-blocking I/O (`epoll` on Linux) — no per-connection thread at all, which is why Netty scales to far more concurrent connections per box than thread-per-connection servers. But you still explicitly wire up **handlers** in a pipeline (`ChannelPipeline`) that decide what happens on connect/read/close events, and you still configure idle-state handlers (`IdleStateHandler`) to decide when a quiet connection should be proactively closed — because Netty, being general-purpose, has no idea whether "quiet for 30 seconds" means "client went away" or "client is legitimately still thinking."

**Java's `HttpClient` (client side):** abstracts connection reuse and HTTP/2 multiplexing entirely — you just call `.send()`. But you still set `.connectTimeout()` on the client builder and `.timeout()` per-request, because only you know whether a 500ms API call hanging for 30 seconds is acceptable for your use case or should fail fast.

The pattern across all of these: **the framework guarantees protocol correctness (it will never produce a malformed TCP/TLS/HTTP exchange); it cannot guarantee your operational tradeoffs are sensible, because those are business decisions, not protocol facts.**

## Part 5 — Common developer confusions, answered directly

**"I called `.close()` on my HTTP response — why does `netstat`/`ss` still show the TCP connection as ESTABLISHED?"**
Because you closed the *response* (pool release), not the *connection* (see 3.2). This is correct, intended pooling behavior, not a leak — unless the pool itself never shrinks, which would be a real leak.

**"Why did my request fail with 'Connection reset by peer' on a connection that worked fine 5 minutes ago?"**
The server (or a load balancer/NAT in between) closed it due to an idle timeout you didn't know about, and your client-side pool didn't detect that until it tried to reuse the stale connection. Fix: configure your client pool's max idle time to be *shorter* than whatever the server/LB's idle timeout is, so your pool proactively retires connections before the other side kills them out from under you.

**"My server has plenty of CPU/memory headroom but still can't handle more concurrent connections — why?"**
Almost always thread-pool sizing (Tomcat-style) or OS-level file descriptor limits (`ulimit -n` — every open socket is a file descriptor on Linux) — a policy/config ceiling, not an actual hardware ceiling. Check `ulimit -n` and the server's max-thread/max-connections setting before assuming you need a bigger machine.

**"Does HTTP/2 mean I don't need connection pooling anymore?"**
You still need *a* connection open per origin (host+port+scheme), and most HTTP/2 clients pool a small number of connections per origin (sometimes just one) since a single connection already multiplexes many concurrent streams — so "pooling" shifts from "many TCP connections" to "many streams over few connections," but you still configure max-concurrent-streams and connection limits per origin.

**"Why does my Java client talking to a Python server occasionally hang forever with no error?"**
Almost always a missing read/connect timeout — most HTTP client libraries do **not** set a timeout by default, meaning "wait forever" is the literal default policy until you override it. This is the single most common production incident cause tied to everything in this note: the framework correctly abstracted the mechanics, but nobody set the policy.

---
Related concepts: [[TLS]] · [[TCP]] · [[HTTP/2]] · [[Netty]] · [[Connection Pooling]]
