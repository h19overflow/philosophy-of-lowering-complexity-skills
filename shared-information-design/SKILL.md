---
name: shared-information-design
description: "Apply John Ousterhout's shared-information principle to consolidate duplicated knowledge, eliminate parallel implementation drift, and identify deeper shared primitives. Use when finding duplicated structures, synchronized state, multiple modules interpreting the same data, repeated parsing/normalization, cross-module invariants, or deciding between shared primitives and god objects."
---

# Shared Information Design: Consolidating Knowledge and Finding Primitives

Based on Chapter 9 of John Ousterhout's *A Philosophy of Software Design* ("Better Together Or Better Apart?"), this skill guides how to handle information that exists or interacts across multiple locations in a codebase.

---

## Core Philosophy

Information leakage occurs when a single design decision, policy, or invariant is reflected in multiple independent components:
- If a data format changes, multiple files must be updated.
- If a business rule changes, multiple modules risk falling out of sync.

> **Code should be brought together when multiple pieces depend on the same hidden knowledge, policy, invariant, representation, or design decision.**

However, combining code does **not** mean creating a monolithic god class. When two concepts interact or share knowledge, search for a **deeper shared primitive** that captures the common foundation without conjoining unrelated responsibilities.

---

## The 5 Knowledge Classifications

When encountering duplicated logic, data structures, or synchronized code, classify the shared information:

```text
┌─────────────────────────────────────────────────────────────┐
│ 1. Representation Knowledge  → Schema, byte layouts, offsets│
│ 2. Policy Knowledge          → Fallbacks, retries, rates    │
│ 3. Protocol Knowledge        → Wire formats, envelopes      │
│ 4. Invariant Knowledge       → State rules, safety limits   │
│ 5. Coincidental Similarity   → Accidental syntactic match   │
└─────────────────────────────────────────────────────────────┘
```

### 1. Representation Knowledge
- **Example**: Multiple modules independently know that a citation offset is encoded as `f"{doc_id}#{page}:{offset}"`.
- **Action**: Put the representation behind a single dedicated type or parser (e.g. `SourceLocation.format_location()`). No other module should parse raw coordinate strings.

### 2. Policy Knowledge
- **Example**: Three different service callers independently know the fallback sequence `["gpt-4o", "claude-3-5-sonnet", "gemini-1.5-pro"]`.
- **Action**: Move the fallback policy inside the model gateway. Callers should request capability or quality tier, not coordinate fallback sequences.

### 3. Protocol Knowledge
- **Example**: Several microservices construct identical event envelopes with timestamp, event_id, and producer metadata.
- **Action**: Encapsulate envelope serialization and validation inside a shared messaging module. Callers pass only domain payloads.

### 4. Invariant Knowledge
- **Example**: Both the checkout route and the inventory worker check `if ticket.status == "reserved" and ticket.locked_by == order_id`.
- **Action**: The `Tickets` service/module must own guarded state transitions (`ticket.release(order_id)`). External code must never perform raw state checks that enforce domain invariants.

### 5. Coincidental Similarity (False DRY)
- **Example**: Two unrelated validation loops look almost identical but one validates discount coupon codes and the other validates shipping postal codes.
- **Action**: **DO NOT COMBINE.** Combining them creates artificial coupling between two independent domains that will evolve differently.

---

## Parallel Implementation Drift

Whenever the system has parallel implementations such as:
```text
PostgreSQL / in-memory
real provider / fake provider
local storage / cloud storage
production adapter / test adapter
sync / async
```

Ask: **Are they implementing the same policy twice?**

Separate **mechanism** (I/O, storage, network) from **policy / algorithm** (ranking, validation, scoring, lifecycle transitions).

### Anti-Pattern: Duplicated Policy in Parallel Mechanisms
```text
PostgreSQL Search  ──► Contains Reciprocal Rank Fusion (RRF) formula
In-Memory Search   ──► Contains duplicate RRF formula in Python
```
*Risk*: When ranking math, boost factors, or constants are tuned in one path, the test or fallback path silently drifts.

### Remedy: Extract the Policy Function
```text
PostgreSQL Candidates ──┐
                         ├──► fuse_rankings(candidates, rrf_k=60)
In-Memory Candidates  ──┘
```

*Rule: Do NOT respond to parallel implementations by automatically building abstract class hierarchies (`AbstractRetriever`, `PostgresRetriever`, `MemoryRetriever`, `RetrieverFactory`). Prefer extracting only the pure function or shared calculation that owns the policy.*

---

## The Shared Primitive Technique

When two domain concepts frequently interact or share representation knowledge, developers often make one of two mistakes:
1. **Pass-Through Manager**: Creating a shallow third service (`CitationEvidenceManager`) that delegates to both.
2. **Coupled God Object**: Merging both concepts into one bloated class (`CitationAndEvidence`).

### The Solution: Search for the Lower-Level Primitive
Identify the fundamental shared concept upon which both concepts depend:

| Interacting Concepts | Hidden Shared Primitive | What the Primitive Encapsulates |
| :--- | :--- | :--- |
| `Citation` + `EvidenceChunk` | `SourceLocation` | File/document ID, source URI, page number, heading breadcrumb, line range. |
| `ToolCall` + `ToolResult` | `ToolCallId` | Unique execution identifier and invocation correlation. |
| `AgentTurn` + `ModelInvocation` | `TraceContext` | Distributed trace ID, span ID, parent span, sampling flag. |
| `Retriever` + `Reranker` | `ScoredCandidate` | Document ID, raw content, preliminary retrieval score. |
| `TokenLimiter` + `Chunker` | `TokenBudget` | Max tokens, counting algorithm, reserved response buffer. |

**Benefit**:
- `Citation` and `EvidenceChunk` remain completely independent domain concepts.
- Neither depends on the other.
- Both depend cleanly on `SourceLocation`, which enforces coordinate formatting once.


### Named Domain Types Over Generic Containers
Generic containers (`Pair<A, B>`, `Tuple2`, `Map.Entry<K, V>`, `tuple[str, int]`, untyped dicts/hashmaps) force callers to memorize positional indexes (`.getLeft()`, `item[0]`, `.getKey()`). The reader cannot tell what the data represents without reading upstream code.
- **Rule**: Ban generic pairs/tuples for domain data.
- **Remedy**: Define a lightweight named domain record, struct, or interface (e.g. `record UserScore(String username, int points)` or `@dataclass class UserScore`).
- **Why**: Modern languages make records/structs syntactically free. A named domain type documents intent at both declaration and call site without requiring explanatory comments.
---

## False DRY: When NOT to Combine

Duplication means **duplicated knowledge**, not merely **similar text**.

Ask:
> **"If the business requirement for Feature A changes, is it inevitable that Feature B must change in the exact same way?"**

- If **YES**: It is true shared knowledge. Consolidate it behind one owner.
- If **NO**: It is coincidental similarity. Keep the implementations separate to preserve independent evolution.

---

## Explicit Anti-Dogma Rules

1. **Multiple implementations do not automatically require inheritance**: Prefer extracting pure shared algorithms (e.g. `fuse_rankings`) over creating abstract base classes, factories, or strategy patterns.
2. **DRY applies to knowledge, not syntax**: Merging two syntactically similar 10-line functions that represent independent business concepts creates artificial coupling.
3. **Shared primitives must be small and cohesive**: A shared primitive (like `SourceLocation` or `TraceContext`) is an immutable value object, not a dumping ground for general-purpose utility methods.

---

## Bundled References

- `references/shared-primitives-guide.md`: Real-world shared primitive patterns and parallel implementation drift examples.
