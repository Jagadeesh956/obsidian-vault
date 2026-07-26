# Authentication vs Authorization — History, Mechanisms, and RBAC

> Related: [[OSI Layer 7 - Application Layer Protocols and Technologies]] · [[OSI Layer 6 - Presentation Layer Protocols and Technologies]] · [[Application Connection Lifecycle - Mapping Protocols to Code]]

## Part 1 — The core distinction

**Authentication (AuthN): "Who are you?"** — verifying identity. The output of authentication is a confirmed identity (a user, a service, a device).

**Authorization (AuthZ): "What are you allowed to do?"** — given a confirmed identity, deciding what actions/resources it can access. The output of authorization is a yes/no (or scoped) decision on a specific action.

The classic analogy that actually holds up: **a passport is authentication** (proves who you are), **a visa is authorization** (proves what you're allowed to do once your identity is established). Notice these are deliberately issued by *different authorities* in real life, and the same separation shows up in software: it's entirely normal (and good practice) for the system that authenticates you to be a completely different component from the system that authorizes each specific action — this is why you'll see dedicated "Identity Providers" (Okta, Auth0, Keycloak) handling AuthN while application-level RBAC/policy engines handle AuthZ separately.

**Why conflating them causes real bugs:** the most common security mistake in web apps is checking "is this request authenticated" (has a valid session/token) and stopping there — forgetting to *also* check "is this specific authenticated user allowed to do *this specific thing*." This exact class of bug has a name: **IDOR (Insecure Direct Object Reference)** — e.g., a logged-in user changing `/api/orders/1234` to `/api/orders/1235` in the URL and successfully viewing someone else's order, because the endpoint checked AuthN (valid session) but never checked AuthZ (does *this* session's user own *this* order).

## Part 2 — How this plays out in a real web request

1. User submits credentials → **AuthN** happens once, at login.
2. Server issues a **credential artifact** (session cookie, JWT, or opaque token) representing "this identity is confirmed."
3. Every subsequent request carries that artifact.
4. On each request, middleware/gateway **re-validates** the artifact (is the session still valid? Is the JWT signature valid and unexpired?) — this re-validation is authentication happening *again*, cheaply, without re-asking for a password.
5. Then, separately, the application checks **AuthZ** for the specific action: does this user's role/permissions allow `DELETE /api/orders/1235`? This can happen at multiple points — API gateway, middleware, or deep inside business logic (e.g., "can only delete your *own* orders" requires data-level context that a gateway usually can't evaluate, only the application can).

This is why real systems have **two conceptually separate failure modes**: `401 Unauthorized` (actually means "not authenticated" — HTTP's naming here is famously confusing) and `403 Forbidden` (authenticated, but not authorized for this specific action).

---

## Part 3 — Historical evolution: what problem each mechanism solved that its predecessor couldn't

### 3.1 HTTP Basic Auth (1996, RFC 7617 — originally part of HTTP/1.0)
- **Problem it solved:** Early web apps had *no* standardized way to gate access at all — sites rolled their own ad hoc schemes.
- **How it works:** Client sends `Authorization: Basic base64(username:password)` on every single request.
- **Fatal weakness:** Base64 is *encoding*, not encryption — trivially reversible. Without TLS (see [[OSI Layer 6 - Presentation Layer Protocols and Technologies]]), credentials are sent in the clear on every request. Even over TLS, sending the raw password on every single request is bad practice — it maximizes how often the most sensitive secret you own is transmitted.
- **Still used today:** Only for low-stakes internal tooling or as a fallback, almost always paired with mandatory TLS.

### 3.2 Session-based auth (cookies) — the web's real default for decades
- **Problem it solved:** Basic Auth's "resend the password every request" problem. Instead, authenticate once, get a random opaque **session ID** stored in a cookie, and the server maps that ID to identity in server-side storage.
- **How it works:** Login → server creates a session record (in memory, Redis, a DB table) → sends `Set-Cookie: session_id=...` → browser automatically attaches this cookie on every subsequent request to that domain → server looks up the session ID against its store to re-derive identity.
- **Advantages:** The actual password is only ever sent once (at login); sessions can be instantly revoked server-side (just delete the session record) — this instant-revocation property is something later stateless approaches (JWT) explicitly gave up, as you'll see below.
- **Disadvantages:** **Stateful** — every server instance needs access to the same session store, which becomes a real scaling/architecture constraint (this is exactly why "sticky sessions" or a shared Redis session store became standard infrastructure). Also vulnerable to **CSRF** (Cross-Site Request Forgery) if not paired with additional protections, because browsers attach cookies automatically to *any* site's requests to your domain — a malicious site can trigger authenticated requests on your behalf without ever seeing your session ID.
- **Modern hardening:** `HttpOnly` (JS can't read the cookie, blocks XSS-based theft), `Secure` (only sent over TLS), `SameSite=Strict/Lax` (blocks the cross-site CSRF vector directly at the browser level — this flag effectively solved most of CSRF without extra server-side tokens once browsers adopted it).
- **Owner:** IETF, cookie mechanics themselves are RFC 6265.

### 3.3 API keys
- **Problem solved:** Session cookies assume a browser with a human logging in interactively — machine-to-machine and third-party API access needed something simpler, long-lived, and non-interactive.
- **How it works:** A static secret string issued once, sent typically as a header (`X-API-Key` or `Authorization: Bearer <key>`), checked against a stored value.
- **Disadvantages:** No built-in expiry, no built-in scoping (a leaked key is often all-or-nothing access), no standard revocation/rotation mechanism — entirely up to whoever implements it. This crudeness is exactly what OAuth was later built to fix for third-party access scenarios specifically.

### 3.4 Kerberos (1988, MIT) — the enterprise SSO problem
- **Problem solved:** In a large internal network (a university, then enterprises), users needed to authenticate *once* and access many different internal services without re-entering credentials for each, **without** sending passwords over the network to each service individually.
- **How it works:** A trusted third party (**Key Distribution Center**) issues time-limited **tickets** after initial login; the user presents a ticket to each service instead of a password, and the service trusts the KDC's signature on the ticket rather than verifying credentials itself.
- **Legacy/relevance:** Still the backbone of Windows Active Directory domain authentication today. This "central authority issues short-lived proof, services trust the authority instead of re-checking passwords" pattern is the direct conceptual ancestor of OAuth/OIDC tokens decades later — same idea, different era/protocol.

### 3.5 LDAP / Active Directory — centralized identity directory problem
- **Problem solved:** Before centralized directories, every application maintained its *own* user database — meaning a company with 50 internal tools had 50 separate places an employee's password/permissions lived, a nightmare for offboarding (did IT actually revoke access everywhere when someone left?).
- **How it works:** A hierarchical directory service (LDAP protocol, RFC 4511) storing users, groups, and org structure in one place; applications authenticate *against* the directory rather than maintaining their own user table.
- **Relevance today:** Still the backbone of most corporate identity (Active Directory), though modern cloud-native companies increasingly use cloud IdPs (Okta, Azure AD/Entra ID, Google Workspace) that speak newer federated protocols (SAML/OIDC) instead of/alongside raw LDAP.

### 3.6 SAML (2002, OASIS) — federated SSO across organizational boundaries
- **Problem solved:** Kerberos/LDAP solve SSO *within* one organization's network. But by the early 2000s, businesses needed SSO **across** organizational boundaries — e.g., log into your company's IdP once, and get into a third-party SaaS vendor's app without that vendor ever seeing your password.
- **How it works:** XML-based. An **Identity Provider (IdP)** authenticates the user and issues a signed **SAML assertion** ("this is definitely [email protected], confirmed at this time"); the **Service Provider (SP)**, i.e. the third-party app, trusts assertions signed by that IdP's known key and grants access without ever handling the password itself.
- **Disadvantages:** XML-based, verbose, genuinely painful to implement correctly (signature validation edge cases have caused real security vulnerabilities historically); designed for browser-based enterprise SSO, awkward for mobile apps and APIs — which is a big part of why OAuth/OIDC displaced it for anything beyond classic enterprise web SSO.
- **Owner:** OASIS. Still extremely common in enterprise B2B SaaS ("Login with your company SSO") even today.

### 3.7 OAuth 1.0 (2007) → OAuth 2.0 (2012) — delegated access, not authentication
- **Problem solved:** Before OAuth, if you wanted a third-party app to access your data on another service (e.g., a photo-printing site accessing your Flickr photos), the *only* option was giving that third party your actual Flickr password — a massive, obviously bad trust requirement. OAuth's entire point: **delegated, scoped access without sharing credentials.**
- **Critical, widely-confused fact: OAuth is an *authorization* protocol, not an authentication protocol.** An OAuth access token proves "this app has permission to do X on your behalf" — it does **not** inherently prove who the human is. Using raw OAuth as if it were login (a very common early mistake, "Sign in with Facebook" implementations before OIDC existed) is exactly the AuthN/AuthZ conflation from Part 1, and caused real vulnerabilities.
- **How OAuth 2.0 works (Authorization Code flow, the standard web flow):** App redirects user to the provider's login page → user authenticates *directly with the provider*, never revealing credentials to the app → provider redirects back with a short-lived **authorization code** → app exchanges that code (server-to-server, with a secret) for an **access token** → app uses that access token, scoped to specific permissions ("read your profile," "read your calendar"), to call the provider's API on the user's behalf.
- **Why OAuth 2.0 replaced 1.0:** OAuth 1.0 required complex cryptographic request-signing on every call — genuinely painful to implement correctly. OAuth 2.0 simplified this by relying on TLS for transport security instead of per-request signatures, at the cost of being less secure if TLS itself is compromised — a deliberate, debated tradeoff.
- **PKCE (2015 addition, RFC 7636):** Solved a specific new problem — mobile/SPA apps *can't* keep a client secret truly secret (it'd be embedded in a distributable app binary or exposed in browser JS), so a malicious app could intercept the authorization code and exchange it itself. PKCE adds a dynamically generated proof (a code verifier/challenge pair) tying the token exchange to the specific app instance that started the flow, closing that hole without needing a static secret.
- **Owner:** IETF, RFC 6749 (OAuth 2.0 core), RFC 7636 (PKCE).

### 3.8 OpenID Connect — OIDC (2014) — adding authentication back on top of OAuth
- **Problem solved:** Exactly the gap called out above — OAuth alone doesn't standardize "prove who the user is." Everyone kept building incompatible, ad hoc identity layers on top of raw OAuth. OIDC is a thin, standardized identity layer built directly on top of OAuth 2.0.
- **How it works:** Adds a new token type, the **ID token** — a signed JWT (see below) specifically containing identity claims (user ID, email, name, when they authenticated) — alongside OAuth's existing access token. The access token is still "what this app can do"; the ID token is now explicitly "who this user is, attested by the provider."
- **Why this matters practically:** "Sign in with Google/GitHub/Microsoft" buttons on modern websites are OIDC, not raw OAuth — this is the fix for the exact AuthN/AuthZ conflation problem OAuth-as-login caused earlier.
- **Owner:** OpenID Foundation, built on IETF's OAuth 2.0.

### 3.9 JWT — JSON Web Token (2015, RFC 7519) — the stateless session problem
- **Problem solved:** Session-based auth (3.2) requires a server-side lookup on every request — fine for one monolith, painful for distributed microservices where every service would need to hit a shared session store just to validate identity on every call.
- **How it works:** A JWT is `base64(header).base64(payload).signature` — the payload carries claims (user ID, roles, expiry) directly, and the **signature** (typically HMAC or RSA/ECDSA) lets *any* service verify authenticity **without calling back to a central session store**, as long as it has the public key or shared secret to check the signature.
- **Advantages:** Stateless, horizontally scalable, self-contained (a service can extract roles/claims directly from the token instead of querying a user-info endpoint).
- **The tradeoff nobody can avoid:** JWTs traded away session's best property — **instant revocation**. A stolen/compromised JWT remains valid until it *expires*, no matter what the server does, since there's no central record to delete (unlike a session ID). Real systems work around this with **short expiries + refresh tokens** (a longer-lived, revocable token used only to mint new short-lived access JWTs) or a maintained token-blocklist (which re-introduces some statefulness, partially undoing JWT's core advantage — a genuine, unavoidable architectural tradeoff, not a solved problem).
- **Owner:** IETF, RFC 7519.

### 3.10 mTLS — Mutual TLS
- **Problem solved:** Regular TLS (see [[OSI Layer 6 - Presentation Layer Protocols and Technologies]]) only authenticates the *server* to the client (the padlock icon) — the server has no cryptographic proof of who the *client* is beyond whatever app-level token it sends. Service-to-service communication inside a microservices mesh needed strong, cryptographic mutual identity without a shared bearer token that could leak.
- **How it works:** Both sides present X.509 certificates during the TLS handshake (exactly the cert mechanism from your kubeadm PKI note) — the server validates the client's cert against a trusted CA, same as the client validates the server's.
- **Relevance to your K8s work:** this is exactly what **service meshes** (Istio, Linkerd) automate between pods — every pod gets a short-lived cert, and mTLS becomes the default AuthN layer for all pod-to-pod traffic, independent of whatever application-level auth each service also does.

### 3.11 MFA / 2FA — the password-only weakness
- **Problem solved:** Passwords alone are only "something you know" — reused across sites, phishable, guessable, and breach dumps make credential-stuffing attacks cheap and automatable.
- **How it works:** Requires a second, independent factor: "something you have" (SMS/authenticator app TOTP codes, hardware keys) or "something you are" (biometrics). TOTP (Time-based One-Time Password, RFC 6238) is the common authenticator-app standard — a shared secret plus the current time, run through HMAC, regenerates a new 6-digit code every 30 seconds on both sides independently.
- **Known weaknesses:** SMS-based 2FA is vulnerable to SIM-swapping attacks — a large part of why security guidance has shifted toward authenticator apps or hardware keys over SMS.

### 3.12 WebAuthn / FIDO2 / Passkeys (2019 onward) — solving phishing itself, not just weak passwords
- **Problem solved:** Even MFA doesn't stop *phishing* — a fake login page can still capture your password **and** relay a real-time OTP prompt to the real site (real-time phishing proxies exist for exactly this). The industry needed something that's cryptographically impossible to phish, not just "harder to guess."
- **How it works:** Public-key cryptography, generated and stored on your device (a phone's secure enclave, a hardware key like a YubiKey) — the private key **never leaves the device and is never transmitted anywhere, ever**. The site sends a challenge; the device signs it locally; the site verifies with the stored public key. Critically, the credential is cryptographically bound to the *specific domain* it was created for, so even a pixel-perfect phishing clone at a different domain simply cannot obtain a valid signature — phishing is structurally impossible, not just harder.
- **Owner:** FIDO Alliance + W3C (WebAuthn is the W3C-standardized browser API; FIDO2/CTAP is the underlying device protocol). "Passkeys" is the more recent consumer-facing branding (Apple/Google/Microsoft) for the same underlying WebAuthn technology, aimed at finally making this usable for mainstream consumer login, not just enterprise hardware keys.

---

## Part 4 — Authorization models: how "what are you allowed to do" gets structured

Authentication mechanisms (Part 3) answer "who." These models answer "what can they do," and they evolved for their own separate reasons.

### 4.1 ACL — Access Control Lists (earliest model)
- **How it works:** A list directly attached to each resource, naming which specific users/entities can perform which actions on it (classic Unix file permissions — `rwx` per user/group/other — are a primitive ACL).
- **Disadvantages:** Doesn't scale — with thousands of users and resources, maintaining a list *per resource* becomes unmanageable (adding a new employee means updating permissions on potentially thousands of individual resources).

### 4.2 RBAC — Role-Based Access Control
- **Problem it solved:** ACL's per-resource, per-user unmanageability. Instead of granting permissions to individuals directly, define **roles** (Admin, Editor, Viewer) with a fixed permission set, and assign users to roles — permission changes happen at the role level, automatically propagating to everyone in that role.
- **How it's implemented in real web apps:** A `roles` table, a `permissions` table, and a join table mapping roles to permissions; the user's assigned role(s) get embedded in their session/JWT claims, and middleware checks `if (!user.roles.includes('admin')) return 403` (or via a proper policy check, not string comparison, in production code) before executing sensitive actions.
- **Disadvantages:** Struggles with fine-grained, contextual rules ("editors can edit posts, but only their *own* posts, and only before the 24-hour edit window closes") — pure role membership can't express *conditional*, data-dependent logic, which is exactly what the next model solves.

### 4.3 ABAC — Attribute-Based Access Control
- **Problem it solved:** RBAC's inability to express contextual/conditional rules. Decisions are made from **attributes** — of the user (department, clearance level), the resource (owner, sensitivity tag), the action, and the environment (time of day, IP location) — combined via policy logic rather than fixed role membership.
- **How it works:** A policy engine evaluates rules like "allow if `user.department == resource.department` AND `time.hour between 9-18`" — genuinely flexible, but the policies themselves become a real artifact to write, test, and audit.
- **Real implementation:** **OPA (Open Policy Agent)** with its Rego policy language is the dominant modern implementation of this pattern — widely used inside Kubernetes admission controllers, API gateways, and service meshes to enforce exactly this kind of contextual policy, decoupled from application code.

### 4.4 ReBAC — Relationship-Based Access Control (Google Zanzibar model, 2019)
- **Problem it solved:** Neither RBAC nor ABAC cleanly express *relationship-graph* permissions at massive scale — "can user X view this specific document" when access depends on nested sharing (a folder shared with a team, which a user belongs to, which inherits from a parent folder...) the way Google Drive or Docs works. Google's internal **Zanzibar** system (the permissions backend behind Drive, YouTube, Photos) was built specifically for this, and the paper (published 2019) has since spawned open-source implementations.
- **How it works:** Models permissions as a graph of relationships ("user U is a *member* of team T," "team T has *viewer* access to document D") and answers authorization queries by graph traversal, checked at massive scale with strong consistency guarantees.
- **Real implementations:** **SpiceDB** (open-source, directly inspired by Zanzibar), **OpenFGA** (CNCF project — again, the same governance home as Kubernetes and gRPC from your earlier notes).

## Part 5 — Where enforcement actually happens in a real system

| Enforcement point | What it typically checks | Example technology |
|---|---|---|
| API Gateway / reverse proxy | Coarse AuthN (valid token?), sometimes coarse role checks | Kong, NGINX, cloud API Gateway |
| Service mesh sidecar | Service-to-service AuthN (mTLS), coarse network policy | Istio, Linkerd |
| Application middleware | Route-level AuthZ (does this role have access to this endpoint at all) | Express middleware, Spring Security filters |
| Business logic / data layer | Fine-grained, data-dependent AuthZ (does this user own *this specific record*) | Application code — this layer can never be fully pushed down to infra, since it needs business context |
| Policy engine (sidecar or embedded) | Centralized, auditable policy decisions decoupled from app code | OPA/Rego, OpenFGA |

The general industry trend across the last decade: push as much AuthZ as possible **out of scattered application code and into a centralized, declarative policy layer** (OPA, OpenFGA) — for the same reason centralized directories (3.5) replaced per-app user tables decades earlier: consistency, auditability, and not needing to trust every individual developer to get an `if` statement right in every single service.

---
Related concepts: [[TLS]] · [[JWT]] · [[OAuth 2.0]] · [[OpenID Connect]] · [[SAML]] · [[RBAC]] · [[OPA (Open Policy Agent)]] · [[WebAuthn]]
