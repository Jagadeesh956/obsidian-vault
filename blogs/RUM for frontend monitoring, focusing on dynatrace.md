---
title: "RUM for Frontend Monitoring (Dynatrace Focus)"
date: 2026-08-07
tags:
  - SRE
  - Frontend Monitoring
  - RUM
  - Dynatrace
  - Observability
summary: "Visual-first guide: real flow diagrams for Dynatrace RUM from browser telemetry to ActiveGate, Dynatrace backend, and dashboards."
---

## Table of contents

1. Dynatrace RUM: Full End-to-End Model (Top-Level)
2. Why frontend monitoring is hard
3. Why only console logs are not enough
4. Generic RUM solution flow
5. Dynatrace deployment identity A: Auto-injected (OneAgent)
6. Dynatrace deployment identity B: Agentless (Cluster ActiveGate)
7. Frontend-backend correlation path (real mechanism)
8. JS error capture flow
9. Slow page diagnosis flow
10. Local/private network architecture view
11. Validation flow (what to test in order)
12. How pushed data is used in Dynatrace UI (dashboards, alerts, triage)
13. Final takeaway

## Dynatrace RUM: Full End-to-End Model (Top-Level)

```mermaid
flowchart TB
  B["End User Browser<br/>- Runs frontend JS<br/>- Captures actions, errors, timings"]
  E["Edge / Proxy / WAF / CDN"]
  M{Beacon Mode}
  O["OneAgent on your app server<br/>Same-origin endpoint<br/>/rb_<id>"]
  A["Cluster ActiveGate<br/>Cross-origin endpoint<br/>/bf"]
  D["Dynatrace Backend<br/>SaaS or Managed"]
  R1["RUM UX Dashboards<br/>Apdex, page load, user actions"]
  R2["Error Dashboards<br/>JS exceptions, impacted sessions"]
  R3["Service Correlation Dashboards<br/>frontend to backend traces"]
  R4["SRE Dashboards and Alerts<br/>SLOs, trends, anomalies"]

  B -->|HTTPS beacon| E --> M
  M -->|Auto-injected| O --> D
  M -->|Agentless| A --> D
  D --> R1
  D --> R2
  D --> R3
  D --> R4
```

---

## Why frontend monitoring is hard

Frontend code executes in browser, not in your backend process.
Your server usually only serves artifacts, then user experience depends on browser/device/network conditions.

```mermaid
flowchart LR
  S["Frontend server/CDN<br/>serves HTML/CSS/JS"] --> U["User Browser<br/>executes app"]
  U --> X["User experience outcomes<br/>slow render, JS error, UI freeze"]
  X -->|not always visible in backend logs| G["Observability gap"]
```

---

## Why only console logs are not enough

```mermaid
flowchart TB
  L["console.log / console.error in browser"]
  V["User DevTools only"]
  N["No centralized view"]
  C["No correlation with backend traces"]
  O["Operations blind spots"]

  L --> V --> N
  V --> C
  N --> O
  C --> O
```

---

## Generic RUM solution flow

```mermaid
flowchart TB
  U["Browser + RUM Agent"]
  T["Capture telemetry<br/>page timings, actions, errors, XHR/fetch"]
  I["Ingest endpoint"]
  P["Processing + session stitching"]
  Q["Dashboards + alerts + investigation"]

  U --> T --> I --> P --> Q
```

---

## Dynatrace deployment identity A: Auto-injected (OneAgent)

```mermaid
sequenceDiagram
  participant Browser as User Browser
  participant App as App/Web Server (OneAgent)
  participant DT as Dynatrace Backend

  Browser->>App: GET index.html
  App-->>Browser: HTML with auto-injected RUM JS
  Browser->>App: Beacon POST /rb_<id> (same-origin)
  App->>DT: Forward beacon + metadata
  DT-->>DT: Session/action processing + correlation
```

Key points:
- same-origin beacon path
- OneAgent injects script in response
- usually easier for enterprise apps

---

## Dynatrace deployment identity B: Agentless (Cluster ActiveGate)

```mermaid
sequenceDiagram
  participant Browser as User Browser
  participant CDN as CDN/Static Host
  participant AG as Cluster ActiveGate
  participant DT as Dynatrace Backend

  Browser->>CDN: GET index.html
  CDN-->>Browser: HTML + manual RUM script tag
  Browser->>AG: Beacon POST /bf (cross-origin)
  AG->>DT: Forward telemetry
  DT-->>DT: Process + analytics + dashboards
```

Key points:
- cross-origin beacons
- CORS allowlist required for beacon origin
- common for static frontend hosting

---

## Frontend-backend correlation path (real mechanism)

```mermaid
flowchart LR
  A["Browser Action<br/>click/search/checkout"] --> B["XHR/fetch request"]
  B --> C["x-dynatrace header attached"]
  C --> D["Backend service<br/>OneAgent instrumented"]
  D --> E["Backend trace tagged<br/>with same dynatrace context"]
  E --> F["Dynatrace links<br/>frontend action to backend trace"]
```

---

## JS error capture flow

```mermaid
flowchart TB
  J["Runtime JS error / rejected promise"]
  K["RUM agent captures<br/>message + stack + URL + browser"]
  L["Beacon endpoint<br/>/rb_<id> or /bf"]
  M["Dynatrace error analytics"]
  N["Impacted users/pages/time windows"]

  J --> K --> L --> M --> N
```

---

## Slow page diagnosis flow

```mermaid
flowchart TB
  U["User reports slow page"]
  A["RUM captures<br/>navigation timing<br/>resource waterfall<br/>long tasks"]
  B["Dynatrace breakdown<br/>frontend delay vs API delay"]
  C["Owner routing<br/>Frontend / Backend / Infra"]
  D["Fix + verify in next user sessions"]

  U --> A --> B --> C --> D
```

---

## Local/private network architecture view

### SaaS path from enterprise network

```mermaid
flowchart LR
  B["Browser in corporate network"] --> P["Corp Proxy / Egress"]
  P --> A["Dynatrace Cluster ActiveGate (SaaS side)"]
  A --> D["Dynatrace SaaS backend"]
```

Needs:
- egress allow rules
- CSP + CORS config
- TLS/proxy compatibility

### Managed/private path

```mermaid
flowchart LR
  B["Browser internal"] --> S["Internal app server<br/>OneAgent"]
  S --> M["Dynatrace Managed cluster<br/>private DNS/FQDN"]
```

Needs:
- internal DNS and cert trust
- server-to-managed connectivity
- privacy masking policies

---

## Validation flow (what to test in order)

```mermaid
flowchart TB
  A["1. Identify mode<br/>auto-injected or agentless"]
  B["2. RUM JS present in page"]
  C["3. Beacon calls visible<br/>/rb_<id> or /bf"]
  D["4. Beacon blocked?<br/>check CORS/CSP/proxy/firewall"]
  E["5. Data visible in Dynatrace RUM UI"]
  F["6. Frontend-backend link works<br/>x-dynatrace context"]
  G["7. Dashboards + alert rules validated"]

  A --> B --> C --> D --> E --> F --> G
```

---

## How pushed data is used in Dynatrace UI (dashboards, alerts, triage)

Once browser telemetry reaches Dynatrace, it becomes usable in a few practical layers.

### 1) Dashboards (what teams continuously watch)

| UI area | What to place | Why it matters |
| --- | --- | --- |
| User Experience dashboard | Apdex, LCP, INP, CLS, page load p50/p95, bounce by page | Shows if users feel the app is fast and stable |
| JS Error dashboard | Error rate per release, top stack traces, impacted users/sessions | Quickly identifies broken frontend deployments |
| Geography/Device dashboard | Slow pages by region, browser, device class, OS | Helps separate code issues from environment/network issues |
| Conversion journey dashboard | Drop-off by step, action duration by funnel stage | Ties technical signals to business outcomes |
| Frontend-to-backend dashboard | Frontend action -> backend service latency/error trend | Speeds up ownership routing during incidents |

### 2) Alerts (what should wake people up)

Start with low-noise, symptom-based conditions:

- **UX degradation:** Apdex below threshold for N minutes on key apps.
- **Latency regression:** p95 page load or key action duration above baseline.
- **Error spike:** JS error rate increases x% compared to the previous baseline window.
- **Availability symptom:** sudden rise in failed user actions (checkout/search/login).
- **Region-specific incident:** one geography/browser pair suddenly degrades.

Use burn-rate style alerting for SLO-backed journeys if possible (for example: checkout action success rate).

### 3) Triage workflow in Dynatrace UI

1. Open the affected application and time window.
2. Confirm symptom panel first (Apdex, error rate, impacted users).
3. Drill into user actions/pages with the highest impact.
4. Pivot to distributed traces linked via Dynatrace context headers.
5. Split issue ownership: frontend code, backend dependency, or infra/network.
6. Validate recovery by watching the same dashboards after rollout.

### 4) Recommended baseline dashboard pack for your site

- **Executive health panel:** Apdex, error rate, impacted sessions.
- **Top 10 slow actions/pages:** p95 duration and trend.
- **Top 10 JS exceptions:** count, first seen, last seen, release version.
- **Backend correlation panel:** action latency vs upstream service latency.
- **Release comparison panel:** before/after deploy deltas for UX and errors.

This gives both engineering and product teams a shared, objective view of frontend health.

---

## Final takeaway

RUM is the missing observability layer for browser-side truth.
Dynatrace makes it useful by connecting frontend telemetry to backend traces and operational dashboards.

If browser-side telemetry is missing, incident response starts late.
