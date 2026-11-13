---
name: plan-optimization-create
description: Creates an optimization plan document in Markdown from measurement and codebase analysis and stores it in the project — baseline from the documented measurement commands, evidenced problem areas with file:line and the classification measured or suspected, target values per metric, optimization strategies, and steps that leave behavior unchanged and each carry a measurement before and after; structurally compatible with plan-review, plan-lint and plan-coding. Optionally takes a scope (path, module or focus in quotes) and the output language as arguments; without them, the whole codebase and German.
argument-hint: "[scope] [language]"
arguments: bereich sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Bash Write Edit
---

**Output language: `$sprache`** — if nothing is there, or a word that clearly does not name a language, German. Everything you address to the human or write into the plan file, you write in this language: plan document, follow-up questions, abort messages, handoff. That these instructions are in English changes nothing about this.

Plan the optimization for the scope `$bereich`. Think like a performance engineer: measure first, then search, then plan. The goals are runtime, memory and scaling. Behavior stays, the numbers move. The result is a document, not code: the faster code is built step by step via `/plan-coding`, and every step carries its measurement before and after.

## 1. Resolve the arguments

**Scope.** `$bereich` is a path (`src/search`), a module name or a focus in quotes (`"Import-Durchsatz"`). Empty means: the whole codebase. If the scope looks like a path, check it via Glob; if it does not exist, ask instead of silently analyzing everything. Also determine the location and file name now per section 8 and check via Glob whether the file already exists — if so, ask before the test and measurement runs, instead of measuring for five minutes and discarding the result.

**Language.** The output language is the last, optional argument — a language name or code in any spelling (`englisch`, `english`, `en`). The invocation is split like a shell command line, so the full invocation **$ARGUMENTS** applies: if its last word clearly names a language, that is the output language and everything before it is the scope — even if nothing comes before it. Otherwise the whole invocation is the scope, and the language stays German. If the last word could also belong to the scope („auf englisch"), do not guess: add the question to the follow-up questions from section 7 and assume German until answered.

The language changes the text, not the structure: headings, column titles, header block labels and prose are translated; step numbers, the wave codes `W1, W2 …`, the identifiers `P1, P2 …` and the key `Claude-Session` stay, because `/plan-review`, `/plan-lint` and `/plan-coding` reference them later.

## 2. Orient

A large codebase cannot be read completely, and a plan that pretends to have done so invents. Read in this order and stop as soon as a picture forms: (1) directory tree to depth two, manifests, build, test and measurement commands, `CLAUDE.md`, `CONTEXT.md`, ADRs — terminology and decisions already made are binding; (2) measurement tools: benchmark, profiling and performance test files, measurement results, metrics and logs stored in the repository; (3) entry points: main, routes, CLI, handlers, jobs, render components; (4) the hot paths from section 3; (5) the data volumes of these paths from fixtures, migrations and configuration, because a pattern only matters above a certain volume; (6) targeted greps along the hot paths for section 4. You use Bash read-only; the exceptions are the test run and measurement run below.

Numbers in the plan have exactly two sources: measured values from the measurement run or from stored measurement results, and data volumes from fixtures, migrations and configuration. What the baseline does not output is `Verdacht` (suspected; Part A of the catalogue).

**Test run.** Run the documented test or build command — from `CLAUDE.md`, README, manifest scripts or Makefile — exactly once, with a time limit of five minutes and without installing dependencies, and record command and result under „Build und Tests": green, red with the red tests so that no step inherits them, or „unbekannt" if no command is documented. If it exceeds the limit, abort, record that, and do not start a second attempt.

**Measurement run.** Run the documented measurement or benchmark command from the same sources, with a time limit of five minutes for the entire measurement (`timeout 300 <command>`, the first run under `time`) and without installing anything. If the first run took less than a minute, repeat it twice and take, per metric, the median of three runs with the range as spread; otherwise it stays at one run, and the baseline carries the note „Einzelmessung, Streuung unbekannt". Run and environment go under „Baseline-Messung", as the template prescribes; the values per metric go only in the target values. If there is no command, the baseline reads „nicht gemessen", and every finding is `Verdacht`; if the command exceeds the limit or fails red, the same applies, with the reason next to it. After the run, check `git status --porcelain`; you do not clean up what it left behind.

What you did not read goes in the architecture summary under „Nicht gelesen".

## 3. Trace hot paths

Pick two to three paths — first the ones the measurement command measures, otherwise the entry points with the largest data volume or frequency — and trace each from entry to result as a chain of `file:line`, with the data volume at each station. This shows which loop runs over which volume: a quadratic search over ten elements is not a finding; over ten thousand, it is the main operation. Whatever lies on none of these paths is a cold path.

## 4. Collect problem areas

Now read `${CLAUDE_SKILL_DIR}/references/optimization-categories.md` in full before you note the first finding — Part A for the findings, Part B for section 6. The introduction of Part A governs what counts as `gemessen` (measured) and what as `Verdacht`, and what does not become a `P`. Every finding becomes a `P<n>` with location, finding, evidence, severity and the classification `gemessen` or `Verdacht`. Search the files of the hot paths and their direct callers; you do not search cold paths — whatever you come across there while reading becomes a `Niedrig` (low) `P` under non-goals.

## 5. Set target values

For each metric that a hot path from section 3 moves, one row of the target values: baseline value with spread, target, measurement command; you do not carry over other outputs of the measurement command. The target is a follow-up question from section 7 with your proposal relative to the baseline — „halbieren", „unter 200 ms" —, because local numbers are not production; without an answer, „messbare Verbesserung, Zielwert offen" stands as an assumption. Without a baseline, the row carries „nicht gemessen" and the measurement command that the measurement step sets up. Numbers appear only here; problem areas and steps refer to the row.

## 6. Choose strategies

Per Part B of the same reference: the smallest set of strategies that resolves every finding to which the severity scale from Part A assigns a strategy.

## 7. Ask follow-up questions

Ask the questions that would change the plan — bundled in one message, at most five, each with your proposal and a short justification, so that a „passt" suffices as an answer: the target values per metric, the priority (runtime, memory or scaling first), the boundaries (result format, ordering, floating-point precision, interfaces that must remain unchanged), areas that must not be touched, and the budget of steps. Do **not** ask about what is in the code or what the measurement shows — you read and measure that yourself.

If a question remains open, continue planning with your assumption, mark it in the plan as an assumption and enter the question in `## Offene Fragen`.

## 8. Determine the location

1. Are there already plans in the project (`docs/plans/`, `.claude/plans/`, `plans/`)? Then put it there, in the naming scheme used there.
2. Otherwise create `docs/plans/`.
3. File name `optimierung-<bereich>.md` in kebab case; without a scope `optimierung-<projektname>.md` with the name from the manifest or directory; for `english` correspondingly `optimization-<scope>.md`.
4. If the file already exists → do not overwrite. Name it and ask whether to extend it or use a new name.

## 9. Write the plan

The complete structure is in `${CLAUDE_SKILL_DIR}/references/optimization-template.md`; structure and order are binding. The header block carries `Optimierung`, `Bereich`, `Erstellt`, `Claude-Session: ${CLAUDE_SESSION_ID}` and `Status: Entwurf`; step overview and parallel overview are at the very top. You write the file once, after the steps, waves and cross-check from sections 10 to 12 are settled. The template is German because German is the default output language; for another output language, translate its labels as you carry them over, not only during the cross-check.

## 10. Cut steps

A step has one concern and a verifiable result and names result, affected files, dependencies, behavior guard, measurement and acceptance. For an optimization, add: without a baseline, step 1 sets up the measurement, before anything else; if a hot path lacks a test, a characterization test precedes it; a step is one optimization on one path, and the three rules from the introduction of Part B apply per step — never structure together with behavior, after-measurement with the same command as the baseline. The `Messung` point refers to the row of the target values instead of repeating numbers; for measurement and test steps it reads „keine — <Grund>". After each step, build and tests run at the state of the baseline, i.e. without new red tests. A step is too large if it would itself need a plan; too small if its result on its own moves no metric.

## 11. Determine parallelism

Two steps may only go into the same wave if both hold: no functional dependency and no jointly written files. Steps on the same hot path never run in parallel, because otherwise no measurement can be attributed to a step; measurement and characterization-test steps of different paths may go together in W1. Form waves: W1 is all steps without a precondition, W2 everything that depends only on W1, and so on; name the touchpoints for each wave. If there is no real parallelism, write exactly that.

## 12. Cross-check

Check the composed text as if you had not written it yourself — on the text, not by rereading the file:

- Does the sum of the steps cover the goal, and nothing beyond it?
- Do the dependencies in overview, waves and steps agree?
- Does any step change observable behavior?
- Does every step have the `Messung` point, and on a hot path does it refer to a row of the target values? Is every before-value measured, not estimated?
- Does every cache have an invalidation path and a size limit? Does every `Hoch` (high) finding have a strategy, and does no strategy lie on a cold path?
- Does every `P` have a strategy or a non-goal, and does every strategy have at least one step? Do numbers appear only in the target values?
- Does „Nicht gelesen" list what remained unread? Is every invented detail gone, every assumption marked as an assumption, and is everything in the output language?

Correct what you find and check again until a pass yields nothing new. For the thorough review, point afterwards to `/plan-lint` and `/plan-review`.

## 13. Hand off

Name the path of the file, the number of `P` per category with the classification `gemessen` or `Verdacht`, the target values in one line (metric: baseline → target), the number of steps, the waves in one line, the open questions, what remained unread, and what the test and measurement runs left behind in the working directory (from the `git status --porcelain` after the runs). No rehashing of the content — the plan is in the file.

## 14. Stance

The temptations of this skill: optimizing without measurement; trusting a single run; polishing cold paths; planning a cache without invalidation; changing ordering or floating-point results on the side; using micro-benchmarks for macro claims; inventing target values instead of asking; fixing bugs on the side; noting a finding without `file:line`; claiming completeness where samples were read. A plan with two measured problem areas and one open question is better than one with twelve suspected cases and none.

And before you send: are the answer and the plan file in the output language? Otherwise the German template pulls toward German, even if something else was requested above.
