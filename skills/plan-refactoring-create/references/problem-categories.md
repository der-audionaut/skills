# Problem categories and strategy catalogue

Part A steers the collection of problem areas, Part B the choice of strategies. Both lists are a sieve, not a compulsory exercise: what the codebase does not show stays out.

---

## Part A — Problem categories

The evidence for a `P` is the location `file:line` plus the grep pattern that found it, or the metric that measures it — not the impression while reading. What only stands out while reading, such as structurally identical duplicates or comments that contradict the code, is evidenced by the quoted line. Metrics on size and change frequency come from the hotspot list of the orientation, not from a second run. What cannot be evidenced is an open question, not a finding. Suspected bugs get no `P`; they go in their own section of the template.

### Structural problems (`Strukturprobleme`)

**Indicators**
- Layer violation: a call against the dependency direction, such as persistence straight from the handler or business logic in the controller.
- Cyclic dependencies: module A imports B and B imports A, directly or via third parties.
- God modules: large, frequently changed and depended on by many — all three at once; candidates are the large, frequently changed files on the hotspot list.
- Wrong responsibility: a module computes with another module's data instead of asking it.
- Mixed concerns: one module carries input and output, business rules and presentation at once.
- Reaching into internals: access to private fields, subpaths or helper functions of a foreign module instead of its public surface.
- Implicit coupling: wiring via strings, reflection, global state or shared configuration that no import makes visible.

**Evidence**
- Both sides of the dependency.
- Coupling as a number: a grep over all import statements yields the edge list; from it, incoming and outgoing imports per module and the cycles are read off, without a second run. Counts are made for the modules that a `P` or the relocation table names.

**Not a finding**
- A layering that the project itself nowhere adheres to.
- Dependencies on framework base classes that the framework forces — a named exception of the target architecture.

### Duplicates (`Duplikate`)

**Indicators**
- Identical: the same block in two places, except for whitespace.
- Structurally identical: the same sequence with different identifiers or types.
- Divergently copied: common origin, different by now — recognizable by the same shape with differing edge cases. Carries the note „divergent", because merging them can change behavior.

**Evidence**
- Both locations and the pattern — a distinctive expression or function name — that found both; for structurally identical blocks, the quoted lines.

**Not a finding**
- Coincidental similarity without a common reason — merging couples what should stay separate.
- Tests that repeat deliberately to stay readable.

### Performance bottlenecks (`Performance-Engpässe`)

**Indicators**
- Queries or external calls in loops (N+1).
- Unbounded result sets: loading without limit, pagination or streaming.
- Repeated computation without reuse within one pass.
- Blocking work in the request path that belongs in a job.
- Missing indexes for queries the code describes.

**Evidence**
- Found statically means „Verdacht" (suspicion): the pattern that shows the loop or the call. A suspicion can get a strategy; the step's acceptance is then the counted number of queries.
- „belegt" (evidenced) only with a measurement stored in the repository — benchmark result, profiler output. If there is none, it stays a suspicion; no further search is made.

**Not a finding**
- A pattern that does not hold given the data volume: a loop over three configuration entries is not N+1.

### Maintainability risks (`Wartbarkeitsrisiken`)

**Indicators**
- Hotspots without tests: high change frequency and no test on the path.
- Magic values, long parameter lists, boolean flags as behavior switches.
- Comments that contradict the code; dead paths; TODOs without an addressee.
- Configuration in several places for the same thing.

**Evidence**
- Metric from the hotspot list (changes, lines per file), parameters per function — or the grep pattern.

**Not a finding**
- Style that follows the project's conventions, even if it would be solved differently elsewhere.

### Severity

- **Hoch** (high) — hinders every change in the scope or grows with every feature. Needs a strategy.
- **Mittel** (medium) — costs on every touch, otherwise stays stable. Gets a strategy if it lies on the same path as a Hoch finding.
- **Niedrig** (low) — local or a matter of taste. Listed under non-goals with a reason.

---

## Part B — Strategy catalogue

A strategy resolves named `P` identifiers; a strategy without a `P` is anticipation. The target architecture is measured against three standards — concerns separated, modules with a clear surface, coupling lowered — and evidences them with the import numbers from Part A before and after; "after" is counted on the planned import graph.

Three rules apply to every technique. Every order begins with the tests that the behavior guard names — if they are missing, a characterization-test step goes first. Removal happens only once the grep for the old thing is empty; strings, configuration and reflection count as callers, and this grep is the step's acceptance. Structure never changes in the same step as logic.

### Extract (function, module, class)
- **When:** a block does something recognizable that deserves a name, or two concerns share one function.
- **Order:** extract the block → call it at the old place.
- **Behavior guard:** the tests of the surrounding function; the extracted unit gets its own tests only once it has its own surface.
- **Trap:** extracting the block with all the variables of its surroundings — a function with eight parameters is no gain; bundle the data first, then extract.

### Move with redirection of callers
- **When:** a function lives in the wrong module, or a module knows a layer it should not know.
- **Order:** create at the new place → old place delegates → switch callers one by one → remove old place.
- **Behavior guard:** the tests of the callers.
- **Trap:** leaving the delegation at the old place — then there are two places for one function, and the next caller picks the wrong one.

### Merge duplicates
- **When:** identical, structurally identical or divergently copied blocks with a common reason.
- **Order:** extract a shared function from one side → switch the second side over.
- **Behavior guard:** both test suites, for divergent duplicates including the edge cases in which the sides differ.
- **Trap:** unifying divergent duplicates onto the "nicer" side and losing the other side's edge cases.

### Introduce a seam (interface, adapter)
- **When:** a dependency is to become replaceable or testable before it is moved.
- **Order:** derive the interface from the existing usage → put the existing code behind it without changing it → switch callers to the interface.
- **Behavior guard:** the tests of the callers; the interface gets a contract test that old and new code must both pass.
- **Trap:** cutting the interface according to the ideal picture instead of the actual usage — then the existing code does not fit behind it.

### Parallel Change
- **When:** a signature, a data format or a module is to be replaced while the callers keep running.
- **Order:** create the new next to the old → switch callers one by one, each step runnable → remove the old.
- **Behavior guard:** the tests of the callers after each switch.
- **Trap:** cutting the transition phase short and swapping old and new in one step.

### Strangler for large modules
- **When:** a god module that cannot be replaced in one go.
- **Order:** facade in front of the module → pull one responsibility after another behind the facade into a new module until the facade points only to new code → remove the facade.
- **Behavior guard:** characterization tests against the facade that stay the same across all rebuilds.
- **Trap:** pulling two responsibilities at the same time because they "belong together" — then it is unclear which move broke the test.

### Batch queries (against N+1)
- **When:** a loop triggers one query or external call per iteration.
- **Order:** load the data before the loop in one go → switch the loop to the loaded set.
- **Behavior guard:** the loop's result test; the counted number of queries is the acceptance.
- **Trap:** changing order or error behavior — a batched query fails differently than ten individual ones.

### Decouple via port or event
- **When:** two modules know each other mutually, or an inner part knows an outer one.
- **Order:** determine the direction that is to remain → for the other, introduce a port (interface in the inner module, implementation outside) or an event → remove the counter-import.
- **Behavior guard:** the tests of both modules; the import grep evidences that the cycle is gone.
- **Trap:** introducing an event where a port suffices — asynchronous coupling is harder to test than an interface.

### Separate layers (Clean Architecture)
- **When:** business logic, use cases and adapters are jumbled together, and the target architecture demands dependencies pointing inward.
- **Order:** name the domain and free it from outside dependencies via ports → put the use cases on top → connect the adapters on the outside.
- **Behavior guard:** tests of the use cases against the ports; characterization tests at the adapters.
- **Trap:** forcing a template onto a framework that dictates its own layout and its own base classes — the separation happens within its constraints.

### Relocate a module
- **When:** the target architecture gives the module a different place.
- **Order:** change the directory → switch namespace or package, paths and all imports in the same step.
- **Behavior guard:** the test run at the baseline state.
- **Trap:** tidying up during the relocation — then Git no longer detects the rename (`git diff -M` needs similarity), and the diff hides the logic change behind a hundred moved lines.
