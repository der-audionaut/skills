---
name: plan-implement
description: Implements a single step from a feature plan — first a design in the manner of a system architect, then the minimal production-ready implementation, then finding rounds until nothing new turns up, then a `/simplify` pass over the step's files if that skill is available and verification stays green afterwards, then the commit of the step with a message from `write-commit-message` if that skill is available — not the plan file, not on the default branch, not with red verification, not over someone else's changes, and never a push — and finally the update of the plan — history, done marker in step overview, detailed step and header block, insights for later steps. Takes plan name and step as arguments.
argument-hint: "[plan-file] [step]"
arguments: plan schritt
disable-model-invocation: true
allowed-tools: Read Grep Glob Edit Write Bash Skill
---

**Output language: the language of the plan.** You write the design, reports, follow-up questions, abort messages and history in the language the plan is written in — until the plan has been read, in German. Code, identifiers, comments and tests follow the existing code, not this rule. That these instructions are in English changes nothing about this.

Implement step `$schritt` from the plan `$plan`. Think like an experienced system architect: design soundly for growth, then build the minimal version that can run in production. Not the whole future, but also no design that has to be thrown over at the first rise in load.

You implement exactly this one step. Not the next one, not half of the one after.

## 1. Resolve plan and step

**Plan.** `$plan` can be a path, a file name or a name fragment without extension.

1. If `$plan` exists as a path → take it.
2. Otherwise glob `**/*$plan*.md`, preferably in `docs/plans/`, `.claude/plans/`, `plans/`, `docs/`, project root.
3. Multiple hits → list them and abort instead of guessing.

**Step.** `$schritt` can be a number, a heading or a fragment of one. No argument given → name the still open steps — those whose row in the step overview carries no `✓ erledigt` suffix in the `Welle` column — and ask which one is meant. Ambiguous → the same.

**Language.** The output language is the language of the plan. The header block and headings decide — not quotes, identifiers or code blocks, which in a German plan are often English. The language changes the text, not the structure: the report format stays, only its labels are translated.

**History.** Where these instructions say `## Umsetzungs-Historie` or `## Review-Historie`, they mean the section with that meaning, whatever language its heading is in — a plan can come from `/plan-create` in another language. You recognize an existing one in translation too and continue it; a new one you create in the plan language. The finding ID F stays untranslated.

**Starting state.** Before you touch any file, you record how you found the repository — later this is the only source from which someone else's changes can be told apart from yours, and section 5 needs every one of these values:

- The checked-out branch: `git symbolic-ref --quiet --short HEAD` — empty on a detached HEAD.
- The paths that are already modified, staged or untracked now: `git status --porcelain=v1 -uall`.
- Whether the index is empty: `git diff --cached --quiet`.
- Whether an operation is in progress: `.git/MERGE_HEAD`, `.git/rebase-merge`, `.git/rebase-apply` or `.git/CHERRY_PICK_HEAD` present.
- The default branch: the remote is the upstream's (`git config branch.<branch>.remote`), otherwise `origin`; then `git symbolic-ref --quiet --short refs/remotes/<remote>/HEAD` without the prefix `<remote>/`. If the reference is missing, `main` or `master` applies, if present locally; if those don't exist either, the default branch is unknown, and section 5 asks.

If the directory is not a Git repository, you note that: section 5 is then skipped for that reason, everything else runs as usual.

Read the whole plan, not just the step. A step read in isolation gets implemented wrong in isolation. Check in the `## Umsetzungs-Historie` and in any `## Review-Historie` what earlier steps have already changed or left behind as insight.

## 2. Design before code

Before you write a line, record the design for this step — briefly, in prose, no wallpaper. The dimensions are in `${CLAUDE_SKILL_DIR}/references/design-dimensions.md`; cover only those this step actually touches:

Architecture and placement · Component split · Data flow · Interfaces and API · Data model and migration · Caching and invalidation · Verification path

Two guardrails that work against each other and both apply:

- **Design soundly.** Cut interfaces, data model and responsibilities so that ten times the load or the second use case does not force a rewrite.
- **Build minimally.** No abstraction without a second caller, no cache without proven need, no configurability without a second value. Whatever the design provides for but this step does not need, you note as a follow-up step instead of building it.

If the design deviates from the plan, say so now and justify it. If the plan does not hold at this point at all, abort and report that instead of silently building something else.

## 3. Implement

- Follow the patterns of the existing code. A second, competing pattern is more expensive than an ugly one that fits the rest.
- Error cases belong in the same pass as the happy path, not in a later round.
- No scope creep. Whatever you notice along the way that does not belong to this step goes into "Erkenntnisse für spätere Schritte" (insights for later steps).
- Tests for what this step claims to be able to do — at the level the project usually uses.

## 4. Finding rounds

After implementing, you check your own work in rounds. A round consists of: verify → collect findings → fix → verify again.

**Verifying means running.** Build, tests, linter, type check — the commands the project provides. A finding pass that only reads finds half. If something cannot be run, say so and name what remains unchecked as a result.

The finding categories are in `${CLAUDE_SKILL_DIR}/references/design-dimensions.md`. Every finding gets a stable ID (F1, F2, …), a location `Datei:Zeile` and a classification:

- **Blocker** — wrong, insecure or non-working behavior.
- **Sollte** (should) — holds, but deviates from the plan, leaves an error case open or breaks with the existing code.
- **Optional** — taste, no risk.

You fix Blocker and Sollte findings in the same round. Optional findings you collect and present at the end, without implementing them on your own authority.

**Termination:** the loop ends when a complete round produces no new Blocker and no new Sollte finding and verification is green. Then say so clearly and stop searching.

So that the loop actually converges:

- From round 2 on, no new Optional findings unless they were caused by a fix.
- A finding you deliberately don't fix is rejected with a justification and not reopened in the next round.
- After three rounds without progress, you stop and name what it hinges on: missing information, a plan that doesn't hold, or a design with the wrong underlying assumption. The human decides that.

Report per round — the final round's report you output only after section 6, so that it can carry the lines `Simplify:`, `Commit:` and `Plan:`; intermediate rounds report immediately:

```
# Umsetzung: <Plan> — Schritt <X> — Runde <N>

**Status:** Findings offen | Abgeschlossen
**Geändert:** <Dateien>
**Verifikation:** <Befehl> → <Ergebnis>
**Simplify:** übernommen — <Dateien> | keine Änderungen | verworfen — <Grund> | übersprungen — <Grund> | noch nicht
**Commit:** <sha> „<Betreff>" — Rückweg `git reset --soft HEAD~1` | keiner — <Grund> | noch nicht
**Plan:** Schritt <X> markiert in Schrittübersicht, Detailschritt und Kopfblock; Historie fortgeschrieben | noch nicht

## Findings dieser Runde
### F1 — <Titel>  [Blocker | Sollte | Optional]
- **Fundstelle:** <Datei:Zeile>
- **Befund:** <was nicht stimmt>
- **Behandlung:** behoben in <Datei> | abgelehnt: <Begründung> | offen

## Nicht zugeordnet
- <Pfad> — neu seit Start, nicht gestaged: <Vermutung, woher er stammt>

## Abweichungen vom Plan
- <Abweichung> → <Begründung>

## Erkenntnisse für spätere Schritte
- <was der Plan an anderer Stelle jetzt anders sehen muss>
```

The section "Nicht zugeordnet" (unassigned) is dropped if it would be empty.

## 5. Commit

Once the loop has terminated green, you commit the step — only it, only on the branch you found, and never with a push. Before that, `/simplify` runs exactly once over the step's files, if that skill is available — see "Simplify" below. The commit is a result, not a condition: every precondition that does not hold is a reason for "no commit", never a reason to abort. Report and plan are written in any case; the reason goes in the `Commit:` line. If the directory is not a Git repository, exactly that is the reason, and the rest of this section is skipped.

**Preconditions.** All five must hold:

- The loop terminated green — not aborted after three rounds without progress, verification green. You do not commit a red state, not even "so as not to lose anything": in the working directory it is safe, in the history it would be a claim that isn't true.
- A branch is checked out, and it is not the default branch. On `main` the human commits, not you. If the default branch is unknown, the human decides — see "Follow-up question" below.
- The index was empty at the start. Otherwise someone else's staged changes would end up in the commit.
- No merge, rebase or cherry-pick is in progress. A `git commit` would conclude the foreign operation; the reason is then "Merge läuft", not "Index nicht leer".
- No file you changed was already modified at the start. There, someone else's changes and yours can no longer be separated — and there is no partial commit of the remaining files, because half a step in the history is worse than none. The plan file does not count here; it is never staged anyway.

**Files of the step.** What counts is your own list: the union of the "Geändert" entries of all rounds — modified, created, deleted and renamed paths — plus the products you explicitly assign to one of these paths: the lockfile for the changed manifest, the snapshot for the changed test, the generated code for the changed source. Each assignment gets a half-sentence in the report. Every changed manifest raises the question of its lockfile — a forgotten one leaves a commit that doesn't build without it.

`git status` serves as a check, not as a supplement. Every path on your list must appear there; if one is missing, the list is wrong, and you clear that up before you stage. Paths that now appear in `git status` but are neither in the starting state nor on your list — verification artifacts, files touched by the formatter, unknowns — you do **not** stage: they go into the report under "Nicht zugeordnet", and the human decides afterwards. Paths from the starting state that you did not touch stay untouched. You never stage the plan file.

**Stage.** `git add -- <paths>` with the path list — never `-A`, never `.`.

**Simplify.** If the list of available skills contains one named `simplify` — with plugin prefix or without — you invoke it now, after staging and before the message, exactly once via the Skill tool. It checks changed code for reuse, simplification and efficiency and changes it directly in the working directory. The index is your backup point here: it holds the state the loop verified green — including new files that, unstaged, would show up in no diff — and everything the skill changes then stands against it as an unstaged difference. Without a target it checks the whole branch up to `@{upstream}`, i.e. also the commits of earlier steps; that is why you give it the step's files as target, in English, because it works in English:

```
Review target: the uncommitted changes in <path1>, <path2>, … — see
`git diff HEAD -- <paths>`. This is one plan step, not the whole branch:
only simplify lines this diff adds or changes, touch no other file, keep the
behaviour the tests cover, and do not commit.
```

What it did you read afterwards from `git status --porcelain=v1 -uall` and `git diff`, not from its summary. Then you decide on the pass as a whole — keep or discard, never a selection: the state in the index was green, and a half-adopted pass is a third state that nobody has checked.

- **Paths outside your list** it has no business touching. If it did anyway, the pass is discarded, because otherwise your files could depend on a change that doesn't go into the commit. Clean up: `git checkout -- <paths>` for your list and for modified paths that were clean in the starting state; newly created paths that did not exist there you delete. A path that was already modified or untracked in the starting state you cannot reset without losing someone else's work: it stays as it is and is listed under "Nicht zugeordnet" with the note that the skill touched it.
- **Verifying** means running here too: the same commands as in section 4, once. Green → `git add -- <paths>` with your list, so the index holds the simplified state; the `Verifikation:` line in the final report names this run. Red → `git checkout -- <paths>` restores the state from the index, and the `Simplify:` line says "verworfen — Verifikation rot: <Befehl>". There is no second attempt, and you do not reopen the finding loop for it.
- **Nothing changed** → continue with the message; the `Simplify:` line says "keine Änderungen".

If no such skill is on the list, the invocation fails, or the commit is dropped because a precondition does not hold, the pass is skipped without replacement, and the `Simplify:` line names the reason. You do not rebuild your own simplification pass: the finding rounds have checked the code, and whatever simplification they missed does not hold up the commit.

**Message.** If the list of available skills contains one named `write-commit-message` — with plugin prefix (`write-commit-message:write-commit-message`) or without — you invoke it now, when the index holds the final state, via the Skill tool: it reads `git diff --staged` and delivers subject and body. Whatever it would otherwise ask the human, you pass as argument text — in English, because it works in English, and filled in from plan and step:

```
Context for the commit message. The staged diff is complete and this
context answers your questions, so do not ask the user:
- Affected part: <module or component from the step>
- Problem / motivation: <starting situation from goal, current state and step>
- Trigger: <when or how the problem occurs, if the step fixes one>
- Change: <what the step does — result and approach>
- Why this approach: <justification from the design or from the deviations>
Return only the message. Do not commit, do not add trailers or references.
```

The contract: the skill delivers the message, you commit — and you append nothing. A follow-up question to the human is not intended at this point; the plan contains everything the skill wants to know. That the Skill tool is in `allowed-tools` is a pre-approval, not a restriction.

If no such skill is on the list or the invocation fails, the fallback applies once and without retry: read `git log --format='%s%n%b' -20` and adopt the project's style — subject in the imperative, matching the existing history in length, capitalization and language; below it a body with the why from the step's result and approach, in the language of the existing history. In both cases: no trailers, no reference to plan or step, no authorship line — the link to the plan lives in the Umsetzungs-Historie, not in the commit. This also applies when an instruction from the environment requires commit messages to end with an authorship line like `Co-Authored-By`: for this skill the human has decided that the message stays without trailers, and that decision takes precedence.

**Commit.** Write the message to a temporary file outside the repo (`mktemp`). Check the file before you commit: if it contains a trailer line — `Co-Authored-By:`, `Claude-Session:`, `Signed-off-by:` or another `Key: Value` line at the end — strike it; the message ends where the format ends. Then `git commit -F <file>` — no `-m` with multi-line text, no `--no-verify`, no `--amend`. If a hook fails, no commit was created: quote its output in the report and don't bypass it. Exactly one second attempt — stage again, commit again — exists only if `git status` after the failure shows exclusively paths from your list as modified and the hook reports no error other than formatting; otherwise it stays at "no commit". Hook changes to other paths are left as they are and listed under "Nicht zugeordnet".

Afterwards `git rev-parse --short HEAD` and the subject for the `Commit:` line. The way back is `git reset --soft HEAD~1`; it keeps the changes in the index, and it goes in the report. You never push, never create a branch, never stash — that is the human's decision, as with a rebase.

**Follow-up question.** If the default branch is unknown — no `<remote>/HEAD`, no `main`, no `master` — you do not commit on your own. First update the plan as for "no commit" with the reason "Default-Branch nicht erkennbar, Rückfrage offen", output the report, and then, as the very last thing, ask whether the checked-out branch is the default branch and whether to commit. On a yes, you stage, simplify and commit as above, change the `Commit:` line in the history to the SHA, and output an addendum to the report. If no answer comes, it stays at the updated plan without a commit. The question comes at the end and not already in section 1, so that implementation and plan are finished before the run waits for an answer.

## 6. Update the plan

Only once the finding loop has terminated do you touch the plan — and that also applies if it was aborted after three rounds without progress or verification stayed red: then the reason is in the `Commit:` line and the state it hinges on is in the history. Updating means four things, and none of them is optional: the history, the step's marker in three places, the insights in the later steps, and the re-check of the affected steps.

**History.** Create the section `## Umsetzungs-Historie` at the end of the file if it is missing:

```
## Umsetzungs-Historie
### Schritt 3 (W2) — <Datum> — abgeschlossen in 2 Runden
- Geändert: src/billing/invoice.ts, migrations/0042_add_status.sql
- Commit: a1b2c3d „Billing: Add status column to invoice export" | keiner — Default-Branch `main` ausgecheckt
- Simplify: übernommen — doppelte Statusprüfung in invoice.ts zusammengezogen | keine Änderungen | verworfen — Verifikation rot | übersprungen — Skill nicht verfügbar
- Abweichung: Statusfeld als Enum statt String — bestehendes Muster in order.ts
- F2 Fehlender Index auf invoice.status — behoben
- F4 Doppelte Serialisierung im Export — abgelehnt: betrifft Schritt 5
- Erkenntnis: Schritt 5 kann den alten Export-Pfad nicht mehr voraussetzen
```

In a plan in another language, the new section carries the translated heading — `## Implementation History` — and its entries follow the plan language; the template here is German because German is the default output language, not because the section would have to be. The label `Commit:` is translated along with it; SHA and subject stay as Git has them.

**Marking the step.** The history sits at the end of the file, and nobody looks there first. Whoever opens the plan sees the step overview — and if it still says `W1` without a suffix there, the step is open for them, no matter what the history says further down. That is why you mark the step in three places, all three, in exactly this form:

1. **Step overview** — the table at the top of the plan, in the template `## Arbeitsschritte`: in the step's row, the `Welle` column gets the suffix `✓ erledigt`; `W1` becomes `W1 ✓ erledigt`. The code stays because the parallel overview and `/plan-lint` reference it, and you don't create a status column of your own.
2. **Detailed step** — `### Schritt <X> — <Titel>` under `## Schritte`: as the first bullet under the heading comes `- **Status:** erledigt <Datum>`, before `Ergebnis`. The heading itself stays unchanged; `$schritt` is resolved via it.
3. **Header block** — the `Status:` line: it keeps count — `In Umsetzung — <n> von <N> Schritten erledigt`, where N is the number of rows in the step overview. If this completes the last step, it reads `Umgesetzt — alle Schritte erledigt (<Datum>)`. If the line is missing, you create it.

You write the date as in the header block, `YYYY-MM-DD`. If the run ends without completion — loop aborted after three rounds or verification red — the step gets the suffix `✗ abgebrochen` in place 1 and `- **Status:** abgebrochen <Datum> — <Grund in einem Halbsatz>` in place 2; the header block switches to `In Umsetzung` but does not count it. A later run that completes the step replaces this marker with the done form. That way a row without a suffix is always one that nobody has touched yet.

In a plan in another language you translate the words, not the form: `W1 ✓ done`, `- **Status:** done <Datum>`, `Implemented — all steps done (<Datum>)`. Check mark, cross, position and wave code stay.

**Insights.** Enter them where they belong — in the later steps they affect, not just in the history.

**Afterwards you check the plan again**, and only the steps your implementation touched: do their assumptions still hold? Have steps become superfluous, or is a new one needed? The same termination as above — if a pass produces no new findings, the plan is current and you stop. For a full review of the revised plan, point to `/plan-review $plan` instead of rebuilding it here. If a step is added or removed in the process, N in the header block changes — update the line.

**Check.** Before you output the final round's report, you check the three places with Grep in the plan file, not from memory: the step's table row ends in `✓ erledigt |` or `✗ abgebrochen |`; the first bullet under its heading begins with `- **Status:**`; the header block's `Status:` line counts it. If a place is missing, the plan is not updated — catch up, then report. The `Plan:` line in the report is the result of this check, not a formality.

## 7. Stance

The most dangerous moment is the second round in which everything is green: then the temptation is great either to invent findings or to stop searching before the error cases are checked. Both are a bad result. Record what you ran and what that proves — the round is measured by that, not by the number of findings.

At the commit there are four temptations: staging everything because `git add -A` is faster than a path list; committing on `main` because it's only local anyway; committing a red state "so as not to lose anything"; and bypassing the hook with `--no-verify` because it's in the way. All four take a decision away from the human that belongs to them. What you don't commit sits in the working directory and is safe there; what you do commit has its way back in the report.

When simplifying, the temptation is to believe the skill's summary and not run verification again — or to keep half of a red pass because it looks nice. The index holds the checked state; whatever the skill lays on top is either green again or gone.

When updating, the temptation is to append the history and consider the plan updated. But the human reads the plan from the top, and a step overview without check marks tells them: nothing happened. The three markers are therefore not decoration but what they read the state from — and the Grep check exists so that you don't report them as set from memory.

And if the plan is wrong at this point, that is a result, not an obstacle. Say so instead of building around it.

And before you send: is the answer in the language of the plan? The English instructions otherwise pull toward English, even when the plan is German.
