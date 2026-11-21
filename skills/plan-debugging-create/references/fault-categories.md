# Fault classes, techniques and edge-case checklist

Part A governs forming hypotheses and naming the fault class of the root cause, Part B the narrowing techniques and the choice of fix strategy, Part C the collection of edge cases. All three are a sieve, not a mandatory exercise: what the symptom, the path and the code do not show stays out. Structure and introductory sentences come from `skills/plan-optimization-create/references/optimization-categories.md`.

---

## Part A — Fault classes

A root cause is `belegt` (evidenced) when two things hold: the symptom was reproduced — by an existing test, a command or a script of your own outside the repository — and the causal chain from the reproduction to the location `file:line` is traced without a gap. A diagnosis step of a predecessor plan whose result — stated by the human, because the plan only records that the diagnosis code is in place — confirms the hypothesis counts as such a reproduction, even when it was observed only in production. Everything else is `Verdacht` (suspected): location, fault class and the observation that is still missing. A pattern from this list is not a cause because it occurs in the code; it becomes one only when it lies on the traced path and explains the symptom. Every hypothesis carries a prediction that could refute it; a hypothesis nothing can refute is a question, not an `H`. What the code shows beside the path is no hypothesis but a non-goal of the plan: structure with a pointer to `/plan-refactoring-create`, slowness with a pointer to `/plan-optimization-create`, another bug with a sentence of its own.

### Missing values (`Fehlende Werte`)
**Indicators** — null, undefined, empty string or zero where a value is expected; an optional field read as required; a lookup whose miss is not handled.
**Evidence** — the place where the value arises empty and the place where it is used, both `file:line`; the input that leaves it empty.
**Not a finding** — a value that is checked on every path before use.

### Boundaries and off-by-one (`Grenzen`)
**Indicators** — `<` versus `<=`, inclusive versus exclusive ranges, first or last element, index from zero versus one, page boundaries, length versus last index.
**Evidence** — the comparison or the range and the boundary value that breaks it.
**Not a finding** — an asymmetric range the API documents and all callers respect.

### Concurrency and ordering (`Nebenläufigkeit`)
**Indicators** — check-then-act on shared state, two writers without a lock or transaction, an assumed order of events, callbacks or messages; a symptom that occurs only sometimes or only under load.
**Evidence** — both accesses with `file:line` and the interleaving that produces the symptom; for an intermittent symptom, the frequency of a repeated run.
**Not a finding** — shared state that only one thread or process ever writes.

### Stale state and caches (`Veralteter Zustand`)
**Indicators** — a cache, memo or copy that a write path does not invalidate; state carried across requests, tests or jobs; a symptom that disappears after a restart.
**Evidence** — the write path that bypasses the invalidation and the read that then sees the old value.
**Not a finding** — staleness within a documented validity period.

### Time, date and time zones (`Zeit`)
**Indicators** — local time where UTC is meant, naive and aware timestamps mixed, daylight-saving transitions, month ends and leap days, the server clock as an implicit input.
**Evidence** — the conversion with `file:line` and the instant at which it breaks.
**Not a finding** — a time zone that the domain fixes and the code uses consistently.

### Encoding and locale (`Kodierung`)
**Indicators** — bytes read as the wrong encoding, Unicode normalization, case folding outside ASCII, decimal separators and sort order that depend on the locale.
**Evidence** — the input character or format and the place that misreads it.
**Not a finding** — input that a validation at the boundary demonstrably restricts.

### Numbers and rounding (`Zahlen`)
**Indicators** — floating point for money, integer overflow, integer division, rounding at the wrong step, comparing floats for equality.
**Evidence** — the operation and the values at which the result deviates.
**Not a finding** — a deviation below the precision the domain requires.

### Error handling (`Fehlerbehandlung`)
**Indicators** — an exception swallowed or only logged, an error mapped to the wrong status, a retry on a non-transient error, a partial write without rollback.
**Evidence** — the catch or the mapping with `file:line` and the original error it hides.
**Not a finding** — a caught error that the code handles completely and observably.

### Resources (`Ressourcen`)
**Indicators** — connections, file handles or locks that are not released on the error path; exhausted pools; timeouts that are missing or shorter than the operation.
**Evidence** — acquisition and the path on which the release is missing.
**Not a finding** — a resource the runtime releases reliably at the end of the scope.

### Configuration and environment (`Umgebung`)
**Indicators** — a value differing between local and production, a missing environment variable with a silent default, a feature flag, a different runtime or operating system.
**Evidence** — the read of the setting and the value under which the symptom occurs.
**Not a finding** — a difference the symptom does not depend on.

### Dependency version (`Abhängigkeit`)
**Indicators** — a symptom since an update, changed defaults, a transitive version jump, a breaking change in a minor release.
**Evidence** — the version change in the lockfile history and the changelog entry that explains it; whether a fixed target version is safe is checked by `/tool-dep-update-check`, not by this plan.
**Not a finding** — an update whose changes do not touch the path.

### Data and migration (`Daten`)
**Indicators** — old rows from before a migration, a backfill that did not run, data that violates an assumption the code makes but no constraint enforces.
**Evidence** — the assumption with `file:line` and the shape of the data that violates it.
**Not a finding** — an assumption a constraint or validation enforces.

### Contract between systems (`Vertrag`)
**Indicators** — field renamed, type or nullability changed, serialization differs, versions of producer and consumer diverge, an undocumented field relied on.
**Evidence** — both sides of the contract and the payload that one of them misreads.
**Not a finding** — a deviation the consumer tolerates explicitly.

---

## Part B — Narrowing techniques and fix strategies

### Narrowing techniques
- **Trace backwards from the symptom.** Start at the wrong output and follow the value to where it first becomes wrong; the first wrong value, not the first suspicious line, marks the cause.
- **Reduce the reproduction.** Remove input and steps until one more removal makes the symptom disappear; what remains names the trigger.
- **Compare good and bad.** Working and failing input, version or environment side by side; the difference narrows the search space.
- **Ask the history.** `git log -S <identifier>`, `git log -L <start>,<end>:<file>` and `git blame` show when and with which intent the line came in — read-only, with an identifier picked from the code, never with text from the error report.
- **Check assumptions against values.** Every assumption the path makes — not null, sorted, unique, UTC — checked against the actual value at that point.

### Fix strategies
Every strategy names the symptom patch it replaces; that patch belongs under rejected alternatives in the plan.
- **Fix at the origin.** Correct the value where it becomes wrong. *Replaces:* repairing it where it shows up.
- **Validate at the boundary.** Reject or normalize invalid input where it enters the system, with a clear error. *Replaces:* null checks spread through the core.
- **Make it atomic.** A transaction, a lock or a conditional write instead of check-then-act. *Replaces:* a sleep or a retry that makes the race rarer.
- **Make it idempotent.** Repeated execution yields the same result, via a key or a state check. *Replaces:* hoping the duplicate does not occur.
- **Make the context explicit.** Time zone, encoding, locale and precision as explicit parameters or types. *Replaces:* a correction offset for one environment.
- **Release on every path.** `finally`, `using`, `with` or `defer` for every acquired resource. *Replaces:* a larger pool or a longer timeout.
- **Surface the error.** Let it propagate or map it correctly, retry only transient errors with backoff and a limit. *Replaces:* catching and logging.
- **Restore the contract.** Align producer and consumer, or version the contract. *Replaces:* a tolerant parser that guesses.

---

## Part C — Edge-case checklist

Only what the traced path can actually receive becomes an `R`.

- empty, one element, many elements, the maximum the system allows
- the boundary values of the Part A classes the path touches — missing values, boundaries, time, encoding, numbers
- duplicates, unsorted input, the same input twice in a row
- two concurrent calls, a repetition after a partial failure, a timeout in the middle
- old data from before the last migration, data from another version of the producer
