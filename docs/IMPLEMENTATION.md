# Implementation Plan

## Tech Stack

### Frontend
- **Next.js 14** (App Router) - React framework
- **ReactFlow** - Node editor component
- **Monaco Editor** - Code editor (VSCode's editor)
- **tailwind-quartz** - Your custom Tailwind variant
- **Zustand** - State management for IR
- **Zod** - Runtime validation of IR

### Backend
- **FastAPI** - API framework (simpler than Django for this)
- **Pydantic** - Data validation and IR schema
- **Jinja2** - Template engine for code generation
- **HSFS** - Hopsworks Feature Store SDK
- **OpenAI/Anthropic SDK** - LLM integrations

### Infrastructure
- **Docker** - Containerization
- **KServe** - Model serving on Kubernetes
- **Redis** - Session state storage (optional)
- **PostgreSQL** - Metadata storage (optional)

## Project Structure

```
hopsworks-brewer/
├── packages/
│   ├── web/                    # Next.js frontend
│   │   ├── app/
│   │   │   ├── page.tsx       # Main editor page
│   │   │   ├── api/           # API routes if needed
│   │   │   └── layout.tsx
│   │   ├── components/
│   │   │   ├── FlowEditor.tsx
│   │   │   ├── ChatPanel.tsx
│   │   │   ├── CodeView.tsx
│   │   │   ├── NodePalette.tsx
│   │   │   └── NodeInspector.tsx
│   │   ├── lib/
│   │   │   ├── ir-store.ts    # Zustand store
│   │   │   ├── ir-schema.ts   # Zod schemas
│   │   │   ├── chat-client.ts
│   │   │   └── compiler-client.ts
│   │   └── styles/
│   │       └── tailwind-quartz.css
│   │
│   ├── compiler/               # Python backend
│   │   ├── api/
│   │   │   ├── main.py        # FastAPI app
│   │   │   ├── routes.py
│   │   │   └── websocket.py   # Real-time sync
│   │   ├── core/
│   │   │   ├── ir_schema.py   # Pydantic models
│   │   │   ├── compiler.py    # IR → Python
│   │   │   ├── validator.py
│   │   │   └── graph.py       # DAG analysis
│   │   ├── templates/
│   │   │   ├── agent.py.jinja2
│   │   │   ├── node.py.jinja2
│   │   │   └── kserve.yaml.jinja2
│   │   └── operations/
│   │       ├── hopsworks.py   # Hopsworks ops
│   │       ├── llm.py         # LLM ops
│   │       └── transform.py   # Data ops
│   │
│   └── shared/                 # Shared types/schemas
│       ├── ir-types.ts
│       └── ir_types.py
│
├── examples/
│   ├── flows/
│   │   ├── customer-service.yaml
│   │   ├── churn-predictor.yaml
│   │   └── recommendation.yaml
│   └── generated/
│       └── agents/
│
├── docs/                       # Documentation
├── scripts/                    # Build/deploy scripts
└── tests/
    ├── frontend/
    ├── compiler/
    └── e2e/
```

## Implementation Phases

### Phase 1: Foundation (Week 1)

#### Day 1-2: Project Setup
```bash
# Frontend setup
npx create-next-app@14 packages/web --typescript --tailwind --app
cd packages/web
npm install reactflow zustand zod @monaco-editor/react

# Backend setup
cd packages/compiler
poetry init
poetry add fastapi pydantic jinja2 pyyaml click uvicorn
poetry add --dev pytest pytest-asyncio black ruff

# Shared schemas
# Create TypeScript and Python versions of IR schema
```

#### Day 3-4: Core IR Implementation
```python
# packages/compiler/core/ir_schema.py
from pydantic import BaseModel, Field
from typing import Dict, List, Optional, Literal
from enum import Enum

class PortType(str, Enum):
    STRING = "string"
    NUMBER = "number"
    BOOLEAN = "boolean"
    OBJECT = "object"
    ARRAY = "array"
    DOCUMENT = "document"
    TABLE = "table"

class PortSchema(BaseModel):
    name: str
    type: PortType
    schema: Optional[Dict] = None
    optional: bool = False

class Transform(BaseModel):
    mode: Literal["declarative", "code"]
    spec: Union[str, Dict]

class PromptLayer(BaseModel):
    system: Optional[str]
    template: str
    model: Optional[ModelConfig]
    output_schema: Optional[Dict]

class Node(BaseModel):
    id: str
    type: Literal["transform"] = "transform"
    inputs: List[PortSchema]
    outputs: List[PortSchema]
    error: Optional[PortSchema]
    prompt: Optional[PromptLayer]
    transform: Transform
    runtime: Optional[RuntimeConfig]
    state: Optional[StateConfig]
    position: Optional[Dict[str, float]]

class AgentFlow(BaseModel):
    version: Literal["1.0"] = "1.0"
    metadata: FlowMetadata
    context: ContextRequirements
    nodes: Dict[str, Node]
    edges: List[Edge]
```

#### Day 5: Basic Compiler
```python
# packages/compiler/core/compiler.py
class IRCompiler:
    def __init__(self):
        self.template_env = Environment(loader=FileSystemLoader('templates'))

    def compile(self, flow: AgentFlow) -> str:
        # Validate
        errors = self.validate(flow)
        if errors:
            raise CompilationError(errors)

        # Analyze graph
        graph = self.build_dependency_graph(flow)
        execution_order = self.topological_sort(graph)

        # Generate code
        template = self.template_env.get_template('agent.py.jinja2')
        return template.render(
            flow=flow,
            execution_order=execution_order,
            nodes=flow.nodes,
            timestamp=datetime.now()
        )
```

### Phase 2: UI Development (Week 1-2)

#### Day 1-2: Flow Editor
```tsx
// packages/web/components/FlowEditor.tsx
import ReactFlow, { Node, Edge, useNodesState, useEdgesState } from 'reactflow'
import { useIRStore } from '@/lib/ir-store'

export function FlowEditor() {
  const { ir, updateNode, addEdge } = useIRStore()
  const [nodes, setNodes, onNodesChange] = useNodesState([])
  const [edges, setEdges, onEdgesChange] = useEdgesState([])

  // Sync IR to ReactFlow
  useEffect(() => {
    setNodes(irToReactFlowNodes(ir.nodes))
    setEdges(irToReactFlowEdges(ir.edges))
  }, [ir])

  return (
    <ReactFlow
      nodes={nodes}
      edges={edges}
      onNodesChange={onNodesChange}
      onEdgesChange={onEdgesChange}
      onConnect={handleConnect}
      nodeTypes={nodeTypes}
    >
      <Background />
      <Controls />
    </ReactFlow>
  )
}
```

#### Day 3: IR State Management
```typescript
// packages/web/lib/ir-store.ts
import { create } from 'zustand'
import { AgentFlow, Node } from '@/shared/ir-types'

interface IRStore {
  ir: AgentFlow
  setIR: (ir: AgentFlow) => void
  addNode: (node: Node) => void
  updateNode: (id: string, updates: Partial<Node>) => void
  deleteNode: (id: string) => void
  addEdge: (edge: Edge) => void
  deleteEdge: (id: string) => void
}

export const useIRStore = create<IRStore>((set) => ({
  ir: createEmptyFlow(),
  setIR: (ir) => set({ ir }),
  addNode: (node) => set((state) => ({
    ir: {
      ...state.ir,
      nodes: { ...state.ir.nodes, [node.id]: node }
    }
  })),
  // ... other methods
}))
```

#### Day 4-5: Code View & Chat Panel
```tsx
// packages/web/components/CodeView.tsx
import MonacoEditor from '@monaco-editor/react'
import { useIRStore } from '@/lib/ir-store'
import yaml from 'js-yaml'

export function CodeView() {
  const { ir, setIR } = useIRStore()
  const [code, setCode] = useState('')

  useEffect(() => {
    setCode(yaml.dump(ir))
  }, [ir])

  const handleCodeChange = (value: string | undefined) => {
    try {
      const parsed = yaml.load(value || '')
      // Validate with Zod
      const validated = AgentFlowSchema.parse(parsed)
      setIR(validated)
    } catch (e) {
      // Show validation errors
    }
  }

  return (
    <MonacoEditor
      height="100%"
      language="yaml"
      value={code}
      onChange={handleCodeChange}
      options={{
        minimap: { enabled: false },
        fontSize: 14,
      }}
    />
  )
}
```

### Phase 3: Chat Intelligence (Week 2)

#### Day 1-2: Chat Backend
```python
# packages/compiler/api/chat.py
from openai import AsyncOpenAI
from .prompts import SYSTEM_PROMPT

class ChatAgent:
    def __init__(self):
        self.client = AsyncOpenAI()

    async def process_command(self, command: str, ir: AgentFlow) -> AgentFlow:
        # Build context
        context = f"""
        Current IR:
        {ir.model_dump_json(indent=2)}

        Available operations:
        - Add node: hopsworks.read_features, llm.generate, transform.filter
        - Connect nodes: Create edges between compatible ports
        - Modify node: Update properties, add prompt layer
        """

        response = await self.client.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": f"{context}\n\nCommand: {command}"}
            ],
            response_format={"type": "json_object"}
        )

        # Parse and validate modified IR
        modified = AgentFlow.model_validate_json(response.content)
        return modified
```

#### Day 3: WebSocket Sync
```python
# packages/compiler/api/websocket.py
from fastapi import WebSocket
import json

class ConnectionManager:
    def __init__(self):
        self.connections: List[WebSocket] = []

    async def connect(self, websocket: WebSocket):
        await websocket.accept()
        self.connections.append(websocket)

    async def broadcast_ir_update(self, ir: AgentFlow):
        message = json.dumps({"type": "ir_update", "data": ir.dict()})
        for connection in self.connections:
            await connection.send_text(message)

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await manager.connect(websocket)
    try:
        while True:
            data = await websocket.receive_text()
            message = json.loads(data)

            if message["type"] == "chat_command":
                ir = AgentFlow.parse_obj(message["ir"])
                modified = await chat_agent.process_command(
                    message["command"], ir
                )
                await manager.broadcast_ir_update(modified)
    except WebSocketDisconnect:
        manager.disconnect(websocket)
```

### Phase 4: Hopsworks Integration (Week 2-3)

#### Day 1-2: Feature Store Operations
```python
# packages/compiler/operations/hopsworks.py
import hsfs

class HopsworksOperations:
    @staticmethod
    def compile_read_features(node: Node, args: Dict) -> str:
        """Generate code for reading features"""
        return f"""
fg = self.fs.get_feature_group(
    name='{args["feature_group"]}',
    version={args.get("version", 1)}
)

# Point-in-time read
df = fg.read(
    time_travel_format="HUDI",
    as_of=ctx.event_time
)

# Filter by key
result = df[df['{args["key_column"]}'] == inputs['{args["key"]}']]
return {{'features': result.to_dict('records')[0]}}
"""
```

#### Day 3: Deployment Pipeline
```python
# packages/compiler/deploy/kserve.py
class KServeDeployer:
    def __init__(self, namespace: str):
        self.namespace = namespace
        self.k8s_client = kubernetes.client.ApiClient()

    def deploy(self, agent_code: str, flow: AgentFlow):
        # Build Docker image
        image = self.build_container(agent_code, flow)

        # Create InferenceService
        inference_service = self.create_inference_service(flow, image)

        # Deploy
        serving_client = kserve.KServeClient()
        serving_client.create(inference_service, namespace=self.namespace)

    def build_container(self, agent_code: str, flow: AgentFlow) -> str:
        # Generate Dockerfile
        dockerfile = self.generate_dockerfile(flow)

        # Build and push
        image_name = f"agent-{flow.metadata.name}:{version}"
        # ... Docker build logic
        return image_name
```

### Phase 5: Testing & Polish (Week 3)

#### Testing Strategy
```python
# tests/compiler/test_compiler.py
import pytest
from compiler.core import IRCompiler, AgentFlow

@pytest.fixture
def sample_flow():
    return AgentFlow.parse_file("examples/customer-service.yaml")

def test_compilation(sample_flow):
    compiler = IRCompiler()
    code = compiler.compile(sample_flow)

    # Check generated code
    assert "class CustomerServiceAgent" in code
    assert "async def predict" in code
    assert "hsfs.connection()" in code

def test_parallel_execution(sample_flow):
    compiler = IRCompiler()
    graph = compiler.build_dependency_graph(sample_flow)

    # Check that independent nodes are identified
    parallel_groups = compiler.identify_parallel_groups(graph)
    assert len(parallel_groups[0]) == 2  # fetch_customer, get_conversation
```

## Development Workflow

### Local Development
```bash
# Terminal 1: Frontend
cd packages/web
npm run dev  # http://localhost:3000

# Terminal 2: Backend
cd packages/compiler
poetry run uvicorn api.main:app --reload  # http://localhost:8000

# Terminal 3: Hopsworks Mock (for testing)
poetry run python scripts/mock_hopsworks.py
```

### Git Workflow
```bash
# Feature branches
git checkout -b feature/node-inspector
# ... make changes
git add .
git commit -m "Add node property inspector panel"
git push origin feature/node-inspector
# Create PR
```

### Testing
```bash
# Frontend tests
cd packages/web
npm test

# Backend tests
cd packages/compiler
poetry run pytest

# E2E tests
npm run test:e2e
```

## Key Implementation Decisions

### 1. State Synchronization
- **Single source of truth**: IR in Zustand store
- **Optimistic updates**: UI updates immediately, syncs to backend
- **Conflict resolution**: Last write wins, with undo support

### 2. Code Generation
- **Template-based**: Jinja2 templates for flexibility
- **Incremental**: Only regenerate changed nodes
- **Validated**: Generated code is syntax-checked

### 3. Error Handling
- **Graceful degradation**: Fallback to simpler operations
- **User feedback**: Clear error messages in UI
- **Recovery**: Auto-save and restore on crash

### 4. Performance
- **Lazy loading**: Load node definitions on demand
- **Debounced sync**: Batch IR updates to backend
- **Virtual scrolling**: For large flows

## Deployment

### Development Environment
```yaml
# docker-compose.yml
version: '3.8'
services:
  frontend:
    build: packages/web
    ports:
      - "3000:3000"
    environment:
      - API_URL=http://backend:8000

  backend:
    build: packages/compiler
    ports:
      - "8000:8000"
    environment:
      - HOPSWORKS_API_KEY=${HOPSWORKS_API_KEY}

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

### Production Deployment
```bash
# Build and push containers
docker build -t agent-builder-frontend packages/web
docker build -t agent-builder-backend packages/compiler
docker push agent-builder-frontend
docker push agent-builder-backend

# Deploy to Kubernetes
kubectl apply -f k8s/
```

## Monitoring & Observability

### Metrics to Track
- IR compilation time
- Node execution latency
- Error rates by node type
- User actions (nodes added, flows created)
- Deployment success rate

### Logging
```python
import structlog

logger = structlog.get_logger()

logger.info("compiling_flow",
    flow_id=flow.metadata.name,
    node_count=len(flow.nodes),
    edge_count=len(flow.edges)
)
```

## Security Considerations

### Input Validation
- Validate all IR modifications
- Sanitize code transforms
- Limit resource usage

### Authentication
- API key for backend access
- Hopsworks credentials encrypted
- Session management with Redis

### Code Execution
- Sandboxed compilation
- No arbitrary code execution
- Validated declarative operations

## Success Metrics

### Week 1
- [ ] IR schema complete
- [ ] Basic UI with 3 nodes working
- [ ] Simple compilation to Python

### Week 2
- [ ] Chat can modify flows
- [ ] Hopsworks integration working
- [ ] State management functional

### Week 3
- [ ] Deploy to KServe successful
- [ ] Full customer service example
- [ ] Error handling robust

### Week 4
- [ ] Performance optimized
- [ ] Documentation complete
- [ ] Ready for beta users