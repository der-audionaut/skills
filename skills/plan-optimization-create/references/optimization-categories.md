# Optimization categories and strategy catalogue

Part A governs collecting the problem areas, Part B the choice of strategies. Both lists are a sieve, not a mandatory exercise: what the measurement and the codebase do not show stays out. Structure and introductory sentences come from `skills/plan-refactoring-create/references/problem-categories.md`; where both catalogues say something about performance, this one applies.

---

## Part A — Problem categories

The evidence of a `P` is `gemessen` (measured) or `Verdacht` (suspected). Measured: the value is in the baseline or in a measurement stored in the repository, and the finding refers to the row of the target values. Only what the baseline outputs counts as measured — a run of your own to back up a suspicion is the "Measure first" step, not evidence. Suspected: location `file:line`, the pattern, and the data volume above which it matters. A pattern on a volume that, according to fixtures, configuration or the domain, stays small is not a finding; neither is style without runtime impact — readability is a matter for `/plan-refactoring-create`, at most one sentence under non-goals. What cannot be evidenced is an open question, not a finding. Suspected bugs get no `P`; they go in their own section of the template.

### Bottlenecks (`Engpässe`)

**Indicators**
- Queries or external calls in loops (N+1).
- Blocking work in the request path that belongs in a job.
- Locks and serial waits that force independent work to run one after another.
- Synchronous input and output in hot paths that the operation needs — repeatedly reading the same source is Unnecessary work.

**Evidence**
- Measured: the path's share of runtime. Suspected: the loop or the call and the number of iterations.

**Not a finding**
- Blocking work whose result the response needs — it cannot move into a job.

### Inefficient logic (`Ineffiziente Logik`)

**Indicators**
- Quadratic or worse search in lists where a set or map would serve.
- Repeated computation of the same value per call or per iteration.
- Unnecessary copies and conversions between representations.
- A data structure that does not fit the access pattern.

**Evidence**
- Measured: runtime of the operation over the data volume. Suspected: the complexity the code shows.

**Not a finding**
- A computation that the code executes only once anyway.

### Unnecessary work (`Unnötige Arbeit`)

**Indicators**
- Rendering unchanged parts.
- Reading or loading the same unchanged source per iteration — file, configuration, query.
- Loading data that is not needed, or entire lists where a slice or virtualization suffices.
- Work before first need whose result often goes unused.

**Evidence**
- Measured: calls, renderings or loaded records per operation versus those used. Suspected: the place that triggers work without need.

**Not a finding**
- Preparatory work whose result is demonstrably always needed.
- Rendering patterns in a project without a UI.

### Memory (`Speicher`)

**Indicators**
- Caches and collections that grow without bound.
- Materializing entire results where streaming or iterating suffices.
- Holding large objects beyond their use.
- Leaks via registrations, subscriptions and closures that are never released.

**Evidence**
- Measured: peak usage or its course over time. Suspected: the place that holds or materializes.

**Not a finding**
- Memory that ends with the operation.

### Scaling (`Skalierung`)

**Indicators**
- In-process state that prevents multiple instances.
- Missing pagination or limits at interfaces.
- Work that grows with the total volume instead of with the result.
- Missing indexes for queries the code describes.

**Evidence**
- Measured: runtime or throughput with growing data volume. Suspected: the work that grows with the total volume.

**Not a finding**
- An interface without pagination whose volume the domain caps — countries, status values.

### Severity

- **Hoch** (high) — lies on the hot path and is measured, or determines the complexity of the main operation. Needs a strategy.
- **Mittel** (medium) — lies on the hot path as a suspicion. Gets a strategy if it lies on the same route as a `Hoch` finding or if the planned measurement step covers its metric.
- **Niedrig** (low) — lies on a cold path. Listed with a reason under non-goals.

---

## Part B — Strategy catalogue

A strategy resolves named `P` identifiers; a strategy without a `P` is anticipation.

Three rules apply to every technique. Every optimization begins with the measurement of its metric and ends with the same measurement, using the same command. Structure and runtime never change in the same step as behavior — result, ordering, floating-point precision and error behavior stay. A cold path is not optimized.

### Measure first
- **When:** the baseline is missing or does not cover the metric.
- **Metric:** the one the plan wants to move first; the step delivers its baseline value.
- **Behavior guard:** none needed — the step changes no production code.
- **Trap:** building a micro-benchmark that does not measure the hot path, and then optimizing its number.

### Switch algorithm and data structure
- **When:** quadratic search or the wrong structure for the access pattern.
- **Metric:** runtime of the operation over the realistic data volume.
- **Behavior guard:** equality of results including ordering and edge cases — empty set, duplicates, boundary values.
- **Trap:** introducing a set and thereby losing the ordering a caller relies on.

### Memoization and caching
- **When:** the same input is repeatedly computed or loaded.
- **Metric:** number of computations or accesses per operation, then runtime.
- **Behavior guard:** one test per write path that proves invalidation; one test for the size limit.
- **Trap:** no invalidation path or no size limit — the first case delivers stale data, the second becomes a memory finding.

### Batching and preloading
- **When:** queries or calls per loop iteration (N+1).
- **Metric:** number of queries or calls per operation.
- **Behavior guard:** error behavior when an element is missing or fails.
- **Trap:** preloading everything, including what the operation never touches — then the finding migrates to Memory.

### Defer, paginate, stream
- **When:** entire results are materialized or loaded although only a slice is needed.
- **Metric:** peak usage and time to first result.
- **Behavior guard:** completeness and ordering across all pages; behavior when data changes during iteration.
- **Trap:** pagination without stable sorting — elements appear twice or not at all.

### Indexes for described queries
- **When:** a query filters or sorts on a field without an index, and the volume grows.
- **Metric:** runtime of the query at realistic volume.
- **Behavior guard:** the query test and the measurement of a write path.
- **Trap:** an index for a rare query — it makes every write more expensive.

### Asynchronous or parallel
- **When:** the measurement shows waiting time on independent input and output or external calls.
- **Metric:** waiting time per operation versus total runtime.
- **Behavior guard:** equality of results for any order of responses; error behavior when one part fails.
- **Trap:** parallelizing CPU-bound work that shares the interpreter or a lock — overhead without gain.

### Rendering: memoization, stable keys, virtualization
- **When:** unchanged parts are re-rendered or long lists are drawn in full — only in projects with a UI.
- **Metric:** number of renderings per interaction, time to display.
- **Behavior guard:** visible result and state of input fields stay; snapshot or component tests.
- **Trap:** lifting state up or deriving keys from the index — then more renders, not fewer.

### Reduce allocations, reuse buffers
- **When:** a hot path creates many short-lived objects per iteration.
- **Metric:** allocations per operation, then runtime.
- **Behavior guard:** a test with two concurrent operations.
- **Trap:** sharing a buffer that two operations see at the same time.

### Remove unnecessary work
- **When:** work is triggered whose result demonstrably goes unused.
- **Metric:** number of executions per operation, then runtime.
- **Behavior guard:** the test showing that the result is needed nowhere — a grep for the users and a run without the work.
- **Trap:** removing work that is needed via a side effect — logging, counters, cache warming.
