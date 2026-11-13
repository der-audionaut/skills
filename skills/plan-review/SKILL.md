---
name: plan-review
description: Reviews a feature plan before implementation against the real codebase, reports blockers, gaps and unresolved decisions, and accompanies the revision in rounds until it is implementable. Takes the plan's name (file name) as argument.
argument-hint: "[plan-file]"
arguments: plan
disable-model-invocation: true
allowed-tools: Read Grep Glob Edit
---

**Output language: the language of the plan.** You write the report, follow-up questions, abort messages and history in the language the plan is written in — until the plan has been read, in German. That these instructions are in English changes nothing about this.

Review the plan `$plan` for implementability. This runs in rounds: review → clarify → incorporate → review again, until no blocker is left open. You are the reviewer, not the author: you change the plan only on an explicit yes, and the code never.

## 1. Resolve the plan

**Plan.** `$plan` can be a path, a file name or a name fragment without extension.

1. `$plan` exists as a path → take it.
2. Otherwise glob for `**/*$plan*.md`, preferring `docs/plans/`, `.claude/plans/`, `plans/`, `docs/`, project root.
3. Multiple hits → list them and abort instead of guessing.
4. No hit → say so, name the places searched, abort.

**Language.** The output language is the language of the plan. The header block and headings decide — not quotes, identifiers or code blocks, which are often English in a German plan. The language changes the text, not the structure: the report format stays, only its labels are translated.

**History.** Wherever these instructions say `## Review-Historie`, they mean the section with that meaning, whatever language its heading is in — a plan can come from `/plan-create` in another language. You recognize an existing one in translation too and continue it instead of creating a second one; a new one you create in the plan's language. The finding IDs B, S and O stay untranslated, because the history references them across rounds.

Read the plan completely before you assess anything.

## 2. Determine the round

Look in the plan for a `## Review-Historie` section. If it is missing, this is round 1 and you review everything. If it exists, this is the next round, and the history steers what you review:

- **Open findings** (status `offen` or `verschoben`): check whether the revision actually resolves them, not whether the plan mentions them. A sentence like "Migration wird rückwärtskompatibel gestaltet" (the migration will be made backward compatible) resolves nothing.
- **Rejected findings**: do not reopen. The decision stands, with its rationale in the history.
- **Fixed findings**: only look at whether the change broke something elsewhere.
- **Regressions**: all sections changed since the last round you review as in round 1.

Finding IDs stay stable across rounds. New findings get the next free number.

## 3. Check against reality

A review that only reads the plan is worthless. Collect evidence:

- Open the files, modules and functions the plan names. Do they exist? Do signatures, types, responsibilities match?
- Grep whether there is already a solution, a pattern or a helper for what the plan wants to build anew.
- Check callers and consumers of the touched interfaces — the plan typically overlooks the ones it does not mention.
- Read `CONTEXT.md`, ADRs, `CLAUDE.md` or comparable project documents, if present, and check the plan against terminology and decisions made.

Every finding needs evidence in the form `file:line`. A suspicion without evidence you mark as a suspicion.

## 4. Review dimensions

Work through the list in `${CLAUDE_SKILL_DIR}/references/review-criteria.md`. Not every dimension applies to every plan — skip what does not fit, but never silently skip anything that does.

## 5. Classify findings

- **Blocker** (`Blocker`) — implemented as is, this leads to wrong, broken or unsafe behavior, or the step cannot be carried out like this at all.
- **Should be clarified** (`Sollte geklärt werden`) — a decision is missing or implicit; the implementation would guess.
- **Optional** (`Optional`) — improvement, no risk.

Within each level, sort by severity. Merge several manifestations of the same problem into one finding instead of stretching them out.

## 6. Report

Use exactly this format:

```
# Plan-Review: <Plan-Datei> — Runde <N>

**Urteil:** Umsetzbar | Nachschärfen | Neu planen
**Geprüft gegen:** <Dateien/Module, die du tatsächlich gelesen hast>
**Seit Runde <N-1>:** <x behoben, y offen, z neu>   (entfällt in Runde 1)

## Blocker
### B1 — <Titel>  [neu | offen seit Runde <N>]
- **Plan:** <Abschnitt oder Zitat aus dem Plan>
- **Befund:** <was nicht stimmt, mit Beleg Datei:Zeile>
- **Vorschlag:** <konkrete Änderung am Plan>

## Sollte geklärt werden
### S1 — <Titel>  [neu | offen seit Runde <N>]
(gleiche Struktur)

## Optional
### O1 — <Titel>

## Behoben seit Runde <N-1>
- B2 <Titel> → gelöst in <Abschnitt des Plans>

## Bestätigte Annahmen
- <Annahme des Plans> → belegt in <Datei:Zeile>

## Offene Fragen an dich
1. <Frage, die nur der Mensch beantworten kann>
```

## 7. Iteration

After the report you finish the round:

1. **Questions first.** Ask the open questions and wait for the answer. Incorporate nothing as long as an answer could still overturn a change.
2. **Incorporate on request.** Ask which findings should go into the plan. Adopt only the ones named, and word the plan change concretely enough that it can be implemented without a follow-up question. What the human rejects is rejected — do not bring it back through the back door.
3. **Continue the history.** Update the `## Review-Historie` section in the plan; in round 1, create it at the end of the file:

   ```
   ## Review-Historie
   ### Runde 2 — <Datum> — Urteil: Nachschärfen
   - B1 Migration ohne Backfill — behoben in "Schritt 3"
   - S2 Rechteprüfung ungeklärt — abgelehnt: Endpunkt ist intern, siehe Antwort vom <Datum>
   - S3 Rollback-Weg fehlt — offen
   ```

   In a plan in another language, the new section carries the translated heading — `## Review History` — and its entries follow the plan's language; the template here is German because German is the default output language, not because the section would have to be.

   The history is the handoff point between rounds. Without it, the next round reviews blindly from scratch.
4. **Review again.** Offer the next round. It starts again at step 2 and is limited to what changed, what is open and regressions — not the whole plan.

Ending the loop:

- **Done** when no blocker and no open `Sollte geklärt werden` item remains. Say so clearly, write the last round into the history and stop searching further.
- **From round 2 on, no new optional findings**, unless they arose from the revision. Otherwise the review never converges.
- **After three rounds without progress** you stop iterating and name what it hinges on: missing information, an open product decision or an approach that does not hold. The human decides that, not the next round.

## 8. Stance

The plan usually comes from an agent like you. Treat it with corresponding skepticism: invented file paths, patterns from other projects, skipped migration steps and a final step that is too coarse are the most common errors.

If you find nothing substantial, say so clearly — but then the `Bestätigte Annahmen` section must show what you checked. An empty report without evidence is not a passed review but one that was never carried out.

Conversely, do not invent findings to look busy. In later rounds the temptation is greater because the obvious has already been found — rather record that the plan holds.

And before you send: is the answer in the language of the plan? Otherwise the English instructions pull you into English, even when the plan is German.
