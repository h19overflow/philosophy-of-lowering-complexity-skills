---
name: module-boundary-design
description: "Decide whether a software boundary should exist, evaluate whether it hides enough complexity to justify its cost, and determine if an architectural framework (like LangGraph, Temporal, Celery, or Kafka) is genuinely warranted. Use when creating classes, modules, services, repositories, managers, adapters, interfaces, protocols, public APIs, or graph nodes."
---

# Module Boundary Design: Earning Every Abstraction

Based on John Ousterhout's *A Philosophy of Software Design* (Chapters 4–7, 9), this skill provides the decision framework for placing boundaries in code, designing deep modules, and evaluating architectural frameworks.

---

## Core Philosophy

A software boundary is not free. Every boundary introduces:
- New names, types, and files to learn.
- Interface overhead and indirection.
- Potential change amplification across boundaries.
- Cognitive burden on future engineers.

> **Caller expresses intent. Module owns mechanics.**
> **Every boundary and framework must earn its existence by hiding meaningful complexity.**

Never split code simply because:
- "The file or function is long."
- "Steps execute in chronological order."
- "Clean code says everything deserves a class or service."
- "The problem vocabulary sounds like an agent or workflow."

---

## The 6-Question Boundary Workflow

For every proposed class, module, service, repository, interface, or graph node, answer these six questions:

```text
┌─────────────────────────────────────────────────────────────┐
│ 1. Owned Knowledge      → What secrets does it own?         │
│ 2. Information Hiding   → What can callers stop knowing?    │
│ 3. Abstraction Depth    → Hidden complexity / interface?    │
│ 4. Independent Thought  → Can both sides be understood alone│
│ 5. Present Value        → Real logical/operational value?   │
│ 6. Concept Cost         → Does the new concept pay its way? │
└─────────────────────────────────────────────────────────────┘
```

### Question 1 — What Knowledge Does This Component Own?
Clearly articulate the hidden knowledge owned by this component:
- **Policy**: Business rules, retry budgets, fallback sequences, rate limits.
- **Invariants**: State transition rules, validity constraints, safety checks.
- **Infrastructure**: SQL queries, wire protocols, vendor SDKs, connection pools.
- **Representation**: Data layouts, internal schemas, normalization logic.
- **Algorithms**: Ranking fusion, token budgeting, chunking heuristics.
- **Lifecycle**: Resource allocation, cleanup, transaction commit/rollback.

*Rule: If you cannot articulate specific knowledge or invariants this boundary uniquely owns, challenge the boundary.*

### Question 2 — What Does It Hide?
Ask: **What can callers stop knowing because this abstraction exists?**

- **Strong Abstraction**:
  - `KnowledgeStore.search_hybrid(tenant_id, query, limit)` hides pgvector cosine distance `<=>`, PostgreSQL tsvectors, HNSW indexes, SQL queries, and Reciprocal Rank Fusion math. Callers express pure search intent.
  - `ModelGateway.invoke(messages)` hides OpenAI/Anthropic SDKs, asyncio timeouts, exponential backoff with jitter, HTTP 429 translation, and telemetry. Callers express pure model execution intent.
- **Weak / Shallow Abstraction**:
  ```python
  # SHALLOW: Caller learns two names for one thing. Hides nothing.
  class DocumentService:
      def __init__(self, repo: DocumentRepository):
          self.repo = repo
      def get_document(self, doc_id: str):
          return self.repo.get(doc_id)
  ```
  *Rule: If callers still need to know how the implementation works, or if the interface exposes nearly everything it wraps, the abstraction is shallow. Delete the wrapper.*

### Question 3 — Is It Deep Enough?
Estimate the depth ratio:

$$\text{Depth Ratio} = \frac{\text{Hidden Implementation Complexity}}{\text{Exposed Interface Complexity}}$$

- **Deep Module**: Small interface surface, substantial hidden implementation. Pushes mechanical complexity downward.
- **Shallow Module**: Wide interface relative to tiny implementation. Disperses complexity across callers.

*Watch for:*
- **Pass-through methods**: Forwarding arguments without adding policy, validation, normalization, resilience, or caching.
- **Layer chains**: `Controller -> Service -> Manager -> Repository -> Database`. Collapse layers that do not own distinct domain policy.

### Somewhat General-Purpose Interfaces (Level 2)
Avoid both extremes:
- **Level 1: Overly Specialized**: The interface is tied to one caller's immediate screen, button, or loop. It leaks the caller's transient workflow into the module.
- **Level 2: Somewhat General-Purpose (The Target)**: The interface represents fundamental domain operations (e.g. `delete_range(pos, len)` instead of `backspace()` and `delete_selection()`), while the implementation is focused strictly on current concrete requirements.
- **Level 3: Over-Generalized Framework**: Speculative generics, dynamic registries, and plugin hooks for hypothetical requirements (YAGNI).

### Strategic Programming vs. Tactical Shortcuts
- **Tactical Mindset**: Minimizes immediate diff size at the expense of long-term maintainability (adding boolean flags to callers, copy-pasting queries, patching around symptoms).
- **Strategic Mindset**: Invests small, bounded effort to place behavior in its natural domain owner, solve root causes, and keep boundaries clean.
- **The Ponytail Balance**: Strategic programming is *not* an invitation to over-engineer. Design carefully for requirements we understand today; reject speculative architecture for imagined future needs.

### Question 4 — Can Both Sides Be Understood Independently?
- Can a caller use this component effectively by reading only its interface documentation?
- Can an engineer safely modify the internal implementation without inspecting every call site?

- **Interface vs Implementation Comments**: Interface documentation tells users *what* the module provides and *how to use it*; implementation comments tell maintainers *how it works*. If interface comments must explain internal mechanics to be usable, or if callers must read the code to use the interface, the abstraction is leaky or shallow.
- **Types and Names Over Comments**: Encode units (`timeoutMs`, `Duration`), boundaries (`startInclusive`, `endExclusive`), and constraints in types and names rather than prose comments. Comments only earn their place when the code genuinely cannot carry the information (why not the obvious thing, cross-module invariants, or historical context).
*Rule: If understanding Component A requires reading Component B, they are conjoined. Separating them scatters complexity. Keep them together.*

### Question 5 — Does the Boundary Create Present Value?
Every boundary must justify itself through one or both of these present values:

#### A. Logical Value
- **Information Hiding**: Shielding callers from volatile mechanics or vendor details.
- **Domain Semantic Compression**: Combining low-level steps into a meaningful domain operation.
- **Change Locality**: Isolating likely future edits to a single file.

#### B. Operational Value
- **Durable State / Checkpoints**: Persisting progress across crashes or restarts.
- **Failure & Security Isolation**: Containing errors or establishing trust boundaries.
- **Concurrency & Asynchrony**: Managing parallel tasks, worker pools, or background jobs.
- **Independent Retries**: Retrying a flaky network call without re-running the entire workflow.
- **Human Approval**: Suspending execution for out-of-band authorization.

*Rule: If a boundary provides neither logical nor operational value today, reject it.*

### Question 6 — What Is the Concept Cost?
Every new type, interface, configuration knob, and file spends cognitive budget.
Choose the lowest-complexity outcome:
1. **Keep Inline**: Single-use logic that reads naturally in the parent workflow.
2. **Keep Together**: Cohesive logic sharing state or invariants in one module.
3. **Extract Function**: Pure calculation or reusable helper with clean parameter boundaries.
4. **Create Deep Module**: Encapsulate multi-step subsystem behind an intent-facing API.
5. **Shared Primitive**: Extract a lower-level value object (e.g. `SourcePosition`, `TraceContext`) when concepts interact.
6. **Remove Wrapper**: Eliminate shallow passthrough services, managers, or repositories.

---

## Architectural Framework Evaluation

Frameworks impose heavy architectural gravity. When considering:
- **LangGraph**, **Temporal**, **Celery**, **Kafka**, **event buses**, **DI frameworks**, **workflow engines**:

Ask: **What runtime or architectural semantic does this framework provide that ordinary application code cannot?**

### Evaluating LangGraph & Agent Orchestration

```text
               Deterministic Sequence?
             (analyze -> retrieve -> tool -> synthesize)
                         │
        ┌────────────────┴────────────────┐
        ▼                                 ▼
   No cycles, no pauses,             Needs: cycles, dynamic
   no durable checkpoints?           replanning, durable checkpoints,
        │                            human suspension, independent retry?
        ▼                                 ▼
Ordinary Python / TypeScript        LangGraph Justified
(Async function or simple class)     (Operational semantics earn cost)
```

- **LangGraph is justified when requirements demand:**
  1. **Cyclic Execution**: ReAct loops where the model dynamically determines the next step and may loop back based on observations.
  2. **Durable Checkpointing**: Persisting execution state to survive process restarts or pod eviction.
  3. **Human-in-the-Loop Suspension**: Pausing graph execution indefinitely until external user input or confirmation arrives.
  4. **Dynamic Multi-Agent Routing**: Autonomous message exchange between non-deterministic agents.
- **LangGraph is NOT justified when:**
  - The execution is a deterministic sequence or fixed DAG.
  - The system is called an "agent" merely because it calls an LLM.
  - Streaming responses force you to bypass the compiled graph to invoke methods manually.
  - Routing is handled by static regex or hardcoded heuristics.

*Rule: Judge semantics, not topology. A DAG may justify a workflow engine if it requires durable checkpoints and distributed retries; a graph without operational needs belongs in ordinary procedural code.*

### Evaluating Event-Driven Architectures & Pub/Sub
Event-driven code inverts control flow and scatters the call graph across runtime message handlers (e.g. Spring `@EventListener`, Kafka topics, Redis streams). Reading source code alone no longer reveals what executes next.
- **Event-driven decoupling is justified when:**
  1. **Autonomous Lifecycles**: Publishers and subscribers run on independent failure domains, deployment schedules, or scaling tiers.
  2. **True Asynchrony**: The publisher's response must not block on subscriber processing.
  3. **One-to-Many Fanout**: Multiple independent systems consume the same business fact without the producer coupling to them.
- **Event-driven decoupling is NOT justified when:**
  - The flow is a linear sequence where ordering matters (e.g., checkout steps, validation -> processing).
  - The subscriber must succeed for the producer's transaction to be valid.
  - An in-memory event bus is used merely to avoid calling a method directly.
- **Mandatory Tracing Invariant**:
  Any event-driven system must carry distributed trace context (`traceId`, `correlationId`). An untraced event-driven system is unobservable and unmaintainable.

---

## Explicit Anti-Dogma Rules

1. **Deep modules are not an excuse for god classes**: A deep module owns a coherent domain or infrastructure slice. Do not pack unrelated responsibilities into one monolithic class.
2. **Fewer files are not automatically better**: Combining unrelated concerns into one file increases cognitive load. Separate when concerns are genuinely independent.
3. **One implementation does not make an interface premature**: An abstraction with one implementation is fully justified if it hides significant current complexity (e.g., `KnowledgeStore` hiding pgvector and tsvector SQL).
4. **Abstract present complexity, not imagined variation**: Build abstractions for complexity that exists today. Do NOT build factories, registries, or strategy frameworks for hypothetical future providers.
5. **"Every boundary must earn its existence" does not mean "avoid boundaries"**: Strong, deep boundaries are essential for maintainability. The rule simply demands that boundaries provide real information hiding or operational isolation.
6. **Prefer composition over implementation inheritance**: Implementation inheritance tightly couples subclasses to parent internals and easily breaks domain invariants (the classic `Stack extends Vector` flaw where Stack exposes `.add(int, E)` and breaks LIFO). Use interface inheritance for polymorphic contracts and compose behavior.
7. **Design patterns must earn their place (Rule of Three)**: Write direct procedural branches (like a switch expression or function) first. Extract an architectural pattern (Strategy, Factory, Visitor) only when a third distinct caller or variant appears and measurably benefits.
8. **Ban shallow getters and setters**: Classes with hand-rolled getters/setters for every field hide nothing and merely widen the surface area. Use immutable records, dataclasses, or properties.

---

## Bundled References

- `references/anti-patterns.md`: Concrete boundary anti-patterns (shallow services, framework gravity, speculative factories, table-per-repository).
