# Breaking points

Where breaks are written down, how to recognize them, how much they weigh — and what is not one. For everything: first the source, then the match against usage in the project, then the classification. A break without a quote is a guess; a "not affected" without a grep pattern is not a check.

---

## A — Where they are written down

- **Migration and upgrade guides first.** `UPGRADE.md`, `UPGRADE-<major>.md`, `MIGRATION.md`, an "Upgrading" or "Migration guide" chapter in the documentation. They are written for exactly this question and name the order of the adaptations. If they are missing for a major jump, that itself is a hint about the package's maturity.
- **The repository's changelog.** `CHANGELOG.md`, `CHANGES`, `HISTORY.rst`, `NEWS`. In monorepos it lives in the package's subfolder (`packages/<paket>/CHANGELOG.md`), and the tag is called `<paket>@<version>`.
- **Release pages.** `/releases/tag/<tag>` and the releases API. Some projects maintain only these, some only the changelog, some generate the notes from commit subjects — then `feat!:` and `BREAKING CHANGE:` are the markers.
- **The diff between the tags.** `/compare/<tag-ausgang>...<tag-ziel>`, limited to the public surface: type definitions (`*.d.ts`, `py.typed` stubs, headers), export lists, README, examples. After the trial run the target is installed in the worktree and the source in the working copy: `diff -r` of the two package directories is then the diff without network.
- **The pre-releases in between.** `5.0.0-rc.1` often carries the details that `5.0.0` condenses into "see RC notes".
- **The source version's deprecations.** What the source version already marks as deprecated — in types, docstrings, runtime warnings — is the list of what the next major removes. That often appears nowhere as "removed", only as "deprecated" two versions earlier.

## B — How to recognize them

**Explicitly marked.** `BREAKING`, `⚠️`, `!:` in subject lines, "removed", "dropped", "no longer", "renamed", "now requires", "minimum … version", "is now the default", "throws … instead of", sections "Breaking changes", "Migration", "Upgrading". That is the easy part.

**Not marked, but a break.** The hiding places where updates fail in production:

- **Changed defaults.** "Strict mode is now enabled by default", "connection pooling on by default", "trailing slashes are no longer normalized". The code compiles, the API is the same, the behavior is not. Read the call sites: is the option set explicitly? If not, the new default now applies.
- **Stricter validation.** "Now rejects invalid …", "now throws on unknown keys". Tolerant inputs that used to pass become errors — usually only in production, where the inputs are messier than in the tests.
- **Changed return and error types.** `null` becomes an exception, array becomes iterator, synchronous becomes asynchronous (Promise, Future), one error class becomes another. Tests that check error messages or class names are the first victims, and `catch` blocks on the old type silently catch nothing anymore.
- **Order, precision, format.** Sorting no longer stable, floating-point rounding different, timestamps with instead of without time zone, JSON key order, date format, log format. Breaks snapshots, comparisons, downstream parsers.
- **Module format and distribution.** ESM-only without CommonJS, removed `exports` subpaths (`<paket>/lib/…` no longer exists), renamed entry points, no type bundle anymore (types moved out to `@types/…` or the other way round), removed browser builds.
- **Dropped platforms — with side effects.** "Dropped Node 16 / PHP 8.0 / Python 3.8" often means: the package now uses syntax the old runtime cannot parse, and fails on load, not on call. For TypeScript packages a newer `typescript` version may be needed to read the types.
- **Only types changed.** Stricter generics, `readonly`, removed overloads. Runtime unchanged, `tsc`, `mypy`, `phpstan` red. A break for the build, not for the application — still an adaptation.
- **Peer and co-requirements raised.** The target requires `react >= 19`, `symfony/http-kernel ^7`, `tokio 1.40`. The break is not in the package but in what it pulls along.
- **Configuration schema.** Renamed options, changed plugin API, new configuration file format, renamed property names (Spring Boot), renamed environment variables, new mandatory fields. Grep for the old names in YAML, JSON, XML, `.env*`, property files.
- **Command line.** Flags renamed or removed, changed exit code, changed output that scripts parse. Grep in `package.json` scripts, `Makefile`, CI, shell scripts.
- **Bundled migrations and file layout.** ORMs and frameworks ship database migrations; asset, template or fixture paths change. An update that requires a migration is possible, but not without a deployment step — that belongs in the next steps.
- **Feature flags and compile options.** Cargo features renamed, build tags changed, optional extras (`paket[extra]`) restructured.
- **Security fixes that tighten behavior.** A CVE fix that makes parsing stricter is a break for tolerant inputs. It is still right — the report names it as an adaptation, not as a reason to stay.
- **License change.** Not technical, but a blocker for many projects: a switch to AGPL, BUSL, SSPL or a commercial license. It is in the changelog, in `LICENSE`, in the registry metadata.
- **Order of removal.** What the target removes was deprecated before. If the source did not yet have the deprecation — because it only came in an intermediate version — the project never saw it. That is why you read the whole range.

## C — How much they weigh

- **Blocker.** Hits the used surface and cannot be resolved within this update: runtime or platform not satisfied, another root dependency holds the package in place, the target does not exist or is withdrawn, a different module path (Go major), a license change the project cannot accept. Always with the precondition that would resolve the blocker.
- **Adaptation.** Hits the used surface, and the way is known: follow a rename, set an option explicitly, adjust an error class, convert configuration, plan a migration. Each with `Datei:Zeile` and what has to be done there. Ten locations of the same rename are one adaptation with ten locations, not ten adaptations.
- **Note.** Does not hit the surface (proven with a grep pattern), or is a deprecation that still works, or a behavior change that, after reading the call sites, does not reach the project. Goes into the report so that it is not researched anew on the next jump.

For transitive dependencies the question shifts: not "does the project tolerate the target", but "does every package that requires it tolerate the target" — and that is in their constraints and changelogs, not in the project's code.

For frameworks the relationship reverses: do not hold the list of breaks against the code, but the code against the source state's deprecation output and the migration tool's diff. The list of breaks of a framework major is too long to grep point by point, and the tool knows it better.

## D — What is not a break

Do not inflate. A report with twenty points, seventeen of which do not affect the project, does not get read.

- Internal restructuring, performance improvements, new features, documentation, changes to the package's own build.
- Removed or changed APIs the project does not use — proven with the pattern. As a one-line note, not as a breaking point with three sub-bullets.
- Deprecations, as long as the project does not turn warnings into errors. They belong in the „Deprecations" section, not in „Bruchstellen".
- Raised peer or co-requirements the project already satisfies.
- A red result that is just as red on the source state.
- Changes between source and lock version when the lock is already further along than `$sourceVersion`: the project already has them.
