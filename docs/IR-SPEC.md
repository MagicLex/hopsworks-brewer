# Intermediate Representation Specification

## Overview

The IR (Intermediate Representation) is the canonical format for agent flows. It's designed to be:
- **Human-readable**: YAML/JSON format that developers can edit directly
- **LLM-friendly**: Structured for chat-based manipulation
- **Type-safe**: Strong validation with schemas
- **Extensible**: Can add fields without breaking existing flows

## Core Schema

### Agent Flow

```typescript
interface AgentFlow {
  version: "1.0"
  metadata: FlowMetadata
  context: ContextRequirements
  nodes: Record<string, Node>
  edges: Edge[]
  deployment?: DeploymentConfig
}

interface FlowMetadata {
  name: string
  description?: string
  author?: string
  created_at?: string
  modified_at?: string
  tags?: string[]
}

interface ContextRequirements {
  requires: string[]  // Required context fields
  optional?: string[] // Optional context fields
}
```

### Node Specification

```typescript
interface Node {
  // Identity
  id: string
  type: 'transform'  // Always 'transform' for now

  // Data flow
  inputs: PortSchema[]
  outputs: PortSchema[]
  error?: PortSchema

  // Optional prompt augmentation
  prompt?: PromptLayer

  // Implementation
  transform: Transform

  // Runtime configuration
  runtime?: RuntimeConfig

  // State management
  state?: StateConfig

  // UI positioning (for visual editor)
  position?: { x: number, y: number }
}

interface PortSchema {
  name: string
  type: PortType
  schema?: JSONSchema7  // Optional JSON schema validation
  optional?: boolean
  description?: string
}

type PortType =
  | "string"
  | "number"
  | "boolean"
  | "object"
  | "array"
  | "document"  // For text/markdown
  | "table"     // For structured data
  | "embedding" // For vector representations
```

### Transform Implementation

```typescript
interface Transform {
  mode: 'declarative' | 'code'
  spec: string | DeclarativeOp
}

interface DeclarativeOp {
  op: string  // Operation name
  args: Record<string, any>  // Operation arguments
}

// Declarative operations catalog
type BuiltinOps =
  | 'hopsworks.read_features'
  | 'hopsworks.write_features'
  | 'hopsworks.get_model'
  | 'http.get'
  | 'http.post'
  | 'transform.filter'
  | 'transform.map'
  | 'transform.reduce'
  | 'io.format_response'
```

### Prompt Layer

```typescript
interface PromptLayer {
  system?: string
  template: string  // Jinja2 template
  model?: ModelConfig
  output_schema?: JSONSchema7  // For structured outputs
  temperature?: number
  max_tokens?: number
  tools?: ToolSpec[]  // For function calling
  fallback?: FallbackConfig
}

interface ModelConfig {
  provider: 'openai' | 'anthropic' | 'azure' | 'bedrock'
  name: string
  endpoint?: string  // For custom endpoints
}

interface ToolSpec {
  name: string
  description: string
  parameters: JSONSchema7
  maps_to?: string  // Node ID to route tool calls to
}

interface FallbackConfig {
  on: 'timeout' | 'error' | 'validation_fail'
  model?: ModelConfig
  static_response?: string
}
```

### Runtime Configuration

```typescript
interface RuntimeConfig {
  timeout_ms?: number      // Default: 30000
  retries?: number         // Default: 1
  retry_backoff_ms?: number // Default: 1000
  requires_context?: string[] // Context fields needed
  concurrency?: number     // Max parallel executions
}
```

### State Management

```typescript
interface StateConfig {
  scope: 'request' | 'session'
  key?: string  // State key template, default: "{node.id}"
  ttl_seconds?: number  // For session scope
  backend?: 'memory' | 'redis' | 'hopsworks'  // Default: memory
}
```

### Edge Specification

```typescript
interface Edge {
  from: string  // "node_id.port_name" or "node_id" (default output)
  to: string    // "node_id.port_name" or "node_id" (default input)
  condition?: string  // Python expression, e.g., "value > 0"
  transform?: string  // Optional inline transform
}
```

### Deployment Configuration

```typescript
interface DeploymentConfig {
  target: 'kserve' | 'lambda' | 'local'
  kserve?: {
    namespace?: string
    service_name?: string
    min_replicas?: number
    max_replicas?: number
    target_rps?: number
    resources?: {
      cpu?: string     // e.g., "1"
      memory?: string  // e.g., "2Gi"
      gpu?: string     // e.g., "1"
    }
  }
}
```

## Built-in Declarative Operations

### Hopsworks Operations

```yaml
# Read features from feature group
transform:
  mode: declarative
  spec:
    op: hopsworks.read_features
    args:
      feature_group: "customers_v3"
      feature_view: "customer_360"  # Optional
      key: "${user_id}"
      as_of: "${context.event_time}"
      features: ["age", "tenure", "churn_risk"]  # Optional selection

# Write to feature group
transform:
  mode: declarative
  spec:
    op: hopsworks.write_features
    args:
      feature_group: "predictions"
      data: "${predictions}"
      event_time: "${context.event_time}"

# Get model from model registry
transform:
  mode: declarative
  spec:
    op: hopsworks.get_model
    args:
      name: "churn_predictor"
      version: "latest"  # or specific version
```

### HTTP Operations

```yaml
transform:
  mode: declarative
  spec:
    op: http.post
    args:
      url: "https://api.service.com/endpoint"
      headers:
        Authorization: "Bearer ${context.auth.token}"
      body: "${request_data}"
      timeout_ms: 5000
```

### Data Transformations

```yaml
# Filter operation
transform:
  mode: declarative
  spec:
    op: transform.filter
    args:
      expression: "item['score'] > 0.5"
      input: "${items}"

# Map operation
transform:
  mode: declarative
  spec:
    op: transform.map
    args:
      expression: "item['value'] * 2"
      input: "${items}"
```

## Code Transform Examples

When declarative isn't enough, use code:

```yaml
transform:
  mode: code
  spec: |
    def transform(inputs, context, state):
        # Access inputs
        user_data = inputs['user_data']

        # Access context
        user_id = context.user_id

        # Access state (if configured)
        history = await state.get('history', [])

        # Custom logic
        processed = complex_business_logic(user_data)

        # Update state
        history.append(processed)
        await state.set('history', history[-10:])

        # Return outputs
        return {
            'result': processed,
            'metadata': {'processed_at': context.event_time}
        }
```

## Complete Example Flow

```yaml
version: "1.0"

metadata:
  name: customer_service_agent
  description: AI agent for customer support with context awareness
  tags: [production, customer-facing]

context:
  requires: [user_id, session_id, query]
  optional: [auth_token]

nodes:
  fetch_customer:
    type: transform
    inputs:
      - name: user_id
        type: string
    outputs:
      - name: customer_data
        type: object
        schema:
          type: object
          properties:
            age: {type: number}
            tenure: {type: number}
            churn_risk: {type: number}
    transform:
      mode: declarative
      spec:
        op: hopsworks.read_features
        args:
          feature_group: customers_360_v3
          key: "${user_id}"
          as_of: "${context.event_time}"
    runtime:
      timeout_ms: 2000
      retries: 2
    position: {x: 100, y: 100}

  get_conversation:
    type: transform
    inputs:
      - name: session_id
        type: string
    outputs:
      - name: history
        type: array
    state:
      scope: session
      key: "conversation_{session_id}"
      ttl_seconds: 3600
    transform:
      mode: code
      spec: |
        def transform(inputs, context, state):
            history = await state.get('history', [])
            return {'history': history[-5:]}  # Last 5 messages
    position: {x: 100, y: 250}

  generate_response:
    type: transform
    inputs:
      - name: query
        type: string
      - name: customer_data
        type: object
      - name: history
        type: array
    outputs:
      - name: response
        type: object
        schema:
          type: object
          properties:
            answer: {type: string}
            confidence: {type: number}
            citations: {type: array, items: {type: string}}
    error:
      name: generation_error
      schema:
        type: object
        properties:
          reason: {type: string}
          fallback_available: {type: boolean}
    prompt:
      system: |
        You are a helpful customer service agent.
        Use the customer data to provide personalized responses.
        Be concise and professional.
      template: |
        Customer Data:
        - Age: {{ customer_data.age }}
        - Tenure: {{ customer_data.tenure }} months
        - Churn Risk: {{ customer_data.churn_risk }}

        Conversation History:
        {% for msg in history %}
        - {{ msg.role }}: {{ msg.content }}
        {% endfor %}

        Customer Query: {{ query }}

        Provide a helpful response in JSON format.
      model:
        provider: openai
        name: gpt-4o-mini
      temperature: 0.3
      output_schema:
        type: object
        properties:
          answer: {type: string}
          confidence: {type: number, minimum: 0, maximum: 1}
          citations: {type: array, items: {type: string}}
        required: [answer, confidence]
      fallback:
        on: validation_fail
        static_response: |
          {"answer": "I'm having trouble understanding. Could you rephrase?", "confidence": 0}
    runtime:
      timeout_ms: 5000
      retries: 1
    position: {x: 300, y: 175}

  update_conversation:
    type: transform
    inputs:
      - name: query
        type: string
      - name: response
        type: object
    outputs:
      - name: success
        type: boolean
    state:
      scope: session
      key: "conversation_{context.session_id}"
    transform:
      mode: code
      spec: |
        def transform(inputs, context, state):
            history = await state.get('history', [])
            history.append({
                'role': 'user',
                'content': inputs['query'],
                'timestamp': context.event_time
            })
            history.append({
                'role': 'assistant',
                'content': inputs['response']['answer'],
                'timestamp': context.event_time
            })
            await state.set('history', history)
            return {'success': True}
    position: {x: 500, y: 175}

  format_response:
    type: transform
    inputs:
      - name: response
        type: object
      - name: success
        type: boolean
    outputs:
      - name: final_response
        type: object
    transform:
      mode: declarative
      spec:
        op: io.format_response
        args:
          template:
            status: success
            data: "${response}"
            session_updated: "${success}"
    position: {x: 700, y: 175}

edges:
  # Data flow
  - from: _input.user_id
    to: fetch_customer.user_id

  - from: _input.session_id
    to: get_conversation.session_id

  - from: fetch_customer.customer_data
    to: generate_response.customer_data

  - from: get_conversation.history
    to: generate_response.history

  - from: _input.query
    to: generate_response.query

  - from: _input.query
    to: update_conversation.query

  - from: generate_response.response
    to: update_conversation.response

  - from: generate_response.response
    to: format_response.response

  - from: update_conversation.success
    to: format_response.success

  - from: format_response.final_response
    to: _output

  # Error handling
  - from: generate_response.generation_error
    to: error_handler.error

deployment:
  target: kserve
  kserve:
    namespace: production
    service_name: customer-service-agent
    min_replicas: 2
    max_replicas: 10
    target_rps: 50
    resources:
      cpu: "2"
      memory: "4Gi"
```

## Validation Rules

1. **Node IDs** must be unique and match pattern `[a-z][a-z0-9_]*`
2. **Port names** must be unique within a node
3. **Edges** must connect existing nodes and compatible port types
4. **Context requirements** must be satisfied by deployment configuration
5. **Cycles** are not allowed (DAG only)
6. **Prompt templates** must be valid Jinja2
7. **Code transforms** must define a `transform` function
8. **State keys** must include scope variables (session_id, user_id, etc.)

## Type Compatibility

Port types can be connected with these rules:

| From Type | To Type | Allowed |
|-----------|---------|---------|
| string | string | ✅ |
| string | document | ✅ |
| number | number | ✅ |
| number | string | ✅ (with cast) |
| boolean | boolean | ✅ |
| boolean | string | ✅ (with cast) |
| object | object | ✅ |
| array | array | ✅ |
| document | string | ✅ |
| document | document | ✅ |
| table | object | ✅ (single row) |
| table | array | ✅ |
| embedding | array | ✅ |

## IR Manipulation by Chat

The chat assistant can manipulate the IR with commands like:

```
"Add a node that fetches user features"
→ Creates a new node with hopsworks.read_features

"Connect the customer data to the LLM"
→ Adds edge from fetch_customer.customer_data to generate_response.customer_data

"Add error handling to the response generation"
→ Adds error port and fallback node

"Make the LLM response more concise"
→ Updates the prompt.system field

"Add caching to the feature fetch"
→ Adds cache configuration to the node.runtime
```

The chat understands the complete IR structure and can make valid modifications while preserving flow integrity.