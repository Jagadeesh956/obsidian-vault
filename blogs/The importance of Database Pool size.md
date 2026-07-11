---
title: "The Importance of Database Pool Size: A Midnight Page I Won't Forget"
date: 2026-07-02
tags:
  - SRE
  - Database
  - Kubernetes
  - Reliability
  - PostgreSQL
summary: "A real production incident where 900 connections from a single service quietly starved a shared database cluster — and what it taught me about connection pools."
---

## The page that started it all

July 1st, 2026. Midnight.

A page fires for failures on a backend service supporting the "disputes management" customer journey. The symptom at the front-facing edge: `504 timeouts` on a Java function running on our internal FaaS platform.

Within minutes, multiple teams are pulled onto the bridge — cloud operations, database operations, and the backend service owners. Everyone starts looking at their own layer, and for a while, nothing lines up.

---

## Finding the smoking gun

After digging through logs for a while, one line stands out:

> `Connection is not available, request timed out after x seconds`

There it is. The database is the culprit — or at least, that's where the trail leads.

The database operations team dug further and traced it to a service causing heavy blocking sessions and long-running queries, with a cascading effect on the backend service that actually sat in the customer journey.

---

## What we found

1. **One service was quietly hoarding connections.** Each pod from a specific service was holding 50+ active connections, across 3 pods spread over 6 availability zones — roughly **900 connections opened by a single application** against a shared database cluster with a hard cap of **1,000 connections**.
2. **Those connections weren't idle.** Database operations identified many long-running, blocking queries coming from that same service.
3. **The service that customers actually felt the impact through was innocent.** It was running fine with just 1-2 open connections — it simply couldn't acquire a *new* one, because the pool was effectively exhausted.

---

## The real root cause

Three days before the page, changes had gone out to production for two separate services. One of those services runs a heavy monthly batch workload on the 1st of every month.

Here's the twist: the service that actually broke — the one throwing `Connection is not available` — hadn't been touched at all in terms of data processing logic. The real change was in the *other* service: it had been updated to process files from S3 faster, with a shorter time gap between runs and a higher file count per run.

That change didn't fail loudly on its own. It just quietly consumed more and more of the shared connection pool every run, until there was nothing left for anyone else — including a completely unrelated service sitting in a critical customer journey.

It took **4-5 hours** end to end: understanding the symptom, tracing it back through the DB layer, identifying the actual offending service, pulling in its owners, and getting the change reverted.

---

## What I learned from this

### 1) A database connection is a shared, finite resource — treat it like one

It's easy to think of your own service's connection pool as your own problem. It isn't. On a shared cluster, every pod acquiring connections is drawing from the same well as everyone else.

### 2) "Unrelated" services can take each other down

The service that broke and the service that caused the break had nothing to do with each other functionally. The only thing connecting them was the database cluster they both depended on. Blast radius doesn't respect your service boundaries.

### 3) Symptom location and root cause location are often different

The alerts, the timeouts, and the customer impact all showed up on a service that had zero relevant changes. The actual change was three days old and living in a completely different service. If we'd only looked at "what changed in the failing service," we'd have found nothing.

### 4) Connection limits need to be enforced, not assumed

Nothing stopped one application from opening 900 out of 1,000 available connections on a shared cluster. Pool sizing per application, and alerting on cluster-wide connection saturation, should exist *before* this happens — not get added afterward.

### 5) Batch/bulk workloads deserve extra scrutiny before shipping

The change that caused this wasn't reckless on its face — faster S3 processing, more files per run, shorter intervals. But on a monthly batch job hitting a shared resource, "faster and more" translates directly into "more sustained pressure on the pool." That's exactly the kind of change that needs a connection-usage review before it ships.

---

## What I'd do differently going forward

- **Cap per-application connection pools explicitly**, sized against the cluster's real ceiling, not against what "feels safe."
- **Alert on cluster-wide connection utilization**, not just per-service health — a service can look perfectly healthy while starving everyone else.
- **Treat shared database clusters as a blast-radius boundary** during change review, the same way we'd treat a shared network or shared cache.
- **Flag batch/bulk-processing changes for extra review** whenever they touch shared infrastructure, even if the change looks purely like a performance improvement.

---

## Final takeaway

> The service that failed wasn't the service that was wrong.

That's the part of this incident that stuck with me. On shared infrastructure, the blast radius of a bad change rarely stays inside the service that made it — and a database connection pool is one of the easiest places for that to happen quietly, until it isn't quiet anymore.
