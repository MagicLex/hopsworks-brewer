# Vellum — Technology Stack (Agent‑Focused)

> Scope: Agent platform and runtime only. Excludes marketing/analytics. Based on public signals and product behavior; flags assumptions.

## Snapshot
- Product: Agent builder + workflows/orchestration, evals, monitoring.
- Delivery: SaaS with hosted agent execution, UI, SDKs, and APIs.
- Confirmed: Public API served by a Python app (Gunicorn) behind Google’s load balancer.

## Agent Platform Architecture (Inferred)
- Orchestrator/API
  - Python web service (Django or FastAPI) served by Gunicorn.
  - Coordinates runs, planning, tool calls, state transitions, timeouts/retries.
- Workers/Executors
  - Queue‑backed background execution for long/parallel steps and tools.
  - Likely uses Google Pub/Sub or Cloud Tasks, or Redis/Celery.
- Workflow Engine
  - DAG execution with an Agent Node; maps inputs/outputs; branching/loops.
- Tools & Models
  - Built‑in/custom tools (HTTP/DB/code). Model routing/fallbacks; structured I/O.
- Memory & Retrieval
  - RAG support for grounding; short‑term scratchpad per run.
- State & Storage
  - Relational DB (likely Postgres/Cloud SQL) for runs, versions, prompts, datasets.
  - Object storage (likely GCS) for artifacts/logs and large traces.
- Observability
  - Traces of tool/model calls, tokens, latencies; production monitoring and alerts.

## SDKs & APIs
- Client SDKs expected for Python/JavaScript; call agents/workflows, stream or await results, fetch traces.
- REST/JSON APIs with webhooks/callbacks for run status and tool results (typical for this class).

## Model & Data Integrations
- Major LLM providers via unified interface and routing policies.
- Retrieval to common vector stores/databases; HTTP/DB tool connectors.

## Security & Enterprise (Unverified)
- Expect API keys per env, RBAC, audit logs, retention controls, PII handling; specifics not public.

## Unknowns / To Verify
- Framework (Django vs FastAPI) and queue/orchestrator choice (Celery/Redis vs Pub/Sub/Cloud Tasks vs Temporal).
- Exact SDK packages, API base URLs, streaming endpoints.
- Runtime platform (Cloud Run vs GKE), isolation/concurrency limits, cost controls.
- Supported vector stores and model provider matrix.

## Quick Takeaways
- Stack suggests a Python‑based orchestrator with queue‑backed workers on GCP, plus a robust workflow engine for agent composition.
- Strong build → eval → deploy → monitor loop with versioning and guardrails.
