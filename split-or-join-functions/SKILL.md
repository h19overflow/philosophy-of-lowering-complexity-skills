---
name: split-or-join-functions
description: "Make deliberate function extraction and inlining decisions based on software complexity rather than line count. Prevents fragmentation into one-use helpers and conjoined methods. Use when refactoring large functions, extracting methods, reviewing private helper functions, breaking workflows into steps, or inlining tightly coupled code."
---

# Split or Join Functions: Complexity-Driven Decomposition

Based on John Ousterhout's *A Philosophy of Software Design* (Chapter 9: "Better Together Or Better Apart?"), this skill governs when to extract a function and when to inline or join functions together.

---

## Core Philosophy

> **Line count is not a software design metric.**

Never extract a function simply because:
- "The function exceeds 30/50/70 lines."
- "Steps execute in chronological order."
- "A linter complained about cyclomatic complexity on a simple sequential dispatch."

A longer cohesive function is vastly superior to several fragmented, conjoined helpers. Function extraction must **reduce complexity**, not merely relocate it into a maze of private methods.

---

## The 8-Question Extraction Test

Before extracting code into a helper function, answer these 8 questions:

1. **Coherent Abstraction**: Does the extracted code form a single, standalone concept that makes sense on its own?
2. **Intent Name**: Can it be named by **WHAT** it does rather than **WHEN** it executes (e.g. `calculate_discount` vs `_step3_apply_discounts`)?
3. **Parent Comprehension**: Can the parent function be understood without reading the child's implementation?
4. **Child Comprehension**: Can the child be understood and tested without reading the parent function?
5. **Information Hiding**: Does the child hide meaningful implementation details (e.g., complex math, regex parsing, schema traversal)?
6. **State Cleanliness**: Does it avoid giant context/state plumbing and take narrow, explicit parameters?
7. **Independent Reuse/Testing**: Is the helper genuinely reusable elsewhere or valuable to test in isolation?
8. **Independent Evolution**: Will the parent and child usually change independently in future requirements?

### The Decision Rule
- **Mostly YES (5+)**: Extract the function. It represents a genuine, independent abstraction.
- **Mostly NO**: **DO NOT EXTRACT.** Keep the logic inline in the parent function.
- **Existing helper failing this test**: **INLINE IT.** Join it back into the parent.

---

## Detecting Conjoined Functions

Two functions are **conjoined** when understanding either one requires reading both. Conjoined functions scatter a single algorithm across multiple scopes.

### Warning Signs of Conjoined Functions
- **Single Caller**: The helper is called from exactly one place.
- **Caller-Specific State**: The helper reads or mutates variables specifically prepared by its caller.
- **Shared Local Variables**: The helper requires 5+ local variables passed from the parent scope.
- **Lockstep Evolution**: Every bug fix or feature change requires editing both caller and helper simultaneously.
- **Temporal Naming**: Watch for names such as:
  ```text
  _prepare_*
  _process_*
  _handle_*
  _update_*
  _finalize_*
  ```
  *(Note: Do not treat naming alone as evidence; verify whether the function actually hides independent complexity or merely divides chronological steps).*
- **Context Bag Plumbing**: The helper receives a giant state dictionary or context object just to use two fields.
- **Increased Navigation**: Jumping back and forth between functions increases cognitive load compared to reading linear code.

**Rule**: When functions are conjoined, **prefer joining them into one cohesive function**.

---

## Detecting the Opposite Problem: Overly Accumulated Functions

Avoid dogma in either direction: **Fewer functions are not automatically better.**

Inspect whether a large function has accumulated multiple independent abstractions that should be extracted:
- **Mixed Levels of Abstraction**: High-level workflow orchestration mixed with low-level byte manipulation, regex parsing, or SQL query construction.
- **Multiple Unrelated Responsibilities**: A function that parses input, computes a business algorithm, formats an external message, and persists to a database.
- **Hidden Reusable Calculations**: A mathematical formula, normalization rule, or data transformation embedded in a giant controller that cannot be tested in isolation.

**Remedy**: Extract the pure, reusable calculation or normalization step. Leave the workflow narrative in the caller.

---

## Explicit Anti-Dogma Rules

1. **Fewer functions are not automatically better**: If a function does three unrelated things, extracting cohesive, pure helpers improves readability and testability.
2. **Line count is neither a reason to split nor a reason to keep together**: Evaluate independence, information hiding, and cognitive load.
3. **Pure functions are natural extraction candidates**: If a block of code takes explicit data, performs a calculation, and returns a result without touching caller state, extraction is almost always safe and beneficial.
