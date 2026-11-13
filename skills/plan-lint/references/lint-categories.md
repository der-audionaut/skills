# Lint categories

Recognition criteria for the three kinds of finding. Part D at the end concerns the architecture level and is the richest source of contradictions, because the same decision is described there in several places.

---

## T — Dead knowledge

The hallmark is always: the statement describes a state that no longer exists or never existed.

- **Done work in the future tense.** An implemented step that still describes what *is to be* built. Cross-check against `## Umsetzungs-Historie` and the code.
- **Settled questions.** TODOs, "muss noch geklärt werden" (still needs clarifying), "abhängig von X" (depends on X) — where the answer now stands elsewhere in the plan or in the history.
- **References into nothing.** Named files, functions, fields, endpoints, tables, tickets that do not exist (anymore). Check with Grep, do not estimate.
- **Placeholders.** "TBD", "hier später ergänzen" (fill in later), empty sections, example values from a template.
- **Duplicates.** The same matter spelled out in two places. The weaker version goes, the stronger one stays — not the other way round just because it is higher up.
- **Orphaned context.** A section that no step and no decision uses anymore, because that part of the feature was dropped.
- **Outdated descriptions of state.** "Aktuell macht das der Legacy-Service" (currently the legacy service does this) — where the legacy service was replaced in step 1.

**Not dead knowledge:** rationales, rejected alternatives with rationale, non-goals, risks, open questions to the human, requirements without a corresponding step (that is a gap and gets reported, not deleted), plan-internal references into nothing — a step number, a `W<n>` or a `P<n>` that does not exist (that is numbering, see I, and stays).

---

## I — Inconsistencies

Hallmark: the same thing, represented differently. Both places may be meant correctly.

- **Naming.** `invoice_status` / `billingState` / "Rechnungsstatus" for one field. Cross-check with code and `CONTEXT.md`.
- **Types and formats.** Field as integer in the schema section, as string in the API. Timestamps sometimes with, sometimes without time zone.
- **Numbering and ordering.** Steps that jump after an insertion; references to "step 4" that is now step 5, or to a `P<n>` that does not exist under `## Problemfelder`. Unify only when the target is unambiguous, otherwise present it and leave it — the reference may indicate a missing problem area.
- **Problem areas** (refactoring plans). A `P` without strategy and without non-goal — present, do not delete: the strategy or the reason is missing.
- **Granularity.** One step in five sub-points, the next as one sentence — a hint that the second one has not been thought through.
- **Register.** Domain term here, technical synonym there, for the same concept.
- **Structure.** Steps with acceptance criterion and steps without, in the same plan.

**Not a finding:** the progress marking from `/plan-coding` — `W1 ✓ erledigt` or `W1 ✗ abgebrochen` in the step overview next to `W1` in the parallel overview, the `**Status:**` bullet in the detailed step, the header block status `In Umsetzung` or `Umgesetzt`. The wave identifier is the same; the suffix is state, not a second name, and it stays.

---

## W — Contradictions

Hallmark: the two statements cannot both be true. Always quote both locations.

- **Execution model.** Synchronous in the request in one place, asynchronous in the job in the other.
- **Data model.** Field optional versus field required; a single value versus a list; unique versus multiple.
- **Source of truth.** Two sections that describe different systems as authoritative.
- **Numeric values.** Validity durations, limits, page sizes, time windows stated differently in two places.
- **Ordering.** Step A requires step B, while according to the plan B comes after A.
- **Scope.** Something is in the non-goals or excluded by a commitment of the plan — such as „Verhalten bleibt unverändert" (behavior stays unchanged) or „kein Refactoring-Gegenstand" (not subject to refactoring) — and reappears as a task in a step.
- **Against documented decisions.** The plan contradicts an ADR or a decision from the `## Review-Historie` without naming the break.

---

## D — Architecture level

The same decision usually appears four or five times in a plan, in different chapters. That is exactly where the versions drift apart:

- **Architecture versus data flow.** Does the data flow describe paths the component split does not allow?
- **Data flow versus API.** Do the described endpoints deliver the data the flow assumes at that point?
- **API versus schema.** Is every API field present in the data model, with matching type and matching nullability? And conversely: fields in the schema that nobody ever reads?
- **Schema versus access patterns.** Do described queries assume indexes or relations the schema does not have?
- **Caching versus data flow.** Who writes the data, and does invalidation in the described order even arrive before the next read? A cache without a named invalidation path is a contradiction, not a detail.
- **Caching versus consistency promise.** The plan promises immediate visibility in one place and sets a validity duration in another.
- **Design versus step sequence.** Does the design describe an end state the steps together never reach — such as a row of the relocation table without a step? And conversely: a step that leads to a place the design — in a refactoring plan, the folder tree — does not know?
