# Design dimensions and finding categories

Part A steers the design before the code, Part B the finding rounds afterwards. Both lists are a sieve, not a mandatory exercise: whatever the step does not touch, you leave out.

---

## Part A — Design

### Architecture and placement
- Which layer does the change belong to, and which layers may it know about?
- Does it run synchronously in the request or asynchronously in a job? What speaks against that?
- Which boundary is being drawn here — and who may cross it?

### Component split
- Which component gets the responsibility, and why not the obvious other one?
- What is public, what stays an implementation detail?
- Does the responsibility already exist somewhere? Extending beats building new.

### Data flow
- Where does the data come from, who transforms it, where does it end up?
- Where is the source of truth, and where do copies arise that can go stale?
- What happens if part of the flow fails or runs twice?

### Interfaces and API
- Signature, error format, status codes: do they match the existing code?
- Breaking change for existing callers? If so: transition or version?
- Contracts phrased stably, not coupled to internal structure?
- Pagination, filtering, sorting — only if this step needs them, but considered in the design.

### Data model and migration
- Field types, nullability, defaults, constraints — fitting rather than convenient?
- Indexes for the queries this step introduces.
- Migration backward compatible, so the old and new version can briefly run side by side?
- Backfill needed? Way back available?

### Caching and invalidation
- Does this step need a cache at all, or is that anticipation? Without proven need: no.
- If so: level (request, application, distributed, HTTP), key, lifetime.
- Invalidation is the hard part — who writes, and how does the cache learn about it?
- What happens on a cache miss under load, and on a cold start?

### Verification path
- How does it become visible that this step works — which command, which test case, which manual path?
- What cannot be checked automatically, and how is it secured instead?

---

## Part B — Findings

### Correctness against the plan
- Does the code do what the step promised — completely, not just at the core?
- Are deviations deliberate and justified, or did they arise unnoticed?

### Error cases and edge conditions
- Empty set, very large set, invalid input, timeout, partial failure.
- Duplicate execution: idempotent or guarded?
- Concurrency: shared state, race conditions, transaction boundaries.

### Security
- Authorization checked, not just authentication.
- Inputs validated, outputs encoded for their context, queries parameterized.
- No secrets in the code, no personal data in the log.

### Performance
- Queries in loops, N+1, missing indexes, unbounded result sets.
- Blocking work in the request path that belongs in a job.
- Does this also behave like this with a realistic data volume, or only in the test?

### Consistency with the existing code
- Naming, error handling, structure as in the surrounding code?
- A second pattern introduced for an already solved problem?
- Terms used as in the project's domain model?

### Tests
- Do they cover the step's promise and not just the happy path?
- Do they test behavior or only implementation?
- Do they actually run green, and in isolation too?

### Leftovers
- Commented-out code, debug output, TODOs without an addressee, dead code.
- Half-finished abstractions with exactly one caller.
- Forgot to update configuration, migrations, documentation?
