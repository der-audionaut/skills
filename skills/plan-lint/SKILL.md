---
name: plan-lint
description: Cleans up a feature plan — removes dead knowledge, unifies inconsistencies and resolves contradictions between architecture, data flow, API, schema and caching, iterating until a pass yields no new findings. Takes the plan's name as argument.
argument-hint: "[plan-file]"
arguments: plan
disable-model-invocation: true
allowed-tools: Read Grep Glob Edit
---

**Output language: the language of the plan.** You write the report, follow-up questions, abort messages and lint history in the language the plan is written in — until the plan has been read, in German. That these instructions are in English changes nothing about this.

Clean up the plan `$plan`. Read it like an experienced system architect taking over someone else's design document: does the design carry the same picture of the system throughout, or do chapter 2 and chapter 6 describe two different systems?

You are not checking here whether the plan is good — that is what `/plan-review` does. You check whether it is internally consistent, and you restore what you can restore with evidence.

## 1. Resolve the plan

**Plan.** `$plan` can be a path, a file name or a name fragment without extension.

1. `$plan` exists as a path → take it.
2. Otherwise glob `**/*$plan*.md`, preferring `docs/plans/`, `.claude/plans/`, `plans/`, `docs/`, project root.
3. Multiple hits → list them and abort instead of guessing.

**Language.** The output language is the language of the plan. The header block and headings decide — not quotes, identifiers or code blocks, which are often English in a German plan. The language changes the text, not the structure: the report format stays, only its labels are translated.

**History.** Wherever these instructions say `## Review-Historie`, `## Umsetzungs-Historie` or `## Lint-Historie`, they mean the section with that meaning, whatever language its heading is in — a plan can come from `/plan-create` in another language. You recognize an existing one in translation too — and remove it just as little —, a new one you create in the plan's language. The finding IDs T, I and W stay untranslated so that the history and the human can reference them unambiguously.

You lint the plan itself for content, not for language. A plan that mixes German and English is only an I finding when the same thing ends up with two names because of it.

Read the plan completely before you change anything. Inconsistencies only show in the whole — the section that looks wrong is often the right one.

## 2. Collect findings

Three categories; the recognition criteria are in `${CLAUDE_SKILL_DIR}/references/lint-categories.md`:

- **T — Dead knowledge.** Statements about a state that no longer exists: done TODOs, placeholders, duplicates, descriptions of already implemented steps in the future tense, references to files or fields that do not exist, sections no step uses.
- **I — Inconsistencies.** The same thing, named differently or typed differently. A field is called something else in the schema section than in the API. Step numbers that no longer add up after an insertion.
- **W — Contradictions.** Two statements that cannot both hold. Synchronous in the request and at the same time in the job. Field nullable and required. Cache validity in two places with two values.

Every finding gets a stable ID (T1, I1, W1, …), the location as section or line, and a quote of the affected sentence. A contradiction needs **both** locations.

Back up your classification with evidence. Whether a statement is dead is decided not by its age but by the codebase: does the file, the field, the function exist? Use Grep and Read instead of guessing. What you cannot back up with evidence is not a finding but a question.

## 3. Handle findings

- **T → remove.** Without replacement. If removing it leaves a reference elsewhere pointing into nothing, fixing that up belongs to the same change.
- **I → unify.** On the variant backed by the code, by `CONTEXT.md` or by an ADR. If there is no evidence, take the variant that predominates in the plan, and say in the report that this is a convention, not evidence.
- **W → resolve only if one side is demonstrably wrong.** The evidence can lie in the code, in a later decision in the `## Review-Historie` or in an implementation that has already happened. If it is statement against statement without evidence, that is a decision, not cleanup: present both sides, ask, change nothing.

**What you never remove:**

- Rationales and rejected alternatives. They look superfluous and are the reason the plan looks the way it does.
- `## Review-Historie`, `## Umsetzungs-Historie`, `## Lint-Historie`.
- Open questions to the human, including unanswered ones. Especially those.
- Requirements, just because no step covers them. That is a gap, not dead knowledge — report it, do not delete it.

When in doubt: do not remove, present it as a finding. A plan that is too full is a smaller problem than a plan missing the decisive line.

## 4. Iteration

One round is: collect → handle → read the changed plan completely again.

The re-reading is the core, because cleanup produces new findings. Removed passages leave references pointing into nothing. Unified terms suddenly turn two paragraphs into duplicates. A resolved contradiction reveals that a third section followed the same wrong side.

**Termination:** The loop ends when a complete pass yields no new T, I or W finding. Then say so clearly and stop searching.

So that it actually terminates:

- From round 2 on, no pure style findings — wording, ordering, heading levels are not lint as long as they are not ambiguous.
- A finding the human rejected is not reopened.
- After three rounds without progress you stop. Whatever is still open then is as a rule an unresolved contradiction, and that belongs to the human.

## 5. Report

```
# Plan-Lint: <Plan-Datei> — Runde <N>

**Ergebnis:** <x entfernt, y vereinheitlicht, z Widersprüche offen>
**Geprüft gegen:** <Dateien/Dokumente, die du für die Belege gelesen hast>

## Entfernt (T)
### T1 — <Titel>
- **Fundstelle:** <Abschnitt>
- **Zitat:** "<der entfernte Satz>"
- **Beleg:** <warum tot, z.B. Datei:Zeile existiert nicht / Schritt 2 abgeschlossen>

## Vereinheitlicht (I)
### I1 — <Titel>
- **Vorher:** "<Variante A>" (<Abschnitt>) / "<Variante B>" (<Abschnitt>)
- **Nachher:** <gewählte Variante>
- **Beleg:** <Datei:Zeile / ADR / Konvention im Plan>

## Widersprüche (W)
### W1 — <Titel>  [aufgelöst | offen — Entscheidung nötig]
- **Aussage A:** "<Zitat>" (<Abschnitt>)
- **Aussage B:** "<Zitat>" (<Abschnitt>)
- **Beleg / Frage:** <auflösender Beleg, oder die Frage an dich>

## Unverändert gelassen
- <was nach Finding aussah, aber keins ist, mit einem Satz warum>
```

Quote every removal. The report is the only place where what disappeared remains reconstructible — and the basis for being able to bring back a deletion without reading the diff.

## 6. History

After the last round, one compact line in `## Lint-Historie` at the end of the plan, created if it does not exist:

```
## Lint-Historie
### <Datum> — 2 Runden — 4 entfernt, 3 vereinheitlicht, 1 Widerspruch offen (W1: Cache-Gültigkeit)
```

In a plan in another language, the new section carries the translated heading — `## Lint History` — and its lines follow the plan's language; the template here is German because German is the default output language, not because the section would have to be.

No log of the individual lines — that is what version control is for. The history only records what remained open, so the next pass picks up there.

## 7. Stance

The temptation of a cleanup pass is to confuse brevity with quality. A plan is not better because it is shorter — it is better when nothing in it misleads anymore. Context, rationale and constraints are not ballast.

The second temptation is to resolve a contradiction by picking a side. That is not cleanup but an architecture decision that is not yours to make as long as no evidence supports it.

And before you send: is the answer in the language of the plan? Otherwise the English instructions pull you into English, even when the plan is German.
