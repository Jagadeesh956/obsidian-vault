---
title: "SRE to Java Backend Engineer — Transition Plan"
date: 2026-08-09
tags:
  - career
  - java
  - backend
  - transition-plan
summary: "Honest roadmap from Amex SRE to hireable backend engineer: what to build, how to reframe real experience truthfully, and what the resume should say."
---

## The starting reality (say this to yourself once, then move on)

- You have genuine, deep Java/Spring/JVM internals knowledge — verified in this exact conversation, not claimed.
- You have real distributed systems production exposure: Cassandra, Spark, Kafka, Postgres, Kubernetes, at scale, across multiple platforms.
- You have led cross-team incident response and root-caused issues other teams missed.
- You do NOT have: a shipped, end-to-end backend service; a public portfolio; system design interview reps; testing discipline evidence.
- Your GPA (6.6/10, 3.35/4) will be filtered on by ATS/HR screens at some companies. Nothing fixes this except making it irrelevant — a strong GitHub + a strong project talks louder than a transcript at every company worth targeting.

**Do not fabricate experience.** Reframe truthfully, then close the real gap by building. Fabrication doesn't survive a live system-design round, a pair-programming round, or a background check — and this profile doesn't need it.

---

## What to build, starting today (12-week plan)

Pick ONE project. Do not scatter across five toy apps. Depth on one real thing beats five tutorials.

### Recommended project: "Incident Intelligence Platform" (mirrors what you actually know)

Build a real backend service that ingests incident/alert data, correlates it, and surfaces root-cause suggestions — this is literally your Amex domain, so you can build it with real judgment instead of guessing requirements like a bootcamp clone.

**Core scope (weeks 1–4): Ship a real service**
- Spring Boot REST API (not a hello-world — real domain: incidents, services, alerts, on-call ownership)
- Postgres schema with actual relationships (incidents → services → alerts → resolutions), Flyway/Liquibase migrations
- Proper layered architecture: controller → service → repository, DTOs vs entities, validation
- Authentication: Spring Security + JWT (or OAuth2) — this is a near-universal JD requirement you currently have zero evidence of
- Pagination, filtering, proper error handling (`@ControllerAdvice`, structured error responses)

**Testing (weeks 4–5): close your biggest silent gap**
- JUnit 5 + Mockito unit tests for service layer
- Testcontainers for real Postgres integration tests (this specifically signals "production-minded," not just "can write code")
- Aim for meaningful coverage on business logic, not vanity 100%

**Production-mindedness layer (weeks 5–7): this is your unfair advantage — use it**
- Add the observability work you already know how to do for real: structured logging, distributed tracing (OpenTelemetry), custom metrics (Micrometer)
- Add a `/health`, `/metrics` endpoint, basic SLO thinking documented in the README (define an SLO for your own API, show you understand error budgets)
- Add resilience patterns: circuit breaker (Resilience4j), retry with backoff, explicit timeout configuration — tie this back to everything we discussed about server-side timeouts and DB query cancellation earlier in this conversation. Document *why* you made each choice.

**Deployment (weeks 7–9): use your existing K8s strength**
- Containerize it, deploy to a real Kubernetes cluster (kind/minikube is fine) with a proper Helm chart or raw manifests
- Basic CI/CD: GitHub Actions pipeline — build, test, containerize, deploy
- This is where your CKAD/CKA knowledge becomes a genuine differentiator instead of a separate bullet point — most backend candidates can't do this part at all

**Polish + write-up (weeks 9–12)**
- Clean README: architecture diagram, decisions and trade-offs explained, "what I'd do differently at scale"
- A short blog post (reuse your existing blog-writing habit from the vault) walking through one hard decision you made — e.g., "why I added a circuit breaker here" or "how I designed the incident correlation logic"
- Push all of it to GitHub, public, pinned on your profile

By week 12 you have: a real service, real tests, real deployment, real docs, and a defensible story for every line of your resume.

---

## Resume — what it should actually contain

### Framing principle
Every bullet should answer: *what did I build/change, what was the technical mechanism, what was the measurable outcome.* Cut anything that's just "monitored X" or "supported Y" with no technical verb.

### Header positioning
Don't title yourself "Site Reliability Engineer" if you're applying to backend roles. Use: **"Backend Engineer | Java, Spring, Distributed Systems"** — and let the SRE title live inside the experience bullets, not the headline. Recruiters and ATS filters weight the title line heavily.

### Reframing real Amex experience (honest examples)

Weak/ops-sounding (avoid):
> "Handled production incidents and monitored Java applications across platforms."

Reframed truthfully, developer-language (use):
> "Diagnosed and root-caused defects in production Java services processing billions of daily events across 3–4 platforms; read and reasoned about unfamiliar codebases under time pressure to isolate concurrency, connection-pool, and transaction-boundary bugs."

Weak:
> "Improved observability of Java apps."

Reframed:
> "Designed and implemented distributed tracing and structured log tagging across Java/Spring services, reducing mean time to root-cause for cross-service incidents; advised engineering teams on instrumentation standards adopted across multiple platforms."

Weak:
> "Wrote automation scripts to reduce manual work."

Reframed:
> "Built Ansible-based validation tooling for Spark/Cassandra clusters and a multi-cluster Kubernetes automation CLI (bash-wrapped kubectl), eliminating recurring manual verification work across [N] clusters."

Weak:
> "Led production calls with multiple teams."

Reframed:
> "Led cross-functional incident response across Database, Network, and Cloud Operations teams for high-severity production issues; frequently identified root causes ahead of the owning team, informing postmortem and remediation decisions."

### New section: Projects (this carries the most weight for a career-shift resume)
```
Incident Intelligence Platform — Personal Project [GitHub link]
Java 21, Spring Boot, Postgres, Kubernetes, Resilience4j, OpenTelemetry

- Designed and built a backend service for incident correlation and root-cause
  suggestion, modeled on real production observability patterns from SRE work
- Implemented JWT-based auth, layered architecture, and Testcontainers-backed
  integration tests covering core business logic
- Added circuit breakers, configurable timeouts, and DB query cancellation to
  prevent cascading failures under load; documented trade-offs in design notes
- Deployed via Helm to Kubernetes with a GitHub Actions CI/CD pipeline;
  instrumented with distributed tracing and custom metrics
```

### Skills section — group by what backend interviewers actually screen for
```
Languages: Java (Core, Concurrency, JVM internals), Python, SQL
Frameworks: Spring Boot, Spring Security, Spring Data JPA
Data: PostgreSQL, Cassandra, schema design, Flyway migrations
Messaging/Streaming: Kafka
Testing: JUnit 5, Mockito, Testcontainers
Infra/Platform: Kubernetes (CKAD; CKA in progress), Docker, Helm, Terraform (learning)
CI/CD: GitHub Actions
Observability: OpenTelemetry, distributed tracing, Micrometer, log correlation
```
This groups your genuine SRE strengths (data, infra, observability) alongside real developer signals (Spring, testing) so the SRE background reads as an asset, not something to hide.

### Education section
List it factually, don't hide it, don't apologize for it in the resume itself:
```
M.S., Cleveland State University — Dec 2022
B.Tech, Lovely Professional University — 2021
```
No GPA line is required on an Indian tech resume unless specifically asked — omit it rather than draw attention to it. Your projects and experience carry the weight instead.

### What NOT to put on the resume
- Do not claim a title, project, or ownership you didn't have.
- Do not list "Java Developer" as a past job title if it wasn't — list your real title, and let the bullets do the reframing.
- Do not pad the skills list with tech you've only read about (e.g., don't list Terraform as a skill until you've actually used it in the project above — list it under "currently learning" if you want it visible honestly).

---

## Interview prep priorities, in order

1. **System design (application layer)** — Grokking System Design / ByteByteGo. You already have the infra-layer intuition; you're filling in API design, data modeling, caching, consistency trade-offs.
2. **DSA, moderate depth** — not competitive-programming level, but enough for standard product-company screens (arrays, strings, trees, graphs, DP basics). Your algorithms note in the vault is a fine starting point — revisit it with interview framing.
3. **Spring/Java deep-dive** — you're already ahead here after this conversation. Be ready to explain proxies, transactions, concurrency, GC basics out loud, clearly, in interview format — that's a genuine differentiator once you're in the room.
4. **"Tell me about a production issue you debugged"** — this is your strongest card. Prepare 2–3 real Amex stories in STAR format, translated into developer language using the reframing patterns above.

---

## The 90-day sequence, summarized

- **Weeks 1–9:** Build the project (above), in public, committing regularly (visible GitHub activity matters to screeners).
- **Weeks 9–10:** Rewrite resume using the reframing patterns above; get 2–3 people to review it who work in backend roles.
- **Weeks 10–12:** Start applying — target product companies and strong GCCs (Razorpay, PhonePe, Flipkart, Swiggy, Zoho, Freshworks, well-run bank/finance GCCs), not tier-2 IT services firms. Simultaneously start system design + DSA prep on a steady weekly cadence so you're interview-ready by the time responses start coming in.

## One last honest note
This plan is deliberately not fast. It's real. The version of this plan that's fast is the fabricated one, and it fails in the interview room, not on the resume screen — which is a worse place to fail. Twelve weeks of honest building gets you a story that survives a staff engineer grilling you for 45 minutes. That's the actual bar at the companies worth joining.


---

## Functional Requirements — Jira-Style Stories (Incident Intelligence Platform)

Format follows real sprint-ready stories: user story, acceptance criteria, data/implementation notes, and a self-validation checklist so you can grade your own work like a reviewer would.

---

### STORY-1: Service and Incident data model foundation

**As a** platform engineer
**I want** a normalized data model for services, incidents, and alerts
**So that** the system has a real foundation to correlate events against

**Description / how data is ingested:**
No external ingestion yet — this story is schema-first. Define entities: `Service` (id, name, owner_team, tier), `Incident` (id, service_id, severity, status, opened_at, resolved_at, root_cause_summary), `Alert` (id, incident_id, source, raw_payload JSONB, received_at). Use Flyway/Liquibase migrations, not `hibernate.ddl-auto=update`, so schema evolution is explicit and reviewable — this is itself a signal of production maturity.

**Acceptance criteria:**
- Migrations run cleanly from empty DB via `flyway migrate` / equivalent
- Foreign keys and indexes exist on `service_id`, `incident_id`, `status`, `opened_at`
- Entity relationships correctly modeled (one service → many incidents → many alerts)
- Seed script populates 5+ services and 20+ sample incidents for local dev

**Validate yourself:**
- Can you explain, out loud, why you chose JSONB for `raw_payload` instead of fixed columns? (trade-off: flexibility vs. queryability — be ready to defend it)
- Run `EXPLAIN ANALYZE` on a query filtering by `service_id` + `status` — confirm the index is actually used, not a full table scan

---

### STORY-2: Alert ingestion API (external data entry point)

**As an** external monitoring system (e.g., Prometheus Alertmanager, a synthetic script standing in for Dynatrace/Datadog)
**I want** to POST alerts to a REST endpoint
**So that** the platform has a real ingestion path, not just seeded data

**Description / how data is actually ingested:**
`POST /api/v1/alerts` accepts a JSON payload: `{service_name, severity, message, source, timestamp, metadata}`. This simulates what a real RUM/APM tool (tying back to our Dynatrace conversation) would push via webhook. Validate payload shape with Bean Validation (`@Valid`, `@NotNull`, custom severity enum). Unknown `service_name` should either auto-register a placeholder service or reject with 422 — pick one, document why in the README.

**Acceptance criteria:**
- Endpoint returns 201 with the created alert's ID on success
- Returns 400 with a structured error body (not a stack trace) for malformed payloads
- Alerts persist correctly linked to their service
- Load test: endpoint handles 50 concurrent POSTs without data corruption (use a simple JMeter/k6 script)

**Validate yourself:**
- Deliberately send malformed JSON, missing fields, and an unknown service — confirm each fails predictably and safely, not with a 500
- Can you explain what happens to your DB connection pool under the 50-concurrent-request test? Tie this back to the HikariCP timeout discussion from earlier in this vault

---

### STORY-3: Incident auto-creation from correlated alerts

**As a** system
**I want** to automatically open an Incident when N alerts for the same service arrive within a time window
**So that** noisy individual alerts get grouped into an actionable incident, mirroring real correlation engines

**Description:**
Implement a correlation rule: if 3+ alerts for the same `service_id` arrive within 5 minutes, and no open incident exists for that service, create one automatically and link the alerts to it. This is genuinely the "hard part" of the whole project — it's where you demonstrate actual engineering judgment, not CRUD.

**Acceptance criteria:**
- Given 3 alerts within the window, exactly one incident is created and all 3 alerts are linked to it
- A 4th alert within the same open incident's lifetime attaches to the existing incident, doesn't create a duplicate
- Alerts outside the time window, or for a service with no existing open incident and only 1-2 alerts, do not trigger incident creation
- Logic is covered by unit tests with time-based edge cases (alert at exactly the window boundary)

**Validate yourself:**
- Write a test for the boundary condition (alert #3 arrives at exactly 5:00 vs 4:59 vs 5:01) — does your logic behave the way you intended, or did you just eyeball it?
- Explain your choice of window-tracking approach (in-memory sliding window vs. DB query with timestamp range) and its trade-off at scale

---

### STORY-4: Incident status lifecycle and transitions

**As an** on-call engineer
**I want** to transition an incident through OPEN → ACKNOWLEDGED → RESOLVED states with valid rules
**So that** the system enforces real incident management discipline, not free-form status edits

**Description:**
`PATCH /api/v1/incidents/{id}/status` with a target status. Enforce a state machine: OPEN → ACKNOWLEDGED → RESOLVED only (no skipping, no going backward except explicitly allowed RESOLVED → REOPENED). Record `resolved_at` timestamp and require a `root_cause_summary` before allowing RESOLVED.

**Acceptance criteria:**
- Invalid transitions (e.g., OPEN → RESOLVED directly) return 409 Conflict with a clear message
- RESOLVED requires a non-empty `root_cause_summary` in the request body, or the transition is rejected
- Valid transitions update `status` and relevant timestamps correctly
- State transition logic is isolated in its own class/method, unit-testable without spinning up the full API

**Validate yourself:**
- Can you diagram your state machine on a whiteboard in under 2 minutes if asked in an interview?
- Did you use an enum + explicit transition map, or scattered if/else? The former is what a reviewer expects at this level — refactor if it's the latter

---

### STORY-5: Root-cause suggestion engine (rule-based, not ML — be honest about scope)

**As an** on-call engineer
**I want** the system to suggest a likely root cause category based on incident patterns
**So that** I get a head start on investigation, similar to what Dynatrace's Davis AI does at a much simpler level

**Description:**
Do NOT build actual ML here — that's scope creep that will sink the whole project. Build a rule-based suggestion engine: if alerts reference specific keywords (timeout, connection refused, OOM, 5xx rate), map to a category (Network, Database, Memory, Application Error) and attach a `suggested_category` to the incident. This is honest, scoped engineering — document explicitly in the README that this is intentionally rule-based v1, with ML noted as a future direction. That honesty is itself a good interview talking point.

**Acceptance criteria:**
- Given alert text containing known keywords, the correct category is suggested
- Unmatched text results in `category = UNKNOWN`, not a crash or empty string
- Suggestion logic is a separate, swappable component (interface-based), so "swap in ML later" is a real architectural claim, not just a README comment

**Validate yourself:**
- Can you explain why you made this rule-based instead of pretending to do ML with three if-statements? (Answer: honesty about scope + real interfaces beats fake sophistication — this is a legitimate engineering story to tell)

---

### STORY-6: Authentication and authorization

**As a** platform operator
**I want** the API secured with JWT-based auth and role-based access
**So that** only authorized users can modify incidents, while read access can be broader

**Description:**
Implement Spring Security with JWT. Two roles: `VIEWER` (read-only GET access) and `RESPONDER` (can POST alerts, PATCH incident status). Issue tokens via a simple `/auth/login` endpoint backed by a `users` table with hashed passwords (BCrypt).

**Acceptance criteria:**
- Unauthenticated requests to protected endpoints return 401
- `VIEWER` role attempting a PATCH returns 403
- Passwords are never stored or logged in plaintext, verify by grepping logs after a login attempt
- Token expiry is enforced; expired token returns 401 with a clear error, not a generic failure

**Validate yourself:**
- Try to log the JWT secret or a raw password accidentally — did your logging config leak it anywhere? This is a real production mistake worth deliberately testing for
- Explain the difference between authentication and authorization out loud, using your own endpoints as the example (ties back to the earlier vault note on this exact topic)

---

### STORY-7: Resilience — timeouts, retries, circuit breaker on a downstream call

**As a** system
**I want** calls to a simulated downstream dependency (e.g., a "notification service" stub) to fail safely
**So that** the platform doesn't cascade-fail when a dependency is slow or down

**Description:**
Add a downstream call (can be a deliberately-flaky local stub service) — e.g., "notify on-call via Slack" when an incident opens. Wrap it with Resilience4j: circuit breaker, retry with exponential backoff, and an explicit timeout. This directly implements everything covered earlier in this vault about server-side timeouts and DB/downstream call cancellation — don't just copy config, be able to explain each parameter.

**Acceptance criteria:**
- When the stub service is artificially slowed beyond the configured timeout, the call fails fast rather than hanging the request thread
- After N consecutive failures, the circuit breaker opens and subsequent calls fail immediately without hitting the stub (verify via logs/metrics)
- Circuit breaker half-opens and recovers correctly once the stub is healthy again
- Behavior is demonstrated with a test or a recorded terminal session, not just claimed

**Validate yourself:**
- Can you explain, from memory, the difference between what happens to the client-facing request timeout versus what happens to the actual downstream call when the circuit is open? (This is directly the client-vs-server timeout distinction from earlier in this vault — if you can't explain it cleanly here, revisit that note first)

---

### STORY-8: Observability — tracing, metrics, structured logs

**As a** platform engineer
**I want** distributed tracing and custom metrics on every request
**So that** the system is debuggable the way you'd actually expect from your own SRE background

**Description:**
Add OpenTelemetry instrumentation (auto or manual spans) across the alert-ingestion → correlation → incident-creation flow. Add Micrometer custom metrics: `alerts_ingested_total`, `incidents_created_total`, `incident_resolution_duration_seconds`. Structured JSON logging with correlation/trace IDs on every log line.

**Acceptance criteria:**
- A single alert POST produces a trace showing every internal step (ingestion → correlation check → incident creation) as child spans
- `/actuator/prometheus` (or equivalent) exposes the custom metrics correctly
- Logs include a trace ID that matches the span ID from the trace, so a log line can be correlated back to its trace — this is the real-world mechanism, not a toy version
- README includes a short "how I'd use this to debug a real incident" walkthrough

**Validate yourself:**
- Deliberately trigger a failure (bad payload, downstream timeout) and confirm you can find it end-to-end starting from a single trace ID — that's the actual test of whether your observability setup works, not just whether the endpoints exist

---

### STORY-9: Deployment to Kubernetes with CI/CD

**As a** platform engineer
**I want** the service containerized, Helm-charted, and deployed via a CI/CD pipeline
**So that** the project demonstrates real platform ownership, not just local `mvn spring-boot:run`

**Description:**
Dockerfile (multi-stage build, non-root user, minimal base image). Helm chart with configurable replicas, resource requests/limits, readiness/liveness probes tied to your `/actuator/health` endpoint. GitHub Actions pipeline: build → test → build image → push to a registry → deploy to a local kind/minikube cluster (or document the equivalent for a cloud cluster if you have access).

**Acceptance criteria:**
- `docker build` produces a working image under a reasonable size (document the size, explain any optimization choices)
- Helm chart deploys successfully with `helm install`, pods reach Ready state
- Liveness/readiness probes correctly reflect real app health — kill the DB connection and confirm readiness fails appropriately rather than staying falsely green
- CI pipeline runs on every push, fails the build if tests fail (prove this by deliberately breaking a test and pushing)

**Validate yourself:**
- Can you explain the difference between your liveness and readiness probe configuration, and justify why each threshold/timeout value is what it is? Generic copy-pasted probe config is an obvious tell to an interviewer — make sure yours reflects actual reasoning about this specific app

---

### STORY-10: Load testing and documented performance characteristics

**As a** platform engineer
**I want** to know how this service actually behaves under load
**So that** I can speak to real numbers in an interview instead of vague claims

**Description:**
Run a k6 or JMeter load test against the alert-ingestion endpoint at increasing concurrency (10, 50, 100, 200 concurrent). Record p50/p95/p99 latency, error rate, and DB connection pool behavior at each level. Document where the system starts degrading and why (connection pool exhaustion? DB query time? GC pauses?).

**Acceptance criteria:**
- A load test script exists in the repo and is runnable by anyone (documented command)
- A results table/graph exists in the README showing latency and error rate at each concurrency level
- At least one genuine bottleneck is identified, explained, and either fixed or explicitly documented as a known limitation with a proposed fix
- If you tune something in response (e.g., increase pool size, add caching), the before/after numbers are both recorded — this "I measured, found a problem, fixed it, measured again" loop is exactly what a senior engineer does and exactly what most portfolio projects skip

**Validate yourself:**
- This is the story most candidates skip entirely, which is exactly why doing it properly is disproportionately valuable in an interview. Can you tell a 90-second story: "at X concurrency, I saw Y degrade, I diagnosed it was Z, I changed W, and confirmed the fix with numbers"? If yes, this story alone is worth more in an interview than the other nine combined.

---

## How to use these 10 stories

- Work them roughly in order — 1-2-3-4 form the core domain, 5 is your "smart" differentiator, 6-7-8 are the production-maturity layer, 9-10 are what actually separates this from a tutorial project.
- Time-box each to your 12-week plan (roughly 1-1.5 stories per week, adjust as needed).
- Treat "Validate yourself" sections as a personal Definition of Done — if you can't answer them out loud without looking at your own code, the story isn't actually done yet, it just compiles.


---

## Update: Real Data Ingestion Sources (replaces manual-POST-only approach in STORY-2)

Decision: use REAL free data sources instead of purely synthetic/manual POSTs, so the project demonstrates a genuine ingestion pattern, not a toy.

### Primary source: GitHub Webhooks (real events, official, free)
- Register a webhook on one of your own public repos: Settings → Webhooks → Add webhook
- Subscribe to `workflow_run` (CI failures are a legitimate real incident trigger) and optionally `issues`, `push`
- Local dev exposure: use `smee.io` (free, purpose-built for local webhook development) or `ngrok` to forward GitHub's POSTs to your local Spring Boot app
- Verify the `X-Hub-Signature-256` HMAC header on every incoming payload — this is a real security practice (webhook signature verification) worth explicitly implementing and documenting, not skipping
- Map `workflow_run.conclusion == "failure"` events into your existing `Alert` model (STORY-1): `source = "github_actions"`, `raw_payload` = the webhook JSON, `service_name` = repo name

### Secondary source: Self-built synthetic monitoring (zero external dependency)
- Add a `@Scheduled` job that pings a configurable list of real public URLs (your own sites, or well-known public endpoints) every N seconds
- Record real response time and status code; if latency exceeds a threshold or status isn't 2xx, generate an `Alert` internally — same shape as GitHub-sourced alerts, feeding the SAME correlation engine (STORY-3)
- This mirrors exactly what Dynatrace/UptimeRobot-style synthetic monitoring does at its core, and requires no API key, no rate limits, no external dependency risk

### Optional third source: Public Statuspage.io incident feeds (real historical incident data)
- Poll `https://www.githubstatus.com/api/v2/incidents.json`, `https://www.cloudflarestatus.com/api/v2/incidents.json`, or other Statuspage-hosted companies' public APIs
- No auth needed; good for backfilling realistic incident history data for demos/screenshots without waiting for live events to occur naturally

### Why two real sources instead of one
Having GitHub webhooks (push-based, event-driven) AND synthetic monitoring (pull-based, scheduled) hitting the same correlation pipeline demonstrates you can design an ingestion layer that abstracts over different delivery models — a real architectural decision, and a strong thing to explain in an interview: "my system accepts both push-based webhooks and pull-based polling through the same internal Alert contract."

### Updated STORY-2 acceptance criteria (supersedes original manual-POST-only version)
- A real GitHub Actions failure on your test repo produces a correctly persisted `Alert` within seconds, verified end-to-end (trigger a real CI failure, watch it land in your DB)
- Webhook signature verification correctly rejects a forged/tampered payload (test this deliberately — send a fake payload with a wrong signature, confirm it's rejected)
- The synthetic monitor correctly detects a deliberately broken/slow endpoint (point it at something you control and intentionally break) and generates an alert
- Both ingestion paths converge on the same `Alert` entity and flow into the same correlation logic from STORY-3, with no source-specific branching in the correlation code itself
