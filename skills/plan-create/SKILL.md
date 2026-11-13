---
name: plan-create
description: Creates a plan document in Markdown from a feature description and stores it in the project — with session ID, step overview, parallelization overview and a design for architecture, data flow, API, schema and caching. Takes the feature description as argument, optionally followed by the output language (default German).
argument-hint: "[feature-description] [language]"
arguments: feature sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Write Edit
---

**Output language: `$sprache`** — if nothing is given there, or a word that recognizably names no language, German. Everything you address to the human or write into the plan file, you write in this language: plan document, follow-up questions, abort messages, handoff. That these instructions are in English changes nothing about this.

Plan the feature from this invocation: **$ARGUMENTS**

Think like an experienced system architect: design so that growth does not overturn the design, and then cut the smallest production-ready version into steps. The result is a document, not code — you implement nothing here.

## 1. Resolve language and feature

**Language.** The output language is the last, optional argument — a language name or code in any spelling (`englisch`, `english`, `en`). Without it, German stays, even if the feature description or the codebase is in English; whoever wants the plan in another language specifies it.

The language changes the text, not the structure: layout and order of the template stay, and so do the tables; headings, column titles, labels and prose get translated. The step numbers, the wave codes `W1, W2 …` and the key `Claude-Session` in the header block stay the same in every language, because `/plan-review`, `/plan-lint` and `/plan-coding` reference them later.

**Feature.** The feature description is free text, but the invocation is split into words like a shell command line. If the description is in quotes, the mapping is unambiguous — `$feature` is the description, `$sprache` the language — and that is how the invocation is meant: `/plan-create "Rechnungsexport als CSV" english`. Without quotes, `$sprache` holds only the second word of the description; then the full invocation above counts:

1. If its last word recognizably names a language, that is the output language and everything before it is the description.
2. Otherwise the whole invocation is the description, and the language stays German.
3. If the last word could also belong to the description ("Login-Seite auf englisch"), don't guess: add the question to the follow-up questions from section 3 and assume German until answered.

## 2. Understand before you plan

First the codebase, then the plan. A plan written without looking into the project invents file paths and patterns.

- Where in the system does the feature sit? Which modules, endpoints, tables does it touch?
- Does it already half exist? Grep for neighboring functionality before you plan something new.
- Which patterns apply here — layering, error handling, tests, migrations?
- Read `CLAUDE.md`, `CONTEXT.md`, ADRs and existing plans in the project. Terminology and decisions already made are binding.

## 3. Ask follow-up questions

Ask the questions that would change the plan — bundled in one message, at most five, each with your proposal and a short justification, so that a "passt" ("fine") suffices as an answer.

Ask about: product decisions, scope and boundaries, behavior in special cases, compatibility promises, priorities. Do **not** ask about what is in the code — you read that yourself.

If a question remains open, keep planning with your assumption, mark it in the plan as an assumption and enter the question under `## Offene Fragen`. A plan waiting for an answer is worthless; an unmarked assumption is dangerous.

## 4. Determine the storage location

1. Are there already plans in the project (`docs/plans/`, `.claude/plans/`, `plans/`)? Then put it there, in the naming scheme used there.
2. Otherwise create `docs/plans/`.
3. File name from the feature description in kebab case, short and descriptive, in the output language: `rechnungsexport-csv.md`, with `english` `invoice-export-csv.md` — unless the project prescribes a different naming scheme.
4. If the file already exists → do not overwrite. Name it and ask whether to extend it or use a new name.

## 5. Write the plan

The full structure is in `${CLAUDE_SKILL_DIR}/references/plan-template.md`. Binding are:

- **Header block** with feature, date, status and `Claude-Session: ${CLAUDE_SESSION_ID}` — so it can later be traced which session the plan came from.
- **Step overview at the very top**, as a table, before any prose. Whoever opens the plan sees the work steps first.
- **Parallel overview** directly below, in waves.
- Then goal, non-goals, current state, design, the steps in detail, test strategy, rollout, risks, open questions.

The design covers architecture and placement, component split, data flow, interfaces, data model and migration, and caching — each only as far as this feature touches it. Caching only gets a section if there is a proven need; otherwise one sentence on why not. Whatever the design provides for but the feature does not need yet belongs under non-goals, not in a step.

The template is German, because German is the default output language. For a different output language you translate its headings, column titles and header block labels while carrying them over — not only during the cross-check.

## 6. Cut the steps

- A step has **one** concern and a verifiable result. If the description ends at "implement", it is not thought through.
- After each step the system runs. No broken intermediate state.
- Each step names: result, affected files, dependencies, acceptance criterion.
- Migration before use, interface before consumer, feature flag before rollout.
- A step is too big if it would itself need a plan. Too small if its result means nothing on its own.

## 7. Determine parallelism

Two steps may only go into the same wave if **both** hold: no functional dependency, and no files written by both. Changing the same file at the same time is no parallelization gain but a merge conflict with a detour.

Typical collision points that never run in parallel: migrations and schema changes, central configuration, route and container registration, shared types.

Form waves from this: W1 is all steps without a precondition, W2 everything that depends only on W1, and so on. For each wave, name the steps and the files where they could touch. If no real parallelism results, write exactly that — an invented wave costs more later than an honest chain.

## 8. Cross-check

Read the written plan once completely, as if you had not written it yourself, and check:

- Does the sum of the steps cover the goal, and nothing beyond it?
- Do the dependencies in the overview match those in the steps? And the waves?
- Is every invented detail gone — files, fields, functions you have not verified?
- Does every assumption stand as an assumption?
- Is the plan in the output language throughout — including the headings and table headers that come from the German template?

Fix what you find and read again. You are done when a pass turns up nothing new. For the thorough review, point to `/plan-lint` and `/plan-review` afterwards — you don't rebuild those here.

## 9. Hand off

Name the path of the file, the number of steps, the waves in one line and the open questions. No rehashing of the content — the plan is in the file.

A plan with three steps and one open question is a better result than one with twelve steps and none. If the feature is described too vaguely to cut steps, say so instead of inventing structure.

And before you send: are the answer and the plan file in the output language? The English instructions and the German template otherwise pull toward their own language, even when something else was requested above.
