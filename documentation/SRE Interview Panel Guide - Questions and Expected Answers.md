---
title: "SRE Interview Panel Guide - Questions and Expected Answers"
date: 2026-07-05
tags: [SRE, interview, hiring, linux, kubernetes, devops, cloud]
summary: "Consolidated interview report for hiring a Site Reliability Engineer: panel expectations, technical and behavioral questions, and what strong answers look like."
---

# SRE Interview Panel Guide - Questions and Expected Answers

## 1) Hiring context and what "good fit" means

This role is not just operations support. It is **software-driven operations at scale**.

A strong candidate should show:

- Solid Linux and networking fundamentals
- Python automation ability (not just scripts, but maintainable engineering)
- Clear understanding of cloud and Kubernetes architecture
- Incident response maturity under pressure
- Metrics-first, reliability-first thinking
- Ability to learn new open source systems quickly (OpenStack, Kafka, OpenSearch, databases, etc.)

---

## 2) Core competencies to evaluate

## A) Linux and systems fundamentals

Expect candidate to be confident with:

- processes, scheduling, memory, filesystems
- systemd, logs, troubleshooting flow
- networking tools (`ss`, `ip`, `tcpdump`, `dig`, `traceroute`)
- kernel/user-space boundaries (at least practical understanding)

## B) Python and automation engineering

Expect candidate to:

- write clean, testable automation
- use idempotent workflows
- handle retries/timeouts/backoff safely
- think in terms of maintainability and observability

## C) Networking and distributed systems basics

Expect candidate to understand:

- DNS, TCP, TLS, load balancing, NAT
- L4 vs L7 behavior
- service discovery and failure modes

## D) Cloud and Kubernetes operations

Expect candidate to explain:

- pod/service/ingress/network policy/storage basics
- cluster-level debugging
- deployment safety patterns (canary, rollback, health checks)

## E) Incident management and reliability mindset

Expect candidate to:

- lead with triage, impact, containment, communication
- reason from signals/metrics, not guesses
- define post-incident prevention actions

## F) Communication and ownership

Expect candidate to:

- communicate clearly with dev, platform, and management
- write high-quality runbooks and incident notes
- own outcomes end-to-end

---

## 3) Suggested interview structure (panel flow)

1. **Linux + networking round** (45 min)
2. **Coding/automation round (Python)** (60 min)
3. **Kubernetes + cloud systems round** (45 min)
4. **Incident simulation / troubleshooting round** (45 min)
5. **Behavioral + ownership round** (30-45 min)

---

## 4) Question bank with expected answers

## 4.1 Linux / Systems Round

### Q1: A service is "up" according to systemd, but users see timeouts. How do you debug?

**Strong answer should include:**
- define impact scope first (all users? one region? one endpoint?)
- check process health + actual readiness (not only process existence)
- inspect socket/listen state (`ss -ltnp`)
- check resource pressure (CPU steal, memory, IO wait)
- inspect logs and recent deploy/config changes
- verify upstream dependency health
- confirm network path and LB behavior

### Q2: Explain the difference between CPU load and CPU utilization.

**Strong answer:**
- utilization = percentage of CPU busy
- load average = runnable + uninterruptible tasks waiting for CPU/IO
- high load with low CPU can indicate IO bottlenecks

### Q3: What does OOM killer do and how do you investigate OOM events?

**Strong answer:**
- kernel kills process under memory pressure
- use `dmesg` / kernel logs for OOM traces
- inspect process memory growth patterns
- check cgroup/container memory limits
- apply fixes: limits tuning, leak fixes, workload shaping

---

## 4.2 Python / Automation Round

### Q4: Write automation to restart unhealthy services across many nodes safely.

**Expectations:**
- idempotent logic
- health check gate before/after action
- concurrency control / batching
- retries with exponential backoff
- timeout and rollback strategy
- structured logging and metrics

### Q5: How do you make automation production-safe?

**Strong answer should mention:**
- dry-run mode
- staged rollout
- guardrails and blast-radius control
- versioned configs
- tests (unit + integration)
- audit logs and alerting

### Q6: How would you design a Python tool to collect diagnostics from 500 nodes?

**Strong answer:**
- async/concurrent execution with rate limit
- deadline/timeout per node
- partial-failure tolerant aggregation
- clear output schema (JSON)
- retries only for transient failures
- observability of the collector itself

---

## 4.3 Networking Round

### Q7: App returns intermittent 503. Where can it originate besides app code?

**Strong answer:**
- load balancer target health failure
- ingress/controller upstream timeout
- service mesh proxy resets/circuit breaker
- DNS resolution instability
- backend connection pool exhaustion

Candidate should avoid "503 always app problem" thinking.

### Q8: Explain DNS TTL and how stale DNS can impact reliability.

**Strong answer:**
- resolvers cache based on TTL
- long TTL can delay failover visibility
- app/runtime DNS caching can keep stale endpoints
- should combine DNS strategy with connection lifecycle strategy

### Q9: L4 LB vs L7 LB - practical difference for SRE debugging?

**Strong answer:**
- L4: transport-level, no HTTP route awareness
- L7: HTTP-aware, path/host routing, richer telemetry
- debugging differs in where failures are visible and how health checks behave

---

## 4.4 Kubernetes / Cloud Round

### Q10: Pod is Running but endpoint is failing. What checks do you do?

**Strong answer should include:**
- readiness/liveness probes status
- service selector/endpoints correctness
- container logs and recent restarts
- resource throttling (CPU/memory)
- network policy blocks
- DNS/service discovery in cluster
- ingress and service mesh routing

### Q11: How would you reduce deployment risk in Kubernetes?

**Strong answer:**
- progressive rollout (canary/blue-green)
- SLO/SLI guardrails during rollout
- auto rollback on error budget burn
- pre-deploy checks and smoke tests
- immutable image and config versioning

### Q12: What cloud concepts are essential for SRE?

**Expected:**
- VPC/network segmentation
- IAM least privilege
- autoscaling behavior and limits
- managed LB/DB failure modes
- multi-AZ/region tradeoffs
- cost/performance/reliability balance

---

## 4.5 Incident Simulation Round

### Q13: Simulated incident: elevated latency, 5xx spikes, no clear single alert. What now?

**Strong answer process:**
1. establish incident severity and user impact
2. define timeline and assign roles (commander/driver/comms)
3. check golden signals (latency, traffic, errors, saturation)
4. isolate failing dependency/path quickly
5. apply low-risk mitigation first (rate limit, rollback, scale, failover)
6. communicate regularly with clear updates
7. collect evidence for post-incident review

### Q14: What makes a strong postmortem?

**Strong answer:**
- factual timeline
- clear root cause + contributing factors
- detection/response gaps
- concrete action items with owners and due dates
- follow-up verification, not just ticket creation

---

## 4.6 Behavioral / Ownership Round

### Q15: Tell us about a production incident you owned.

**What to listen for:**
- candidate explains context and impact clearly
- shows prioritization under pressure
- distinguishes symptom vs root cause
- speaks about prevention and systemic fixes
- takes responsibility without blame language

### Q16: How do you balance reliability vs feature velocity?

**Strong answer:**
- references SLO/error-budget thinking
- not "always reliability first" or "always ship fast"
- shows data-based decision making

### Q17: You disagree with a dev team’s rollout decision. What do you do?

**Strong answer:**
- brings data and risk framing
- proposes safe alternative rollout plan
- escalates respectfully if needed
- focuses on shared outcome, not politics

---

## 5) Scoring rubric (recommended)

Score each area 1-5:

- Linux/systems
- Python/automation
- Networking
- Kubernetes/cloud
- Incident handling
- Communication/ownership

**Hire signal:**
- mostly 4s and 5s
- no major 1-2 in Linux or incident response for this role

**Strong hire indicators:**
- thinks in systems, not tools only
- explains tradeoffs clearly
- can automate safely at scale
- demonstrates calm, structured incident behavior

**No-hire indicators:**
- shallow fundamentals hidden by buzzwords
- no structured debugging process
- unsafe automation mindset (no guardrails)
- blames other teams without accountability

---

## 6) Final expectation from a strong candidate

A strong SRE candidate for this role should sound like:

> "I understand how systems fail across layers, I use code and metrics to operate them safely at scale, and I can lead incidents with clarity and ownership."

That is the profile that will fit model-driven, automation-first infrastructure operations across cloud and on-prem environments.
