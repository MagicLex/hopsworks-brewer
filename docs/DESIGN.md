# Agent Builder Platform - Architecture Design

## Problem Statement

We need a visual agent builder that:
1. Provides a ReactFlow-based UI for designing agent workflows
2. Allows chat-based manipulation of the same workflow
3. Shows live code representation that can be edited directly
4. Compiles to Python agents deployable on Hopsworks KServe
5. Integrates natively with Hopsworks feature stores

The platform is **not** Hopsworks itself, but a specialized tool for creating agents that deploy to Hopsworks.

## Core Abstractions

### 1. The Transform Node

Everything is a transformation node. No special types, just capabilities.

```typescript
type Node = {
  id: string
  type: 'transform'  // Only one type for now

  // Data flow
  inputs: PortSchema[]
  outputs: PortSchema[]
  error?: PortSchema

  // Optional LLM augmentation
  prompt?: PromptLayer

  // Implementation
  transform: Transform

  // Runtime behavior
  runtime?: RuntimeConfig

  // State management
  state?: StateConfig
}
```

**Key Insight**: Any node can be augmented with an LLM prompt layer. This isn't a separate node type - it's a capability that can be added to any transformation.

**Conditional Logic**: If/then branching is handled through edge conditions, not special nodes:
```typescript
edge: {
  from: "risk_scorer.score",
  to: "high_risk_handler",
  condition: "score > 0.7"  // Only routes if condition is true
}
```
This keeps nodes focused on transformation, while edges handle routing logic.

**Future Node Types**: While everything is currently a `transform`, the type field exists for potential future extensions:
- `boundary` - Explicit entry/exit points for sub-flows
- `subflow` - Reference to another complete flow (composition)
- `stream` - Different execution semantics for streaming data
- `batch` - Batch processing with different parallelization

These would only be added if transform nodes with different edge conditions prove insufficient.

### 2. Context Propagation

Every flow has an implicit context that threads through all nodes:

```typescript
type Context = {
  user_id: string
  session_id: string
  trace_id: string
  event_time: string  // For point-in-time reads
  auth?: AuthToken
}
```

Context is automatically propagated without explicit wiring in the UI. Nodes declare what context they need via `runtime.requires_context`.

### 3. State Management

Two scopes of state, no more:

- **request**: Lives for one execution
- **session**: Lives across multiple calls (conversation memory)

```python
# State is accessed through a simple API
history = await state.get("key", default=[])
await state.set("key", new_value)
```

### 4. The Intermediate Representation (IR)

The IR is the contract between all components:

```yaml
version: "1.0"
metadata: {...}
context: {requires: [...]}
nodes: {node_id: NodeSpec, ...}
edges: [{from: "node.port", to: "node.port"}, ...]
```

The IR is:
- **Declarative first**: Nodes declare what they want, not how
- **Chat-native**: Structured for LLM understanding and manipulation
- **Bidirectional**: Can generate UI from code or code from UI

## System Architecture

### Three-Panel Interface

```
┌─────────────────┬──────────────────┬──────────────────┐
│   Flow Editor   │   Chat Assistant │   Code View      │
│   (ReactFlow)   │                  │   (Monaco)       │
│                 │                  │                  │
│  Visual nodes   │  "Add a node     │  version: "1.0"  │
│  and edges      │   that fetches   │  nodes:          │
│                 │   user data"     │    fetch_user:   │
└─────────────────┴──────────────────┴──────────────────┘
         ↓                 ↓                   ↓
         └─────────────────┴───────────────────┘
                           ↓
                    [Shared IR Store]
                           ↓
                 [Compiler + Validator]
                           ↓
                    [Python Agent Code]
                           ↓
                    [KServe Container]
```

### Component Responsibilities

#### UI Layer (ReactFlow)
- Visual node editing with tailwind-quartz components
- Drag-and-drop node creation
- Edge drawing with validation
- Node property panels
- Real-time IR sync

#### Chat Intelligence
- Understands complete IR schema
- Can add, modify, delete nodes
- Suggests optimizations
- Explains flow behavior
- Reviews generated code

#### Code View
- Live IR representation in YAML
- Syntax highlighting and validation
- Direct editing with immediate UI updates
- Shows generated Python preview

#### Compiler
- Traverses IR to generate Python
- Handles context propagation
- Manages state lifecycle
- Generates KServe predictor class
- Validates against Hopsworks schema

## Integration Points

### Hopsworks Integration

Native integration through declarative operations:

```yaml
transform:
  mode: declarative
  spec:
    op: hopsworks.read_features
    args:
      feature_group: "customers_v3"
      key: "${user_id}"
      as_of: "${context.event_time}"
```

The compiler translates these to HSFS API calls with proper authentication and point-in-time semantics.

### KServe Deployment

Generated agents compile to standard KServe predictors:

```python
class GeneratedAgent(Model):
    def __init__(self):
        super().__init__("agent-name")
        self.hsfs_conn = hsfs.connection()
        self.llm = LLMClient()

    async def predict(self, request):
        # Generated flow execution
        return response
```

### LLM Integration

Prompt layers compile to appropriate LLM client calls:

```python
# Generated from prompt layer
response = await llm.generate(
    system=node.prompt.system,
    user=template.render(**inputs),
    schema=node.prompt.output_schema
)
```

## Design Principles

### 1. Progressive Disclosure
- Start with minimal nodes
- Add capabilities as needed
- Hide complexity until required
- Sensible defaults everywhere

### 2. Declarative First, Code Second
- 80% of nodes should be declarative
- Escape to code when needed
- Declarative nodes are easier to understand and modify

### 3. IR as Source of Truth
- The IR is canonical
- UI and code are views
- All modifications go through IR
- Validation happens at IR level

### 4. Production-Ready Defaults
- Timeouts on everything
- Retries where sensible
- Error handling built-in
- Context propagation automatic

### 5. Hopsworks Native
- Use Hopsworks patterns
- Don't reinvent feature stores
- Respect existing schemas
- Deploy to standard KServe

## What This Is Not

- **Not a generic workflow engine**: Airflow exists
- **Not a Hopsworks replacement**: It's a companion tool
- **Not a DAG scheduler**: Compiles to Python, runs in KServe
- **Not multi-tenant**: Single workspace initially
- **Not a distributed system**: Single container execution

## Success Criteria

1. **Developer Experience**
   - Build a working agent in < 5 minutes
   - Chat can modify flows without breaking them
   - Generated code is readable and debuggable

2. **Production Quality**
   - Deploys work first time
   - Agents respond in < 200ms P95
   - Proper error handling and fallbacks
   - Point-in-time correct feature reads

3. **Integration**
   - Native Hopsworks feature access
   - Standard KServe deployment
   - Works with existing tools

## Security Considerations

- Context carries authentication
- Nodes declare resource access
- Compiled code includes validation
- Secrets never in IR
- Audit logging for data access

## Future Extensions (Not V1)

These can be added without changing core abstractions:

- Multi-agent orchestration
- Advanced caching strategies
- Schema evolution support
- Multi-tenancy
- Distributed execution
- Custom node plugins

The architecture is designed to support these without fundamental changes.