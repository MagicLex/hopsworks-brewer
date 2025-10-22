# Hopsworks Agent Builder

A visual agent authoring platform for creating AI agents that deploy to Hopsworks KServe.

## What This Is

An **agent IDE** with three synchronized views:
- **Visual Flow Editor** (ReactFlow) - Drag and drop nodes
- **Chat Assistant** - Natural language flow manipulation
- **Code View** - Direct IR editing in YAML

The platform compiles visual flows to Python agents deployable on Hopsworks KServe, with native integration to Hopsworks feature stores.

## Core Innovation

**Everything is a Transform Node** with optional capabilities:
- Any node can be augmented with an LLM prompt layer
- Declarative operations with code escape hatches
- Context propagation and state management built-in
- Point-in-time correct feature reads from Hopsworks
- Seamless integration of ML models (XGBoost, scikit-learn) with LLMs

## Documentation

- [`docs/DESIGN.md`](docs/DESIGN.md) - Overall architecture and philosophy
- [`docs/IR-SPEC.md`](docs/IR-SPEC.md) - Complete IR schema and node model
- [`docs/COMPILATION.md`](docs/COMPILATION.md) - How IR compiles to Python/KServe
- [`docs/IMPLEMENTATION.md`](docs/IMPLEMENTATION.md) - Tech stack and build plan

## Examples

- [`examples/customer-service.yaml`](examples/customer-service.yaml) - Full customer support agent with knowledge base
- [`examples/simple-feature-enrichment.yaml`](examples/simple-feature-enrichment.yaml) - Simple LLM-augmented feature engineering

## Key Design Principles

1. **Pragmatic Abstraction** - Not too simple, not overengineered
2. **IR as Truth** - Everything flows through the Intermediate Representation
3. **Progressive Disclosure** - Start simple, reveal complexity as needed
4. **Hopsworks Native** - Use existing patterns, don't reinvent
5. **Production Ready** - Timeouts, retries, error handling by default

## Architecture

```
┌─────────────┬──────────────┬──────────────┐
│  Flow UI    │     Chat     │  Code View   │
│ (ReactFlow) │  Assistant   │   (Monaco)   │
└─────────────┴──────────────┴──────────────┘
         ↓            ↓             ↓
         └────────────┴─────────────┘
                     ↓
              [Shared IR State]
                     ↓
            [Python Compiler]
                     ↓
            [KServe Deployment]
```

## Tech Stack

**Frontend:**
- Next.js 14, ReactFlow, Monaco Editor, tailwind-quartz

**Backend:**
- FastAPI, Pydantic, Jinja2, HSFS

**Deployment:**
- Docker, KServe, Hopsworks

## Getting Started

See [`docs/IMPLEMENTATION.md`](docs/IMPLEMENTATION.md) for the complete build plan.

## What This Is Not

- Not a generic workflow engine (use Airflow)
- Not a Hopsworks replacement (it's a companion)
- Not a distributed system (single container)
- Not multi-tenant (single workspace initially)

## Status

This is the architecture and design phase. Implementation follows the plan in `docs/IMPLEMENTATION.md`.