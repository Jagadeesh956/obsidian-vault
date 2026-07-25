---
title: "The Blind Spots of a Client-Server Architecture: What I Learned at Work"
date: 2026-02-22
tags:
  - SRE
  - Observability
  - Microservices
  - Reliability
  - Kubernetes
summary: Simple lessons from real production incidents where client errors did not match app logs.
---

## Why I started doubting our dashboards

In production, I kept seeing this pattern:

- Client side showed `503 Service Unavailable`
- App team said: "we do not return 503"

At first, I thought it was a logging bug.

Later I understood the real issue: we were only looking at one part of the path.

---

## The real path is longer than we think

In my projects, request flow is usually:

**Client -> Load Balancer -> Ingress Gateway -> Sidecar Proxy -> App Container**

If we only watch app logs, we miss failures in LB, ingress, and sidecar.

That was my blind spot in the beginning.

---

## What I learned from real incidents

### 1) App logs alone are not enough

You can have clean app logs and still have user-facing failures.

I have seen this many times:

- request blocked before controller logging starts
- ingress timeout
- sidecar reset/circuit break

### 2) "No app errors" does not mean service is healthy

Some dashboards look great because they only track app-level success.

But users still fail on the edge path.

Now I treat user-side success rate as the main signal.

### 3) Missing trace IDs slows incident response

When IDs are not passed from edge to app, teams waste time arguing where issue is.

I learned this the hard way on bridge calls.

### 4) Reliability numbers can look better than reality

If we ignore LB/Ingress/mesh failures, availability % looks higher than actual user experience.

Now I question any metric that is not end-to-end.

---

## What I changed after these incidents

### Better observability across layers

Now I push for metrics in all layers:

- **LB:** target health changes, backend 5xx
- **Ingress:** status by path/host, retries, upstream latency
- **Sidecar:** reset counts, circuit break events
- **App:** early filter metrics + structured logs

### Better incident handling

In incidents, I now ask only 3 things first:

1. Where exactly did failure start?
2. What evidence do we have right now?
3. What is the fastest safe action to reduce user impact?

### Better reporting

There should be a better way to document the metrics .

---

## My simple checklist now

- Start metrics before controller methods.
- Pass one correlation ID across all hops.
- Alert on full-path error budget burn.
- Run synthetic checks from client side.
- Include upstream dependency behavior in every RCA.

---

## Final takeaway

This was a big mindset change for me:

> If we start observing too late in the request path, we also start debugging too late.

Reliability is a full-path job, not just an application-team job.
