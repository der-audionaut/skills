---
name: tool-dep-update-check
description: Checks whether a project dependency can be updated from one version to a target version — finds it in the manifest, measures the jump via the registry, reads changelog and release notes for the entire range, matches breaking points against the actual usage in the code, checks platform and dependency conflicts, does a trial run in a throwaway worktree and hands over a verdict with evidence, without changing the project. Takes dependency, source version and target version as arguments, optionally followed by the branch to check on — without one, the checked-out branch.
argument-hint: "[dependency] [sourceVersion] [targetVersion] [branch]"
arguments: dependency sourceVersion targetVersion branch
allowed-tools: Bash Read Grep Glob Edit WebFetch WebSearch
---

**Output language: German.** You write the report, follow-up questions and abort messages in German, even if the project, changelog and release notes are in English. Quotes from changelogs, identifiers, commands and error messages stay as they are.

Check whether the dependency `$dependency` in this project can be updated from version `$sourceVersion` to version `$targetVersion` — on the branch `$branch`, or, if none is given, on the checked-out one. Work like someone who, before the update PR, answers the question the PR would otherwise only raise in review: What breaks, what has to be adapted, what stands in the way? You do not perform the update; you answer whether and how it can be done.

Three rules carry the whole procedure: You change nothing that remains — no commit, no modified manifest file in the working copy, no push; the trial run happens in a throwaway worktree that disappears at the end. Every statement about a break has a source, and every statement "not affected" has the grep that proves it. And the verdict follows the evidence, not the version number: a major jump is not automatically blocked, a patch not automatically harmless.

## 1. Resolve the arguments

**Dependency, source and target version.** All three are mandatory. If one is missing, state the invocation — `/tool-dep-update-check <dependency> <sourceVersion> <targetVersion> [branch]` — and abort. Do not guess a version from the lock file: whoever explicitly names two versions wants exactly this pair checked, and whoever does not name them should name them. Strip a leading `v`, `^` or `~`; a range instead of a version (`^5.0`, `5.*`) is not a target — name the highest matching version from the registry as a candidate and ask.

Compare the two versions before you check anything: if they are equal, there is nothing to do, say so. If the target is below the source, it is a downgrade — a different task in which changelogs are read backwards; say so and ask whether that is what was meant. Where the versions do not follow semver (`1.2.3.RELEASE`, `2.0.0rc1`, calendar versions), compare with the ecosystem's tool instead of by gut feeling; the commands are in `${CLAUDE_SKILL_DIR}/references/ecosystems.md`.

**Branch.** The branch is the state that is checked — not a branch that would be switched to.

1. `$branch` is empty → the checked-out state (`HEAD`). A detached HEAD is no obstacle here, because you only read and branch a worktree off it; the report then names the commit instead of a branch name.
2. `$branch` exists as a local branch (`git rev-parse --verify refs/heads/<name>`) → take it. If it has an upstream and is behind it, that goes into the report as a line — what is checked is the state the human has.
3. It exists only on a remote (`origin/<name>`) → `git fetch <remote>` first, then the remote branch; note whether it moved during the fetch. Unlike with a rebase, no local branch is needed: reading and branching a detached worktree work from any ref.
4. It does not exist at all → name the existing branches and abort.

With `$branch` you read everything via `git show <ref>:<pfad>` and search with `git grep <muster> <ref> -- <pfade>`, without switching — you do not touch the human's working copy. Without `$branch` you read the working copy as it is, including uncommitted changes, because that is the state that would be updated. If manifest or lock differ from the commit (`git status --porcelain -- <manifest> <lock>`), that goes into the report: the trial run in section 6 branches off the commit, not off the working copy.

## 2. Find the dependency in the project

Find the manifests: `package.json`, `composer.json`, `pom.xml`, `build.gradle(.kts)` together with `gradle/libs.versions.toml`, `pyproject.toml` and `requirements*.txt`, `Cargo.toml`, `go.mod`, `Gemfile`, `*.csproj` together with `Directory.Packages.props`. Search by glob across the whole project, but not in `node_modules`, `vendor`, `target`, `build`, `dist`, `.venv` — those hold the manifests of foreign packages. The ecosystem determines lock file, registry queries and commands; read `${CLAUDE_SKILL_DIR}/references/ecosystems.md` as soon as it is settled, not only when a command is missing.

**Resolve the name.** First the exact name (`lodash`, `symfony/console`, `org.springframework.boot:spring-boot-starter-web`, `requests`), then a case-insensitive substring. Multiple hits → list them and abort instead of guessing; `guzzle` hits `guzzlehttp/guzzle` and `guzzlehttp/psr7`, and checking the wrong package is a convincing-looking wrong answer. No hit in a manifest → look in the lock file. If it is there, it is a transitive dependency: its version is chosen by whoever requires it. The check then asks which direct dependencies pull it in and whether each of them allows the target, and ends with the two paths — raise the parents or pin the package itself as a root requirement (npm `overrides`, Composer root `require`, Maven `dependencyManagement`, with Cargo as a direct dependency with a fixed constraint); you do neither, you name them. If it is not in the lock either → say so, name the searched manifests, abort.

If it appears in several manifests (monorepo, workspaces, multi-module) → check all of them. The update is only possible once it is possible everywhere; the report separates by manifest. If they carry different versions, `$sourceVersion` applies to the matching ones; list the others with their state.

**Constraint versus lock.** Read the constraint in the manifest (`^4.17.20`) and the resolved version in the lock (`4.17.21`). What actually runs is the lock version, and it is the starting point for changelogs and trial run; `$sourceVersion` stays in the report header. If both are equal, good. If the lock lies between source and target, the project is already part of the way there — say so and measure the jump from the lock version. If the lock is below the source or above the target, something is wrong — wrong package, wrong branch, wrong manifest — and you ask instead of continuing. Without a lock (Maven with fixed versions, projects without a lock file) the manifest version is the starting point; if it holds a range, take the installed version if there is one, otherwise the current resolution of the range, and say which one you took.

Also note whether the target lies within the existing constraint: then the update is a pure lock update without a manifest change, otherwise the manifest changes. That belongs in the report's next steps, because it determines the command.

## 3. Measure the jump

Ask the registry about the target, with the queries from the ecosystem reference. A private registry (`.npmrc`, Composer `repositories`, `settings.xml`, `pip.conf`) is picked up by the project, not by you.

- **Does the target exist?** No → abort and name the nearest existing versions below and above it; maybe it is a typo. If it is withdrawn (yanked, retracted), marked deprecated or a pre-release → that is a finding in the report header, not an abort.
- **How far is it?** Release dates of source and target and the number of versions in between, separated by major, minor, patch. Three years between two versions say more than the numbers. Whether there is anything newer than the target is added as one line — not as a recommendation, as information.
- **What kind of jump?** Patch, minor, major per semver — as a hint, not as a verdict. With `0.x`, minor is the break; Spring Boot, Django and many frameworks break in minors; calendar versions say nothing at all. The changelog decides.
- **What does the target require?** Runtime and platform — Node `engines`, PHP version and extensions, Java version, `requires-python`, MSRV, Go directive, Ruby version, target framework — against three states: what the project itself declares (`engines`, `require.php` and `config.platform`, `maven.compiler.release`, `requires-python`, `rust-version`, `go` directive, `.ruby-version`, `TargetFramework`), what CI and a Dockerfile actually use, and what is installed locally. The trial run runs with the local runtime; if it differs from CI, a green trial run is valid only for the local one, and that goes into the report.
- **What does it pull along?** The target's own dependencies and peer requirements against the project's root dependencies: a target that requires `react >= 19` while the project is on 18 is a conflict candidate before a single command has run.
- **Who else depends on it?** Reverse dependencies in the project (`npm explain`, `composer why` and above all `composer why-not <paket> <ziel>`, `mvn dependency:tree -Dincludes=`, `gradle dependencyInsight`, `cargo tree -i`, `go mod why`, `dotnet nuget why`): a plugin that requires `^4` blocks the jump to 5 until it is updated itself. That is a blocker with a precondition, not a no.

Two special cases: in Go, a major from 2 upwards is a different module path (`…/v2`) — the update is then a change of every import path, and section 5 lists them all. And in Cargo, two versions of the same crate can exist side by side; the jump then produces not a conflict but a duplicate, which only breaks when types of the two versions meet.

## 4. Read the changes

Sources in this order: the migration or upgrade guide (`UPGRADE.md`, `UPGRADE-<major>.md`, a "Migration" or "Upgrading" chapter in the documentation) → `CHANGELOG.md`, `CHANGES`, `HISTORY` → the repository's release pages (`/releases/tag/<tag>`; the tag scheme is `v1.2.3`, `1.2.3` or, in monorepos, `<paket>@1.2.3` — `git ls-remote --tags` or the releases API shows it) → last, the diff between the tags (`/compare/<a>...<b>`) or, after the trial run, the diff between installed source and installed target, limited to the public surface: type definitions, exports, README. The repository URL comes from the registry metadata; WebSearch serves to find the changelog, not as a source for its content.

**Read the whole range.** Every version after the starting point up to and including the target. A jump from 4.9 to 5.2 has its breaks in 5.0.0, and those are exactly what someone misses who only opens the notes for 5.2. For long ranges: majors and minors in full, patches skimmed for `breaking`, `security`, `revert`, `deprecat`, `drop` and `remove`. Pre-releases in between often carry the details the final version only summarizes.

Sort what you find into three buckets — the patterns and hiding places are in `${CLAUDE_SKILL_DIR}/references/breaking-points.md`, and you read them before you open the first changelog, so you know what you are looking for:

- **Breaks.** Removed or renamed APIs, changed signatures, changed defaults with visible effect, dropped platform support, changed configuration format, removed subpath exports, changed feature flags.
- **Deprecations.** Still works, warns, goes away later. Relevant because they are the work of the next jump — and because a project that turns warnings into errors already experiences them as breaks now.
- **Behavior changes without an API break.** Not marked as "breaking", but output, order, precision, error class or timing change. Grep does not find them; only tests or reading the call sites do.

Every entry gets its source: version, URL or file, a quoted line. A break without a quote is a guess and is marked as such.

## 5. Match against usage

Determine the surface of the dependency the project actually uses. The package name is rarely what appears in the code: `Pillow` imports as `PIL`, `symfony/console` as `Symfony\Component\Console\`, a Maven artifact as the Java packages in the jar, a crate with a hyphen as an underscore. The mapping per ecosystem is in the reference. Then you search — with `$branch` via `git grep … <ref>`, otherwise in the working copy, always without the vendor directories, so that you do not count the dependency's own code:

- Import, require and use statements, including subpath imports (`<paket>/…`) and pure type imports.
- Indirect usage: configuration in YAML, XML, JSON or environment variables that names classes, options or plugins of the dependency; framework wiring via service definitions, attributes, annotations, decorators; build tools and test setup that load it as a plugin or preset; class names in strings; its command line in scripts, Makefile and CI.
- Re-exports through the project's own modules: a wrapper in the project that imports the dependency once has many callers that never name it — follow the wrapper.

Then each break from section 4 against this surface: **affected** with `Datei:Zeile` — all locations, not the first — together with the grep pattern that found them, or **not affected** with the grep pattern that stayed empty. The pattern belongs in the report in both cases: a "not used" without a pattern cannot be verified, and an "affected" without a pattern cannot be repeated after the adaptation to show that no location is left. For behavior changes you read the call sites and say whether the change reaches them and whether a test would notice it.

If the dependency is the application's framework — Symfony, Spring Boot, Django, Rails, Angular — the surface is the entire application, and matching by grep does not hold up. Then the route via deprecations is the most important one: run the tests on the source state with deprecations visible (the switches per ecosystem are in the reference); what the source already warns about, the target removes. If there is a migration tool for exactly this jump — Rector sets, OpenRewrite recipes, `ng update`, framework codemods — run it in the worktree from section 6; its diff is the list of necessary adaptations. It is a source of findings, not the verdict.

## 6. Trial run

The static check says what ought to break; the trial run says what breaks. Both are needed: the changelog keeps quiet about the undocumented, and the trial run misses what the tests do not cover.

**Create the worktree.** `git worktree add --detach <tmp>/dep-check-<paket> <ref>` — `<tmp>` from `mktemp -d`, `<ref>` is `HEAD` or the resolved `$branch`. `--detach`, because Git does not let a branch be checked out twice and because you do not want to move any branch. If the project has submodules, `git -C <worktree> submodule update --init --recursive`. From here on every modifying command runs in the worktree — with `-C <worktree>` or from there — and before each one you check the path; the human's working copy is a read-only zone. Untracked files such as `.env` or local configuration are not present in the worktree, and you do not copy them: they may point at real systems. If the source state fails because of that, it is an environment finding, not a verdict on the update.

**Order:**

1. **Install the source state**, with the lock as it is (`npm ci`, `composer install`, `mvn dependency:resolve`, `cargo fetch`, `go mod download`, `bundle install`, `dotnet restore`, `uv sync` or `poetry install`). If that fails — toolchain missing, no network, private registry without access — the check stays static: report it under „Nicht geprüft" and continue with section 7. `command -v` tells you what is missing locally.
2. **Resolution as a dry run**, where the tool has one (`npm install <paket>@<ziel> --dry-run`, `composer update <paket> --with <paket>:<ziel> --dry-run`, `pip install --dry-run`, `cargo update -p <paket> --precise <ziel> --dry-run`, …): it shows conflicts before a long install runs. First the strict variant that moves only the package and its own dependencies; then the one that may also pull along other root dependencies (Composer `-w`, then `-W`; with Cargo and npm this happens on its own). Every package that has to move along goes into the report — whether that is acceptable is the human's decision. `--legacy-peer-deps`, `--force`, `--ignore-platform-reqs` and relatives are not a resolution, they hide the conflict; a trial run that only passes with them reports the conflict as a blocker.
3. **Apply the update**, exactly to the target and not as a range: `^5.2.0` could resolve 5.2.3, and then you are checking a different version than the one asked about. The commands per ecosystem are in the reference; for manifests without a command (`pom.xml`, `build.gradle`, `requirements.txt`) you change the version in the worktree copy and run the install. Then check that lock and installation really hold the target, and read the lock diff (`git -C <worktree> diff --stat`): which other packages moved?
4. **Build, type check, linter**, as the project provides them — `CLAUDE.md`, `README`, `package.json` scripts, `Makefile`, CI configuration. Compiler and type errors are the sharpest findings: `Datei:Zeile`, which API. Then the **tests**, the whole suite; if it is very long, first the modules of the affected surface, then the rest, and the report says what ran. Collect deprecation output during the tests: that is what the target now warns about, i.e. the work for the jump after next.
5. **Red?** First clarify whether the update is the cause: in the worktree, restore manifest and lock (`git -C <worktree> checkout -- <manifest> <lock>`, reinstall) and rerun only the red tests. Red on the source state too → not the update's fault, report as pre-existing. Red only with the update → finding, with the error message in one line and the breaking point from section 4 it belongs to — or with "undocumented" if no changelog entry fits; that is a finding of its own and worth an extra sentence.
6. **Do not repair.** You do not adapt code in the worktree to make the trial run green; that is the update PR, not the check. One exception: if a single obvious adaptation — a renamed import — blocks everything else so that no other finding is reachable, you may make it in the worktree to see the rest, and you say exactly what you changed; in the report it appears as a necessary adaptation. Anything more: stop, the human migrates with the report in hand.
7. **Clean up.** `git worktree remove --force <worktree>`, then `git worktree prune`, and `git worktree list` must again show what was there before. A forgotten worktree only becomes noticeable when `git switch` reports the branch as checked out elsewhere. The package managers' caches live outside the repo and stay — that is what they are for.

**Termination:** If the trial run fails to get through install and build in three attempts for environmental reasons — not because of the update — stop, remove the worktree and lay out what you tried and what remains unchecked as a result. The static verdict then stands with lower reliability. A trial-run result is never invented, and a test run that did not finish is reported as not finished.

## 7. Verdict and handoff

The verdict has three levels, and it is derived from the evidence, not from gut feeling:

- **Möglich** (possible) — no break hits the used surface, no conflict, trial run green. Without a trial run, „nur statisch" goes next to it.
- **Möglich mit Anpassungen** (possible with adaptations) — the update works, but code or configuration has to change; each adaptation with `Datei:Zeile` and what has to be done.
- **Blockiert** (blocked) — something stands in the way that this update alone does not resolve: a platform requirement, another root dependency that holds the package in place, a target that does not exist, a Go major with a new module path. Always with what would have to happen first — a blocker with a precondition is an order of steps, not a no.

Separately from that, the **reliability** (**Belastbarkeit**): trial run green, trial run red, or checked statically only — and whether the tests touch the affected surface at all. A green trial run on untested code only proves that it compiles.

```
# Dependency-Update: <dependency> <sourceVersion> → <targetVersion> — Möglich | Möglich mit Anpassungen | Blockiert

**Geprüft auf:** <branch> @ <sha> — ausgecheckt | ohne Wechsel, per `git show`, `git grep` und Worktree
**Belastbarkeit:** Probelauf grün (<Befehle>) | Probelauf rot: <n> Tests | nur statisch — <warum>; Tests decken die betroffene Oberfläche ab | nicht ab
**Fundstelle:** <manifest>:<Zeile> `<constraint>` — Lock hält <version> (= sourceVersion | abweichend: <was>)
**Sprung:** <patch | minor | major> — <n> Versionen dazwischen (<Majors/Minors>), <source> vom <Datum>, <target> vom <Datum>; neuer als das Ziel: <version> | Ziel ist aktuell
**Voraussetzungen:** Ziel verlangt <Laufzeit> — Projekt <Stand>, CI <Stand>, lokal <Stand> → erfüllt | nicht erfüllt: <was>
**Auflösung:** ohne weitere Änderungen | zieht mit: <paket a→b, …> | Konflikt mit <paket> (verlangt <constraint>)

## Bruchstellen
### <API oder Verhalten> — seit <version>
- **Änderung:** <ein Satz, mit Quelle: URL oder Datei, zitierte Zeile>
- **Betroffen:** <Datei:Zeile, …> — gefunden mit `<grep-muster>` | nicht verwendet — geprüft mit `<grep-muster>`
- **Anpassung:** <was zu tun ist> | keine

## Deprecations
- <was ab dem Ziel angemahnt wird und wo das Projekt es benutzt — die Arbeit für den nächsten Sprung>

## Verhaltensänderungen ohne API-Bruch
- <Default, Reihenfolge, Fehlerklasse … — Aufrufstelle, und ob ein Test es bemerken würde>

## Probelauf
- Install: <Befehl> → <Ergebnis>
- Build/Typen/Lint: <Befehl> → <Ergebnis>
- Tests: <Befehl> → <n bestanden, m rot>; rot: <Test> — <Fehler in einer Zeile> → <Bruchstelle X | auch auf Ausgangsstand rot | undokumentiert>
- Mitgezogen im Lock: <paket a→b, …>
- Hinweise: <Advisories gegen das Ziel, Deprecation-Ausgabe, Lock-Format …>

## Nicht geprüft
- <was nicht geprüft werden konnte, warum — und was das für das Urteil heißt>

## Nächste Schritte
- <der Befehl für das echte Update in der Arbeitskopie — den führst nicht du aus>
- <Dateien, die anzupassen sind, in der Reihenfolge, in der es Sinn ergibt>
- <bei Blockiert: was vorher passieren muss — `<paket>` auf <version>, Laufzeit auf <version>>
```

Empty sections get a line „keine", so that it stays visible that they were checked. The report is the output; you write nothing into the project unless the human asks for it.

## 8. Stance

The first temptation is to judge by the version number. "Patch, it'll be fine" and "major, won't work" are both guesses; the check is worth as much as you have read.

The second is to read only the target's release notes. The break is in the first version after the starting point, and that is exactly what someone skips who only looks at the end.

The third is to write "not used" after a grep for the import. Configuration, strings, framework wiring, subpath imports and the project's own wrappers are the places where updates fail in production. Name the pattern — if you cannot name it, you have not checked.

The fourth is to perform the update because it is "almost the same work". It is a change to the human's project. The check ends with a report and an unchanged working copy; the PR is their business.

The fifth is to repair the worktree to green. Every correction you make there is a finding you hide. Report the necessary adaptations instead of making them — apart from the one exception in section 6, and that one then appears in the report.

The sixth is to trust a green trial run. Green means: what the tests cover works. Say what they do not cover, especially if the affected surface lies in that gap.

The seventh is to blame the update for what was already red before. Compare with the source state before you write "breaks".

The eighth is to leave the worktree standing. It sits under `/tmp`, nobody sees it — until `git switch` says the branch is checked out elsewhere. Remove, prune, verify.

And the answer to "does this work?" may be "yes, but not on its own". If the jump requires other packages or the runtime to catch up first, that is not blocked but an order of steps — and you write it down as an order of steps.
