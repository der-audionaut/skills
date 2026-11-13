---
name: tool-git-rebase
description: Rebases the checked-out branch or a named branch onto a base branch — fetches the base's current state, checks the starting situation, resolves conflicts commit by commit so that the intent of both sides survives, verifies the result with the project's own commands and hands off with a report, without pushing. Takes the base branch as an argument, optionally followed by the branch to rebase — without it, the checked-out branch.
argument-hint: "[basebranch] [branch]"
arguments: basebranch branch
disable-model-invocation: true
allowed-tools: Bash Read Grep Glob Edit
---

**Output language: German.** You write the report, follow-up questions and abort messages in German, even if the repository and commit messages are in English. Commit messages and code follow the existing codebase, not this rule.

Rebase the target branch — `$branch`, or without it the checked-out branch — onto `$basebranch`. Work like someone carrying other people's commits through other people's changes: afterwards, every commit should do exactly what its author wanted — just on the new state of the base. You build nothing new here and tidy up nothing on the side.

Three rules carry the whole procedure: You do not push. You change neither the order nor the message nor the cut of the commits. And every conflict is read before it is resolved — taking one side wholesale is not a resolution but a hidden revert.

In the commands below, `<basis>` is the base branch, `<ziel>` the target branch, `<datei>`/`<dateien>` the conflicted file(s) and `<vorheriger Branch>` the branch that was checked out before; the report template uses the same placeholders.

## 1. Resolve base branch and target branch

**Base branch.** `$basebranch` is the branch the target branch is rebased onto — as a local name (`main`) or with remote (`origin/main`). Check with `git rev-parse --verify` which of these exist, and then:

1. `$basebranch` is empty → name the candidates that exist (the target branch's upstream, `main`, `master`, `develop`) and ask which one is meant. Do not guess: a rebase onto the wrong branch can be undone, but the way back costs trust.
2. The name exists only on a remote (`origin/<name>`) → take the remote branch.
3. It does not exist at all → name the existing branches and abort.

**Target branch.** The target branch is the branch being rebased. It must exist locally, because the rebase checks it out — a remote branch can only be a base. Check with `git rev-parse --verify refs/heads/<name>`, and then:

1. `$branch` is empty → the checked-out branch is the target branch (`git symbolic-ref --quiet --short HEAD`; if that returns nothing, no branch is checked out, and section 2 aborts).
2. `$branch` exists as a local branch → that is the target branch.
3. The name exists only on a remote (`origin/<name>`) → abort and name the one-liner that creates it locally: `git switch <name>`. Creating branches is not your job; the rebase is.
4. It does not exist at all → name the existing branches and abort.
5. Target branch and base branch point to the same ref → abort: a branch cannot be rebased onto itself.

## 2. Check the starting situation

Before `git rebase` runs, the situation must be unambiguous. Each of the following points is a reason to abort — report it and stop instead of repairing:

- **Working tree not clean.** `git status --porcelain` shows modified or staged files. Name them and suggest commit or stash; you do not stash yourself, because a forgotten stash is worse than an aborted rebase. Untracked files only get in the way if the base brings a file at the same path — then the same applies.
- **A rebase, merge or cherry-pick is already in progress.** `git status` says so, `.git/rebase-merge`, `.git/rebase-apply` or `.git/MERGE_HEAD` confirm it. Cleaning up someone else's intermediate state is not your place.
- **No branch checked out** (detached HEAD). This also applies with `$branch`: switching away from a detached commit can make commits unreachable, and whether anyone needs them, you do not know — the human does the one `git switch` line beforehand.

Then fetch the current state — always, not only when `main` looks old, because whether it is old you only know afterwards: `git fetch <remote>` for the remote the base branch lives on, and note whether `<remote>/<basis>` moved; that goes into the report as a line. A rebase onto a stale local `main` resolves yesterday's conflicts and brings the same ones again tomorrow. If the local base branch has an upstream, compare both states: equal → take the local name; local behind → take the upstream (`origin/main`) and say so in the report; local ahead, i.e. unpublished commits on the base → ask which state is meant. You do not move the local base branch in the process: you rebase onto `origin/main` instead of first fast-forwarding `main` — the rebase is your job, maintaining other branches is not.

Now look at the work being rebased — everything on `<ziel>` and without switching branches; without `$branch`, `<ziel>` is the checked-out branch, and nothing changes compared to the usual procedure:

- `git merge-base <basis> <ziel>` and `git log --oneline <basis>..<ziel>` — these are the commits about to be rewritten. If the base is already an ancestor of `<ziel>` (`git merge-base --is-ancestor <basis> <ziel>`), there is nothing to do: say so and stop.
- If the target branch has an upstream, the rebase will rewrite published history; the push afterwards needs `--force-with-lease`. That is not a reason to abort, but it belongs in the report. It is a reason to abort if the upstream has commits that are missing locally (`git log --oneline <ziel>..<ziel>@{u}` is not empty): fetch those first, otherwise the later push overwrites someone else's work.

Secure the starting point before you change anything: `git rev-parse <ziel>` for the report, and the branch's diff against the old merge base into a file outside the repo, e.g. `git diff $(git merge-base <basis> <ziel>) <ziel> > /tmp/rebase-<ziel>-vorher.diff`. Git also sets `ORIG_HEAD` during the rebase; that is the way back as long as the target branch is checked out — after switching back, no longer, see section 5.

## 3. Run the rebase

If the target branch is not checked out, switch now — and only now: note the checked-out branch for the way back, then `git switch <ziel>`. If the switch fails, abort and report it; up to this point you have changed nothing. The switch comes this late so that all checks and follow-up questions before it end on the human's branch, with nothing to switch back.

`GIT_EDITOR=true git rebase <basis>` — plain, without `-i`, without autosquash, without reordering. The prefixed `GIT_EDITOR=true` prevents an editor from opening and the run from hanging. If the rebase runs through without stopping, continue with section 5.

If it stops, there is a conflict. `git status` shows the affected files, `git show --stat REBASE_HEAD` the commit currently being applied.

## 4. Resolve conflicts

Understand first, then change. For each stop:

**Assign the sides correctly.** In a rebase it is the reverse of what you know from a merge: `ours` — `HEAD`, the upper block from `<<<<<<<` — is the **base** plus everything already rebased; `theirs` — `REBASE_HEAD`, the lower block down to `>>>>>>>` — is the **branch's commit** currently being applied. Whoever mixes this up throws away their own work with `--ours` and believes they kept it.

**Determine both intents.** `git show REBASE_HEAD` shows what the commit wanted; `git log -p <merge-base>..<basis> -- <datei>` shows what the base wanted at the same spot and why. Read the commit messages — they are the only justification you get.

**Resolving means: both intents survive.** The base's change stays, and the commit does the same thing on the new state as before — adapted to new names, new signatures, moved code. If the base has meanwhile taken care of the commit's concern itself, the commit's side may be dropped; you prove that with the location in the base instead of assuming it. If both sides want something different at the same spot, and both are valid on their own, that is not conflict resolution but a decision: present both sides with quotes, say what you would propose and why, ask, and leave the rebase stopped until the answer. Name the way out `git rebase --abort` — and, if you switched for the rebase, the switch back `git switch <vorheriger Branch>`, which follows after `--continue` as well as after `--abort`.

Recurring patterns — lock files, generated code, import blocks, deleted versus modified, renames, formatting runs, colliding migrations — are listed with their handling in `${CLAUDE_SKILL_DIR}/references/conflict-patterns.md`. Read the file at the first conflict, not at the third.

**Before you move on:**

- The base's change is still there: `git diff <merge-base> <basis> -- <datei>` shows what the base changed in this file, and every one of those lines must be in your resolution — new names, new signatures, new calls. If one is missing, you have reverted the base; that is exactly what happens when you simply take the lower block because it is "your own work".
- The commit's change is still there: hold `git show REBASE_HEAD -- <datei>` against your version. Its intent must remain recognizable, even if it now carries different names.
- No marker left in the affected files: `grep -n '^<<<<<<<\|^=======\|^>>>>>>>' <dateien>`. A forgotten marker is the most embarrassing commit a rebase can produce.
- Look at `git diff` against the index: is really only what the conflict required changed?
- `git add <dateien>`, then `GIT_EDITOR=true git rebase --continue`.
- If Git reports that the commit has become empty, check with `git diff --cached`: if the index is empty, the change is already in the base, and `git rebase --skip` is right — note the skipped commit for the report. If the index is not empty, something else is going on, and you read again.

Then the next stop, until the rebase is through. The same conflict comes back in several commits if the branch touched the same spot multiple times; resolve it each time consistently with the first resolution, because Git does not remember it for you.

**Termination:** If the same commit remains unresolved after three attempts — because every resolution brings new conflicts or errors — you stop, leave the rebase standing and present the state: which commit, which files, what you tried, how to proceed (`git rebase --continue` after their own resolution or `git rebase --abort`, then `git switch <vorheriger Branch>` if applicable). A half-finished rebase with an honest report is better than one that runs through on invented resolutions.

## 5. Verify

A rebase without conflict markers is not yet correct. Compilers and tests see conflicts Git does not see: the base renamed a function, the branch calls it at a new spot under the old name — no textual conflict, but broken.

- **Run what the project provides.** Build, tests, linter, type check — as described in `CLAUDE.md`, `README`, `package.json`, `Makefile` or the CI configuration. Verifying means running: a verification line in the report that does not name an executed command with its result is not one. If something does not run, say so and name what remains unchecked as a result.
- **Compare the diff.** `git diff <basis> HEAD` against the saved `vorher.diff`: the difference may only lie where you resolved conflicts and where the base changed the same files. A file that was in the branch diff before and is missing now is a lost change — unless you skipped the corresponding commit with evidence.
- **Count the commits.** `git log --oneline <basis>..HEAD` must show the same subject lines in the same order as before the rebase, minus the skipped ones.

If a check fails, first clarify whether the rebase is the cause: does it concern files both sides changed? Did it already fail on the starting point? If the rebase is the cause, fix it in a separate, clearly named commit on top — the fix would only get into the original commit with an interactive rebase, which you do not have here, and you say so in the report. If the check was already red before, report it and do not touch it: that is not your construction site.

If you switched in section 3 and no rebase is running anymore — it is through, including a repair commit if any — then switch back: `git switch <vorheriger Branch>`. Whoever names the target branch wants nothing to do with it; staying on it would be a surprise. If the switch back fails, e.g. because untracked artifacts from verification collide with files of the previous branch, stay put, report it and force nothing. If, on the other hand, the rebase is waiting for a decision, the target branch stays checked out: you cannot leave a stopped rebase without aborting it — the report says so and names the switch back as the next step.

## 6. Hand off

```
# Rebase: <ziel> auf <basis> — abgeschlossen | wartet auf Entscheidung | abgebrochen

**Ausgecheckt:** <vorheriger Branch> — für den Rebase nach <ziel> gewechselt und zurück | steht auf <ziel>, Rebase wartet   (Zeile entfällt ohne `$branch`)
**Ausgangspunkt:** <sha, alter Stand des Zielbranchs> — zurück mit `git reset --hard <sha>` (auch als `ORIG_HEAD`) | nach dem Rückwechsel: `git branch -f <ziel> <sha>`, nicht `git reset --hard ORIG_HEAD`
**Geholt:** `git fetch <remote>` → <remote>/<basis> unverändert | von <sha> auf <sha> bewegt
**Basis:** <basis> @ <sha> — <n> Commits, die der Branch vorher nicht hatte
**Neu aufgesetzt:** <m> Commits, <k> übersprungen
**Verifikation:** <Befehl> → <Ergebnis>

## Konflikte
### <Datei> — in <sha> „<Betreff>"
- **Basis wollte:** <ein Satz>
- **Commit wollte:** <ein Satz>
- **Auflösung:** <was jetzt drinsteht und warum>

## Übersprungen
- <sha> „<Betreff>" — bereits in der Basis durch <sha der Basis>

## Offen
- <Entscheidung, die der Mensch treffen muss / Prüfung, die nicht laufen konnte>

## Nächste Schritte
- <`git push --force-with-lease`, sofern der Zielbranch einen Upstream hat — den Push machst nicht du; nach einem Rückwechsel mit Branchnamen: `git push --force-with-lease <remote> <ziel>`>
- <wartet der Rebase: `git rebase --continue` nach eigener Auflösung oder `git rebase --abort`, danach `git switch <vorheriger Branch>`>
```

Every conflict gets its entry. The report is the only place where it remains traceable why a spot looks the way it does now — the commits themselves no longer show the resolution, because they look as if the author had written it that way from the start.

With `$branch`, the way back deserves special care. After the switch back, `ORIG_HEAD` still points to the old state of the target branch, but `git reset --hard ORIG_HEAD` hits the checked-out branch — the human's — and would destroy it. That is why the report names `git branch -f <ziel> <sha>`; as long as the rebase is stopped, the way back remains `git rebase --abort`. And the push needs the branch name, because `git push` without a refspec would take the checked-out branch.

## 7. Stance

The first temptation is `--ours` or `--theirs`. It is quick, the markers are gone, and one side is silently reverted — in a rebase, only whoever misses the change notices.

The second is tidying up on the side during the conflict: formatting, a rename, a bug you happen to notice. Every change the conflict does not require makes the rebase unreadable; whoever reviews it can no longer separate resolution from unrequested liberties. Observations belong in the report, not in the commit.

The third is "fixing" a test that is red after the rebase by adjusting it. The test may be the only one that noticed the semantic conflict.

The fourth is pushing because everything is green. The rebase rewrites published history, and that is the human's decision — with `--force-with-lease`, never with `--force`.

And the rebase can be undone at any time: `git rebase --abort` during, `git reset --hard ORIG_HEAD` afterwards — or `git branch -f <ziel> <sha>` if you have already switched back to the previous branch —, `git reflog` always. Write that in the report instead of hoping it is known.
