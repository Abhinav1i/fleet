# Fleet

A modern, language-agnostic job queue with a beautiful real-time UI.

Submit jobs over HTTP/gRPC. Workers in any language pull them, execute them, report back. Fleet handles routing, retries, scheduling, dead letters, and visibility. Think Sidekiq / Celery / BullMQ — but language-agnostic, UI-first, and self-hostable.

---

## Why Fleet

Most job queues are tied to one language (Sidekiq → Ruby, Celery → Python, BullMQ → Node). Their dashboards are afterthoughts. Fleet flips that:

- **Language-agnostic** — SDKs for TS, Python, Go at launch; any language that speaks HTTP/gRPC can be a worker.
- **UI-first** — real-time dashboard is the headline feature, not a bolt-on.
- **Governed** — a job catalog (`job_types`) is the source of truth. Typos and ghost queues are impossible.
- **Ops-friendly** — self-hosted via Docker/Helm, Prometheus/OTel metrics, multi-tenant, RBAC.

---

## Core concept — Job Master

Fleet treats job types as **first-class, catalogued entities**, not arbitrary strings.

A `job_types` table is the source of truth. Workers may only serve types that exist in the catalog. Submissions are validated against the catalog at submit time, not at execute time.

```
job_types
---------
id                   uuid
tenant_id            uuid
name                 string       -- "billing.send_invoice"
version              string       -- "1.2.0"
display_name         string
description          text
input_schema         jsonb        -- JSON Schema, validated on submit
output_schema        jsonb
default_queue        string
default_priority     int
default_timeout_ms   int
default_retry_policy jsonb        -- { max_attempts, backoff, ... }
owner_team           string
status               enum         -- draft | active | deprecated | archived
created_at, updated_at
unique(tenant_id, name, version)
```

**Why this matters:**
- Typos caught at submit, not silently queued forever
- UI always knows the full catalog — dropdown of valid types, auto-generated payload forms
- Schema + routing config live on the type, not scattered across workers
- Governance, RBAC per type, explicit versioning, clean deprecation flow

---

## What counts as a "job"

A job is a discrete unit of work with a defined start and end.

- **Job type** — a registered entry in the job catalog (e.g. `send_email`, `billing.invoice`, `media.resize`).
- **Payload** — JSON, up to 256 KB. Larger data goes to blob storage; payload carries a reference.
- **Metadata** — priority, queue, retry policy, timeout, idempotency key, scheduled-at, trace IDs.
- **Result** — JSON on success, structured error on failure.

Fleet routes, tracks, and retries. Fleet does **not** run user code.

### In scope

- Fire-and-forget background tasks
- Delayed and scheduled jobs
- Recurring (cron) jobs
- Long-running jobs with heartbeats and progress
- Fan-out / fan-in batches
- Linear chains (job B after job A, with A's output as B's input)
- Priority and queue-based routing

### Out of scope

- Streaming / continuous data (use Kafka)
- Sub-100 ms request/response (use gRPC/HTTP directly)
- Full workflow-as-code engines (use Temporal / Cadence)
- Arbitrary DAGs with dynamic branching and conditional joins (use Airflow / Prefect)
- Running user code / container orchestration (use Kubernetes)
- ETL semantics, ML gang scheduling, human-approval workflows

### How we say no

A feature is rejected if it requires any of:
1. Fleet executing user code
2. A durable programming model beyond `function + retry`
3. Sub-100 ms end-to-end latency as a contract
4. Coupling to a specific language, framework, or cloud
5. Turing-complete workflow semantics
6. Owning blob storage, secrets, or compute provisioning

---

## Roadmap — Phase-wise design & effort

Effort is in **engineer-weeks** assuming 2 backend + 1 frontend engineers.

---

### Phase 0 — Foundations &nbsp; · &nbsp; _2 weeks_ &nbsp; · &nbsp; ~6 eng-weeks

**Design**
- Repo layout: `/engine`, `/sdk-ts`, `/sdk-py`, `/sdk-go`, `/ui`, `/proto`, `/docs`
- ADRs: storage backend (Postgres), transport (HTTP first, gRPC later), delivery semantics (at-least-once), multi-tenancy (row-level)
- Data model draft: `tenants`, `job_types`, `queues`, `jobs`, `attempts`, `workers`, `schedules`
- Protobuf / OpenAPI definitions for the wire protocol
- CI/CD, licensing, code of conduct, contribution guide

**Deliverables:** repo scaffolding, ADRs merged, data model diagram, API spec.

---

### Phase 1 — Job Master + Core engine MVP &nbsp; · &nbsp; 4 weeks &nbsp; · &nbsp; ~12 eng-weeks

**Design**
- `job_types` table + CRUD API (`POST /job_types`, `PATCH`, `GET`)
- Worker registration: `POST /workers/register` with claimed types; Fleet rejects types not in the catalog
- Submit API: `POST /jobs` validates `type` exists, `status=active`, payload matches `input_schema`
- Claim loop: `POST /jobs/claim` (long-poll), `POST /jobs/:id/ack`, `nack`, `heartbeat`
- Named queues with priority (0–9)
- Visibility timeout, at-least-once delivery
- Retries with exponential backoff + jitter, dead-letter queue
- Idempotency keys (dedupe window)
- API-key auth

**Deliverables:** running engine, you can register a type, boot a worker (curl + bash), submit a job, see it processed, see it retry, see it dead-letter.

---

### Phase 2 — SDKs &nbsp; · &nbsp; 2 weeks &nbsp; · &nbsp; ~6 eng-weeks

**Design**
- Consistent surface across languages:
  - `client.submit(type, payload, opts)`
  - `worker.on(type, handler)`
  - `worker.run()` (long-poll loop, heartbeats, graceful shutdown)
- TypeScript SDK (worker + client)
- Python SDK (worker + client)
- Go SDK (worker + client)
- End-to-end examples: `send_email`, `resize_image`, `nightly_report`
- SDK-level schema validation mirroring the server

**Deliverables:** three SDKs published to npm / PyPI / Go modules; worked examples in `/examples`.

---

### Phase 3 — Dashboard MVP &nbsp; · &nbsp; 4 weeks &nbsp; · &nbsp; ~12 eng-weeks

**Design — pages**
- **Catalog** — list of job types, create/edit (schema editor + preview form)
- **Queues** — depth, throughput, p50/p95 latency, error rate; drill into a queue
- **Workers** — roster with host, version, claimed types, in-flight count, last heartbeat
- **Job detail** — payload, result, attempts timeline, live log tail, parent/children links
- **Submit** — form auto-generated from `input_schema` of the selected type
- **Dead letters** — filter, inspect, one-click replay (single or bulk)

**Design — tech**
- Next.js + Tailwind + shadcn/ui
- WebSocket / SSE for live updates
- Monaco editor for JSON Schema editing

**Deliverables:** usable self-service dashboard. First "wow moment" demo.

---

### Phase 4 — Scheduling & chains &nbsp; · &nbsp; 3 weeks &nbsp; · &nbsp; ~9 eng-weeks

**Design**
- Scheduled jobs (`run_at` timestamp) — stored separately, promoted into queue at due time
- Recurring jobs — cron syntax, next-fire computed on each run
- Calendar view in UI (upcoming scheduled + recurring)
- Linear chains — `next_job_on_success`, `next_job_on_failure`; parent/child links
- Fan-out / fan-in — submit a batch of child jobs under a parent; parent completes when all children do; aggregate status in UI

**Deliverables:** `cron.every("0 3 * * *").submit("nightly_report")` works end-to-end; chain visualizer in job detail.

---

### Phase 5 — Observability & ops &nbsp; · &nbsp; 3 weeks &nbsp; · &nbsp; ~9 eng-weeks

**Design**
- Prometheus `/metrics` endpoint — queue depth, throughput, retry counts, worker fleet size, type-level breakdowns
- OpenTelemetry traces — submit → claim → ack as a span tree
- Structured audit log for sensitive events (type changes, replay, cancel)
- Alerts with webhook delivery — queue depth, error rate, worker fleet drop, scheduled miss
- Rate limiting per queue and per job type

**Deliverables:** Grafana-ready metrics, trace export, first-class alerting UI.

---

### Phase 6 — Scale & multi-tenancy &nbsp; · &nbsp; 4 weeks &nbsp; · &nbsp; ~12 eng-weeks

**Design**
- gRPC transport alongside HTTP (same proto both sides)
- Horizontal scaling of API layer (stateless) + leader election for the scheduler
- Storage: sharded Postgres by `tenant_id`, or pluggable (Redis Streams for hot queues, SQLite for dev)
- Tenants, projects within tenants, RBAC per role and per `job_type` pattern (`billing.*`)
- API-key scopes (submit-only, worker-only, admin)
- Target: 10 k jobs/sec sustained per cluster, 1 M in-flight

**Deliverables:** multi-tenant SaaS-ready engine, load-tested to targets.

---

### Phase 7 — Packaging & launch &nbsp; · &nbsp; 4 weeks &nbsp; · &nbsp; ~12 eng-weeks

**Design**
- Docker images for engine + UI
- Helm chart for Kubernetes deploys
- Hosted control plane (managed cloud offering)
- Docs site (Mintlify / Docusaurus) with quickstarts per SDK
- Marketing site + public beta signup
- Pricing model (seat-based for UI users, throughput-based for jobs)

**Deliverables:** `helm install fleet` works; `app.fleet.dev` is live; public beta open.

---

### Totals (v1)

| Phase | Duration | Effort |
|---|---|---|
| 0 — Foundations | 2 wk | 6 eng-wk |
| 1 — Job Master + Core engine | 4 wk | 12 eng-wk |
| 2 — SDKs | 2 wk | 6 eng-wk |
| 3 — Dashboard MVP | 4 wk | 12 eng-wk |
| 4 — Scheduling & chains | 3 wk | 9 eng-wk |
| 5 — Observability & ops | 3 wk | 9 eng-wk |
| 6 — Scale & multi-tenancy | 4 wk | 12 eng-wk |
| 7 — Packaging & launch | 4 wk | 12 eng-wk |
| **Total** | **~26 wk (~6 months)** | **~78 eng-wk** |

### Deferred to v2+
- Arbitrary DAGs
- Workflow-as-code SDK
- Cross-region replication
- BYO storage backend
- Plugin / extension system

---

## Non-functional targets (v1)

| Metric | Target |
|---|---|
| Submit → visible in queue | < 50 ms p99 |
| Sustained throughput | 10 000 jobs / sec / cluster |
| Max in-flight jobs | 1 000 000 |
| History retention (default) | 30 days |
| Max payload | 256 KB |
| Max result | 256 KB |

---

## Status

Pre-alpha. Design phase. Not yet usable.

## License

TBD (likely Apache-2.0 for the engine, source-available for the managed cloud bits).
