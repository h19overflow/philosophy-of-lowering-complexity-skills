---
name: exception-design
description: "Design error semantics so callers handle the smallest possible number of exceptional conditions. Define errors out of existence, mask recoverable failures, aggregate at boundaries, and fail fast on corruption. Use when designing, modifying, or reviewing failure semantics, retries, idempotency, and error propagation."
---

# Exception Design

## Purpose

Design error semantics so callers handle the smallest possible number of exceptional conditions.

The goal is NOT to add comprehensive exception hierarchies.

The goal is:

> Minimize the number of failure conditions that callers must understand while preserving correctness, observability, and recoverability.

Use this skill when creating, modifying, reviewing, or propagating errors and exceptions.

The primary decision framework is:

1. Define the error out of existence.
2. Mask the error inside the responsible abstraction.
3. Aggregate related errors at an appropriate higher-level boundary.
4. Crash or fail fast when meaningful recovery is impossible or unsafe.

Prefer eliminating exceptional states over adding handlers.

---

# Trigger Conditions

Use this skill when:

* adding a new exception class;
* adding `try/except` or `try/catch`;
* deciding whether a function should raise/throw;
* designing return types involving failure;
* translating infrastructure errors;
* implementing retries;
* handling cache misses;
* handling missing resources;
* handling duplicate operations;
* processing queues/events;
* designing idempotent APIs;
* implementing RAG retrieval failure behavior;
* handling LLM/provider failures;
* creating HTTP/FastAPI/Express exception handlers;
* designing background workers;
* handling distributed-system retries;
* deciding whether a process should terminate;
* reviewing an exception hierarchy;
* seeing repeated exception handling across callers;
* auditing failure semantics, retries, or idempotency in a substantial change.

Typical trigger phrases:

* exception
* error handling
* raise / throw
* retry
* fallback
* timeout
* not found
* duplicate
* already exists
* cache miss
* invalid state
* provider failure
* error mapping
* idempotency
* failure recovery
* edge cases
* design holes

---

# Governing Principle

Every exposed exception increases interface complexity.

An exception effectively adds another possible output to a function.

Conceptually:

```text
result = operation(...)
```

is not merely:

```text
Input → Result
```

if callers must understand:

```text
Input
  ├── Result
  ├── TimeoutError
  ├── NotFoundError
  ├── AlreadyExistsError
  ├── ProviderError
  ├── RetryableError
  └── ValidationError
```

Treat exception surface area as part of API surface area.

Before introducing or propagating an exception, determine whether the caller genuinely needs to know about it.

---

# The Four Decisions

For every exceptional condition, evaluate these strategies IN ORDER.

```text
Potential Failure
       │
       ▼
1. Can semantics make this normal?
       │
      YES ───► DEFINE OUT OF EXISTENCE
       │
       NO
       ▼
2. Can this abstraction recover correctly?
       │
      YES ───► MASK
       │
       NO
       ▼
3. Can this propagate to a boundary that
   handles many related failures together?
       │
      YES ───► AGGREGATE
       │
       NO
       ▼
4. Is meaningful recovery impossible,
   unsafe, or unjustifiably expensive?
       │
      YES ───► FAIL FAST / CRASH
```

Do not jump immediately to:

```python
try:
    ...
except:
    ...
```

First question whether the exception needs to exist.

---

# Strategy 1 — Define Errors Out of Existence

## Question

> Is this actually an exceptional condition from the caller's perspective?

If the answer is no, redesign the operation's semantics.

Prefer normal values or idempotent semantics over exceptions for normal domain states.

---

## Example — RAG No Evidence / Empty Results

Avoid:

```python
class NoEvidenceFound(Exception):
    pass


async def search(query: str) -> list[Evidence]:
    evidence = await ...
    if not evidence:
        raise NoEvidenceFound(query)

    return evidence
```

"No relevant evidence" or "no search results" is usually a valid search result.

Prefer:

```python
@dataclass(frozen=True)
class SearchResult:
    evidence: tuple[Evidence, ...]

    @property
    def has_evidence(self) -> bool:
        return bool(self.evidence)
```

```python
result = await knowledge.search(query)

if not result.has_evidence:
    return Answer.insufficient_evidence()
```

The API now means:

```text
search(query) → zero or more evidence items
```

instead of:

```text
search(query)
    → evidence
    OR exception
```

---

## Example — Idempotent Delete

Avoid:

```python
async def delete_document(document_id: UUID) -> None:
    document = await load(document_id)

    if document is None:
        raise DocumentNotFound(document_id)

    await delete(document)
```

If the caller's desired postcondition is:

```text
document does not exist
```

then an already-missing document may be success.

Prefer:

```python
async def delete_document(document_id: UUID) -> None:
    await db.execute(
        "DELETE FROM documents WHERE id = $1",
        document_id,
    )
```

This makes repeated deletion naturally idempotent.

---

## Example — Duplicate Queue Delivery

Avoid:

```python
if await already_processed(event.id):
    raise DuplicateEvent(event.id)
```

At-least-once delivery means duplicates are expected.

Prefer semantics such as:

```python
async def process(event: DocumentIndexed) -> None:
    if await processed_version(event.document_id) == event.version:
        return

    await apply_event(event)
```

or enforce idempotency through storage.

The duplicate disappears as an exceptional state.

---

## Strong Candidates for Defining Away

Challenge exceptions such as:

* `CacheMiss`
* `NoResults`
* `AlreadyDeleted`
* `AlreadyApproved`
* `AlreadyProcessed`
* `DuplicateEvent`
* `EmptySearchResult`
* `AlreadyExists`

They may still be legitimate errors in some domains.

Do not remove them blindly.

Ask what postcondition the caller actually wants.

---

# Strategy 2 — Mask the Error

## Question

> Can this module recover correctly without requiring the caller to participate?

If yes, handle the failure inside the abstraction.

The caller should not need to know about mechanical infrastructure failures that the module can successfully compensate for.

---

## Example — Model Rate Limiting & Transient Retries

Application code should usually not contain:

```python
try:
    response = await openai_client.invoke(messages)
except RateLimitError:
    await asyncio.sleep(...)
    ...
```

if retry policy belongs to model infrastructure.

Prefer:

```python
response = await model_gateway.invoke(messages)
```

where the gateway internally handles:

```text
provider call
   │
   ├── 429
   │    ↓
   │   backoff
   │    ↓
   │   retry
   │
   └── response
```

The transient 429 is masked.

If retries are exhausted, the gateway may expose a higher-level error:

```python
raise ModelUnavailable(...) from exc
```

The caller knows:

```text
model unavailable
```

not:

```text
OpenAI SDK HTTP 429 error
```

---

## Example — Cache

Avoid exposing:

```python
CacheMiss
```

to every caller.

A deeper abstraction may implement:

```python
async def get_embedding(text: str) -> Vector:
    cached = await cache.get(text)

    if cached is not None:
        return cached

    embedding = await provider.embed(text)
    await cache.set(text, embedding)

    return embedding
```

The caller does not need to know the cache exists.

---

# Masking Rules

Mask when:

* recovery is deterministic;
* recovery policy belongs to the abstraction;
* callers would all respond identically;
* exposing the lower-level failure would leak implementation details.

Typical masking mechanisms:

* bounded retry;
* fallback;
* cache population;
* reconnect;
* transparent refresh;
* normalization;
* duplicate suppression.

Do NOT mask when doing so:

* changes domain semantics unexpectedly;
* hides permanent failure;
* creates unbounded retry loops;
* loses important information;
* can violate correctness.

---

# Strategy 3 — Error Aggregation

## Question

> Do several failures eventually require the same higher-level response?

If yes, allow them to propagate to one common boundary.

Do not write identical handlers at every intermediate layer.

---

## Example — HTTP / API Boundary

Deep application code may raise domain errors:

```python
raise InvalidDocument(...)
raise ConversationNotFound(...)
raise ModelUnavailable(...)
raise PermissionDenied(...)
```

Intermediate layers should not repeatedly convert them into HTTP.

Prefer centralized translation:

```python
@app.exception_handler(AtlasError)
async def handle_atlas_error(request, exc):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "error": exc.code,
            "message": exc.message,
        },
    )
```

Then:

```text
InvalidDocument ───────┐
ConversationNotFound ──┤
ModelUnavailable ──────┼──► HTTP Error Boundary
PermissionDenied ──────┘
```

One boundary owns HTTP representation.

---

# Aggregation Is Not Exception Swallowing

Bad:

```python
try:
    ...
except Exception:
    return {"error": "Something failed"}
```

Good aggregation:

* preserves meaningful distinctions;
* centralizes representation;
* maintains observability;
* keeps root cause information;
* translates at the correct architectural boundary.

For example:

```python
class DomainError(Exception):
    code: str


class ModelUnavailable(DomainError):
    code = "model_unavailable"


class InvalidDocument(DomainError):
    code = "invalid_document"
```

The hierarchy should encode meaningful handling categories, not every technical detail.

---

# Strategy 4 — Fail Fast / Crash

## Question

> Can the current process safely and meaningfully recover?

If no, do not build elaborate recovery machinery merely to avoid failure.

Examples:

* corrupted persistent invariant;
* incompatible database schema;
* required configuration missing at startup;
* impossible internal state indicating a programming defect;
* inability to initialize mandatory security infrastructure.

Example:

```python
if database_schema_version != REQUIRED_VERSION:
    raise RuntimeError(
        "Application database schema is incompatible"
    )
```

Allow the process supervisor to restart or keep the service unhealthy.

---

# Crash Does Not Mean Ignore

Fail-fast behavior should provide:

* useful diagnostics;
* structured logs;
* correlation IDs where appropriate;
* meaningful exit status;
* observability.

Do not silently continue from an invalid state.

---

# Distributed Systems Judgment

Distributed systems create many states that LOOK exceptional but are normal consequences of unreliable communication.

Challenge exceptions involving:

* duplicate messages;
* repeated commands;
* retries;
* stale reads;
* already-completed operations;
* connection refresh;
* lease expiration;
* optimistic concurrency conflicts.

Prefer designing around:

* idempotency;
* desired postconditions;
* monotonic state transitions;
* deduplication keys;
* compare-and-set;
* transaction boundaries.

---

# Postcondition-Oriented Design

When evaluating a distributed operation, ask:

> What state does the caller want after this operation finishes?

Example:

```text
delete(resource)
```

desired state:

```text
resource absent
```

Therefore:

```text
already absent
```

may be success.

Example:

```text
approve(action)
```

desired state:

```text
action approved
```

Therefore:

```text
already approved
```

may be success.

Example:

```text
ensure_indexed(document, version)
```

desired state:

```text
version indexed
```

Therefore duplicate delivery may be success.

This style frequently eliminates exception states while improving idempotency.

---

# AI Engineering Judgment

## Retrieval

Usually normal:

* zero results;
* weak evidence;
* partial evidence.

Usually exceptional:

* vector database unavailable;
* corrupted stored representation;
* unauthorized tenant access.

---

## Model Invocation

Usually mask internally:

* transient 429;
* temporary 5xx;
* short network timeout when retry is safe.

Usually expose as normalized domain failure after recovery fails:

```text
ModelUnavailable
ModelTimeout
ContextLimitExceeded
```

Avoid exposing provider-specific SDK exceptions through the application.

---

## Structured Output

Do not force every application caller to handle:

* JSON parsing;
* markdown fences;
* provider-specific schema syntax;
* repair attempts.

If structured output is part of the abstraction contract, the model layer should absorb appropriate mechanics.

Expose only the failure that matters after recovery fails, for example:

```python
StructuredOutputUnavailable
```

if the application genuinely needs to react.

---

## RAG Grounding

Do not turn:

```text
insufficient evidence
```

into a generic infrastructure exception.

It is usually a domain outcome.

Represent it explicitly:

```python
Answer(
    status=AnswerStatus.INSUFFICIENT_EVIDENCE,
    text=...,
    citations=[],
)
```

---

# Error Translation

Infrastructure-specific failures should generally stop at infrastructure boundaries.

Avoid leaking:

```text
asyncpg.UniqueViolationError
openai.RateLimitError
redis.ConnectionError
httpx.ReadTimeout
```

through domain/application layers.

Translate when crossing the boundary:

```python
try:
    ...
except asyncpg.UniqueViolationError as exc:
    raise DocumentConflict(...) from exc
```

But do not create one domain exception for every vendor exception.

Translate according to what callers need to distinguish.

---

# Exception Hierarchy Design

Before creating a new exception class ask:

1. Will a caller handle this differently?
2. Does it represent meaningful domain semantics?
3. Does it protect callers from infrastructure details?
4. Does it improve aggregation?
5. Would a result value be clearer?
6. Could the error instead be masked?

If the only justification is:

> "This situation has a unique name."

do not automatically create a class.

---

# Retry Judgment

Retries are exception masking only when they are safe.

Before retrying ask:

1. Is the failure plausibly transient?
2. Is the operation idempotent?
3. Could the first attempt have succeeded despite the client seeing failure?
4. Is retry bounded?
5. Is there backoff?
6. Will retry amplify an outage?
7. Who owns retry policy?

Never blindly retry:

* non-idempotent writes;
* authorization failures;
* validation failures;
* deterministic schema errors;
* malformed requests.

---

# Layer Ownership

A useful default:

```text
User / HTTP
     ↑
     │ aggregate domain failures
     │
Application
     ↑
     │ domain errors
     │
Deep Modules
     ↑
     │ mask recoverable mechanics
     │
Infrastructure / Providers
```

And before any of this:

```text
Can we define the condition out of existence?
```

---

# Anti-Patterns

## Exception Explosion

```text
EmbeddingTimeoutError
EmbeddingRateLimitError
EmbeddingNetworkError
EmbeddingProvider500Error
EmbeddingRetryExhaustedError
```

when every caller handles all of them as:

```text
embedding unavailable
```

Prefer one meaningful abstraction-level error.

---

## Catch-and-Rethrow Noise

Avoid:

```python
try:
    return await repository.load(id)
except DatabaseError as exc:
    raise DatabaseError(str(exc))
```

This adds no semantic value.

Translate only when crossing abstraction boundaries or adding meaningful context.

---

## Catch Everywhere

Avoid exception handling at every call layer.

```text
controller catches
service catches
manager catches
repository catches
adapter catches
```

when each merely logs and rethrows.

Prefer one appropriate owner.

---

## Generic Exception Suppression

Avoid:

```python
except Exception:
    pass
```

unless failure is genuinely optional and intentionally isolated.

---

## Logging at Every Layer

Avoid:

```text
repository logs error
service logs same error
API logs same error
worker logs same error
```

producing four copies of one failure.

Prefer logging at the layer that owns recovery or final failure reporting.

---

## Exceptions for Expected Branching

Avoid using exceptions as ordinary control flow:

```python
try:
    user = get_user(id)
except UserNotFound:
    create_user()
```

when absence is a normal possibility and a result API is clearer.

---

# Post-Implementation Review Pass: Edge Cases & Holes

When reviewing newly created work or non-trivial implementations, systematically check:

1. **Semantic Inversion**: Are normal conditions (empty lists, 0 search results, cache misses, already-deleted records) thrown as exceptions instead of returned as ordinary values?
2. **Concurrency & Distributed Races**: What happens on duplicate requests, network retries, out-of-order queue events, expired leases, or simultaneous writes? Are operations idempotent and transitions monotonic?
3. **Leaky Boundaries**: Are vendor SDK errors (HTTP 429/5xx, DB connection/unique constraint codes) escaping deep modules into application callers?
4. **Retry Storms & Unbounded Loops**: Are retries bounded with backoff? Are non-idempotent writes, 4xx validation, or auth failures protected from blind retry?
5. **Double Logging & Catch-Rethrow**: Is an error logged at every layer it traverses, or caught only to be wrapped in an identical exception?
6. **Silent Swallowing & Half-States**: Does any `catch`/`except` block silently suppress errors or leave database/entity state partially written without rollback?
7. **Caller Cognitive Budget**: Does the caller have to know more than 2-3 failure modes to use this abstraction safely?

---

# Decision Procedure

When writing or reviewing code containing an error path:

## Step 1 — Name the condition

What exactly happened? Do not begin with exception type names. Describe the state.

## Step 2 — Determine whether it is abnormal

Does the operation's contract reasonably include this state? If yes: DEFINE IT OUT OF EXISTENCE.

## Step 3 — Determine recovery ownership

Does this module know enough to recover correctly? If yes: MASK IT.

## Step 4 — Search for common handling

Do many failures ultimately require the same higher-level behavior? If yes: AGGREGATE.

## Step 5 — Determine whether recovery is meaningful

If continuing would be unsafe, corrupt, misleading, or impossible: FAIL FAST.

## Step 6 — Minimize exposed failure concepts

How many different failure concepts must the caller understand now? Reduce this number without hiding meaningful domain distinctions.

---

# Review Checklist

When reviewing substantial changes involving errors, ask:

1. Is every new exception genuinely exceptional?
2. Could any failure be represented as a normal result?
3. Could idempotent semantics eliminate an error?
4. Are recoverable infrastructure failures masked at the correct layer?
5. Are provider/vendor errors leaking upward?
6. Is repeated handling scattered across callers?
7. Can related failures be aggregated?
8. Are we creating too many exception subclasses?
9. Are retries safe and bounded?
10. Is duplicate delivery treated as expected where appropriate?
11. Are errors logged multiple times while propagating?
12. Is any code attempting unsafe recovery instead of failing fast?
13. Does the caller understand fewer failure concepts after this design?

---

# Expected Agent Behavior

Do NOT automatically create exceptions when requirements mention failure conditions.

Instead:

```text
failure discovered
      ↓
classify semantics
      ↓
define away?
      ↓
mask?
      ↓
aggregate?
      ↓
fail fast?
```

When implementation choices are equivalent, prefer the one with the smaller externally visible error surface.
