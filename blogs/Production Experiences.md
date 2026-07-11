---
title: "Production Learnings: What a 4-Hour Incident Taught Me"
date: 2026-06-22
tags:
  - SRE
  - PostgreSQL
  - JVM
summary: A real incident from work and the practical things I learned from it.
---

## Why this one stayed with me

I have seen production issues before, but this one really changed how I think.

The app started failing with this error:

> `cannot UPSERT on a read-only DB`

At first, it looked like a normal DB issue. But fixing it and understanding it took almost 4 hours.

---

## What happened

- A Java app was trying to write to a PostgreSQL cluster.
- Somehow writes were going to non-primary nodes.
- Errors increased very fast.
- In about 15 minutes, alerts crossed 10K failures.

A big bridge call started. Many teams joined:

- App support
- DB team
- Load balancer team
- Infra team
- Downstream teams

Even with many people, we still took time to get to the real reason.

---

## What I learned from this incident

### 1) "DB looks healthy" does not mean users are fine

One team saw the DB as healthy, but app errors were still going up.

Big learning for me: always check the full request path, not one layer.

### 2) DNS + old connections can keep failures going

Even after LB health came back, JVM processes kept using old TCP connections.

So traffic still hit wrong nodes for some time.

That is why failures continued longer than we expected.

### 3) Different check intervals can cause short bad windows

Patroni checks and LB checks were running on different timing.

That timing difference created small windows where routing was unstable.

Now I always check interval alignment in reviews.

### 4) Fast fixes must be written in runbooks

A controlled restart of impacted app instances helped quickly.

We found this action too late in the incident.

Now I make sure runbooks include:

- quick safe actions to reduce impact
- and separate steps for deeper root cause work

### 5) MTTR is also about people and process

The issue was technical, yes. But the long duration was also because of:

- unclear ownership at some points
- repeated questions across teams
- slow decision on mitigation

So this was not only a systems lesson. It was also a teamwork lesson.

---

## What I changed in my daily work

After this incident, I started doing these more strictly:

1. Track LB + ingress + app together, not app only.
2. Define bridge roles early (incident lead, technical driver, updates owner).
3. Keep a list of known safe mitigations.
4. Call out control-plane timing mismatch early.
5. Write post-incident notes focused on learning, not blame.

---

## Final thought

For me, the biggest lesson is simple:

> Production reliability is not just about code. It is about how systems behave under stress and how teams respond in real time.

This incident made me sharper as an engineer and much calmer during long bridge calls.
