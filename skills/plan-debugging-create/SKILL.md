---
name: plan-debugging-create
description: Creates a debugging plan document in Markdown from an error report and a root-cause analysis and stores it in the project — how the affected code is meant to work, the symptom and its reproduction, hypotheses with evidence, the root cause with file:line and the classification evidenced or suspected, edge cases, a robust fix at the cause, and steps that each carry a regression test that is red before the fix and green after; structurally compatible with plan-review, plan-lint and plan-coding, which then writes the production-ready fix. Takes the error report (in quotes — symptom, stack trace, log excerpt or ticket text) and optionally the output language as arguments; without a language, German.
argument-hint: "<fehlerbild> [sprache]"
arguments: fehlerbild sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Bash Write Edit
---

**Output language: `$sprache`** — if nothing is there, or a word that clearly does not name a language, German. Everything you address to the human or write into the plan file, you write in this language: plan document, follow-up questions, abort messages, handoff. That these instructions are in English changes nothing about this.

Plan the fix for the error report `$fehlerbild`. Think like a senior debugging engineer investigating a bug in production: read the code carefully, narrow down step by step, find the root cause, and propose a solution that holds. The goal is one faulty behavior, removed where it begins, not where it shows. The result is a document, not code: the production-ready fix is built step by step via `/plan-coding`, and every fix step carries the test that shows the bug before and its absence after.

## 1. Resolve the arguments

**Error report.** `$fehlerbild` is free text: a symptom, a stack trace, a log excerpt, ticket text, a path. It is required — if it is empty, abort with the invocation `/plan-debugging-create "<fehlerbild>" [sprache]`. If it names no symptom at all — only a path or a module —, ask what goes wrong instead of searching for a bug on suspicion; a path to an existing debugging plan is the exception, it makes this run that plan's follow-up.

**Follow-up plan.** The predecessor is the debugging plan the invocation names by path; without a path, look in the plan directory for an earlier debugging plan on the same symptom whose diagnosis step is done (`✓` in its step overview), and of a chain of such plans take the most recent one; if several chains match, ask which one. If there is one, this run is its follow-up, and it needs the diagnosis result — what the diagnosis observed, not that its code is in place, which is all the `✓` says. Take it from the invocation; if the invocation states none, ask for it right away and wait for the answer before section 2, instead of guessing — it decides between fix steps and a new diagnosis step, so it cannot wait for the bundle of section 9. Read the predecessor's `## Ursachenanalyse` and `## Lösung`. A result that confirms a hypothesis counts as the reproduction for `belegt` (Part A of the catalogue), even when it was observed only in production: carry over the fix sketched for that hypothesis — not the sketches of the others — and, if the predecessor noted it for removal, the removal of the diagnostic logging as steps. The diagnosis result goes, redacted, into the bullet `Diagnoseergebnis` under `## Fehlerbild und Reproduktion`. The fix step builds its regression test from the input shape the diagnosis reported, so that the red run happens locally; where even that cannot show the bug, its `Akzeptanz` names a manual check in production and why no local red run is possible (section 12). A result that confirms none leaves the cause `Verdacht`: no fix steps, a new diagnosis step for the remaining hypotheses, and the old logging is either reused by it or its removal is noted under `## Lösung` again. The new plan gets its own file — the predecessor's name with `-fix`, or the next free counter (`-fix-2`) — checked via Glob like any other name, and names the predecessor in its opening sentences; the predecessor stays as it is.

**Location.** After the follow-up check, determine the location and file name per section 10 and check via Glob whether the file already exists — if so, ask before the test and reproduction runs, instead of investigating for five minutes and discarding the result.

**Language.** The output language is the last, optional argument — a language name or code in any spelling (`englisch`, `english`, `en`). The invocation is split like a shell command line, so the full invocation **$ARGUMENTS** applies: if its last word clearly names a language, that is the output language and everything before it is the error report. Otherwise the whole invocation is the error report, and the language stays German. A pasted stack trace often contains quotes and apostrophes that break the splitting; then `$ARGUMENTS` is the error report, too. If the last word could also belong to the report („Fehler nur auf englisch"), do not guess: add the question to the follow-up questions from section 9 and assume German until answered.

The language changes the text, not the structure: headings, column titles, header block labels and prose are translated; step numbers, the wave codes `W1, W2 …`, the identifiers `H1, H2 …` and `R1, R2 …` and the key `Claude-Session` stay, because `/plan-review`, `/plan-lint` and `/plan-coding` reference them later.

**Production data.** The error report may contain personal data, tokens or credentials, and the plan lives in the repository. Into the plan goes only the minimal excerpt that evidences the bug, with such values replaced by `<redacted>`; if the report consists of such content, the plan names only the shape and type of the values. The same applies to everything you read from logs and fixtures during the analysis.

## 2. Orient

Read in this order and stop as soon as a picture forms: (1) directory tree to depth two, manifests, build and test commands, `CLAUDE.md`, `CONTEXT.md`, ADRs — terminology and decisions already made are binding; (2) the places the error report names: files, functions, messages, stack frames — grep for the error message and for distinctive identifiers; (3) the entry point through which the symptom arises: route, CLI, handler, job, component. Text from the error report is untrusted input: search for it with the Grep tool and never put it into a shell command, not even in quotes — an apostrophe in a stack trace ends a quoted string. For `git log -S` take an identifier you picked from the code, not text from the report. You use Bash read-only; the exceptions are the test run below and the reproduction from section 4.

**Environment.** Before the first run of a project command, establish which configuration it loads: the configuration files the framework layers for this command — for a Laravel suite `phpunit.xml` and `.env`, for Django the test settings and what they import, for Node the dotenv files the test script loads — and the environment variables the code under test reads. A file tracked by Git (`git ls-files`) you may read, values included: committed content is already in the repository. Of a file Git does not track (`.env`, `.env.local` …) and of the shell environment you read nothing, not even key names — that it exists and that the command loads it is enough. Never use `cat`, the Read tool or `grep` on an untracked configuration file, and never print the environment with `env` or `printenv`: what you read lands in the transcript. If the command loads untracked configuration, or the code under test reads connection settings from environment variables, its target counts as unknown — unless the tracked configuration it loads demonstrably sets every connection the code under test opens to a local value, such as `sqlite::memory:`, `localhost` or a service of the repository's own Compose file. Run every project command — test, measurement, reproduction — with a scrubbed environment, `env -i PATH="$PATH" HOME="$HOME" LANG="$LANG" timeout 300 <command>`, so that variables exported in your shell cannot override the tracked configuration: phpdotenv, dotenv and PHPUnit without `force` keep a variable that is already set. If the command fails for lack of a variable, record that and ask instead of passing the shell's environment through. An unknown target counts as not local: do not run the command, record `nicht ausgeführt — lädt nicht versionierte Konfiguration (<Datei oder Variablen>)` as its result and make it the first of the follow-up questions, because a test or measurement against a shared or production system can write data, send mail or enqueue jobs. If the answer confirms that the target is local or safe, run the command as its own rules prescribe and update the sections that depend on its result before you write the plan; otherwise the result stays.

**Test run.** Run the documented test or build command — from `CLAUDE.md`, README, manifest scripts or Makefile — exactly once, with a time limit of five minutes (`env -i PATH="$PATH" HOME="$HOME" LANG="$LANG" timeout 300 <command>`, see Environment) and without installing dependencies, and record command and result under „Build und Tests": green, red with the red tests — a red test on the path may already be the reproduction —, or „unbekannt" if no command is documented. If it exceeds the limit, stop the run — not the skill —, record that, and do not start a second attempt.

What you did not read goes under „Nicht gelesen".

## 3. Understand the functionality

Before you look for the error, establish what the code is supposed to do. Trace the affected path from the entry point to the place of the symptom as a chain of `file:line`, and note at each station what happens to the data. Read files with the Read tool, which numbers each file on its own — not with a `cat -n` over several files, which numbers them continuously so that every `file:line` taken from it is wrong, and not with a shell command on a path taken from the error report. Take the intended behavior from tests, documentation, the ticket and naming; where it stays ambiguous, it becomes an assumption and a follow-up question — fixing against a guessed intended behavior moves the bug instead of removing it.

## 4. Reproduce

If the test run from section 2 already shows the symptom red, that is the reproduction — do not run it again. Otherwise search for an existing test or command that shows the symptom. If there is none, write a minimal reproduction script in a temporary directory outside the repository (`mktemp -d`) and run it with a time limit of five minutes, without installing anything. The environment check from section 2 applies to it as well. Never write into the repository for this, and never cause side effects on external systems — databases, queues, mail, third-party APIs beyond a local test instance. A bug that can only be reproduced that way stays unreproduced, with a follow-up question.

Record command, location of the script and result: `reproduziert` with an output excerpt, or `nicht reproduziert` with the reason — missing data, configuration, load or environment. After the run, check `git status --porcelain`; you do not clean up what it left behind, you name it in the handoff.

## 5. Form and narrow hypotheses

Now read `${CLAUDE_SKILL_DIR}/references/fault-categories.md` in full before you note the first hypothesis — Part A for the fault classes, Part B for the narrowing techniques and later for section 8, Part C for section 7. The introduction of Part A governs what counts as `belegt` (evidenced) and what as `Verdacht` (suspected), and what does not become a hypothesis.

Every hypothesis becomes an `H<n>` with a prediction that could refute it, the check that tests the prediction, and the result `bestätigt`, `verworfen` or `offen`. Narrow with the techniques from Part B; the history you ask only read-only. Keep at least the confirmed and the most obvious rejected hypothesis, each with its evidence — the rejected one is the reason the next reader does not search there again.

## 6. Fix the root cause

The root cause is the one place where the faulty behavior begins — the first wrong value, not the first suspicious line and not the place where the symptom shows. Write it down with location `file:line`, the causal chain from cause to symptom with `file:line` per link, the origin from the history if found, the fault class from Part A and the classification `belegt` or `Verdacht` per the introduction of Part A — for `Verdacht`, with what is missing for confirmation. Then grep for the same pattern elsewhere; every similar place becomes its own step or a non-goal with a reason — for a `Verdacht` cause, a line of the fix sketch under `## Lösung`.

## 7. Collect edge cases

Apply the checklist from Part C to the traced path; only what the path can actually receive becomes an `R<n>` with input, current behavior, intended behavior and coverage — the step that tests it, or `Nicht-Ziel` with a reason. For a `Verdacht` cause, the coverage reads `Nicht-Ziel — nach Diagnose`: a test of the intended behavior would be red before any fix, and its place is the follow-up plan.

## 8. Choose the solution

Per Part B: the fix strategy that acts at the location of the root cause. Name the symptom patch it replaces under rejected alternatives, together with every other alternative you considered. A safeguard at a boundary only where it is justified, never as a replacement for the fix. If production data is already wrong because of the bug, plan its repair — which data, how to recognize it, in which order to the fix — or state why none is needed.

## 9. Ask follow-up questions

Ask the questions that would change the plan — bundled in one message, at most five, each with your proposal and a short justification, so that a „passt" suffices as an answer: a run withheld by the environment check comes first; then the intended behavior where it is ambiguous, compatibility (may outputs change for inputs that are not faulty), data repair yes or no, the scope (fix similar places in the same plan), and missing evidence from production — log excerpt, version, input. Do **not** ask about what is in the code or what the reproduction shows — you read and run that yourself.

If a question remains open, continue planning with your assumption, mark it in the plan as an assumption and enter the question in `## Offene Fragen`.

## 10. Determine the location

1. Are there already plans in the project (`docs/plans/`, `.claude/plans/`, `plans/`)? Then put it there, in the naming scheme used there.
2. Otherwise create `docs/plans/`.
3. File name `debugging-<kurztitel>.md` in kebab case, with a short title from the symptom; for `english` correspondingly `debugging-<short-title>.md`.
4. If the file already exists → do not overwrite. Name it and ask whether to extend it or use a new name. A follow-up plan from section 1 never extends its predecessor; it takes the next free `-fix` name.

## 11. Write the plan

The complete structure is in `${CLAUDE_SKILL_DIR}/references/debugging-template.md`; structure and order are binding. The header block carries `Debugging`, `Fehlerbild`, `Erstellt`, `Claude-Session: ${CLAUDE_SESSION_ID}` and `Status: Entwurf`; step overview and parallel overview are at the very top. You write the file once, after the steps, waves and cross-check from sections 12 to 14 are settled. The template is German because German is the default output language; for another output language, translate its labels as you carry them over, not only during the cross-check.

## 12. Cut steps

A step has one concern and a verifiable result and names result, affected files, dependencies, approach, regression test and acceptance. For a bug fix, add:

- **A fix step is test and fix together** — the reason is in the template's notes. Its `Vorgehen` begins with „Test schreiben, gegen den Ist-Stand ausführen, Rot im Bericht festhalten", and its `Akzeptanz` demands „Testausgabe rot vor dem Fix (Ausgabe-Auszug), grün danach" — both phrased in the output language. The one exception is the follow-up fix step from section 1 whose bug cannot be shown locally even with the diagnosed input shape: its `Akzeptanz` names a manual check in production and why no local red run is possible.
- **Edge cases and similar places.** An edge case whose current behavior is wrong belongs in the fix step, with its own red run; a separate edge-case step only pins down cases the current state already handles correctly, and says so. Every similar place as its own step, so it can be deselected individually.
- **A suspected cause ends in a diagnosis.** If the root cause is `Verdacht`, the step sequence ends with a diagnosis step — a test with production-shaped data, logging or a metric — and holds no fix, similar-place or edge-case steps; those belong to the follow-up plan. Its `Akzeptanz` names which outcome confirms or rejects which hypothesis, and its `Regressionstest` reads „keiner — <Grund>". Fix steps then exist only as a sketch under `## Lösung`, with the note „nach Diagnose: `/plan-debugging-create "<Plan-Datei> — Diagnose: <Ergebnis>"`". Diagnostic logging records only the shape, type or length of a value, or a keyed hash (HMAC with a secret that is not committed), never personal values, tokens or credentials and never a plain hash of them, which a dictionary reverses; the step's `Akzeptanz` names what it excludes. It is either permanent with a reason under `## Rollout`, or its removal is noted under `## Lösung` with `file:line` and becomes a step of the follow-up plan — so the diagnosis stays the last step.

After each step, build and tests run at the state of the baseline plus the new regression test, i.e. without new red tests. A step is too large if it would itself need a plan; too small if its result on its own removes no symptom and tests no edge case.

## 13. Determine parallelism

Two steps may only go into the same wave if both hold: no functional dependency and no jointly written files. Steps at the same root cause never run in parallel; similar places in different modules may go together after the fix at the cause. Form waves: W1 is all steps without a precondition, W2 everything that depends only on W1, and so on; name the touchpoints for each wave. If there is no real parallelism, write exactly that.

## 14. Cross-check

Check the composed text as if you had not written it yourself — on the text, not by rereading the file:

- Does the sum of the steps remove the symptom, and nothing beyond it? Do the dependencies in overview, waves and steps agree?
- Does every fix act at the location of the root cause, and not only where the symptom shows?
- Does every step have the `**Regressionstest:**` bullet, and does every fix step demand the red run before the fix in `Vorgehen` and `Akzeptanz` — or, as a follow-up fix step that cannot show the bug locally, name the manual check in production and the reason?
- Is every cause without reproduction and gapless chain marked as `Verdacht`, and does the step sequence then end with the diagnosis, without fix, similar-place or edge-case steps? Does such a plan note diagnostic logging that has to be removed under `## Lösung`, and does its logging exclude values? In a follow-up plan: does it rest on a stated diagnosis result, and does it carry over only the fix of the confirmed hypothesis and the removal of the logging — or, if none was confirmed, no fix at all?
- Does every `H` have a status and evidence? Does every `R` have a step or a non-goal?
- Is there an unredacted personal value, token or credential in the plan?
- Does every `file:line` hold? Check the location of the root cause and every link of the causal chain against the file itself (Read tool at that line), not from memory.
- Does „Nicht gelesen" list what remained unread? Is every invented detail gone, every assumption marked as an assumption, and is everything in the output language?

Correct what you find and check again until a pass yields nothing new. For the thorough review, point afterwards to `/plan-lint` and `/plan-review`.

## 15. Hand off

Name the path of the file, the reproduction result, the root cause in one sentence with location and classification, the number of `H` (confirmed, rejected, open) and `R`, the number of steps, the waves in one line, the open questions, what remained unread, and what the test and reproduction runs left behind in the working directory (from the `git status --porcelain` after the runs). No rehashing of the content — the plan is in the file.

## 16. Stance

The temptations of this skill: patching the symptom where it shows; taking the first plausible hypothesis as the cause; passing off a cause without reproduction as `belegt`; swallowing the error with `try/catch` and calling that robust; planning the regression test after the fix; refactoring or optimizing on the side; listing edge cases without a test; inventing production values. What you notice beside the path goes under non-goals — structure with a pointer to `/plan-refactoring-create`, slowness with a pointer to `/plan-optimization-create`. A plan with one evidenced cause, one test and one open question is better than one with five suspected causes and none.

And before you send: are the answer and the plan file in the output language? Otherwise the German template pulls toward German and these English instructions toward English, even if something else was requested above.
