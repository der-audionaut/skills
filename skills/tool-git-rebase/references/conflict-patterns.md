# Conflict patterns

Recurring conflicts during a rebase and how to handle them. The same order applies to each: first read both intents with `git show REBASE_HEAD` and `git log -p <merge-base>..<basis> -- <datei>`, then act. And do not forget the assignment: in a rebase, `ours` is the base, `theirs` the branch's commit.

---

## A — Mechanical conflicts

Both sides are right, only the text does not fit together. This is the majority of all conflicts, and this is where the most gets lost through carelessness.

- **Adjacent lines.** Both sides insert something at the same spot — imports, list entries, enum values, routes, dependencies. Keep both, in the order the surroundings dictate (alphabetical, by group). Then check for duplicates: often both sides added the same import.
- **Rename on the base, usage in the commit.** The base renamed a function, a field, a class; the commit uses the old name. Take the new name — and then search the commit's entire diff for further uses of the old name. At spots the base did not touch, there is no textual conflict, but the same error. This is the classic semantic conflict that only the compiler or the test finds.
- **Moved code.** The base moved a block into another file or to another spot; the commit changes the block at the old location. Apply the commit's change at the new location; the old location stays as the base left it.
- **Formatting run on the base.** The base reformatted the whole file (Prettier, Black, gofmt, PHP-CS-Fixer); the commit changed a few lines. Take the base's version, reapply the commit's substantive change, run the formatter over this file. Never align line by line by hand — the formatter decides, and it is reproducible.
- **Whitespace and line endings.** `git diff --ignore-all-space` shows whether there is any substantive difference at all. Resolve to the base's convention.

---

## B — Generated and derived files

These are not merged by hand. The source is resolved, the derived file regenerated.

- **Lock files** — `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `composer.lock`, `Cargo.lock`, `poetry.lock`, `Gemfile.lock`, `go.sum`. First take the base's version (`git checkout --ours -- <lock>`; ours is the base here), then resolve the manifest (`package.json`, `composer.json`, …) by hand, then regenerate the lock file with the project's tool — `npm install --package-lock-only`, `composer update --lock`, `cargo generate-lockfile`, depending on the ecosystem — and stage it. Check: `git diff --cached -- <lock>` may only touch the dependencies the commit changed in the manifest. If generation fails (no network, no tool), say so and leave the rebase stopped.
- **Generated code** — API clients, Protobuf, GraphQL types, ORM snapshots, compiled assets in the repo. Resolve the source, run the generator, stage the result. No generator, no resolution — report, do not guess.
- **Test snapshots.** Resolve the code first, then have only the affected tests' snapshots updated. Then read the snapshot diff: it may only show the change the commit intended. A snapshot that locks in something else on the side is a new bug with a green checkmark.

---

## C — Structural conflicts

One side restructured the file, not just changed it.

- **Deleted versus modified.** The base deleted (`deleted by us`), the commit modified: where did the content go? If it was moved, the commit's change goes to the new location and the old path is confirmed with `git rm`. If it is gone without replacement, the commit's change is probably obsolete — but first check what it did: a bugfix in a deleted file is often still needed elsewhere. The other way round (`deleted by them`, the commit deleted, the base modified): the deletion usually stays, but the base's change must land where the commit relocated the content.
- **Renamed.** Git usually detects renames, but not always; then a pair of deletion and new file with a conflict appears. `git log --follow` and `git diff -M` help with the mapping. The changes go to the renamed path.
- **Both created** (`both added`). Two different files at the same path — almost always two different intents. If the base's file does what the commit wanted, the commit's file is dropped. If it is something else, it needs a different name, and that is a decision: ask.
- **Database migrations.** Both sides created a migration; numbers or timestamps collide, or the schema states diverge. The base's migration stays untouched — it may already have run somewhere. The commit's migration gets a number or timestamp after it and must build on the base's schema state. Frameworks with a pointer to the latest migration (Django `dependencies`, Rails `schema.rb`, Prisma migration lock) need the pointer updated too.
- **Registration and collection files** — routes, container configuration, feature flags, CHANGELOG. Both sides append; keep both, in the file's order. In the CHANGELOG, the commit's entry goes under "Unreleased", even if the base has cut a version in the meantime.

---

## D — What is not conflict resolution

Here the human decides. Quote both sides, say what each wants, name your own proposal with reasoning, ask, leave the rebase stopped.

- Both sides change the same behavior differently, and both are valid on their own — e.g. two different validation rules for the same field.
- The base removed a function the commit extends.
- The commit's tests assert a behavior the base deliberately changed.
- Binary files — images, PDFs, compiled artifacts — are changed on both sides. There is no middle ground; one side must be chosen, and that is not your call.
