# OSI Layer 6 — Presentation Layer Protocols and Technologies

> Back to [[Networking Protocols - Master Index (OSI 7-Layer Model)]]

Layer 6's job: make sure data means the same thing to both ends — **encoding, encryption/decryption, and compression** — translating between an application's internal data representation and a common wire format. Like Layer 5, most of what lives here today is implemented as a library an application calls, rather than a distinct protocol with its own header — but the concerns are very real and this is genuinely one of the more important layers in practice, just packaged differently than OSI's designers imagined.

## TLS/SSL — the layer's flagship example

- **Problem solved:** Applications need confidentiality (nobody else can read the data), integrity (nobody tampered with it), and authentication (I'm actually talking to who I think I am) — without every single application protocol reinventing cryptography badly. TLS's predecessor, SSL (Netscape, 1995), was built specifically because early web commerce had zero protection for credit card data in transit.
- **How it works:** A handshake negotiates a cipher suite and exchanges/derives symmetric session keys (using asymmetric crypto only for the initial key exchange, since symmetric crypto is far cheaper for bulk data), authenticates the server via its X.509 certificate chain (exactly the CA/cert mechanism from your kubeadm PKI doc — same underlying X.509 technology, different CA), then encrypts everything that follows.
- **Evolution:** SSL 2.0/3.0 → TLS 1.0/1.1 (both now deprecated, known-weak) → **TLS 1.2** (still widely used) → **TLS 1.3** (2018) which removed weak/legacy cipher options entirely, and cut the handshake from 2 round-trips to 1 (with 0-RTT resumption for repeat connections) — directly analogous to why QUIC folds TLS 1.3 into its own handshake, as covered in [[OSI Layer 4 - Transport Layer Protocols and Technologies]].
- **Advantages:** Ubiquitous, strong when configured correctly, transparent to the application layer above it (HTTPS is just HTTP wrapped in TLS).
- **Disadvantages:** Misconfiguration (weak ciphers, expired/self-signed certs — like the SAN mismatches from your kubeadm PKI discussion) is extremely common and silently weakens or breaks it; adds latency (handshake round trips) that protocols like QUIC exist partly to eliminate.
- **Owner:** IETF (TLS 1.3 is RFC 8446); the CA/trust ecosystem is governed by the CA/Browser Forum.
- **Implementations:** OpenSSL, BoringSSL (Google), every browser and OS crypto library, `kubectl`'s TLS connection to the API server, and literally the certs you inspected under `/etc/kubernetes/pki`.

## MIME — Multipurpose Internet Mail Extensions

- **Problem solved:** Original internet email (SMTP, see Layer 7) could only carry 7-bit ASCII text — no attachments, no non-English character sets, no images.
- **How it works:** Defines a way to label content with a **Content-Type** (e.g. `image/png`, `application/pdf`) and encode binary data into ASCII-safe text (Base64, quoted-printable) for transport over text-only protocols.
- **Relevance beyond email:** MIME types (`Content-Type` header) are exactly what every HTTP response uses today to tell a browser "this is JSON" vs "this is an image" — a Layer 6 labeling concern that HTTP absorbed directly.
- **Owner:** IETF, RFC 2045-2049.

## Data compression (gzip, Brotli, zstd)

- **Problem solved:** Reducing bytes-on-the-wire for cost/latency, without the application needing to know or care about the compression algorithm.
- **How it works:** Sender compresses the payload, signals which algorithm via a header (HTTP's `Content-Encoding: gzip`), receiver decompresses transparently.
- **Evolution:** gzip (1992, DEFLATE algorithm) → Brotli (Google, 2015, better compression ratio for web text/JS/CSS) → Zstandard/zstd (Facebook, 2016, extremely fast at good ratios — now common for large data transfer, container image layers, etc.)
- **Owner:** gzip is IETF RFC 1952; Brotli is RFC 7932; zstd is Meta-originated, now widely adopted independently.

## Character encoding (ASCII → Unicode/UTF-8)

- **Problem solved:** ASCII's 7-bit, 128-character set couldn't represent non-English text at all — a real presentation-layer problem (same bytes meaning different things, or being unrepresentable, across systems/languages).
- **How it works:** UTF-8 encodes the full Unicode character set in a variable-length, ASCII-backward-compatible way (any pure-ASCII text is already valid UTF-8), which is why it won over competing fixed-width encodings (UTF-16, UTF-32) for wire transmission — smaller for the overwhelmingly common case of mostly-ASCII text/code.
- **Owner:** Unicode Consortium.

---
Related concepts: [[TLS]] · [[iptables]] (for where encrypted traffic is/isn't inspectable) · [[CoreDNS]]
