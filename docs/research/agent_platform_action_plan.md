# Agent Platform — High-Level Action Plan

> Time-agnostic phases to build a hosted agent runtime with tool use, workflows, evals, versioning, and observability.

## Objectives & Scope
- Hosted agent runtime with first-class tool calling and a workflow engine (single and multi-agent), plus evaluations, versioning, and observability.
- Ship usable developer surfaces (API/SDKs) and run real workloads (HTTP tools + RAG) before expanding to enterprise features.

## Core Phases
1) Foundations
- Choose core stack (FastAPI or Django), messaging (Redis/Celery vs Cloud Tasks/Pub/Sub/Temporal), Postgres for metadata/runs, and object storage for artifacts.
- Set up identity (API keys), secrets management, and basic telemetry (OpenTelemetry, structured logs).

2) Agent Orchestrator
- Implement minimal planning/acting loop with tool calls, step limits, timeouts, and retries.
- Add structured outputs with JSON schema validation and guardrail hooks.
- Build model adapter layer (start with 1–2 providers).

3) Tools & Retrieval
- Define a tool spec (name, description, IO schemas, timeouts, idempotency) and a registry for built-ins (HTTP, KV, code) and custom tools.
- Add RAG: ingestion, embeddings, vector index, retrieval with filters; prompt assembly policies.

4) Workflow Engine
- Execute DAGs with typed node IO and state mapping; nodes for Start, Agent, Prompt, HTTP/Tool, Branch, Join, Loop, End.
- Per-node policies (retries, timeouts, error routing) and resumable runs.

5) Developer Surfaces
- APIs: create_run, get_run, stream_run (SSE), list_traces, create_dataset, run_eval.
- SDKs: Python/JS thin clients with retry/backoff and streaming helpers.
- Webhooks for run status and human-in-the-loop pauses (optional).

6) Evaluations & Versioning
- Datasets (CSV/JSONL), batch eval runner, metrics (exact/semantic, latency, cost), comparison reports, and regression thresholds.
- Immutable versions and environments (dev/stage/prod) with promotions.

7) Observability & Ops
- Traces for model/tool calls and node steps; metrics for latency, tokens, error rates, cost.
- Dashboards and basic alerting; redact sensitive fields in logs.

8) Security & Governance
- Secrets via KMS/Vault; RBAC (projects/roles), audit logging, data retention controls.
- Sandboxed execution for code tools with resource limits.

9) Deploy & Scale
- IaC; deploy API and workers as separate services (Cloud Run/ECS or GKE/EKS).
- Horizontal scaling and cost guardrails; plan migration path for queue/orchestrator if starting simple.

## Risks & Mitigations
- Orchestration complexity → Start minimal loop; add features behind flags.
- Tool security → Strict schemas, timeouts, isolation, least-privilege secrets.
- Eval quality → Gold datasets + rubric scoring; enforce regression gates.
- Cost → Budgets, rate limits, caching, model routing/fallbacks.
- Vendor lock-in → Abstract providers (LLM, vectors, queue) behind interfaces.

## Immediate Next Steps
- Decide stack (framework, queue, vector store) and provision core infra.
- Scaffold repos (api, workers, sdk-py, sdk-js, ui) and shared contracts.
- Implement minimal agent loop, HTTP tool, first model adapter, runs schema.
- Expose create_run/get_run/stream_run; ship Python SDK alpha and a single demo flow.
