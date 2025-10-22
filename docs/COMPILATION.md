# IR to Python Compilation

## Overview

The compiler transforms the IR (Intermediate Representation) into deployable Python code for KServe. The process is:

1. **Validate** the IR for correctness
2. **Analyze** the dependency graph
3. **Generate** Python code
4. **Package** for KServe deployment

## Compilation Pipeline

```
IR (YAML/JSON)
    ↓
[Validation]
    ↓
[Dependency Analysis]
    ↓
[Code Generation]
    ↓
Python Agent Module
    ↓
[Containerization]
    ↓
KServe Deployment
```

## Generated Code Structure

### Base Template

```python
# Generated from: {flow.metadata.name}
# Generated at: {timestamp}
# IR Version: {flow.version}

import asyncio
import json
from typing import Dict, Any, Optional
from datetime import datetime
import hsfs
from openai import AsyncOpenAI
from kserve import Model, ModelServer
import logging

logger = logging.getLogger(__name__)

class Context:
    """Execution context that flows through the graph"""
    def __init__(self, request: Dict[str, Any]):
        self.user_id = request.get('user_id')
        self.session_id = request.get('session_id')
        self.trace_id = request.get('trace_id', generate_trace_id())
        self.event_time = request.get('event_time', datetime.now().isoformat())
        self.auth = request.get('auth')
        self._data = request

    def get(self, key: str, default=None):
        return self._data.get(key, default)


class StateManager:
    """Manages state across request and session scopes"""
    def __init__(self, scope: str, key_template: str):
        self.scope = scope
        self.key_template = key_template
        self._store = {}  # In-memory for v1, Redis later

    async def get(self, key: str, default=None):
        return self._store.get(key, default)

    async def set(self, key: str, value: Any):
        self._store[key] = value


class {AgentClassName}(Model):
    """Generated agent from IR flow: {flow.metadata.name}"""

    def __init__(self):
        super().__init__("{flow.metadata.name}")
        self.ready = False

    def load(self):
        """Initialize connections and resources"""
        # Hopsworks connection
        self.hsfs_conn = hsfs.connection()
        self.fs = self.hsfs_conn.get_feature_store()

        # LLM client
        self.llm_client = AsyncOpenAI()

        # State managers for stateful nodes
        {state_manager_init}

        self.ready = True

    async def predict(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Main execution entry point"""
        if not self.ready:
            raise RuntimeError("Model not loaded")

        # Create execution context
        ctx = Context(request)

        # Execute flow
        try:
            {flow_execution}
            return result
        except Exception as e:
            logger.error(f"Flow execution failed: {e}")
            return await self._handle_error(e, ctx)

    {node_methods}

    async def _handle_error(self, error: Exception, ctx: Context) -> Dict[str, Any]:
        """Global error handler"""
        return {
            'error': str(error),
            'trace_id': ctx.trace_id,
            'timestamp': ctx.event_time
        }


if __name__ == "__main__":
    model = {AgentClassName}()
    ModelServer().start([model])
```

### Node Method Generation

Each node becomes an async method:

```python
async def node_{node_id}(self, inputs: Dict[str, Any], ctx: Context) -> Dict[str, Any]:
    """Generated from node: {node.id}"""

    # Runtime configuration
    timeout = {node.runtime.timeout_ms} / 1000 if {node.runtime.timeout_ms} else 30
    retries = {node.runtime.retries} or 1

    # State access (if configured)
    {state_access_code}

    # Retry loop
    for attempt in range(retries):
        try:
            # Timeout wrapper
            result = await asyncio.wait_for(
                self._execute_{node_id}(inputs, ctx, state),
                timeout=timeout
            )

            # Validation
            {output_validation}

            return result

        except asyncio.TimeoutError:
            if attempt == retries - 1:
                raise
            await asyncio.sleep({retry_backoff})
        except Exception as e:
            if attempt == retries - 1:
                if self.has_error_handler_{node_id}:
                    return {'error': {'reason': str(e), 'node': '{node_id}'}}
                raise
            await asyncio.sleep({retry_backoff})

async def _execute_{node_id}(self, inputs: Dict[str, Any], ctx: Context, state: Optional[StateManager]) -> Dict[str, Any]:
    """Core execution logic for {node_id}"""
    {node_execution_logic}
```

## Declarative Operation Compilation

### Hopsworks Operations

```python
# hopsworks.read_features compiles to:
async def _execute_{node_id}(self, inputs, ctx, state):
    fg = self.fs.get_feature_group(
        name="{feature_group_name}",
        version={version}
    )

    # Point-in-time read if as_of specified
    if "{as_of}" in ["{context.event_time}"]:
        df = fg.read(
            time_travel_format="HUDI",
            as_of=ctx.event_time
        )
    else:
        df = fg.read()

    # Filter by key
    key_value = inputs.get('{key_field}') or ctx.get('{key_field}')
    result = df[df['{key_column}'] == key_value]

    return {'customer_data': result.to_dict('records')[0]}
```

### LLM Operations (Prompt Layer)

```python
# Node with prompt layer compiles to:
async def _execute_{node_id}(self, inputs, ctx, state):
    # Prepare prompt from template
    from jinja2 import Template

    template = Template('''{node.prompt.template}''')
    user_prompt = template.render(**inputs)

    # LLM call with structured output
    response = await self.llm_client.chat.completions.create(
        model="{node.prompt.model.name}",
        messages=[
            {"role": "system", "content": '''{node.prompt.system}'''},
            {"role": "user", "content": user_prompt}
        ],
        temperature={node.prompt.temperature or 0.7},
        response_format={
            "type": "json_schema",
            "json_schema": {node.prompt.output_schema}
        } if {node.prompt.output_schema} else None
    )

    # Parse response
    content = response.choices[0].message.content
    if {node.prompt.output_schema}:
        result = json.loads(content)
    else:
        result = {'response': content}

    return result
```

### Code Transform Compilation

```python
# Code transforms are wrapped with proper context
async def _execute_{node_id}(self, inputs, ctx, state):
    # User-provided code
    {node.transform.spec}

    # Execute the transform function
    return await transform(inputs, ctx, state)
```

## Flow Execution Generation

The main flow is generated from the dependency graph:

```python
# Topologically sorted execution
async def predict(self, request):
    ctx = Context(request)

    # Initialize node outputs
    outputs = {}

    # Execute nodes in dependency order
    # Node: fetch_customer
    inputs_fetch_customer = {
        'user_id': request.get('user_id')
    }
    outputs['fetch_customer'] = await self.node_fetch_customer(inputs_fetch_customer, ctx)

    # Node: get_conversation (parallel with fetch_customer)
    inputs_get_conversation = {
        'session_id': request.get('session_id')
    }
    outputs['get_conversation'] = await self.node_get_conversation(inputs_get_conversation, ctx)

    # Node: generate_response (depends on both above)
    inputs_generate_response = {
        'query': request.get('query'),
        'customer_data': outputs['fetch_customer']['customer_data'],
        'history': outputs['get_conversation']['history']
    }
    outputs['generate_response'] = await self.node_generate_response(inputs_generate_response, ctx)

    # Check for errors
    if 'error' in outputs['generate_response']:
        return await self.node_error_handler(
            outputs['generate_response']['error'], ctx
        )

    # Continue with success path...
    return outputs['format_response']['final_response']
```

## Optimization Strategies

### 1. Parallel Execution

Nodes without dependencies execute in parallel:

```python
# Parallel execution for independent nodes
results = await asyncio.gather(
    self.node_fetch_customer(inputs_1, ctx),
    self.node_get_conversation(inputs_2, ctx),
    return_exceptions=True
)
```

### 2. Lazy Evaluation

Only execute nodes needed for the output:

```python
# Skip nodes not in the path to output
if not self._is_needed_for_output(node_id):
    return None
```

### 3. Caching

Cache results for expensive operations:

```python
# Simple in-memory cache
cache_key = f"{node_id}:{hash(inputs)}:{ctx.event_time}"
if cache_key in self._cache:
    return self._cache[cache_key]

result = await self._execute_node(inputs, ctx)
self._cache[cache_key] = result
return result
```

## Container Packaging

### Dockerfile Template

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy generated agent
COPY agent.py .
COPY config.yaml .

# KServe expects this
ENV MODEL_NAME={flow.metadata.name}

# Run the server
CMD ["python", "agent.py"]
```

### Requirements Generation

```python
# Generated requirements.txt based on IR analysis
kserve>=0.11.0
hsfs>=3.4.0
openai>=1.0.0
httpx>=0.25.0
pydantic>=2.0.0
jinja2>=3.1.0
redis>=5.0.0  # If using Redis state backend
```

## Validation During Compilation

### 1. Static Validation

Before code generation:

```python
def validate_ir(flow: AgentFlow) -> List[ValidationError]:
    errors = []

    # Check for cycles
    if has_cycles(flow.edges):
        errors.append("Flow contains cycles")

    # Check port connections
    for edge in flow.edges:
        if not are_ports_compatible(edge.from_type, edge.to_type):
            errors.append(f"Incompatible types: {edge.from} -> {edge.to}")

    # Check context requirements
    for node in flow.nodes.values():
        for ctx_field in node.runtime.requires_context or []:
            if ctx_field not in flow.context.requires:
                errors.append(f"Node {node.id} requires context.{ctx_field}")

    return errors
```

### 2. Runtime Validation

Embedded in generated code:

```python
# Output schema validation
from jsonschema import validate

if node_output_schema:
    validate(result, node_output_schema)
```

## Error Handling Compilation

### Error Port Routing

```python
# Nodes with error ports get special handling
try:
    result = await self.node_generate_response(inputs, ctx)

    if 'error' in result:
        # Route to error handler
        error_inputs = {'error': result['error']}
        result = await self.node_fallback_handler(error_inputs, ctx)

    return result
except Exception as e:
    # Unhandled errors
    if self.has_global_error_handler:
        return await self.node_global_error_handler({'error': str(e)}, ctx)
    raise
```

## Deployment Manifest Generation

### KServe InferenceService

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: {flow.metadata.name}
  namespace: {deployment.kserve.namespace}
spec:
  predictor:
    containers:
      - name: kserve-container
        image: {registry}/{image_name}:{version}
        resources:
          requests:
            cpu: {deployment.kserve.resources.cpu}
            memory: {deployment.kserve.resources.memory}
          limits:
            cpu: {deployment.kserve.resources.cpu}
            memory: {deployment.kserve.resources.memory}
    minReplicas: {deployment.kserve.min_replicas}
    maxReplicas: {deployment.kserve.max_replicas}
    scaleTarget: {deployment.kserve.target_rps}
```

## Testing Generated Code

### Unit Test Template

```python
import pytest
from unittest.mock import Mock, AsyncMock
from agent import {AgentClassName}, Context

@pytest.fixture
def agent():
    agent = {AgentClassName}()
    agent.hsfs_conn = Mock()
    agent.llm_client = AsyncMock()
    agent.load()
    return agent

@pytest.mark.asyncio
async def test_{node_id}(agent):
    inputs = {test_inputs}
    ctx = Context({test_context})

    result = await agent.node_{node_id}(inputs, ctx)

    assert 'expected_field' in result
    assert result['expected_field'] == expected_value
```

## Compilation CLI

```python
# compiler/cli.py
import click
import yaml
from .compiler import IRCompiler

@click.command()
@click.argument('ir_file', type=click.Path(exists=True))
@click.option('--output', '-o', default='agent.py')
@click.option('--validate-only', is_flag=True)
@click.option('--generate-tests', is_flag=True)
@click.option('--package', is_flag=True, help='Create Docker container')
def compile(ir_file, output, validate_only, generate_tests, package):
    """Compile IR to Python agent"""

    # Load IR
    with open(ir_file) as f:
        ir = yaml.safe_load(f)

    # Compile
    compiler = IRCompiler()

    if validate_only:
        errors = compiler.validate(ir)
        if errors:
            for error in errors:
                click.echo(f"❌ {error}", err=True)
            raise click.Abort()
        click.echo("✅ IR is valid")
        return

    # Generate code
    code = compiler.compile(ir)

    with open(output, 'w') as f:
        f.write(code)

    click.echo(f"✅ Generated {output}")

    # Generate tests
    if generate_tests:
        tests = compiler.generate_tests(ir)
        with open(f"test_{output}", 'w') as f:
            f.write(tests)

    # Package for deployment
    if package:
        compiler.package(ir, output)
        click.echo("✅ Created Docker container")
```

## Performance Considerations

1. **Async Everything**: All node executions are async for non-blocking I/O
2. **Connection Pooling**: Reuse Hopsworks and LLM connections
3. **Batch Operations**: Combine multiple feature reads when possible
4. **Streaming**: Support streaming responses for LLM nodes
5. **Resource Limits**: Set memory and CPU limits in KServe config

## Future Enhancements

- **Incremental Compilation**: Only recompile changed nodes
- **Source Maps**: Map generated code back to IR nodes
- **Hot Reload**: Update deployed agents without downtime
- **Compilation Cache**: Cache generated code for unchanged IR
- **Multi-Language**: Generate TypeScript/Go agents