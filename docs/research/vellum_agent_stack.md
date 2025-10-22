# Vellum — Agent Stack (One‑Pager)

## What It Is
- Hosted agent runtime plus workflow engine (Agent Node, tools, branching, RAG) with evals, versioning, and observability.

## Confirmed Signals
- `api.vellum.ai` serves a Python app via Gunicorn behind Google’s LB (`server: gunicorn`, `via: 1.1 google`).

## Likely Architecture (Inference)
- Orchestrator/API: Python service (Django or FastAPI) coordinating reasoning, tool calls, step/state transitions.
- Workers/Executors: Queue‑backed background jobs for long/parallel steps, retries, and timeouts (Redis/Celery or GCP Pub/Sub/Cloud Tasks).
- Workflow Engine: DAG execution with typed node IO; Agent/Prompt/HTTP/Tool/Branch/Loop/End nodes; resumable runs.
- Tools & Models: Built‑in/custom tools (HTTP/DB/code); model adapters; routing/fallbacks; structured outputs.
- Memory/Retrieval: RAG for grounding; per‑run scratchpad/state.
- State/Storage: Relational DB (likely Postgres/Cloud SQL) for runs/versions/datasets; object storage (likely GCS) for artifacts/logs.
- Observability: Traces of model/tool calls, tokens, latencies; metrics and alerts.

## Interfaces
- APIs/SDKs to create/stream runs, fetch traces, manage datasets/evals.
- Webhooks/callbacks for run status and human‑in‑the‑loop pauses (typical).

## Controls & Guardrails
- Step/timeout/retry policies; JSON‑schema outputs; validators/safety filters.

## Unknowns To Verify
- Framework (Django vs FastAPI), queue/orchestrator (Celery/Redis vs Pub/Sub/Cloud Tasks vs Temporal).
- Exact SDK packages, API base URLs/streaming endpoints; runtime platform (Cloud Run vs GKE).
- Supported vector stores and provider matrix; isolation/concurrency and cost controls.

## Quick Takeaways
- Agent runtime appears Python‑based with queue‑backed execution on GCP.
- Strong composition: tools, RAG, branching, versioning, and evals with full traces.
