# Module Boundary Anti-Pattern Catalog

This reference documents the most frequent boundary and abstraction anti-patterns produced by coding agents, with concrete code examples, failure modes, and remedies.

---

## 1. Shallow Service (Passthrough Layer)

### Anti-Pattern
A class or service that introduces an extra layer of terminology while merely forwarding calls to an underlying repository or client.

```python
# BAD: 100% indirection, zero added policy or information hiding
class DocumentService:
    def __init__(self, repository: DocumentRepository):
        self.repository = repository

    async def get_document(self, doc_id: str) -> Document | None:
        return await self.repository.get(doc_id)

    async def delete_document(self, doc_id: str) -> None:
        await self.repository.delete(doc_id)
```

### Why It Fails
- Callers must learn two names (`DocumentService` and `DocumentRepository`) for one capability.
- The interface exposes the exact same complexity as the implementation.

### Remedy
- Call the underlying component directly, OR
- Move real business ownership (tenant verification, validation, audit logging, lifecycle management) into the service and make the repository internal.

---

## 2. Framework Gravity (Badge Architecture)

### Anti-Pattern
Introducing a heavy orchestration framework (e.g. LangGraph, Temporal, Celery) simply because the problem domain mentions "agents" or "workflows", even though the execution is a deterministic pipeline.

```python
# BAD: Wrapping a deterministic 4-step sequence in a StateGraph
workflow = StateGraph(AgentState)
workflow.add_node("analyze", self._analyze)
workflow.add_node("retrieve", self._retrieve)
workflow.add_node("execute_tools", self._tools)
workflow.add_node("synthesize", self._synthesize)
workflow.add_edge(START, "analyze")
workflow.add_edge("analyze", "retrieve")
workflow.add_edge("retrieve", "execute_tools")
workflow.add_edge("execute_tools", "synthesize")
workflow.add_edge("synthesize", END)
```

### Why It Fails
- Imposes global state dictionaries (`AgentState` passing 10+ fields).
- Bypassed as soon as real-world I/O (like SSE token streaming) conflicts with graph compilation.
- Increases cognitive load without providing cycles, checkpoints, or human pauses.

### Remedy
Write a straightforward 35-line procedural async method or class:
```python
# GOOD: Clean, readable, debuggable procedural orchestration
async def run_turn(tenant_id: str, query: str) -> Answer:
    intent = detect_intent(query)
    evidence = await knowledge_store.search_hybrid(tenant_id, intent.search_query)
    packed = ContextPacker.pack(evidence)
    tools = await execute_tools(intent.tools_needed)
    return await model_gateway.synthesize(intent, packed, tools)
```

---

## 3. Speculative Provider Factories

### Anti-Pattern
Building multi-provider interfaces, factories, and registries when only one provider is currently used.

```python
# BAD: Speculative multi-provider machinery for a single LLM client
class IModelProvider(ABC):
    @abstractmethod
    async def generate(self, prompt: str) -> str: ...

class OpenAIProvider(IModelProvider): ...
class AnthropicProvider(IModelProvider): ...
class BedrockProvider(IModelProvider): ...

class ModelProviderFactory:
    @classmethod
    def get_provider(cls, name: str) -> IModelProvider: ...
```

### Why It Fails
- Adds 5 files and 150 lines of boilerplate for hypothetical requirements.
- The hypothetical abstraction frequently breaks when a real second provider is introduced because provider streaming and tool calling semantics differ.

### Remedy
Create a single deep `ModelGateway` that wraps the active provider and handles present complexity (retries, backoff, timeouts, error translation). Add a second provider only when required.

---

## 4. Table-per-Repository Ceremony

### Anti-Pattern
Mechanically creating one repository class and interface for every database table.

```python
# BAD: 4 shallow classes mirroring SQL tables
class DocumentRepository: ...
class DocumentChunkRepository: ...
class MessageRepository: ...
class IncidentRepository: ...
```

### Why It Fails
- Documents and chunks are part of the same domain aggregate. Manipulating chunks outside the document's tenant boundary causes consistency bugs.
- Forces callers to coordinate multi-table transactions across separate repository instances.

### Remedy
Organize storage around deep domain stores (e.g. `KnowledgeStore` owning documents and chunks atomically; `ConversationStore` owning sessions and messages).

---

## 5. Context-Bag State Plumbing

### Anti-Pattern
Passing an untyped or weakly typed global dictionary through every layer of a workflow just so downstream steps can access early variables.

```python
# BAD: Giant dictionary passing through all steps
async def step_3_tools(state: dict[str, Any]) -> dict[str, Any]:
    query = state["user_query"]
    service = state["service_to_check"]
    ...
```

### Why It Fails
- Readers cannot tell which fields are read, written, or optional without reading every node's source code.
- Eliminates IDE auto-complete and compile-time type safety.

### Remedy
Pass explicit, typed arguments between functional steps or encapsulate state inside a cohesive coordinator class.

---

## 6. Tiny Contextual Logger Classes

### Anti-Pattern
Creating single-use logger wrappers around standard telemetry.

```python
# BAD: Trivial wrappers relocating one call site
class RetrievalTimeoutLogger:
    def log_timeout(self, query: str):
        telemetry.event("retrieval_timeout", query=query)
```

### Remedy
Call `telemetry.event("retrieval_timeout", query=query)` directly at the call site. Abstract the infrastructure of telemetry (tracing, PII redaction, transport), not the business event naming.
