# Fleet — Detailed Phase Plan

Team assumption: **2 backend + 1 frontend + 0.5 DevOps**.
Effort unit: **engineer-weeks (ew)**.
Total: **~26 calendar weeks**, **~82 engineer-weeks**.

Each phase lists: **Goals · Workstreams · Tasks · Deliverables · Acceptance Criteria · Risks**.

---

## Phase 0 — Foundations (Weeks 1–2) · 6 ew

**Goal:** Lock architecture, data model, and team conventions so Phase 1 ships without rework.

### Workstreams
1. Architecture & ADRs
2. Data model
3. API contracts
4. Dev infrastructure

### Tasks
- [ ] ADR-001: Storage backend → **Postgres 15** (justify over Redis/MySQL)
- [ ] ADR-002: Transport → **HTTP/JSON v1, gRPC in Phase 6**
- [ ] ADR-003: Delivery semantics → **at-least-once + idempotency keys**
- [ ] ADR-004: Multi-tenancy → **row-level `tenant_id` + API-key scoping**
- [ ] ADR-005: Scheduler → **single leader, elected via Postgres advisory lock**
- [ ] Data model ER diagram: `tenants`, `job_types`, `queues`, `jobs`, `attempts`, `workers`, `schedules`, `audit_log`
- [ ] OpenAPI 3.1 spec for v1 endpoints
- [ ] Proto definitions parked for later (`v1.proto` stubbed)
- [ ] Monorepo layout: `/engine`, `/sdk-{ts,py,go}`, `/ui`, `/proto`, `/docs`, `/charts`
- [ ] CI: lint, test, build, container images
- [ ] License, CONTRIBUTING.md, CODE_OF_CONDUCT.md
- [ ] Local dev: `docker compose up` brings up engine + Postgres + UI

### Deliverables
- 5 merged ADRs, ER diagram, OpenAPI spec, green CI, running dev stack.

### Acceptance
- A new engineer can clone, run `docker compose up`, hit `GET /health` in < 10 minutes.
- Design review signed off by tech lead.

### Risks
- Bikeshedding on storage choice — timebox ADR-001 to 2 days.

---

## Phase 1 — Job Master + Core engine MVP (Weeks 3–6) · 12 ew

**Goal:** You can register a type, run a worker, submit a job, see it processed, retried, and dead-lettered — all via curl.

### Workstreams
1. Job Master (catalog) — _backend eng #1_
2. Submit / claim / ack loop — _backend eng #2_
3. Retries + DLQ + idempotency — _shared_
4. Auth + tenancy — _backend eng #2_

### Tasks

**Job Master**
- [ ] `POST /v1/job_types` — create with `name`, `version`, `input_schema`, defaults
- [ ] `PATCH /v1/job_types/:id` — update, deprecate, archive
- [ ] `GET /v1/job_types` — list, filter by status
- [ ] JSON Schema validator wired in at submit time
- [ ] Unique constraint `(tenant_id, name, version)`
- [ ] Status machine: `draft → active → deprecated → archived`

**Worker protocol**
- [ ] `POST /v1/workers/register` — `{ worker_id, host, version, handles: [{type, version}] }`
- [ ] Reject registration if any claimed type is not `active`
- [ ] `POST /v1/workers/:id/heartbeat` — every 10s; marked dead after 30s silence
- [ ] `POST /v1/jobs/claim` — long-poll, up to 30s, `{queues, max_batch}`
- [ ] `POST /v1/jobs/:id/ack` with result
- [ ] `POST /v1/jobs/:id/nack` with error + optional backoff override
- [ ] `POST /v1/jobs/:id/heartbeat` — extend visibility for long jobs
- [ ] `POST /v1/jobs/:id/progress` — `{percent, note}`

**Submit**
- [ ] `POST /v1/jobs` — validate type, payload, queue, idempotency key
- [ ] Queues created lazily on first submit
- [ ] Priority 0–9, default 5
- [ ] `GET /v1/jobs/:id`, `GET /v1/jobs?status=&type=&queue=`
- [ ] `POST /v1/jobs/:id/cancel`

**Retries / DLQ**
- [ ] Exponential backoff with jitter (`base * 2^attempt + rand(0, base)`)
- [ ] `max_attempts` from type default, overridable per submit
- [ ] DLQ: `jobs.status = 'dead_letter'` after final failure
- [ ] `POST /v1/jobs/:id/replay` — re-queues from DLQ

**Auth**
- [ ] API keys per tenant with scopes: `admin | submit | worker | read`
- [ ] `Authorization: Bearer <key>` middleware

### Deliverables
- Engine binary + Postgres schema + curl-based E2E demo script in `/examples/curl-demo.sh`.

### Acceptance
- E2E demo: register type → start worker loop in bash → submit 100 jobs → 95 succeed, 5 fail → DLQ shows 5 → replay → all green.
- Visibility timeout works: kill a worker mid-job, job re-claimed after timeout.
- Idempotency: submit same job twice with same key → one job created.

### Risks
- Postgres claim query becomes a hot row. Use `SELECT ... FOR UPDATE SKIP LOCKED` early.

---

## Phase 2 — SDKs (Weeks 7–8) · 6 ew

**Goal:** `pip install fleet-sdk`, `npm i fleet-sdk`, `go get fleet-sdk` — all three feel native.

### Workstreams
1. Shared protocol & reference client — _backend eng #1_
2. TS + Py SDKs — _backend eng #2_
3. Go SDK + examples — _backend eng #1_

### Tasks

**Shared**
- [ ] Protocol conformance test suite (HTTP-level, language-neutral)
- [ ] Error taxonomy: `TransientError`, `PermanentError`, `TimeoutError`

**Each SDK**
- [ ] `Client.submit(type, payload, opts)` → job id
- [ ] `Worker.on(type, handler)` registration
- [ ] `Worker.run(queues=[...])` — long-poll, heartbeat, graceful shutdown on SIGTERM
- [ ] Automatic schema validation (optional, warn-only toggle)
- [ ] Structured logging hooks
- [ ] Retries on network errors (not on PermanentError from handler)

**Examples (`/examples/`)**
- [ ] `send_email` — one-liner worker, submit from CLI
- [ ] `resize_image` — reads payload ref from S3, writes result ref
- [ ] `nightly_report` — fan-out across 100 child jobs

### Deliverables
- 3 SDK packages, published to npm / PyPI / Go modules.
- 3 worked examples, each with README.

### Acceptance
- Same conformance test suite passes against all three SDKs.
- `send_email` example runs end-to-end in under 5 minutes from a fresh clone.

### Risks
- Go generics ergonomics for `Worker.on` — prototype early.

---

## Phase 3 — Dashboard MVP (Weeks 9–12) · 12 ew

**Goal:** The demo that sells Fleet. A clean, real-time UI that makes other job queues feel dated.

### Workstreams
1. Design system + layout — _frontend_
2. Catalog + Submit pages — _frontend_
3. Queues + Workers + Job Detail — _frontend_
4. Realtime backbone — _backend eng #2_

### Tasks

**Backend**
- [ ] `GET /v1/stream/events` (SSE) — queue depth deltas, job state changes, worker heartbeats
- [ ] `GET /v1/stats/queues` — rolling windows (1m, 5m, 1h)
- [ ] `GET /v1/stats/types` — same
- [ ] `GET /v1/jobs/:id/logs` — tail + SSE

**Frontend (Next.js + Tailwind + shadcn/ui + Tanstack Query)**
- [ ] Shell: sidebar nav, tenant/project switcher, dark mode
- [ ] **Catalog page** — list, create (Monaco JSON Schema editor), preview auto-form, deprecate flow
- [ ] **Submit page** — type picker → auto-rendered form (react-jsonschema-form) → submit → link to job
- [ ] **Queues page** — cards per queue, sparkline of depth, click into detail
- [ ] **Queue detail** — live chart, per-type breakdown table, in-flight list
- [ ] **Workers page** — table of workers, health badges, in-flight count
- [ ] **Job detail page** — header (status, type, timing), payload JSON viewer, result viewer, attempts timeline, live logs tab, parent/children tab
- [ ] **Dead letters page** — filter + bulk select + replay dialog

### Deliverables
- Running UI at `http://localhost:3000`, screenshots in README.

### Acceptance
- Submit a job from UI → watch it move through states in real time without refresh.
- Replay 50 dead letters in one action.
- Lighthouse > 90 on main pages.

### Risks
- SSE scaling — use per-tenant topics, cap connections per key.

---

## Phase 4 — Scheduling & chains (Weeks 13–15) · 9 ew

**Goal:** Fleet handles time (cron, delayed) and dependencies (chains, fan-out).

### Workstreams
1. Scheduler service — _backend eng #1_
2. Chains + fan-out — _backend eng #2_
3. Calendar + chain viewer UI — _frontend_

### Tasks

**Scheduler**
- [ ] `schedules` table: `{id, type_id, payload, cron?, run_at?, next_fire_at, status}`
- [ ] Leader-elected scheduler loop (advisory lock), polls every 1s
- [ ] Promotes due schedules into `jobs`
- [ ] `POST /v1/schedules`, list, pause, resume, delete
- [ ] Cron parsing (quartz-style or standard, pick one)

**Chains**
- [ ] Submit with `on_success: { type, payload_template }` and `on_failure: {...}`
- [ ] `payload_template` can reference parent result via `$.result.field`
- [ ] Child job gets `parent_job_id`

**Fan-out / fan-in**
- [ ] `POST /v1/batches` — submit N children atomically
- [ ] Parent batch tracks `{total, succeeded, failed}`
- [ ] Optional `on_complete` callback job when all children settle

**UI**
- [ ] Calendar view (month/week) of upcoming scheduled + recurring
- [ ] Chain visualizer in Job Detail (tree, each node clickable)
- [ ] Batch progress bar + child list

### Deliverables
- Cron works end-to-end; chain demo; batch demo.

### Acceptance
- Schedule a `nightly_report` for `0 3 * * *` → fires on time, visible in calendar.
- Chain: `fetch_data → transform → upload`, with result threading, visible as tree.
- 10 000-child batch submits in under 5s, parent progress updates in real time.

### Risks
- Scheduler drift under load. Benchmark early with 1 M active schedules.

---

## Phase 5 — Observability & ops (Weeks 16–18) · 9 ew

**Goal:** Fleet is production-grade: you can debug, alert, and rate-limit.

### Workstreams
1. Metrics + traces — _backend eng #1_
2. Alerts engine — _backend eng #2_
3. Rate limiting — _backend eng #2_
4. Alerts UI — _frontend_

### Tasks

**Metrics**
- [ ] Prometheus `/metrics`: `fleet_queue_depth`, `fleet_jobs_total{status,type,queue}`, `fleet_attempts_total`, `fleet_worker_fleet_size`, latency histograms
- [ ] Per-tenant cardinality cap to prevent explosion

**Traces**
- [ ] OTel SDK integrated; spans for submit → claim → ack
- [ ] Trace context propagated via job metadata so worker SDKs can continue the trace

**Audit log**
- [ ] `audit_log` table for type changes, replay, cancel, schedule edits, RBAC changes
- [ ] UI tab per entity showing history

**Alerts**
- [ ] Rule model: `{metric, aggregation, window, threshold, channel}`
- [ ] Channels: webhook, email (SMTP), Slack
- [ ] Rule evaluator loop every 30s
- [ ] UI: rules CRUD, recent firings

**Rate limiting**
- [ ] Per-queue QPS cap
- [ ] Per-type QPS cap
- [ ] Token-bucket impl, soft (slow accept) vs hard (reject)

### Deliverables
- Grafana dashboard JSON in `/ops/grafana/`, alert rules example pack.

### Acceptance
- Trip a queue-depth alert, see webhook fire.
- Rate limit test: submit 10× over cap → reject rate matches config.

---

## Phase 6 — Scale & multi-tenancy (Weeks 19–22) · 12 ew

**Goal:** 10 k jobs/sec sustained, true multi-tenant isolation, SaaS-ready.

### Workstreams
1. gRPC transport — _backend eng #1_
2. Horizontal scaling — _backend eng #1_
3. Tenancy + RBAC — _backend eng #2_
4. Load testing — _DevOps_

### Tasks

**gRPC**
- [ ] Same methods as HTTP, generated from `v1.proto`
- [ ] Bidi stream for claim loop (lower latency than long-poll)

**Scale-out**
- [ ] Stateless API tier behind LB
- [ ] Scheduler + reaper as separate deployments with leader election
- [ ] Postgres: partition `jobs` by `tenant_id` hash and by month on `created_at`
- [ ] Optional Redis Streams backend for hot path (feature-flagged)
- [ ] Read replicas for dashboard queries

**Tenancy**
- [ ] `tenants`, `projects` (sub-tenant), `users`, `memberships`
- [ ] RBAC roles: `admin`, `operator`, `developer`, `viewer`
- [ ] Fine-grained perms per `job_type` pattern (glob: `billing.*`)
- [ ] API-key scopes map to roles

**Load testing**
- [ ] k6 scenarios: 10 k submit/s, 10 k claim/s, 100 k in-flight
- [ ] Soak test 24 h
- [ ] Chaos: kill scheduler, kill Postgres primary, network partition workers

### Deliverables
- Load test report, scaling playbook in `/docs/scaling.md`.

### Acceptance
- Sustained 10 k jobs/s for 30 min, p99 submit < 50 ms, no data loss on chaos tests.

---

## Phase 7 — Packaging & launch (Weeks 23–26) · 12 ew

**Goal:** Anyone in the world can install, try, and sign up.

### Workstreams
1. Distribution artifacts — _DevOps_
2. Managed cloud — _backend eng #1 + DevOps_
3. Docs & marketing site — _frontend + tech writer_
4. Billing + onboarding — _backend eng #2_

### Tasks

**Distribution**
- [ ] Multi-arch Docker images (`linux/amd64`, `linux/arm64`)
- [ ] Helm chart with `values.yaml` for Postgres, ingress, metrics
- [ ] `fleetctl` CLI for admin ops

**Managed cloud**
- [ ] Control plane: tenant provisioning, API-key issuance, billing hooks
- [ ] Data plane: per-tenant namespace, isolated Postgres schema
- [ ] Hosted at `app.fleet.dev`

**Docs**
- [ ] Mintlify / Docusaurus site
- [ ] Quickstart per SDK (5-min to first job)
- [ ] Concepts, API reference, operations guide, pricing

**Billing**
- [ ] Stripe integration
- [ ] Metering: jobs/month, retained history, UI seats
- [ ] Free tier: 100 k jobs/mo, 7-day retention, 1 project

**Launch**
- [ ] HN + Product Hunt + /r/programming
- [ ] Demo video (3 min)
- [ ] Public beta signup

### Deliverables
- `helm install fleet` works · docs live · `app.fleet.dev` live · public beta open.

### Acceptance
- A new user can sign up, create a type, run the TS SDK sample, and see their first job succeed in under 10 minutes.

---

## Rollup

| Phase | Weeks | Eng-weeks | Key outcome |
|---|---|---|---|
| 0 Foundations | 1–2 | 6 | Architecture locked |
| 1 Job Master + Core | 3–6 | 12 | E2E via curl |
| 2 SDKs | 7–8 | 6 | TS/Py/Go clients |
| 3 Dashboard | 9–12 | 12 | Demoable UI |
| 4 Scheduling + chains | 13–15 | 9 | Time + deps |
| 5 Observability | 16–18 | 9 | Metrics, traces, alerts |
| 6 Scale + tenancy | 19–22 | 12 | 10k jobs/s, multi-tenant |
| 7 Packaging + launch | 23–26 | 12 | Public beta |
| **Total** | **26 wk** | **78 ew** | **v1 shipped** |

Add 15–20% buffer for design churn, holidays, and the unknowns Phase 1 will surface.

---

## Dependencies & critical path

- Phase 1 blocks everything.
- Phase 2 (SDKs) and Phase 3 (UI) can run partially in parallel after Phase 1 week 2.
- Phase 4 depends on Phase 1.
- Phase 5 can start as soon as Phase 1 is stable.
- Phase 6 is the biggest risk — start load-testing stubs in Phase 1 to avoid surprises.
- Phase 7 depends on 0–6 complete.

## Hiring triggers

- Add a 2nd frontend engineer at start of Phase 3.
- Add a full-time DevOps at start of Phase 6.
- Add a tech writer at start of Phase 7.

## Definition of Done per phase

Every phase must ship with:
1. Code + tests (unit + integration coverage > 70% on new code).
2. Docs updated in `/docs`.
3. CHANGELOG.md entry.
4. Demo recording (2 min max).
5. Design-review signoff.
