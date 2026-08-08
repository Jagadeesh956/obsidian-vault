---
title: "RUM for Frontend Monitoring (Dynatrace Focus)"
date: 2026-08-07
tags:
  - SRE
  - Frontend Monitoring
  - RUM
  - Dynatrace
  - Observability
summary: "Diagram-first guide for beginners: why frontend monitoring is hard, how RUM works, and how Dynatrace telemetry actually flows for auto-injected vs. agentless deployments."
---

## Dynatrace RUM: Full End-to-End Model (Top-Level)

```text
+-----------------------------+
|  End User Browser           |
|-----------------------------|
| - Loads frontend app        |
| - Runs Dynatrace RUM JS     |
| - Captures user actions     |
| - Captures JS errors        |
| - Captures page timings     |
+-------------+---------------+
              |
              | HTTPS Beacon (RUM telemetry, POST, text/plain)
              v
+-----------------------------------------------------+
|  Beacon Endpoint (ONE of two, decided at setup time) |
|-------------------------------------------------------
| A) OneAgent on your own web/app server (auto-inject) |
|    same-origin, path ends in  /rb_<id>               |
|                                                        |
| B) Cluster ActiveGate in Dynatrace SaaS (agentless)   |
|    cross-origin, path  /bf  (needs CORS allowlist)    |
+-------------+-----------------------------------------+
              |
              | forwarded internally
              v
+-----------------------------+
| Dynatrace Backend           |
| (SaaS Tenant / Managed)     |
|-----------------------------|
| - Session stitching         |
| - Frontend<->backend link   |
|   via x-dynatrace header    |
| - Analytics + storage       |
| - AI-driven anomaly detect  |
+-------------+---------------+
              |
              | Query / visualize
              v
+--------------------------------------------------+
| Consumers of Observability Data                  |
|--------------------------------------------------|
| 1) RUM dashboards (UX health, Apdex, load times) |
| 2) Error dashboards (JS exceptions, impacted users)|
| 3) Service dashboards (frontend -> backend links)|
| 4) SRE/Operations dashboards (SLO/alerts/trends) |
| 5) Business dashboards (journeys, conversion)    |
+--------------------------------------------------+
```

> Real-world correction vs. a generic RUM diagram: Dynatrace doesn't route every browser through a single fixed "Edge -> ActiveGate -> Backend" pipe. **Which endpoint the beacon goes to is a setup-time choice** (auto-injected vs. agentless), and that choice changes whether the call is same-origin or cross-origin, which is exactly why CORS/CSP problems are the #1 real-world RUM outage cause.

---

## Why frontend monitoring is tricky

Frontend code runs in the user browser, not on your backend server.

Your frontend server usually only serves artifacts (HTML/CSS/JS). Once those files are delivered, behavior depends on:

- browser runtime
- user device CPU/memory
- network quality
- third-party scripts
- render and main-thread blocking

So backend logs alone cannot explain full user experience.

```text
+-------------------+          +--------------------------+
| Frontend Server   |  serves  | Browser Runtime          |
| (artifact host)   +--------->| (actual execution place) |
+-------------------+          +--------------------------+
```

---

## What happens if we use only browser logs

```text
+--------------------+
| console.log/error  |
| in browser         |
+---------+----------+
          |
          | stays local
          v
+--------------------------+
| User DevTools only       |
| (not centralized by default)
+--------------------------+
```

This causes gaps:
- no central visibility
- no large-scale trend analysis
- hard to correlate with backend traces

---

## How RUM solves the gap (generic flow)

```text
+-------------------------+
| Browser + RUM JS        |
+-----------+-------------+
            |
            | capture timings, actions, errors
            v
+-------------------------+
| RUM Beacon Endpoint     |
+-----------+-------------+
            |
            | process + aggregate
            v
+-------------------------+
| Observability Platform  |
+-----------+-------------+
            |
            | dashboards/alerts/queries
            v
+-------------------------+
| SRE + Dev + Product     |
+-------------------------+
```

---

## Dynatrace deployment modes — the fork that matters most

This is the piece most beginner diagrams skip, and it's the one that actually determines your network/CSP setup.

## Mode A) Auto-injected frontend (OneAgent is on your web/app server)

```text
+------------------+   GET /index.html   +---------------------------------+
| User Browser     +--------------------->  Web/App Server (has OneAgent)  |
+--------+---------+                     +----------------+-----------------+
         ^                                                |
         |     HTML response, RUM JS AUTO-INJECTED        |
         |     by OneAgent into <head> before it           |
         |     leaves the server                           |
         +--------------------------------------------------+
                                                            v
                                          +-------------------------------+
                                          | Browser executes injected     |
                                          | RUM JS, starts capturing      |
                                          +---------------+---------------+
                                                          |
                                                          | Beacon POST, SAME-ORIGIN
                                                          | path: /rb_<id>
                                                          | (or /myapp/rb_<id> for Java/IIS)
                                                          v
                                          +-------------------------------+
                                          | OneAgent on that SAME server  |
                                          | intercepts request at /rb_*   |
                                          | and forwards internally       |
                                          +---------------+---------------+
                                                          |
                                                          v
                                          +-------------------------------+
                                          | Dynatrace Backend (SaaS/Mgd)  |
                                          +-------------------------------+
```

Key real-world properties:
- No CORS needed — the beacon URL is on the **same origin** as your app, since OneAgent intercepts it right there on your own server.
- No manual script tag required — OneAgent injects the RUM JS server-side, transparently, into the HTML response.
- This is why auto-injected setups are usually the path of least friction for internal enterprise apps.

## Mode B) Agentless frontend (no OneAgent on the web server)

```text
+------------------+   GET /index.html   +---------------------------------+
| User Browser     +--------------------->  CDN / Static Web Server        |
+--------+---------+                     |  (RUM JS manually inserted      |
         ^                                |   as a <script> tag)            |
         |         HTML + manual          +---------------+------------------+
         |         RUM script tag                          
         +-------------------------------------------------+
                                                            v
                                          +-------------------------------+
                                          | Browser executes RUM JS       |
                                          +---------------+---------------+
                                                          |
                                                          | Beacon POST, CROSS-ORIGIN
                                                          | path: /bf
                                                          | REQUIRES CORS allowlist entry
                                                          v
                                          +-------------------------------+
                                          | Cluster ActiveGate             |
                                          | (Dynatrace SaaS infrastructure)|
                                          +---------------+---------------+
                                                          |
                                                          v
                                          +-------------------------------+
                                          | Dynatrace Backend (SaaS)       |
                                          +-------------------------------+
```

Key real-world properties:
- Beacons are **cross-origin by default**, hitting Dynatrace's own Cluster ActiveGate domain directly.
- Because it's cross-origin, the browser enforces CORS — you **must** add the frontend's origin to the beacon origin allowlist (Settings > Real User Monitoring > Beacon origins for CORS), or beacons silently get blocked and RUM data just stops appearing with no obvious error to the end user.
- Static sites, SPAs hosted on plain CDNs, and third-party-hosted frontends where you can't install OneAgent typically use this mode.
- You *can* redirect an agentless frontend's beacons to a Cluster ActiveGate explicitly, or point an auto-injected app's beacons to a Cluster ActiveGate instead of OneAgent — it's a per-application setting, not a fixed architecture.

---

## Dynatrace flow identities (box-arrow views)

## 1) First page load and bootstrap (auto-injected case)

```text
+------------------+      GET index.html      +-------------------------+
| User Browser     +------------------------->| Web/App Server          |
+--------+---------+                          | (OneAgent installed)    |
         ^                                    +------------+------------+
         |                                                 |
         |   HTML with RUM JS injected                    |
         |   into <head> by OneAgent, transparently        |
         +-------------------------------------------------+
                                                           v
                                           +-------------------------------+
                                           | Browser parses HTML, executes |
                                           | injected RUM JS, initializes  |
                                           | session + starts timers       |
                                           +-------------------------------+
```

## 2) User action and backend call correlation (real mechanism: x-dynatrace header)

```text
+--------------------+
| User clicks action |
+---------+----------+
          |
          v
+-------------------------------+
| Browser App (JS)              |
| triggers fetch/XHR            |
| RUM JS attaches               |
| 'x-dynatrace' header to the   |
| outgoing request               |
+---------------+---------------+
                |
                v
+-------------------------------+
| Backend API / Microservice    |
| (has its own OneAgent)        |
| reads x-dynatrace header,     |
| links this request to the     |
| SAME browser session/action   |
+---------------+---------------+
                |
                v
+-------------------------------+
| API response to browser       |
+---------------+---------------+
                |
                | RUM JS sends beacon:
                | action timing, resource
                | timing, outcome
                v
+-------------------------------+
| Beacon endpoint (OneAgent or  |
| Cluster ActiveGate, per mode) |
+---------------+---------------+
                |
                v
+-------------------------------+
| Dynatrace backend stitches    |
| frontend action + backend     |
| trace using the SAME          |
| x-dynatrace tag value         |
+-------------------------------+
```

> This is the concrete mechanism that makes "frontend -> backend" traces possible: it's a shared tag/header (`x-dynatrace`) injected by RUM JS on the way out and read by a backend OneAgent on the way in — not a generic "correlation engine" guessing based on timing.

## 3) JavaScript error capture path

```text
+--------------------------+
| JS error in browser      |
| (runtime/promise/reject) |
+------------+-------------+
             |
             v
+--------------------------+
| RUM JS captures:         |
| - message                |
| - stack                  |
| - URL                    |
| - browser/device info    |
+------------+-------------+
             |
             v
+--------------------------+
| Beacon endpoint          |
| (OneAgent or ActiveGate) |
+------------+-------------+
             |
             v
+--------------------------+
| Dynatrace error analytics|
| + impacted session views |
+--------------------------+
```

## 4) Slow page diagnosis path

```text
+------------------------------+
| User reports "page is slow" |
+---------------+--------------+
                |
                v
+------------------------------+
| RUM timings captured         |
| - nav timing                 |
| - resource waterfall         |
| - long tasks                 |
+---------------+--------------+
                |
                v
+------------------------------+
| Dynatrace breakdown          |
| Frontend delay vs API delay  |
| (using x-dynatrace-linked    |
|  backend trace, not a guess) |
+---------------+--------------+
                |
                v
+------------------------------+
| Team picks correct owner     |
| (frontend / backend / infra) |
+------------------------------+
```

---

## Session Replay note (often bundled with RUM, worth knowing)

```text
+--------------------------+
| Browser captures DOM/    |
| interaction snapshots    |
+------------+-------------+
             |
             | POST, content-type: application/octet-stream
             | (heavier payload than a normal beacon)
             | may be preceded by an OPTIONS preflight
             v
+--------------------------+
| Same beacon endpoint     |
| (OneAgent or ActiveGate) |
+--------------------------+
```

Session Replay rides the same beacon path as regular RUM telemetry but with a different content type and materially larger payloads — worth knowing before you assume "RUM traffic" is uniformly small.

---

## Local/private network telemetry architecture

## A) Dynatrace SaaS, agentless frontend, enterprise network

```text
+------------------+   HTTPS beacon, cross-origin   +----------------------------+
| Browser (corp)   +-------------------------------> | Corp Egress / Proxy / WAF |
+--------+---------+                                 +-------------+--------------+
                                                                    |
                                                                    v
                                                    +-------------------------------+
                                                    | Cluster ActiveGate             |
                                                    | (Dynatrace SaaS infrastructure)|
                                                    +-------------------------------+
```

Needs:
- outbound access to Dynatrace's Cluster ActiveGate endpoint
- proxy/firewall allow rules for that domain
- CSP `connect-src` allowing the beacon domain
- beacon origin CORS allowlist entry for the corp frontend's origin

## B) Dynatrace Managed, auto-injected frontend (internal, no public egress needed for beacons)

```text
+------------------+   Beacon, SAME-ORIGIN    +----------------------------+
| Browser (internal)+------------------------> | Internal Web/App Server   |
+------------------+   path: /rb_<id>          | (OneAgent installed)      |
                                                +-------------+--------------+
                                                              |
                                                              | internal forward
                                                              v
                                                +----------------------------+
                                                | Dynatrace Managed cluster  |
                                                | (private DNS/FQDN)         |
                                                +----------------------------+
```

Needs:
- OneAgent installed and healthy on the hosting server (not the browser's network path — this is a server-side agent)
- internal DNS/TLS trust between that server and the Managed cluster
- privacy controls for captured payloads (masking rules)

> Real-world nuance: in the auto-injected case, the *browser itself* never needs a path to Dynatrace at all — it only talks to your own server. It's your server's OneAgent that has to reach the Managed cluster. This flips the usual "browser needs firewall exceptions" assumption people bring in from the agentless case.

---

## Deployment validation flow (must-check)

```text
+--------------------------------+
| 1. Is this app auto-injected   |
|    or agentless? (check first, |
|    changes everything below)   |
+---------------+-----------------+
                |
                v
+--------------------------------+
| 2. RUM JS present in page?     |
|    (view-source or DevTools)   |
+---------------+-----------------+
                |
                v
+--------------------------------+
| 3. Beacon calls firing?        |
|    Auto-injected: look for     |
|      /rb_<id>                  |
|    Agentless: look for /bf     |
+---------------+-----------------+
                |
                v
+--------------------------------+
| 4. Beacon blocked?             |
|    - Agentless: check CORS     |
|      beacon origin allowlist   |
|    - Auto-injected: check      |
|      OneAgent process health   |
|    - Either: check CSP         |
|      connect-src / proxy       |
+---------------+-----------------+
                |
                v
+--------------------------------+
| 5. Data in Dynatrace RUM UI?   |
+---------------+-----------------+
                |
                v
+--------------------------------+
| 6. x-dynatrace header present  |
|    on backend calls? (confirms |
|    frontend<->backend linking) |
+---------------+-----------------+
                |
                v
+--------------------------------+
| 7. Dashboards + alert rules    |
|    validated                   |
+--------------------------------+
```

---

## Common mistakes

- Assuming backend APM = full user visibility
- Not knowing which mode you're in (auto-injected vs. agentless) before debugging a "missing RUM data" ticket — the fix is completely different for each
- For agentless setups: forgetting the beacon origin CORS allowlist entry, so beacons fire but are silently rejected by the browser
- For auto-injected setups: assuming beacons need a network path to the internet, when they actually just need OneAgent healthy on the local server
- Ignoring CSP `connect-src`/`script-src` restrictions blocking either the RUM JS load or the beacon POST
- No privacy/session-replay masking review before enabling Session Replay
- No business-action tagging for key flows
- Not checking for the `x-dynatrace` header when a frontend-to-backend trace link is "missing" — it's usually a stripped header somewhere in a gateway/proxy hop, not a Dynatrace bug

---

## Final takeaway

Frontend health is real only when browser-side signals are visible.

Dynatrace RUM closes that gap by collecting telemetry from the actual execution point (user browser), but *how* that telemetry gets back to Dynatrace — same-origin via OneAgent, or cross-origin via Cluster ActiveGate — is a concrete architectural choice made per application, not an implementation detail. Most real-world "RUM isn't showing data" incidents trace back to that fork: the wrong assumption about which mode is in play, a missing CORS allowlist entry, or a CSP rule blocking one specific hop.

> If the user experience is not observable at the browser edge, incident response is always delayed — and if you don't know whether you're auto-injected or agentless, you'll debug the wrong layer first.
