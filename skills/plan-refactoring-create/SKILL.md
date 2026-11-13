---
name: plan-refactoring-create
description: Creates a refactoring plan document in Markdown from a codebase analysis and stores it in the project — evidenced problem areas with file:line, a target architecture by Clean Architecture standards with a new folder structure, and steps that leave behavior unchanged; structurally compatible with plan-review, plan-lint and plan-coding. Optionally takes a scope (path, module or focus in quotes) and the output language as arguments; without them, the whole codebase and German.
argument-hint: "[scope] [language]"
arguments: bereich sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Bash Write Edit
---

**Output language: `$sprache`** — if it is empty or a word that clearly does not name a language, German. Everything you address to the human or write into the plan file, you write in this language: plan document, follow-up questions, abort messages, handoff. That these instructions are in English changes nothing about this.

Plan the refactoring for the scope `$bereich`. Think like an experienced developer who takes over a large, unfamiliar codebase and moves it into a clean architecture: separate concerns, increase modularity, lower coupling. The behavior stays, the structure gets better. First understand — architecture and data flow —, then gather evidence, then plan. The result is a document, not code: the improved code is produced step by step via `/plan-coding`.

## 1. Resolve the arguments

**Scope.** `$bereich` is a path (`src/billing`), a module name or a focus in quotes (`"Duplikate im Import"`). Empty means: the whole codebase. If the scope looks like a path, you check it via Glob; if it does not exist, you ask instead of silently analyzing everything.

**Language.** The output language is the last, optional argument — a language name or code in any spelling (`englisch`, `english`, `en`). The invocation is split like a shell command line, so the full invocation **$ARGUMENTS** applies: if its last word clearly names a language, that is the output language and everything before it is the scope — even if nothing comes before it. Otherwise the whole invocation is the scope, and the language stays German. If the last word could also belong to the scope („auf englisch"), do not guess: add the question to the follow-up questions from section 7 and assume German until the answer.

The language changes the text, not the structure: headings, column titles, header block labels and prose are translated; step numbers, the wave codes `W1, W2 …`, the identifiers `P1, P2 …` and the key `Claude-Session` stay, because `/plan-review`, `/plan-lint` and `/plan-coding` reference them later.

## 2. Orient yourself

A large codebase cannot be read completely, and a plan that pretends otherwise makes things up. So read in this order and stop as soon as a picture has formed: (1) directory tree to depth two, manifests, build and test commands, `CLAUDE.md`, `CONTEXT.md`, ADRs — terminology and decisions already made are binding; (2) entry points: main, routes, CLI, handlers, jobs; (3) hotspots: the thirty most frequently changed files from `git log --format= --name-only -n 500 | sort | uniq -c | sort -rn | head -30` and the thirty largest from `git ls-files -- <bereich> | xargs wc -l | sort -rn | head -30`; (4) the data flows from section 3; (5) the greps from section 4. You use Bash read-only; the only exception is the baseline run below. If the directory is not a Git repository, the hotspots are dropped, and the maintainability risks rest on size and tests.

The baseline is the state against which every step is measured later. Run the documented test or build command — from `CLAUDE.md`, README, manifest scripts or Makefile — exactly once, with a time limit of five minutes and without installing dependencies, and record command and result under „Build, Tests und Baseline": green, red with the red tests so that no step inherits them, or „unbekannt" (unknown) if no command is documented. If it runs over the limit, abort it, record that as the result and do not start a second attempt.

What you did not read goes in the architecture summary under „Nicht gelesen".

## 3. Trace the data flow

Pick two to three central flows and trace each one from the entry point to persistence, as a chain of `file:line`. Layer violations and mixed concerns show up there more reliably than in any directory structure.

## 4. Collect problem areas

Now read `${CLAUDE_SKILL_DIR}/references/problem-categories.md` in full before you record the first finding — Part A for the findings, Part B for section 5. The introduction to Part A rules what counts as evidence, where the metrics come from and what does not become a `P`. Every finding becomes a `P<n>` with location, finding, evidence and severity.

## 5. Choose strategies

Following Part B of the same reference: the smallest set of strategies that resolves every finding to which the severity scale from Part A assigns a strategy.

## 6. Derive the target architecture

From the architecture summary and the problem areas you derive the four parts of the target architecture from the template: the layers with the dependency direction pointing inward to the domain, and named, justified exceptions where the framework forces them; the responsibility per module; the folder structure before and after; the relocation table with the step per row. You count the import numbers for "after" on the planned graph, without a second grep. The structure follows the conventions of the ecosystem and the framework: where the framework dictates the layout, it stays, and the separation happens within its constraints.

## 7. Ask follow-up questions

Ask the questions that would change the plan — bundled in one message, at most five, each with your proposal and a short justification, so that a "fine" suffices as an answer: the target architecture (confirm the layer split and folder structure), the priority (structure, performance or maintainability first), the boundaries (public API, database schema, file formats that must stay unchanged), areas that must not be touched, and the budget of steps. Do **not** ask about what is in the code — you read that yourself.

If a question stays open, keep planning with your assumption, mark it as an assumption in the plan and enter the question in `## Offene Fragen`.

## 8. Determine the storage location

1. Are there already plans in the project (`docs/plans/`, `.claude/plans/`, `plans/`)? Then put it there, following the naming scheme used there.
2. Otherwise create `docs/plans/`.
3. File name `refactoring-<bereich>.md` in kebab case; without a scope `refactoring-<projektname>.md` with the name from the manifest or directory; for `english` correspondingly `refactoring-<scope>.md`.
4. If the file already exists → do not overwrite. Name it and ask whether to extend it or use a new name.

## 9. Write the plan

The complete structure is in `${CLAUDE_SKILL_DIR}/references/refactoring-template.md`; structure and order are binding. The header block carries `Refactoring`, `Bereich`, `Erstellt`, `Claude-Session: ${CLAUDE_SESSION_ID}` and `Status: Entwurf`; the step overview and the parallel overview come first. You write the file once, after the steps, waves and cross-check from sections 10 to 12 are settled. The template is German because German is the default output language; for a different output language you translate its labels while adopting them, not only during the cross-check.

## 10. Cut the steps

A step has one concern and a verifiable result and states result, affected files, dependencies, behavior guard and acceptance. For a refactoring, add: a step is one refactoring move, and structure never changes in the same step as behavior — a relocation therefore changes only paths, namespaces or packages, and imports, so that Git detects the rename and the diff stays readable. The order is characterization test before seam, seam before move, move before deletion. After every step, build and tests run at the baseline state, i.e. without new red tests. A step is too big if it would itself need a plan; too small if its result means nothing on its own.

## 11. Determine parallelism

Two steps may go into the same wave only if both hold: no functional dependency and no jointly written files. Steps on the same module never run in parallel; characterization tests for different areas may go together in W1. Form waves: W1 is all steps without a precondition, W2 everything that depends only on W1, and so on; name the touchpoints for each wave. If there is no real parallelism, write exactly that.

## 12. Cross-check

Check the composed text as if you had not written it yourself — on the text, not by re-reading the file:

- Does the sum of the steps cover the goal, and nothing beyond it?
- Do the dependencies in the overview, the waves and the steps agree?
- Does any step change observable behavior?
- Do the dependencies of the target architecture point inward, or is every exception named and justified there?
- Does every row of the relocation table have a step? Does every `P` have a strategy or a non-goal, and does every strategy have at least one step?
- Is what stayed unread listed under „Nicht gelesen"? Is every made-up detail gone, every assumption marked as an assumption, and is everything in the output language?

Fix what you find and check again until a pass turns up nothing new. For the thorough check, point to `/plan-lint` and `/plan-review` afterwards.

## 13. Hand off

Name the path of the file, the number of `P` per category and severity, the target architecture in one sentence (layers, number of relocations), the number of steps, the waves in one line, the open questions, what stayed unread and what the baseline run left behind in the working directory (`git status --porcelain`). No rehashing of the content — the plan is in the file.

## 14. Stance

The temptations of this skill: rewriting instead of refactoring; forcing a template architecture onto a framework that has its own layout; relocating everything at once; fixing bugs along the way; judging performance by gut feeling; merging coincidental similarity as a duplicate; passing off taste as a problem; planning a step after which the tests do not run; recording a finding without `file:line`; claiming completeness where samples were read. A plan with three evidenced problem areas and one open question is better than one with twelve suspected ones and none.

And before you send: are the answer and the plan file in the output language? Otherwise the German template pulls toward German and these English instructions toward English, even when something else was requested above.
