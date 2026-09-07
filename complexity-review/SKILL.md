---
name: complexity-review
description: "Lightweight post-implementation architecture gate. Evaluates boundary justification, framework necessity, module depth, conjoined helpers, and parallel drift. Use only after non-trivial multi-module changes, new architecture, major refactoring, RAG changes, or agent workflow modifications."
---

# Complexity Review: Lightweight Post-Implementation Gate

Based on John Ousterhout's *A Philosophy of Software Design*, this skill acts as a focused, lightweight post-implementation architecture gate. It evaluates whether a change reduced or contained total system complexity rather than dispersing it.

---

## When to Invoke

Invoke **only** after non-trivial work:
- Multi-module features or architectural additions.
- Major refactoring or boundary reorganizations.
- RAG pipeline or retrieval changes.
- Agent orchestration or framework introductions.
- Infrastructure integrations and persistence changes.

*Do NOT invoke for trivial bug fixes, single-line edits, or localized styling changes.*

---

## Scope & Bounded Review

> **Do NOT audit or redesign the entire repository.**

Review is strictly bounded to the **affected design surface**:
- Changed files and newly introduced types/functions/classes.
- Direct public interfaces modified or introduced.
- Directly impacted callers and callees.

Do not review cosmetic formatting or syntax (handled by formatters/linters). Focus strictly on **structural complexity, information hiding, and framework necessity**.

---

## The 12-Point Architecture Checklist

Evaluate the affected design surface against these 12 questions:

```text
┌─────────────────────────────────────────────────────────────┐
│ 1. Boundary Justification  → Did every boundary earn its way?│
│ 2. Shallow Abstractions    → Are there passthrough layers?  │
│ 3. Conjoined Helpers       → Were cohesive algorithms split?│
│ 4. Temporal Decomposition  → Was code split by clock steps? │
│ 5. Duplicated Knowledge    → Is policy living in 2 places?  │
│ 6. Parallel Drift          → Do test & prod duplicate math? │
│ 7. Framework Semantics     → Are framework features used?   │
│ 8. Interface Leakage       → Does infra leak into domain?   │
│ 9. Speculative Extensibility→ Unused factories / strategies?│
│ 10. Concept Budget         → Did new concepts pay their way?│
│ 11. Change Amplification   → Would future changes ripple?   │
│ 12. Simplification Potential→ Could removing a wrapper help?│
└─────────────────────────────────────────────────────────────┘
```

1. **Boundary Justification**: Did every newly introduced class, service, manager, repository, or graph node earn its existence by hiding meaningful complexity or providing operational value (retries, checkpoints, transactions)?
2. **Shallow Abstractions**: Are there classes or methods whose dominant behavior is forwarding arguments (`caller -> service -> manager -> repo`) without adding policy, validation, normalization, or caching?
3. **Conjoined Helpers**: Were cohesive algorithms fragmented into single-caller private helpers (`_prepare_*`, `_process_*`) where reading the parent requires reading the child?
4. **Temporal Decomposition**: Was code divided into separate functions, classes, or nodes solely because operations occur in chronological sequence ("first, then, next, finally")?
5. **Duplicated Knowledge**: Does the same business rule, schema parsing, coordinate format, or invariant exist in multiple places?
6. **Parallel Implementation Drift**: If parallel mechanisms exist (e.g. PostgreSQL vs in-memory, real vs fake provider), are they duplicating algorithms (such as ranking formulas or scoring math) instead of sharing a single policy function?
7. **Framework Semantics**: If an architectural framework was introduced (e.g. LangGraph, Temporal, Celery), are its runtime semantics (cycles, durable checkpoints, human-in-the-loop pauses, independent retries) genuinely required, or is it a simple deterministic sequence that belongs in procedural code?
8. **Interface Leakage**: Do database query details, vendor SDK types, HTTP headers, or transport exceptions leak into domain logic?
9. **Speculative Extensibility**: Were factories, provider registries, generic protocols, or configuration options built for hypothetical future requirements?
10. **Concept Budget**: Did the total number of new types, files, and names grow more than the delivered capability justified?
11. **Change Amplification**: Will a likely future requirement change force synchronized edits across multiple files?
12. **Simplification Potential**: Would a future engineer understand this feature faster if some files, wrappers, or classes were combined or deleted?

---

## Review Outcomes

Conclude the review with one of three outcomes:

### 1. PASS
The implementation introduces no meaningful avoidable complexity. Boundaries are deep, cohesion is high, frameworks earn their existence, and no shallow wrappers were created.
> Conclude with: **"PASS: Architecture is clean and deep; boundaries earn their existence."**

### 2. PASS WITH NOTES
Minor design tradeoffs remain (e.g. slight coincidental syntactic duplication or acceptable pragmatic shortcuts), but do not justify further code changes now.
> Note the acceptable tradeoff concisely and approve.

### 3. REVISE
The implementation introduces meaningful avoidable complexity (e.g. shallow service, pass-through methods, conjoined helpers, framework gravity, or leaky interfaces).
> For each issue, specify:
> - **Location**: `file:line` or symbol.
> - **Violation**: (e.g., Shallow Abstraction, Framework Gravity, Conjoined Helper, Parallel Drift).
> - **Concrete Impact**: Why this increases cognitive load or change amplification.
> - **Minimal Correction**: The exact inline, collapse, or consolidation action to perform.
>
> *Rule: Fix clear violations when safe. Do not launch unrelated architecture rewrites. Preserve good existing abstractions.*
