# Review dimensions

For each dimension: does it apply? If so, what evidence in the code or in the plan supports your assessment? Headings and bullet names a dimension mentions mean the element with that meaning — in a plan in another language you search and grep for their translation.

## Scope and completeness
- Does the plan describe what does **not** belong to it? A plan without non-goals grows during implementation.
- Does it cover the feature completely, or does it stop at the happy path?
- Are there requirements from the ticket or the discussion that are missing from the plan?

## Factual accuracy against the codebase
- Do all named files, classes, functions, tables, endpoints exist?
- Do signatures, return types and parameters match the plan?
- Does the plan rebuild something that already exists (helper, service, utility)?
- Does the proposed approach fit the existing patterns, or does it introduce a second, competing pattern?

## Open decisions
- Wordings like "vermutlich" (presumably), "sollte man klären" (should be clarified), "TODO", "je nachdem" (depending) mark unresolved points.
- Where does the plan implicitly make a decision without naming the alternative?
- Which decision can only the human make (product, priority, risk)?

## Ordering and dependencies
- Is every step executable with the state after the previous step?
- Does the system stay runnable after every step, or is there a broken intermediate state?
- Migration before code usage, feature flag before rollout, consumers before removal of the old interface?
- External dependencies (other team, third-party system, approval) named and scheduled?

## Data model and migration
- Migration backward compatible? What happens to running instances of the old version?
- Is existing data covered (backfill, defaults, nullable)?
- Is there a way back if the migration goes wrong?
- Indexes, constraints, foreign keys considered?

## Interfaces and contracts
- Who consumes the changed API, the event, the library? Are all callers covered?
- Breaking change recognized and versioned, or avoided?
- External contracts (customers, partners, other teams) affected?

## Error cases and edge conditions
- What happens on timeout, partial outage, duplicate execution, empty set, very large set?
- Idempotency needed? Concurrency, race conditions, locks?
- How does the feature behave on invalid input?

## Security and permissions
- Who is allowed to do this? Is authorization checked, not just authentication?
- Personal data touched? Deletion concept, logging, purpose limitation?
- New attack surface: inputs, uploads, external calls, secrets in the code?

## Performance
- Additional queries per request, N+1, unbounded result sets?
- Does the approach scale with the realistic data volume, not just the one in the test?
- Blocking work in the request path that belongs in a job?

## Tests and verification
- What measures "done"? Is that stated per step in the plan?
- Which test level per change (unit, integration, end-to-end)?
- Are the error cases from the edge conditions described in a testable way?
- Is there a manual verification path for what tests do not cover?

## Behavior preservation (plans with problem areas)
Applies when the plan has a `## Problemfelder` section; otherwise skip. A question prefixed with „Bei `## …`:" (with `## …`) applies only when the plan has that section — what the plan does not have, you do not check, even if a step looks like it in substance.
- Does every step have the `**Verhaltensschutz:**` (behavior guard) bullet? A grep with line numbers for `^## |^### Schritt |^- \*\*(Verhaltensschutz|Messung):\*\*` shows it: under `## Schritte`, every step heading is followed by exactly one Verhaltensschutz hit and, with `## Zielwerte`, exactly one Messung hit; strategies and history do not count. A step without behavior guard is a blocker.
- Do the named tests exist (evidence from factual accuracy), or is the move preceded by a characterization-test step in an earlier wave?
- Does a strategy or a step change observable behavior — API, output, persistence, ordering?
- Does every finding of severity `Hoch` have a strategy, or does the reason with which it stands under the non-goals hold?
- Bei `## Zielarchitektur`: Do its dependencies point inward, or is every exception named and justified (framework base classes, Active Record), and does the step sequence reach the target architecture?
- Bei `## Zielwerte`: Does the `**Messung:**` bullet of every step whose `**Betroffen:**` names a file from `### Hot Paths` point to a row of the target values, otherwise to „keine" (none) with a reason? Does every row of the target values carry a measurement command or „nicht gemessen" (not measured), and does a step with its own before-value name the step that produced it? A hot-path step without measurement is a blocker.

## Operations
- Feature flag or direct rollout? Who switches it, by which criterion?
- Rollback possible, and without data loss?
- Logging, metrics, alerts: how does someone notice it is broken?
- Configuration, secrets, environment variables added for all environments?

## Cut of the steps
- Is a step so large that it would itself need a plan?
- Does every step have a verifiable result, or does it end at "implementieren" (implement)?
- Is the effort proportionate to the benefit, or could the goal be reached considerably smaller?

## Language and consistency
- Does the plan use the domain terms the way the project uses them (`CONTEXT.md`, ADRs, existing code)?
- Does the plan contradict a documented decision? If so: does it justify the break?
- Are the same things named consistently throughout the plan?
