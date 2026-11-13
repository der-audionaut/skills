---
name: tool-check-renovate-pr-in-repo
description: Checks the open Renovate PRs of a GitHub repository in a fresh clone — each dependency pair with tool-dep-update-check in its own subagent —, implements necessary adaptations as one commit per package on the PR branch, checks them for completeness, verifies them, and pushes and approves only after one bundled approval by the human; until then nothing leaves the machine. Takes the repo URL as argument, optionally followed by PR numbers in quotes that restrict the run to those PRs.
argument-hint: "[repoUrl] [prs]"
arguments: repoUrl prs
disable-model-invocation: true
allowed-tools: Bash Read Grep Glob Edit Write Agent Skill
---

**Output language: German.** You write the report, follow-up questions and abort messages in German, even if repository, PR titles and changelogs are in English. That these instructions are in English changes nothing about this. Quotes from PR bodies, identifiers, commands and error messages stay as they are.

Check the open Renovate PRs of the GitHub repository `$repoUrl` — all of them, or with `$prs` only the named ones. Work like someone who, before clicking review, answers the question a Renovate PR otherwise answers only after the merge: can this project carry the update, and what would need to be done for it? Every dependency pair — one package with source and target version — is checked by `tool-dep-update-check`; you clone, inventory, classify, delegate, implement necessary adaptations in the clone and report. You change nothing externally without a yes: commits are created in the clone, and whether they reach the repository and the PR gets an approval is decided by the human in exactly one bundled approval; you never touch the human's working copy.

Three rules carry the whole procedure: writing happens only under this run's temp directory — in the clone and in the report files next to it —, never in the directory the skill was started from. Every statement about a PR has a check report behind it, and where none could be produced, the reason stands instead of a verdict. And nothing leaves the machine without a yes: until the approval this skill only reads GitHub; afterwards it writes exactly two things — the push of its own commits and the approval —, and only for the PRs the human named.

## 1. Arguments and prerequisites

**Repository.** `$repoUrl` is mandatory. Accepted are `https://github.com/<owner>/<repo>`, with or without `.git`, with a path suffix like `/pulls`, the SSH form `git@github.com:<owner>/<repo>.git` and the short form `<owner>/<repo>`. All of these are mapped to `<owner>/<repo>`; anything that cannot be mapped — empty, not a GitHub host, only an owner — names the invocation `/tool-check-renovate-pr-in-repo <repoUrl> [prs]` and aborts. Do not guess a repository from the working copy: whoever starts the skill names the repo.

**PR selection.** `$prs` is optional: PR numbers as one argument in quotes (`"12 15"`), separated by spaces or commas, with or without `#`. Without it, every open Renovate PR applies. A named number that has no open Renovate PR is not a reason to abort — it goes in the report under „Übersprungen" with the reason „kein offener Renovate-PR".

**Tools and permissions.** Three checks before you clone, each with the line the human would run:

1. `command -v gh` — if it is missing, abort and point to the installation per https://cli.github.com. There is no substitute via `curl` and a token.
2. `gh auth status` — if it exits with code 1, abort and name `gh auth login`.
3. `gh repo view <owner>/<repo> --json viewerPermission,defaultBranchRef` — `viewerPermission` must be `WRITE`, `MAINTAIN` or `ADMIN`; `READ` or `TRIAGE` means push and approval would be impossible, and you abort and say so. An error here usually means: the repo does not exist, a typo in the URL, or no access; quote `gh`'s response. Remember the name of the default branch from `defaultBranchRef.name`.

Everything else — Node, PHP, Java, package managers — is needed not by this skill but by the trial run of `tool-dep-update-check`, and that reports on its own what it is missing.

## 2. Clone

`TMP=$(mktemp -d)`, then `mkdir -p "$TMP/reports" "$TMP/prs"`, then `gh repo clone <owner>/<repo> "$TMP/renovate-check-<repo>"`. Cloning via `gh` takes auth and protocol choice from the `gh` configuration, and you create the directories before any subagent runs — unlike the Write tool, `cat > <file>` does not create directories, and a subagent that fails on a missing directory looks like one without a verdict.

The clone is the only place where Git commands may change anything, and you give its absolute path to every command: `git -C "$KLON" …`, or for package managers and scripts a subshell `(cd "$KLON" && …)`. A bare `cd` does not exist — the directory the skill was started from is a read zone and stays untouched; the report at the end cites `git status --porcelain` of that directory as evidence, if it is a repository. If the clone fails — no network, no access, full disk —, abort with `gh`'s output; up to this point nothing has been created except the empty temp directory, and you name it with `rm -rf "$TMP"`.

After the clone all branches are present as `origin/<name>`; the PR branches need no local checkout, because diff and check run over refs. If an `origin/<head>` is missing anyway — the branch was rewritten between `gh pr list` and the clone —, `git -C "$KLON" fetch origin <head>` fetches it; if it stays missing, the PR is „nicht prüfbar: Branch nicht erreichbar".

## 3. Inventory and classification

Read `${CLAUDE_SKILL_DIR}/references/renovate-pr.md` before you touch the first PR: it contains, with sources, the recognition of Renovate PRs, the title and body templates, the PR kinds with their checkability and the `gh` commands. This section assumes them and does not repeat them.

**Inventory.** Two lists whose union is the work list — and you fetch both, even if the second contains all open PRs: the author search finds PRs of the Renovate app account even when their branch prefix differs from the one you read from the configuration, and the prefix filter finds a self-hosted Renovate that does not appear as an app. One list alone has a blind spot.

```
gh pr list --repo <owner>/<repo> --state open --limit 500 --search 'author:app/renovate' \
  --json number,title,headRefName,baseRefName,isDraft,mergeable,reviewDecision,statusCheckRollup,author,url,updatedAt
gh pr list --repo <owner>/<repo> --state open --limit 500 \
  --json number,title,headRefName,baseRefName,isDraft,mergeable,reviewDecision,statusCheckRollup,author,url,updatedAt
```

You filter the second list to branches with the prefix from the clone's Renovate configuration — `branchPrefix`, preceded by `additionalBranchPrefix` if present, default `renovate/`; the file names are in the reference, section A, and `grep -o '"branchPrefix"[[:space:]]*:[[:space:]]*"[^"]*"'` is enough for JSON and JSON5 alike. `--limit 500` is mandatory, because `gh pr list` otherwise silently truncates at 30; the total number of open PRs (`gh pr list --repo <owner>/<repo> --state open --limit 500 --json number --jq length`) goes into the report next to the inventory count, so it is visible what was not a Renovate PR. `body` is deliberately not in the field list: Renovate bodies carry collapsed release notes of often ten to fifty kilobytes, and twenty of them in context are the end of the run. With `$prs` you keep only the named numbers.

The inventory is empty? Then say so — total count, author filter, prefix — and end the run with the report; the clone is removed as usual.

**Body and diff per PR.** You fetch both into files, not into context:

- `gh pr view <n> --repo <owner>/<repo> --json body --jq .body > "$TMP/prs/<n>.md"`, and from it only the table rows: `grep -E '^\|.*(→|->)' "$TMP/prs/<n>.md"`. The `Change` column reads `` `a` → `b` `` in current Renovate and `` `a` -> `b` `` in older PRs; a body without such a row is normal for lockfile maintenance (`All locks refreshed`) and otherwise a hint at a kind you look up in the reference.
- `git -C "$KLON" diff --name-only origin/<basis>...origin/<head>` with `<basis>` from `baseRefName`, and for the pairs `git -C "$KLON" diff origin/<basis>...origin/<head> -- <manifeste>`; which files are manifests and locks is stated in `tool-dep-update-check`, section 2 — `package.json`, `composer.json`, `pom.xml`, `build.gradle(.kts)` together with `gradle/libs.versions.toml`, `pyproject.toml` and `requirements*.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `*.csproj` together with `Directory.Packages.props` — and the locks next to them.

**Classification**, in this order, the first match decides; every reason goes verbatim into the report:

1. `isDraft` → übersprungen: Draft.
2. `mergeable` is `CONFLICTING` → übersprungen: Konflikt mit `<basis>`. Rebuilding is Renovate's rebase checkbox; it goes in the report under „Nächste Schritte" — you do not tick it.
3. Own approval on the current head → übersprungen. Proven, not guessed: `gh api repos/<owner>/<repo>/pulls/<n>/reviews --paginate --jq '.[] | select(.state=="APPROVED") | {user: .user.login, commit: .commit_id}'` against `gh pr view <n> --repo <owner>/<repo> --json headRefOid --jq .headRefOid` and your own login from `gh api user --jq .login`. An own approval on an older commit does not count as a reason — the PR has been rewritten since and is checked anew.
4. Only lock files changed and branch or title name lockfile maintenance → übersprungen: Lockfile-Maintenance, kein Paar.
5. Only files under `.github/workflows/`, `Dockerfile*`, `docker-compose*`, `action.y(a)ml`, or only digests in the diff → übersprungen: Docker, Actions oder Digest — kein Ökosystem der Prüfung.
6. Title starts with `Pin`, `Roll back` or `Replace` → übersprungen: Pin-, Rollback- oder Ersetzungs-PR, with the reason from the reference, section C.
7. Manifest diff yields one or more pairs → checkable. Per changed package: name, old and new version from the diff, leading `^`, `~`, `=` and `v` stripped; the same package in several manifests of a monorepo is one pair, because the checking skill finds all manifests itself. If only the lock changed, but the branch carries `-lockfile` or the table names a package, the pair comes from the lock diff (`version` lines of the named package). Then the counter-check: every table row of the body must match a pair and every pair a row; if it does not match — different package, different version, one row too many —, the PR is „nicht prüfbar: Manifest und Body widersprechen sich", with both versions in the report. Several pairs make a group PR.
8. None of this — no manifest, no lock, no table row → „nicht prüfbar: keine Dependency-Änderung erkannt", with the changed paths.

You read the CI state from `statusCheckRollup`: a `FAILURE` or `ERROR` in it means red, only `SUCCESS`, `NEUTRAL` or `SKIPPED` means green, empty or pending means unknown. It decides nothing about the check, only about the recommendation.

**Own PRs.** If the PR's `author.login` equals your own login from `gh api user --jq .login` — with a self-hosted Renovate under your own account, or with handmade PRs carrying a Renovate prefix —, this account cannot approve it: "Pull request authors cannot approve their own pull requests" (reference, section F). The PR is checked and adapted like any other, but its action is `push` instead of `push + approve` and `keine: eigener PR, Approval durch ein anderes Konto` instead of `approve`; section 7 skips the approval with exactly this reason instead of letting it fail at GitHub.

The classification is kept as a table in context, one row per PR — number, title, branch, base, kind, pairs, CI, author equals own account or not —, and you update it after every PR. It is your state; the clone is the other one.

## 4. Check per pair

Every pair is checked by `tool-dep-update-check` — in a subagent with fresh context, because a single check report with changelogs and trial run costs tens of thousands of tokens and a repo easily has ten Renovate PRs. The subagents run one after another, PRs by ascending number, pairs within a PR in the order of the manifests: two trial runs in the same clone would fight over worktrees, caches and CPU.

The check runs against `<basis>` — the PR's `baseRefName` —, not against the PR branch. There the lock already holds the target, and the checking skill would report "lock above the target" and ask instead of judging.

**The contract.** Per pair, start exactly one subagent via the Agent tool, type `general-purpose`, with this prompt and the placeholders filled in:

```
Working directory: <absolute clone path>. Every Bash command runs there (`cd <path> && …`
or `git -C <path>`), Glob and Grep only with absolute paths under this directory.
Invoke the skill `tool-dep-update-check` via the Skill tool (with plugin prefix `skills:`
or without) with the arguments `<package> <source> <target> <base>` and follow it
completely. Write its complete report via Bash (`cat > <file> <<'EOF'`) to
<tmp>/reports/pr-<n>-<package>.md — not with the Write tool, which is not pre-approved.
Return only: the title line with the verdict, the lines Belastbarkeit, Auflösung and
Voraussetzungen, every adaptation from „Bruchstellen" with file:line and the grep pattern
from „Betroffen", the commands from „Probelauf" (install, build/types/lint, tests) and the
section „Nicht geprüft".
If the skill ends with a follow-up question or an abort instead of a verdict, return
„nicht prüfbar: <Grund in einem Satz>". Change nothing in the clone outside the
worktree that the skill itself creates and removes.
```

**The answer** is at most about thirty lines: the first starts with `# Dependency-Update:` and ends with `Möglich`, `Möglich mit Anpassungen` or `Blockiert` — or it starts with `nicht prüfbar:`. Anything else is a failure of the subagent, not a verdict: restart it exactly once, with the same prompt; if the verdict is still missing, the pair carries „nicht prüfbar: Subagent ohne Urteil", and the run continues with the next pair. You never read the whole report file into context; what you need is in the answer, and the path to the file goes into the report.

**From verdicts to recommendations.** Per PR you combine the verdicts of its pairs according to this table; the first matching row applies:

| Verdicts of the pairs | Belastbarkeit (reliability) | CI on the head | Recommendation |
|---|---|---|---|
| at least one pair „nur statisch" (trial run did not run) | — | any | nicht freigeben: unverifiziert — with the reason from „Nicht geprüft" |
| at least one `Blockiert` or `nicht prüfbar` | any | any | nicht freigeben — with precondition or reason |
| at least one `Möglich mit Anpassungen`, all others `Möglich` or likewise `Möglich mit Anpassungen` | trial run ran | any | anpassen (section 5), dann approven — this also applies to group PRs, needing adaptations is not a blocker; after section 5 the recommendation reads „angepasst, approven", „nicht freigeben: Anpassung unvollständig" or „nicht freigeben: Anpassung rot" |
| all `Möglich` | „Probelauf grün" | red | nicht freigeben: CI rot — Mensch entscheidet |
| all `Möglich` | „Probelauf grün" | green or unknown | approven |

A group PR with one blocked, uncheckable or only statically checked pair is not to be approved as a whole, even if the other pairs would be possible; splitting the group is Renovate configuration, not your task. A group PR whose pairs are all possible and one or more of which need adaptations, on the other hand, is „anpassen, dann approven" — just like a single PR with adaptations.

## 5. Adaptation and commit

A PR with the recommendation „anpassen, dann approven" gets its adaptations now — in the clone, on the PR branch, as commits that the human later pushes or discards. You only come here once all pairs of the PR are checked and none of them is blocked, uncheckable or only static; what the check reports name under „Bruchstellen" with `Datei:Zeile` is the whole assignment. An observation on the side — a style issue, a second bug, a deprecation — belongs in the report, not in the commit. And the assignment has limits: if an adaptation requires a new package, a change to manifest or lock, or to locations no check report names, you do not implement it — the PR is „nicht freigegeben: Anpassung außerhalb des Auftrags", the required change goes verbatim into the report, and the human decides whether the checking skill's verdict holds. If you have already changed something for this PR, it is parked, not discarded — see "Verify once".

**Branch and install.** `git -C "$KLON" switch -c <head> --track origin/<head>`, then install once with the PR branch's lock, in a subshell: `(cd "$KLON" && <install command from the trial run>)` — `npm ci`, `composer install`, whatever the check report names under „Probelauf". The lock already holds all target versions of the PR; you do not change manifest and lock.

**One commit per pair.** In the order of the check, for each pair with adaptations: the changes at exactly the locations its check report names, with the Edit tool and absolute paths under `"$KLON"`; then immediately `git -C "$KLON" add -- <paths>` with exactly these paths — never `-A`, never `.` — and the commit. If two packages touch the same file, their changes go into one shared commit that names both packages in the subject. No verification runs between the commits, and not before the first one either: the PR branch's lock holds all target versions at once, a test run after the first package would be red at the second one's breaking point without saying anything about the first — and the verification comes after the last commit, so every attempt exists committed and can be parked instead of vanishing with `git checkout`. If the install changed lock or manifest files, they only belong in a commit if the adaptation requires them — a package pulled along, for instance; otherwise `git -C "$KLON" checkout -- <file>`, and the report mentions it.

**The message.** If the list of available skills contains one named `write-commit-message` — with plugin prefix or without —, you invoke it via the Skill tool for each commit, as soon as the index holds the package's state. It would read `git diff --staged` in the session directory, where the index is empty, and ask the human; the argument text intercepts both, in English, because it works in English:

```
Context for the commit message. The staged diff is complete and this
context answers your questions, so do not ask the user.
Repository: <klon> — read the staged diff with `git -C <klon> diff --staged`.
- Affected part: <paket> <ausgang> → <ziel> in <owner/repo>; the subject
  must name the package and the version jump
- Problem / motivation: <Bruchstelle aus dem Prüfbericht, ein Satz>
- Trigger: <wo es bricht — Datei:Zeile, Fehlermeldung aus dem Probelauf>
- Change: <was die Anpassung tut>
- Why this approach: <warum so, aus dem Prüfbericht>
Return only the message. Do not commit, do not add trailers or references.
```

If no such skill is in the list or the call fails, the fallback applies once: read `git -C "$KLON" log --format='%s%n%b' -20` and adopt the project's style — imperative subject with package and jump, language as in the existing history, below it the why from the check report. In both cases: no trailers, no reference to this skill, no authorship line — not even if a rule of the environment demands `Co-Authored-By`; for this skill it has been decided that the message stays without trailers. The message goes via `mktemp` into a file outside the clone, you strip every trailer line at the end, then `git -C "$KLON" commit -F <file>` — no `-m` with multi-line text, no `--no-verify`, and never `--amend`: Renovate's commit must not be amended, because Renovate rebases over an amend, and your own commits do not need it. If a project hook fails, no commit was created; quote its output and do not bypass it.

**Check completeness.** After the PR's last commit and before verification there is a gate that the trial run does not replace: build, types and tests only find what they hit — a class name in a YAML file, a dynamic call, a location without a test stay green and are still missing. Two static checks against the diff `git -C "$KLON" diff -U1 origin/<head>..<head>`:

1. **Tick off.** Every location that a check report of this PR names under „Betroffen" with `Datei:Zeile` must be covered by a hunk: the file's diff shows `@@ -a,b +c,d @@` per hunk, and the line lies within a … a+b-1 (if `,b` is missing, b = 1; the one context line lets an insertion directly next to the location count too). The report's numbers come from `<basis>`. If the file on `origin/<head>` differs from it because the PR branch lags behind the base — `git -C "$KLON" diff --stat origin/<head> origin/<basis> -- <file>` is then not empty —, you find the line by its content: `git -C "$KLON" show origin/<basis>:<file> | sed -n '<line>p'`, then `git -C "$KLON" grep -n -F -e '<content>' origin/<head> -- <file>`, and check with the number found that way. A location without a hunk is a forgotten adaptation — yours, not the report's.
2. **Re-grep.** Every pattern that a report names under „Betroffen" for an adapted breaking point — if it is missing, you take the API's name from the breaking point's heading — runs once more over the branch head: `git -C "$KLON" grep -n -E '<pattern>' -- ':!vendor' ':!node_modules' ':!*.lock' ':!*-lock.json' ':!*-lock.yaml'`; `git grep` only sees versioned files, so not the installed packages. Every hit must lie on a changed line — a `+` line in the diff, not a context line. A hit on an unchanged line is a candidate that you read with `-C2` before classifying it: a usage no report names — or none, because it is a comment, documentation, or an own symbol of the same name. Every classification goes into the report with its reason; a silently discarded hit is a forgotten location with a clear conscience.

The gate is passed when every reported location is covered and no hit on an unchanged line is a usage. A forgotten location you catch up on, the way "Verify once" describes for red — mixed reset, work it in, commit again per package — and go through the gate again; that is one attempt, and there are at most three. A usage no report names, on the other hand, is not a location you adapt, but the evidence that the check report was incomplete and its verdict does not hold as is: the PR is „nicht freigegeben: Anpassung unvollständig — <Datei:Zeile> verwendet <API>, kein Bericht nennt sie", parked like a red adaptation, and the human decides with the report in hand. The result of both checks — locations covered, patterns run, hits and classification — goes into the report, even if everything stayed empty.

**Verify once.** Once the gate is passed, the commands from the trial run — build, types, lint, tests, as the check report names them — run exactly once on the branch head, in the subshell in the clone. Green → the PR is „angepasst", the recommendation reads „approven", and `git -C "$KLON" log --oneline origin/<head>..<head>` delivers SHAs and subjects for the report. Red → the cause lies in the adaptations or in a breaking point the report did not name; both are your job, as long as the correction stays at the named locations and no package is installed, no test expectation is weakened and no manifest is touched — a test file the check report names as a usage location is an adaptation location like any other; one it does not name is off limits. The correction goes via `git -C "$KLON" reset origin/<head>` — mixed: HEAD and index back, working copy stays; not `--soft`, which would leave all packages staged, and the first new commit would take them all along —, work it in, stage and commit per package again, through the gate again, verify again. At most three attempts, gate and verification together. If the cause instead lies at a location no report names, or it requires a package, manifest or lock, there is no attempt — that is „Anpassung außerhalb des Auftrags" from above. In both cases the PR ends the same way: you park what you changed and discard nothing. Parking means `git -C "$KLON" switch -c <head>-anpassung-rot`; if anything unstaged from the adaptation is still lying around, you secure it there with `git -C "$KLON" add -- <paths>` and a commit `WIP: Anpassungsversuch <paket>` — the only commit whose message you write yourself; then `git -C "$KLON" switch <head>` and `git -C "$KLON" reset --hard origin/<head>`. The PR is „nicht freigegeben: Anpassung rot" or „… außerhalb des Auftrags" respectively, with the error message or the required change in one line; branch and path go into the report, so the human can read the diff before making the adaptation themselves. An adaptation that only turns green by being reverted is none — then the verdict „mit Anpassungen" was wrong, and that is a finding for the report, not a commit.

**Back and onward.** After every PR `git -C "$KLON" switch <default>`, then the next one. The way back per PR is `git -C "$KLON" branch -f <head> origin/<head>` — it goes into the report under „Nächste Schritte" and is never run by you: it would take the commits you just made off the branch. Nothing is pushed here yet — the push comes only after the approval in section 7, and without a yes not at all.

## 6. Approval

When the last PR is checked and, where needed, adapted, you first output the final report (section 8) — complete, with „Freigabe: ausstehend" and the columns Push, Approve and reviewDecision set to „—". Then follows the approval table and exactly one question, and with that your turn ends:

```
| PR | Paket(e) | Sprung | Urteil | Belastbarkeit | Commit | Aktion |
|----|----------|--------|--------|---------------|--------|--------|
| #<n> | <paket a→b> | patch \| minor \| major | <Urteil> | <Belastbarkeit> | <sha> (<paket>), … \| — | approve \| push + approve \| push (eigener PR, Approval durch anderes Konto) \| keine: <Grund> |
Freigeben? „ja" für alle markierten, „nur #12 #15" für eine Auswahl, „nein" für keine.
```

Marked are the PRs with action `approve`, `push + approve` or `push`. If there are none, the question is dropped, and the report is the end of the run. You ask the question as text, not via a question tool: in a non-interactive run there is then simply no answer, and exactly that is meant to mean "nothing happened" — no push, no approval, the clone stays with its commits, and the report has already said everything the human needs to know. The pre-approval of the tools only applies to the turn that invoked the skill; that `git push` and `gh pr review` go through the normal permission prompt again after the yes is desired for exactly these two commands.

**The answer** you read literally: `ja` → all marked PRs; `nein` → none, the run ends with one sentence; `nur #12 #15` (numbers with or without `#`, separated by spaces or commas) → these, provided they are marked — a named number without an action you mention and leave out. Anything else — „ja, aber ohne #15", „erst mal #12", a question — you answer with a one-sentence follow-up question and wait again; you do not guess what was meant, because every misreading pushes into someone else's repository.

## 7. Push and approve

Per approved PR, in the order of the table, three steps — and first all three for one PR, then the next:

**Push**, only if adaptation commits sit on `<head>`: `git -C "$KLON" push origin <head>` — without `--force`, without `--force-with-lease`, never. If the remote rejects the push, Renovate rebuilt the branch between the clone and now (reference, section E): `git -C "$KLON" fetch origin <head>`, then on the branch `git -C "$KLON" switch <head> && git -C "$KLON" rebase origin/<head>` — that carries exactly your commits from `origin/<head>..<head>` onto the new state —, then push again. At most three attempts; a conflict in the rebase is an abort with `git -C "$KLON" rebase --abort`. After that the PR counts as „nicht gepusht: <Grund>", gets **no** approval — an approval on a PR whose adaptation is not on the branch would be wrong — and the next PR is up. A PR with action `approve` and no commits skips this step.

**Approve** — dropped without an attempt if the PR comes from your own account (section 3): the column then says „nicht erteilt: eigener PR", and the push alone is the result. Otherwise the approval text goes via `mktemp` into a file outside the clone — in the language of the project's commit messages (`git -C "$KLON" log --format=%s -20` shows them) and in exactly this form:

```
<paket> <ausgang> → <ziel>: <Urteil>, <Anpassungen in einem Halbsatz oder „no adaptations needed">; trial run
<grün oder rot mit den gelaufenen Befehlen und Testzahlen>.

<details><summary>Details</summary>
<p>

<der vollständige Inhalt von <tmp>/reports/pr-<n>-<paket>.md>

</p>
</details>
```

Per pair one header line and its own `<details>` block with the full check report — with several pairs using `<summary>Details: <paket></summary>` —, because otherwise the report lives only on your machine, and whoever reads the PR should see the evidence without asking you; GitHub needs the blank lines around the content so it renders the markdown inside. With adaptation commits, before the closing line stands the sentence that these commits (SHAs) sit on the branch and Renovate's rebase checkbox would discard them. The closing line `by tool-check-renovate-pr-in-repo` is the same in every language and makes visible that the skill wrote the review; the text contains no account name, because GitHub lists the review under the logged-in account anyway. Neither skill name contains links to its source. No trailer, no pleasantries. Then `gh pr review <n> --repo <owner>/<repo> --approve --body-file <file>`. If that fails — "Can not approve your own pull request", missing permission, network —, the PR is „nicht erteilt: <Ausgabe von gh in einer Zeile>", and you do not try again: a second approval changes nothing the first one could not.

**Read back:** `gh pr view <n> --repo <owner>/<repo> --json reviewDecision,reviews`. Your own review must appear in `reviews` with `author.login` equal to `<login>` and `state` `APPROVED` — otherwise „nicht erteilt" applies, no matter what `gh pr review` reported. You read `reviewDecision` three ways: `APPROVED` → counts; empty → no review requirement configured, the approval stands; `REVIEW_REQUIRED` despite your own approval → does not count, because a branch rule ("Require approval of the most recent reviewable push" after your own push, or "Dismiss stale approvals") does not let it apply — a second human must approve, and you do not approve again; `CHANGES_REQUESTED` → someone else requested changes, and that is stated as such in the report.

**Clean up and addendum.** Once all approved PRs are done, the same rule as in section 8 applies to the clone: it goes only if no PR carries unpushed commits anymore — not even one that was not approved — and no parked branch exists; otherwise it stays, with its path. Then the addendum to the report: the table from section 8 with the columns Push, Approve and reviewDecision filled in, the line „Freigabe" with the answer, the clone line and the next steps — the merge is the human's business, and for every PR that was not approved or not pushed, the commands are listed once more.

## 8. Report and cleanup

Before the report you remove the clone — `rm -rf "$KLON"` —, but only if nothing unpushed is in it: no adaptation commit, no parked branch. Before the approval that is only the case if no PR was adapted; otherwise it stays, with path and way back in the report, because the commits in it are the result of this run and the subject of the approval. `"$TMP/reports/"` and `"$TMP/prs/"` stay in any case, because the report links there; `rm -rf "$TMP"` is the cleanup line for the human when they are done. If the start directory is a Git repository, `git status --porcelain` there shows the same state as before the run, and that goes into the report as a line.

```
# Renovate-PRs: <owner/repo> — <n> offen, davon <r> Renovate-PRs: <a> approven, <b> angepasst und approven, <c> nicht freigeben, <d> übersprungen

**Klon:** <pfad> — entfernt | bleibt stehen (Anpassungs-Commits auf <head>, … | geparkter Branch <head>-anpassung-rot) → nach dem Push `rm -rf <pfad>`; Berichte unter <tmp>/reports/ → `rm -rf <tmp>` räumt alles
**Startverzeichnis:** unverändert (`git status --porcelain` leer) | kein Git-Repository
**Freigabe:** ausstehend — Frage folgt | ja (alle) | Auswahl: #… | nein | keine Antwort — nichts gepusht, nichts approved | entfällt — kein PR mit Aktion

| PR | Paket(e) | Sprung | Urteil | Belastbarkeit | CI | Empfehlung | Commit | Push | Approve | reviewDecision |
|----|----------|--------|--------|---------------|----|------------|--------|------|---------|----------------|
| #<n> | <paket a→b> | patch \| minor \| major | <Urteil> | <Belastbarkeit> | grün \| rot \| unbekannt | approven \| angepasst, approven \| nicht freigeben: <Grund> | <sha> (<paket>), … \| — | gepusht @ <sha> \| nicht gepusht: <Grund> \| — | erteilt \| nicht erteilt: eigener PR \| nicht erteilt: <Grund> \| — | APPROVED \| leer: keine Review-Pflicht \| REVIEW_REQUIRED: zweiter Mensch nötig \| CHANGES_REQUESTED \| — |

## PR #<n> — <Titel>
- **Branch:** <head> → <basis>, Autor <login>, gefunden über Autor | Branchpräfix
- **Prüfung:** <Kopfblock je Paar — Urteil, Belastbarkeit, Auflösung, Voraussetzungen>; voller Bericht: <tmp>/reports/pr-<n>-<paket>.md
- **Anpassung:** je Paket <sha> „<Betreff>" — <dateien>; Verifikation des Branch-Heads: <befehl> → grün (Commits einzeln ungeprüft) | rot: <Fehler in einer Zeile> → Versuche geparkt auf <head>-anpassung-rot, Branch gleich origin/<head> | unvollständig: <Datei:Zeile> verwendet <API>, kein Bericht nennt sie → Versuche geparkt auf <head>-anpassung-rot | keine nötig
- **Vollständigkeit:** <n>/<n> gemeldete Stellen im Diff, Re-Grep über <m> Muster nur auf geänderten Zeilen | <k> Treffer auf unveränderten Zeilen: <Datei:Zeile> — <Einordnung>, … | entfällt: keine Anpassung
- **Push:** origin/<head> @ <sha> | nicht gepusht: <Grund> | nichts zu pushen   (nach der Freigabe)
- **Approve:** erteilt — reviewDecision APPROVED | leer: keine Review-Pflicht, Approval steht | REVIEW_REQUIRED: zählt nicht, zweiter Mensch nötig | nicht erteilt: <Grund>   (nach der Freigabe)
- **Nicht geprüft:** <aus dem Prüfbericht> | nichts

## Übersprungen
- #<n> <Titel> — Draft | Konflikt mit <basis> | eigene Approval auf dem aktuellen Head | Lockfile-Maintenance | Docker/Actions/Digest | Pin | Rollback | Ersetzung | kein offener Renovate-PR (aus `prs`)

## Nicht prüfbar
- #<n> <Titel> — Manifest und Body widersprechen sich: <beide Fassungen> | keine Dependency-Änderung erkannt: <Pfade> | Subagent ohne Urteil | Rückfrage des Prüf-Skills: <Satz> | Branch nicht erreichbar

## Nächste Schritte
- <vor der Freigabe: die Freigabefrage folgt; danach: je approvtem PR der Merge — `gh pr merge <n> --repo <owner/repo>` oder der Weg des Projekts —, den machst nicht du>
- <je „nicht freigegeben" oder „nicht gepusht": `git -C <klon> push origin <head>` und `gh pr review <n> --repo <owner/repo> --approve`, falls der Mensch es doch will; Rückweg `git -C <klon> branch -f <head> origin/<head>`>
- <je „Anpassung rot" oder „Anpassung unvollständig": `git -C <klon> diff origin/<head>..<head>-anpassung-rot` zeigt die Versuche; die Anpassung macht der Mensch mit dem Prüfbericht und der nicht gemeldeten Stelle in der Hand>
- <je „nicht freigeben": die Vorbedingung aus dem Prüfbericht; bei Konflikt: Renovates Rebase-Checkbox; bei CI rot: der Lauf, den der Mensch ansehen sollte>
- `rm -rf <tmp>` — Berichte, Bodies und, nach dem Push, der Klon
```

Empty sections get a line „keine", so it stays visible that they were checked. On GitHub you write only after a yes and only two things: the push of your own commits and the approval.

**Termination:** A PR that cannot be classified, checked, adapted or pushed does not stop the run — it gets its reason, and the next PR is up. Aborting happens only in sections 1 and 2, where without a tool, access or clone not a single PR could be checked. A subagent gets exactly one second attempt, an incomplete or red adaptation at most three together, a rejected push at most three, an approval exactly one; the approval question is asked once, an unclear answer is followed up once per message; a result is never invented.

## 9. Stance

The first temptation is to believe the title. `Update dependency lodash to v4.17.21` is a template, not evidence; the pair is in the diff, and the table is the counter-check. Where the two diverge, that is a finding, not a matter of interpretation.

The second is to pull bodies and reports into context because it is convenient. A body is ten kilobytes of release notes, a check report thousands of tokens — after the third PR the run can no longer be saved. Files and grep, header blocks and paths.

The third is to run the check on the PR branch because "the update is already in there". There the lock already holds the target, and the checking skill asks instead of judging. The base is the state that would be updated — the adaptation, on the other hand, belongs on the PR branch, because it belongs to the update.

The fourth is to let a subagent work in the session directory. Without `cd <klon> &&` in every command it checks the wrong project — and the human's start directory is a read zone. The path is in the contract, and the report proves with `git status --porcelain` that nothing happened there.

The fifth is to turn „nur statisch" (only static) into "good enough". A trial run that did not run has proven nothing; such a pair is not to be approved, and the reason from „Nicht geprüft" belongs in the report, so the environment gets repaired instead of the check being watered down.

The sixth is to start subagents in parallel because it would be faster. They share the clone, and two trial runs at once are two worktrees, two installs and a result nobody can trust.

The seventh is to verify after every commit because it looks more thorough. On a group branch the run after the first package is red at the second one's breaking point — a result that means nothing and costs three attempts. Once, after the last commit.

The eighth is to make a red adaptation green by reverting it, weakening a test or installing an extra package. The first turns "with adaptations" into a false "possible", the second hides the break, the third is a different update. Parked and reported is more honest than green.

The ninth is to push "because everything is green anyway" before the yes is there — or to push after an answer that was not a clear yes. The push goes into someone else's repository and freezes Renovate's PR; it is the human's decision, not your shortcut.

The tenth is to force a rejected push. `--force` would overwrite Renovate's new state; the way is fetch, rebase your own commits, push again — or stop and report.

The eleventh is to amend because Renovate's commit is so close at hand, or to approve a PR whose push failed. The one overwrites Renovate's commit, the other approves a state that does not exist on GitHub.

The twelfth is to delete the clone before `reviewDecision` has been read or while commits are still in it — or the temp directory with the reports. First read back, then clean up; the clone goes only without anything unpushed; the reports stay; the cleanup line is in the report.

And the thirteenth is to infer completeness from a green trial run. Green means build and tests found nothing — not that nothing is there: the class name in the configuration, the dynamic call, the file without a test stay green and break in production. First tick off and grep once more, then verify.
