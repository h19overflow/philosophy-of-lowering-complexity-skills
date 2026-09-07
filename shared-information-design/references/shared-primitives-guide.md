# Shared Primitives & Parallel Drift Guide

This reference provides real-world patterns for identifying lower-level shared primitives and preventing parallel implementation drift.

---

## 1. The Lower-Level Shared Primitive Catalog

When two domain concepts frequently interact, avoid creating a coupled god object or a shallow manager. Search for the shared primitive:

| Domain Concepts | Coupled Anti-Pattern | Clean Shared Primitive | What the Primitive Owns |
| :--- | :--- | :--- | :--- |
| `Citation` + `EvidenceChunk` | `CitationAndEvidenceManager` | `SourceLocation` | Immutable document ID, URI, format type, page, section heading, line range. |
| `ToolCall` + `ToolResult` | `ToolExecutionCoordinator` | `ToolCallId` | Execution identifier, correlation timestamp, tool name. |
| `AgentTurn` + `ModelInvocation` | `TurnLoggingContext` | `TraceContext` | Distributed run ID, tenant ID, span hierarchy, sampling flags. |
| `Retriever` + `Reranker` | `RetrievalPipelineGodObject`| `ScoredCandidate` | Document ID, raw content, initial retrieval score, source metadata. |
| `TokenLimiter` + `Chunker` | `ChunkTokenOrchestrator` | `TokenBudget` | Character-to-token estimation, limit ceiling, reserved response buffer. |

### Characteristics of a True Shared Primitive
1. **Immutable Value Object**: Has no independent identity; two instances with the same values are identical.
2. **Zero Inbound Dependencies**: Does not import or depend on the higher-level concepts that consume it.
3. **Encapsulated Representation**: Owns formatting, parsing, and serialization of its coordinate or identity data (e.g. `SourceLocation.format_location()`).

---

## 2. Preventing Parallel Implementation Drift

### The Problem
Systems frequently maintain dual mechanisms:
- **Production vs. Test**: PostgreSQL (pgvector) vs. in-memory array search.
- **Provider vs. Mock**: Real OpenAI API client vs. deterministic mock LLM.
- **Sync vs. Async**: Interactive REST pipeline vs. background batch worker.

When business policy or mathematical algorithms are embedded inside the mechanism, the two implementations inevitably drift apart.

### The Remedy: Extract Pure Policy Functions

```python
# GOOD: Pure calculation shared by both PostgreSQL and In-Memory retrieval
def fuse_rrf_scores(
    vector_ranks: dict[str, int],
    lexical_ranks: dict[str, int],
    rrf_k: int = 60,
) -> dict[str, float]:
    """Pure mathematical rank fusion. Independent of database or storage."""
    combined_scores: dict[str, float] = {}
    for item_id, rank in vector_ranks.items():
        combined_scores[item_id] = combined_scores.get(item_id, 0.0) + (1.0 / (rrf_k + rank))
    for item_id, rank in lexical_ranks.items():
        combined_scores[item_id] = combined_scores.get(item_id, 0.0) + (1.0 / (rrf_k + rank))
    return combined_scores
```

Both PostgreSQL candidate processors and in-memory test candidate processors call `fuse_rrf_scores(...)`. Ranking logic cannot drift.
