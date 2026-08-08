---
title: "RUM for Frontend Monitoring (Dynatrace Focus)"
date: 2026-08-07
tags:
  - SRE
  - Frontend Monitoring
  - RUM
  - Dynatrace
  - Observability
summary: "Diagram-first guide for beginners: why frontend monitoring is hard, how RUM works, and how Dynatrace telemetry flows in SaaS/private network setups."
---

## Dynatrace RUM: Full End-to-End Model (Top-Level)

```text
+-----------------------------+
|  End User Browser           |
|-----------------------------|
| - Loads frontend app        |
| - Runs Dynatrace RUM agent  |
| - Captures user actions     |
| - Captures JS errors        |
| - Captures page timings     |
+-------------+---------------+
              |
              | HTTPS Beacon (RUM telemetry)
              v
+-----------------------------+
|  Edge / Ingress / Proxy     |
|  (CDN, WAF, Corp Proxy)     |
+-------------+---------------+
              |
              | Routed securely
              v
+-----------------------------+
|  Dynatrace ActiveGate       |
|-----------------------------|
| - Ingest / forward traffic  |
| - Controlled egress path    |
| - Optional buffering/routing|
+-------------+---------------+
              |
              | Processed telemetry
              v
+-----------------------------+
| Dynatrace Backend           |
| (SaaS Tenant / Managed)     |
|-----------------------------|
| - Session stitching         |
| - Frontend-backend correlate|
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
| Browser + RUM Agent     |
+-----------+-------------+
            |
            | capture timings, actions, errors
            v
+-------------------------+
| RUM Telemetry Endpoint  |
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

## Dynatrace flow identities (box-arrow views)

## 1) First page load and bootstrap

```text
+------------------+      GET index.html      +-------------------------+
| User Browser     +------------------------->| Frontend CDN/Server     |
+--------+---------+                          +------------+------------+
         ^                                                 |
         |       HTML + App JS + RUM script reference     |
         +-------------------------------------------------+
                                                           v
                                           +-------------------------------+
                                           | Dynatrace Agent Script Source |
                                           | (tenant script / CDN)         |
                                           +---------------+---------------+
                                                           |
                                                           v
                                           +-------------------------------+
                                           | Browser initializes RUM agent |
                                           +-------------------------------+
```

## 2) User action and backend call correlation

```text
+--------------------+
| User clicks action |
+---------+----------+
          |
          v
+-------------------------------+
| Browser App (JS)              |
| triggers fetch/XHR            |
+---------------+---------------+
                |
                v
+-------------------------------+
| Backend API / Microservice    |
+---------------+---------------+
                |
                v
+-------------------------------+
| API response to browser       |
+---------------+---------------+
                |
                | RUM agent sends action timing,
                | resource timing, outcome
                v
+-------------------------------+
| ActiveGate / Dynatrace ingest |
+---------------+---------------+
                |
                v
+-------------------------------+
| Dynatrace correlation engine  |
| links user action <-> service |
+-------------------------------+
```

## 3) JavaScript error capture path

```text
+--------------------------+
| JS error in browser      |
| (runtime/promise/reject) |
+------------+-------------+
             |
             v
+--------------------------+
| RUM agent captures:      |
| - message                |
| - stack                  |
| - URL                    |
| - browser/device info    |
+------------+-------------+
             |
             v
+--------------------------+
| ActiveGate / Ingest      |
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
+---------------+--------------+
                |
                v
+------------------------------+
| Team picks correct owner     |
| (frontend / backend / infra) |
+------------------------------+
```

---

## Local/private network telemetry architecture

Below are two practical deployment identities.

## A) Dynatrace SaaS with enterprise network

```text
+------------------+      HTTPS beacon       +-----------------------+
| Browser (corp)   +-----------------------> | Internet Egress/Proxy |
+--------+---------+                         +-----------+-----------+
         |                                               |
         |                                               v
         |                               +-------------------------------+
         +------------------------------>| Dynatrace SaaS Tenant         |
                                         | (RUM ingest + analytics)      |
                                         +-------------------------------+
```

Needs:
- outbound access to Dynatrace endpoints
- proxy/firewall allow rules
- CSP rules allowing agent and beacon domains

## B) Dynatrace Managed (inside private environment)

```text
+------------------+      HTTPS beacon      +----------------------------+
| Browser (internal)+---------------------> | Dynatrace Managed endpoint |
+--------+---------+                        | (private DNS/FQDN)         |
         |                                  +------------+---------------+
         |                                               |
         |                                               v
         |                                  +----------------------------+
         +--------------------------------->| Dynatrace Managed cluster  |
                                            | + optional ActiveGate      |
                                            +----------------------------+
```

Needs:
- internal DNS and TLS cert trust
- path open from user network to managed endpoint
- privacy controls for captured payloads

---

## Deployment validation flow (must-check)

```text
+--------------------------+
| 1. Agent script loads?   |
+------------+-------------+
             |
             v
+--------------------------+
| 2. Beacon calls sent?    |
+------------+-------------+
             |
             v
+--------------------------+
| 3. Beacon not blocked by |
|    CSP / proxy / firewall|
+------------+-------------+
             |
             v
+--------------------------+
| 4. Data appears in       |
|    Dynatrace RUM UI      |
+------------+-------------+
             |
             v
+--------------------------+
| 5. Dashboard + alert     |
|    rules validated       |
+--------------------------+
```

---

## Common mistakes

- Assuming backend APM = full user visibility
- Not validating beacon network path in corp networks
- Ignoring CSP/proxy restrictions
- No privacy masking review
- No business-action tagging for key flows

---

## Final takeaway

Frontend health is real only when browser-side signals are visible.

Dynatrace RUM closes that gap by collecting telemetry from the actual execution point (user browser) and making it available for SRE, engineering, and business dashboards.

> If the user experience is not observable at the browser edge, incident response is always delayed.
