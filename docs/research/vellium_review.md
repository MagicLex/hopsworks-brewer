# Vellum Agent Builder — Highlights

> Snapshot of Vellum’s agent-building primitives and characteristics (based on public site/docs, Oct 2025).

## What It Is
- Build agents in plain English, then refine via UI/SDK.
- Execution layer for AI agents with tool calling and workflows.
- Ships with evaluations, versioning, and production monitoring.

## Core Building Blocks
- Agent Builder: Natural‑language spec → agent scaffold with editable config.
- Workflows: Visual/DAG orchestration; agents appear as nodes (Agent Node).
- Tools: Built‑in tool calling; HTTP/DB/code and custom tools.
- Prompts & Templates: Versioned prompt components reused across agents.
- Datasets & Evals: Test sets and metrics to measure agent quality.
- Routing & Policies: Model/variant routing, fallbacks, guardrails.
- Memory & Retrieval: RAG support for context grounding (agentic RAG patterns).

## Agent Characteristics
- Multi‑step Reasoning: Plans, calls tools, iterates toward goals.
- Multi‑Agent Patterns: Examples include multi‑agent chatbots and handoffs.
- Deterministic Controls: Stop conditions, timeouts, retry/backoff.
- Guardrails: Input/output validation, safety filters, structured outputs.

## Orchestration & Execution
- Agent Node: Drop‑in node for workflows; compose with other nodes.
- Tool Use: First‑class function/tool calling for complex tasks.
- State & Context: Passes intermediate results between nodes/agents.
- Observability: Traces of steps, tool calls, tokens, and latencies.

## Quality & TDD
- Test‑Driven Agents: “AI agents meet test‑driven development” mindset.
- Offline/Batch Evals: Run scenarios against datasets; compare variants/models.
- Regression Gates: Protects quality when changing prompts/tools/models.

## Deploy & Operate
- Versioning: Promote agent versions across environments.
- Environments: Staging/production separation; config isolation.
- Monitoring: Live metrics, error/tokens monitoring, alerting.
- Rollbacks & Fallbacks: Safe revert and policy‑based backups.

## Integrations
- Models: Major LLM providers via unified interface.
- Data/HTTP: Connectors for REST/Graph, vector stores, and DBs.
- SDKs & API: Programmatic control (Python/JS) for CICD and runtime.

## Typical Uses
- Customer/chat agents with tools (CRM, knowledge, tickets).
- Agentic RAG for document workflows and internal search.
- Ops automations: multi‑step workflows with approvals and checks.

## Notable References
- Homepage: “Build agents in plain English”.
- Execution layer for AI agents page.
- Docs: Agent Builder (SME) and Agent Node.
- Docs: Multi‑agent chatbot example.
- Blog: Agentic RAG, agentic workflows, built‑in tool calling.

## Quick Mental Model
- Design: Describe → scaffold → refine prompts/tools/workflow → version.
- Validate: Create datasets → evaluations → compare variants/models.
- Ship: Pin versions → monitor traces/metrics → iterate safely.

