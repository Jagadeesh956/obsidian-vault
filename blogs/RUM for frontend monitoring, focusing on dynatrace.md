---
title: "RUM for Frontend Monitoring (Dynatrace Focus)"
date: 2026-08-07
tags:
  - SRE
  - Frontend Monitoring
  - RUM
  - Dynatrace
  - Observability
summary: "Beginner-friendly guide to why frontend monitoring is hard, why plain logs are not enough, and how Dynatrace RUM captures real user experience end-to-end."
---

## Why I wrote this

I mostly worked on backend and platform systems.
Frontend monitoring looked simple at first, but it is actually a different world.

This note is for people like me who did not start as frontend engineers.

---

## 1) What is the real problem in frontend monitoring?

### Short answer

Your frontend code runs in **user browsers**, not on your server.

Your frontend server usually does only this:

- host static files (HTML/CSS/JS)
- send these files when browser requests them

After that, execution happens on user device + user network + user browser runtime.

So, if user says "page is slow", your backend logs alone may look perfectly fine.

### Why this is difficult

Because user experience depends on many things outside backend:

- browser CPU/memory on user machine
- slow or unstable mobile network
- blocked third-party scripts
- JavaScript runtime errors in browser
- DOM rendering delays
- long tasks freezing UI thread

### Basic interaction flow (no RUM yet)

```text
User Browser
   |  GET /index.html, app.js, styles.css
   v
Frontend server / CDN (only serves artifacts)

[Then execution moves to browser]
Browser parses HTML -> downloads JS -> executes JS -> renders UI
Browser calls backend APIs as needed
```

Important: once JS is running in browser, server has limited visibility unless you explicitly collect telemetry from browser.

---

## 2) Why plain log statements are not enough

### If we only write frontend logs

Example in browser code:

```javascript
console.log("button clicked");
console.error("checkout failed", err);
```

Problem:

- these logs stay inside browser devtools
- ops team cannot see them centrally
- users do not send screenshots/logs for every issue
- no automatic correlation to backend traces

### "Can we just send logs ourselves?"

Yes, you can build custom shipping:

```javascript
window.addEventListener("error", (e) => {
  fetch("/frontend-log-collector", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      message: e.message,
      url: location.href,
      userAgent: navigator.userAgent,
      ts: Date.now(),
    }),
    keepalive: true,
  });
});
```

But this quickly becomes hard in production:

- retry/drop/duplication handling
- privacy filtering (PII)
- volume control and sampling
- schema/version consistency
- dashboards and SLOs on top
- linking browser event <-> backend transaction

This is exactly why RUM tools exist.

---

## 3) How observability tools solve this (RUM model)

RUM = Real User Monitoring.

Instead of only backend metrics, RUM captures real browser-side behavior for actual users.

Typical captured signals:

- page load timings
- resource load timings (JS/CSS/images)
- user actions (clicks/navigation)
- JS errors and unhandled promise rejections
- API/XHR/fetch timings from browser
- geographic/device/browser dimensions
- frontend-to-backend correlation IDs

### Generic RUM flow

```text
(1) Browser loads app + RUM agent script
(2) RUM agent observes browser events, timings, JS errors
(3) Agent sends telemetry beacons to observability backend
(4) Backend aggregates, correlates, visualizes
(5) SRE/dev teams troubleshoot from one place
```

---

## 4) What Dynatrace does specifically

Dynatrace RUM provides the browser agent + telemetry pipeline + backend correlation + dashboards.

High-level:

1. Inject Dynatrace RUM JS agent into frontend pages
2. Agent auto-captures user sessions, performance, JS errors, XHR/fetch timings
3. Data is sent to Dynatrace ingest/beacon endpoint
4. Dynatrace links browser events with backend services/traces (when supported)
5. You get user-impact-first troubleshooting

---

## 5) Dynatrace implementation model (practical)

## 5.1 Add Dynatrace RUM agent script

In `index.html` (or via server-side injection):

```html
<!-- Example placeholder script URL - use the exact snippet from your Dynatrace tenant -->
<script
  src="https://js-cdn.dynatrace.com/jstag/<tenant-generated-id>/<agent-file>.js"
  crossorigin="anonymous"
></script>
```

> In real setup, do not guess this URL. Copy from Dynatrace UI: Application -> RUM -> setup/injection.

## 5.2 Optional: add manual business action markers

Auto-capture is good, but manual events improve context:

```javascript
// API names can vary slightly by Dynatrace RUM agent version.
// Use your tenant's JS API docs for exact method signatures.
function onCheckoutStart() {
  if (window.dtrum && window.dtrum.enterAction) {
    const actionId = window.dtrum.enterAction("checkout_start");

    // your app logic
    performCheckout()
      .then(() => {
        window.dtrum.leaveAction && window.dtrum.leaveAction(actionId);
      })
      .catch((err) => {
        window.dtrum.reportError && window.dtrum.reportError(err);
        window.dtrum.leaveAction && window.dtrum.leaveAction(actionId);
      });
  }
}
```

## 5.3 Capture global JS errors (if not already auto-captured)

```javascript
window.addEventListener("error", (e) => {
  if (window.dtrum && window.dtrum.reportError) {
    window.dtrum.reportError(new Error(e.message));
  }
});
```

---

## 6) End-to-end interaction diagrams

## 6.1 Initial app load + RUM bootstrap

```text
Browser -> CDN/Frontend server : GET index.html
Browser <- CDN/Frontend server : HTML + JS bundle + Dynatrace RUM script
Browser : execute app + initialize RUM agent
```

## 6.2 User action + API call + RUM beacon

```text
User clicks "Checkout"
Browser app -> Backend API : POST /checkout
Backend API -> Browser app : 200 / 4xx / 5xx
Browser RUM agent -> Dynatrace beacon endpoint : action timing + error + resource metrics
Dynatrace : links user action with backend service traces (if available)
```

## 6.3 JS error path

```text
Browser JS runtime throws error
RUM agent captures stack + URL + browser metadata
RUM agent sends beacon to Dynatrace
Dashboard shows impacted users/pages/browsers
```

---

## 7) What to expect after enabling Dynatrace RUM

You will be able to answer questions like:

- Which pages are slow for real users?
- Is slowness from frontend render or backend API?
- Which browser/version has most JS errors?
- Which release increased user frustration/drop-off?
- Which geography or device type is impacted?

This is much better than "backend is healthy, so everything is fine".

---

## 8) Practical rollout checklist (for beginners)

1. Start in non-prod with one app page.
2. Verify script loads and sends data.
3. Check privacy masking settings (PII, input fields, tokens).
4. Confirm sampling/retention settings.
5. Add 2-3 custom business actions (login, checkout, search).
6. Build baseline dashboard before release.
7. Add alerts on user-impact metrics (not only server CPU).

---

## 9) Common mistakes to avoid

- assuming backend logs can explain frontend UX issues
- enabling RUM without privacy review
- no custom actions for key user journeys
- collecting too much low-value telemetry (cost/noise)
- not correlating frontend and backend in incident process

---

## 10) Final takeaway

Frontend monitoring is hard because execution moved from your server to user browser.

RUM (especially with Dynatrace) solves this by giving real-user visibility, not synthetic guesses.

For SREs, this is the mindset shift:

> "Service health" is not complete until we measure user experience at the browser edge.
