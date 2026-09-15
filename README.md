# Skills

Custom skills for agentic development tools. A skill is a Markdown work instruction that the agent loads when the occasion fits — either automatically, because the request matches its description, or explicitly via `/name`.

Three families live here: `news-*` produces researched briefings, `plan-*` works on feature plans across their entire lifecycle, `tool-*` takes over everyday Git chores — so far the rebase.

## Overview

| Directory | Invocation | Purpose |
| --- | --- | --- |
| `skills/news-nachrichtenlage` | automatic or `/news-nachrichtenlage` | News briefing from live research that separates confirmed facts, interpretation and unconfirmed claims |
| `skills/news-wirtschafts-briefing` | automatic or `/news-wirtschafts-briefing` | Economics and financial-market briefing: market picture, lead story, economic data, agenda |
| `skills/plan-create` | `/plan-create "<feature>" [sprache]` | Writes a plan document from a feature description: step table and waves first, then design for architecture, data flow, API, schema and caching |
| `skills/plan-review` | `/plan-review <plan>` | Checks a plan against the real codebase before anyone implements it |
| `skills/plan-lint` | `/plan-lint <plan>` | Cleans up a plan: dead knowledge, inconsistencies, contradictions |
| `skills/plan-coding` | `/plan-coding <plan> <schritt>` | Implements exactly one step of the plan, including findings rounds |
| `skills/tool-git-rebase` | `/tool-git-rebase <basebranch> [branch]` | Rebases the current branch, or the branch you name, onto a base branch: fetches first, resolves conflicts commit by commit with both sides' intent preserved, verifies with the project's build and tests, never pushes |

All seven skills answer in German unless something says otherwise. `/plan-create` takes a different output language as its last optional argument (`/plan-create "invoice export as CSV" english` — the free-text description goes in quotes when a language follows), the news skills take the modifier `sprache <language>` in the request, and `/plan-review`, `/plan-lint` and `/plan-coding` take no language at all: they answer in the language the plan is written in. `/tool-git-rebase` always reports in German. What the language covers and what it leaves alone is described under [Conventions](#conventions).

A skill is named after its directory. Because the skills are loaded as the plugin `skills`, the full name is `/skills:plan-review`; the short form `/plan-review` works as long as no other plugin ships a skill of the same name.

The agent may pull the news skills on its own whenever a request matches their description. The plan and tool skills carry `disable-model-invocation: true` — they only run when you start them explicitly, because they touch files, or in the case of `tool-git-rebase` the Git history, that are not their own.

## The plan skills as a chain

The four plan skills are stations of one workflow. `/plan-create` writes the plan; the other three work on it and share a mechanism: each works in rounds, each writes its result into its own history section at the end of the plan file, and each stops as soon as a complete round finds nothing new.

```
/plan-create "<feature>"  → plan document: step table and waves first, then design and steps
     │
     ├─ /plan-review   → does the plan hold up against the real code?  → ## Review-Historie
     ├─ /plan-lint     → is the plan free of internal contradictions?  → ## Lint-Historie
     └─ /plan-coding   → implement step by step                        → ## Umsetzungs-Historie
```

The order is not a rule. `/plan-lint` pays off especially after several steps have been implemented and the plan contains statements about a state that no longer exists. `/plan-review` pays off before the first line of code — and once more when implementation has noticeably changed the plan.

`/plan-create` reads the codebase before it plans, bundles at most five questions that would change the plan, and then writes the document to the project's plan directory (`docs/plans/` unless the project already keeps plans elsewhere). The step table and the wave overview come first, so whoever opens the plan sees the work before the reasoning. It creates no history sections — those appear when the downstream skills run for the first time — and it never overwrites an existing file.

The history sections are the handover point between runs. Without them, every pass starts blind from scratch and reopens findings that were already rejected.

**Finding the plan:** The three downstream skills accept a path, a file name or a name fragment. The search prefers `docs/plans/`, `.claude/plans/`, `plans/`, `docs/` and the project root. With several hits, the skill aborts and lists them instead of guessing.

**Language:** The three downstream skills take no language argument. They write — report, follow-up questions and everything they put into the plan — in the language the plan is written in, judged by its header block and headings rather than by quoted code or identifiers; a plan `/plan-create` wrote in English gets an English review and an English history section. Until the plan is found and read, messages are German. They recognise the history sections in any language and never create them twice; the finding IDs (B/S/O, T/I/W, F) stay untranslated so they remain stable across rounds. A trailing language word from the old calling convention is ignored.

`/plan-create` is the one skill in the chain that takes a language, as its last optional argument — a name or code (`english`, `en`); without it, German. The language is the one the plan itself is written in: headings, table labels and prose are translated, while section order, tables, step numbers, the wave labels `W1, W2 …` and the header key `Claude-Session` stay as they are, so the downstream skills can reference them. Because the feature description is free text, put it in quotes when a language follows — arguments are split like a shell command line, so `/plan-create Rechnungsexport als CSV english` would hand the skill `als` as its second argument. Without quotes the skill falls back to the full argument string and treats a trailing bare language name as the language; when that word could just as well belong to the description, it asks instead of guessing.

## The news skills

Both produce prose in the chat, German by default, no file — and both share the same core: the value lies not in the news item but in its classification.

`news-nachrichtenlage` assigns every statement to one of three levels (confirmed / interpretation / unconfirmed) and works through sources from the bottom up: primary sources, news agencies, German quality media across the spectrum, international press, independent media. The real added value sits in the section *Wo die Berichterstattung auseinandergeht* (where the coverage diverges) — it names whether a difference rests on facts, weighting, interpretation or omission.

`news-wirtschafts-briefing` starts with a single overview fetch that delivers prices and editorial weighting at once, deepens the lead story from there, and checks every figure at its primary source — paying particular attention to the reference period, because search results cheerfully mix up months.

Both skills have defaults (language, length, focus) and one empty list each that is waiting for you: **Dauerthemen** (standing topics) in `news-nachrichtenlage`, **Watchlist** in `news-wirtschafts-briefing`. If you want a setting changed permanently, edit it directly in the `SKILL.md` — that is exactly what those blocks are for. For a single run, a modifier in the request is enough: `kurz` (short), `nur <topic>` (only this topic) or `sprache english`. The language of the request alone does not switch the output — a request phrased in English yields a German briefing unless a language is named.

## The tool skill

`tool-git-rebase` rebases the branch you are on onto the base branch you name: `/tool-git-rebase main`. It fetches first so the rebase lands on the current state of the base rather than a stale local copy, refuses to start on a dirty working tree, over a rebase already in progress or on a detached HEAD, and records the starting commit before it changes anything, so the way back is always known. Name a second branch — the optional `[branch]` argument, `/tool-git-rebase main feature/x` — and it rebases that branch instead. All checks run on `feature/x` without leaving your branch; the switch happens only right before `git rebase`, and after verification it switches back to where you were, so you end up on your own branch with `feature/x` rebased. Only a rebase that stops for your decision stays on `feature/x`, because a standing rebase cannot be left. A branch that exists only on a remote is not created; the skill names the `git switch` that would.

Conflicts are resolved commit by commit, and the rule is that both intentions survive: the base's change stays, and the branch's commit does on the new state what it did before — with new names, new signatures, moved code. Where the two sides want different things at the same spot, the skill stops and asks instead of picking a side; taking one side wholesale with `--ours` or `--theirs` counts as a hidden revert, not a resolution. The recurring cases — lock files, generated code, formatting runs, delete-versus-modify, colliding migrations — are listed with their treatment in `references/konfliktmuster.md`.

After the rebase it runs the project's build and tests, because renames and moved code produce conflicts Git never flags, compares the branch's diff against the base before and after so a lost change shows up, and checks that the commits are still the same ones in the same order. It never pushes and never squashes, reorders or rewords commits. Rewriting published history is your decision; the report ends with the `git push --force-with-lease` line for when you make it — with the branch name when a branch was given, because a bare push would take the branch you are on — and with the way back for when you don't: `git reset --hard ORIG_HEAD` while the rebased branch is checked out, or `git branch -f <branch> <sha>` once the skill has switched back to your branch, where `ORIG_HEAD` would hit the wrong one.

## Installation

The repo is a plugin marketplace and a plugin at the same time: `.claude-plugin/marketplace.json` describes the marketplace, `.claude-plugin/plugin.json` the plugin `skills`, and everything under `skills/` is a skill. Claude Code fetches the plugin straight from GitHub — no clone, no `git pull`, no symlink per skill.

Once per machine:

```bash
claude plugin marketplace add der-audionaut/skills
claude plugin install skills@der-audionaut
```

Then restart Claude Code; `claude plugin list` shows the plugin, `/help` shows the skills. If you previously mounted the skills via symlinks, remove the old links first, otherwise they load twice:

```bash
rm ~/.claude/skills/{news-nachrichtenlage,news-wirtschafts-briefing,plan-create,plan-review,plan-lint,plan-coding,tool-git-rebase}
```

**Updating.** The plugin deliberately carries no `version`: Claude Code then uses the commit SHA as the version, and every push is an update. `claude plugin update skills@der-audionaut` fetches it manually; it happens automatically if auto-update for `der-audionaut` is switched on under *Marketplaces* in the `/plugin` dialog (it is off by default for third-party marketplaces). A new skill is a new directory under `skills/` — no manifest entry, no symlink; it reaches every machine with the next update.

**Development machine.** Where the skills are edited, the GitHub install gets in the way because it loads a copy of the last commit. There, link the whole repo once instead:

```bash
ln -sfn "$PWD" ~/.claude/skills/skills
```

Claude Code loads the directory as the plugin `skills@skills-dir`; changes take effect immediately without an update, and so do new directories under `skills/`. Not both at once: if the plugin is installed from the marketplace, it wins, and the symlink is skipped with a notice.

**Checking.** `claude plugin validate .` checks manifests and skill frontmatter; `claude plugin details skills@der-audionaut` (or `skills@skills-dir`) lists the skills it recognised.

## Anatomy of a skill

```
skills/<directory>/
├── SKILL.md              # frontmatter + instruction — always loaded
└── references/           # optional, loaded only when needed
    └── <topic>.md
```

The frontmatter controls:

- `name` — documentation only; the skill is invoked by its directory name
- `description` — decides whether the agent pulls the skill on its own; that is why it deliberately contains many phrasings of the request
- `allowed-tools` — the tools the skill is restricted to
- `disable-model-invocation` — `true` prevents automatic invocation
- `argument-hint` / `arguments` — named arguments, mapped positionally and usable in the text as `$plan`, `$schritt`, `$basebranch`, `$branch`; an argument that is not passed becomes the empty string, which is why trailing arguments can be optional. The argument string is split like a shell command line, so a value with spaces stays one argument only inside quotes; `$ARGUMENTS` always holds the complete string as typed

The files under `references/` hold the long lists — review criteria, design dimensions, lint categories. They only enter the context when the skill actually reads them via the skill-directory variable (`${CLAUDE_SKILL_DIR}`). That keeps the `SKILL.md` readable and the context small.

## Conventions

- **German by default.** The instructions are written in German, and so are report and output unless a skill is told otherwise: `/plan-create` by its last argument, the news skills by the modifier `sprache <language>`, and the three downstream plan skills by the plan itself, whose language they follow. `tool-git-rebase` carries a fixed directive „Deutsch" in its first line. Wherever the output can differ from German, the language directive is the first line of the skill and is checked once more at the end: a single line in the middle of a German instruction demonstrably is not enough, the model answers in German anyway.
- **The language changes the text, not the structure.** A freshly created plan keeps section order, tables, step numbers and wave labels while headings and prose follow the language; the news briefings keep their sources and focus. The downstream plan skills recognise history sections in any language, never create them twice and write new ones in the plan's language; finding IDs stay untranslated, code and identifiers follow the existing codebase, and commit messages stay as their authors wrote them.
- **Prose instead of bullet lists.** The skills explain *why* a rule holds; a rule without a reason is the first one to be bent during implementation.
- **Every iterative skill needs a termination criterion.** Without one, an agent invents findings in round three to look busy. Hence the same pattern everywhere: no new optional findings from round 2 on, rejected findings stay rejected, after three rounds without progress the human decides — and a rebase that cannot get one commit through in three attempts stays standing for the human instead of running on with invented resolutions.
- **A section „Vor dem Absenden prüfen" (check before sending) or „Haltung" (stance) at the end.** It states what typically goes wrong — the most common source of error is never the missing rule but the well-known temptation.
