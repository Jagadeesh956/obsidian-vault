---
title: "Go Routine Retry Misuse: A Production Incident That Hit Customers"
date: 2026-07-25
tags:
  - SRE
  - Go
  - Kubernetes
  - Incident Management
  - Production Learnings
summary: "A real incident where a retry bug in a Go SDK caused heavy load, affected shared infra, and impacted customer-facing traffic."
---

## Why I am writing this

This is one of the most stressful incidents I have seen.

It looked like a Safekey issue first. Then an Istio issue. Then cloud issue.
In the end, the trigger was a retry bug in a Go SDK path for SSE.

Big lesson: in distributed systems, the first symptom is often far away from the real source.

---

## Quick system context

We have an internal platform called ECM (Enterprise Configuration Management).

- Services use ECM to pull config updates.
- ECM provides SDKs for multiple languages (Go, Java, Python, Node.js).
- A new SSE feature was added so clients can receive config updates continuously.

The Go SDK change had logic that could keep creating clients/routines forever on failures.

---

## What happened (timeline style)

### T

A major bridge started for Safekey (OTP sender) due to intermittent customer impact.

Safekey team had no recent deployment, so debugging started with dependent teams.

### T + 2 hours

Cloud operations found Istio gateway pods showing liveness failures, pattern matching Safekey impact windows.

Since gateway is shared infra, it was not obvious which service was causing pressure.

Many teams were involved including leaders , vendors for service mesh , OCP etc to identify the issue . 

Teams tried to observe a pattern or isolate it to a specific zone if any infra layer is the culprit . 

### T + 10 hours

Cloud SRE noticed ECM pods in some zones were also crashlooping in the same pattern.

One engineer scaled ECM pods in a problem zone to 0.

**Safekey failures stopped immediately.**

That was the first strong correlation.

### T + 15 hours

Our ECM support team got paged around midnight.

At first, we pushed back because direct correlation was not obvious and SSE traffic visibility in ECM logs was weak.

We asked cloud ops to isolate traffic by zone so we could narrow it better.

### T + 16 hours

I reviewed logs in OpenSearch and noticed unusually high SSE-related failures.

We identified the only consumer using the new SSE feature: Loyalty team.

They confirmed they had enabled SSE in production the previous day.

### T + 18 hours

Loyalty logs showed repeated client creation messages and repeated failures when calling ECM.

Now root cause shifted back to SDK behavior provided by ECM.

Eventually, Loyalty reverted the SSE enablement change.

Incident stabilized.

---

## Root cause (simple version)

In Go SDK SSE flow:

- on connection failure, retry logic kept creating routines/clients repeatedly
- retries were not controlled properly
- this created heavy repeated pressure
- shared infra (Istio gateway and related components) became unstable
- customer-facing application (Safekey) saw impact as a downstream effect

So customer pain showed up in one platform, but source load came from another path.

---

## Why this took so long

This is what slowed us down:

1. Shared infrastructure made blame direction noisy.
2. Early evidence was symptom-based, not source-based.
3. SSE observability was weaker than normal REST path visibility.
4. Multiple teams were debugging in parallel with partial context.
5. Rollback in consumer domain took organizational time.

---

## What we changed after incident

### Engineering fixes

- Go SDK retry logic was fixed.
- Retries were moved to exponential backoff pattern.
- Fallback to regular REST polling was added when SSE repeatedly fails.

### Validation improvements

I personally tested the fix with failure simulation:

- many parallel clients through Docker Compose
- forced pod failures/disconnections
- reconnection behavior checks
- stability checks before sign-off

### Process improvements

- Better correlation checks for "unrelated" platform events
- More focus on shared infra blast-radius thinking
- Better logging requirements for SSE path
- Stronger incident ownership transitions

---

## My personal learnings from this incident

1. Retry logic is production-critical code. Bad retry behavior can break healthy systems.
2. Shared platform incidents need correlation-first debugging, not team-first debugging.
3. "No direct dependency" does not mean "no impact path" in distributed systems.
4. Observability gaps in new features (like SSE) can heavily increase MTTR.
5. Fast rollback options in consuming teams are as important as fixes in provider teams.

---

## Final takeaway

This incident changed how I review resilience features.

Now whenever I see reconnection/retry logic, I ask first:

- What is the worst-case retry behavior?
- How do we cap it?
- What is the fallback?
- How will we observe it in production?

Because one retry bug in one SDK can become a customer incident somewhere else.
