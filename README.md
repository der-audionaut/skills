# Skills

Custom skills for agentic development tools. A skill is a Markdown work instruction that the agent loads when the occasion fits — either automatically, because the request matches its description, or explicitly via `/name`.

Two families live here: `news-*` produces researched briefings, `plan-*` works on feature plans across their entire lifecycle.

## Overview

| Directory | Invocation | Purpose |
| --- | --- | --- |
| `skills/news-nachrichtenlage` | automatic or `/news-nachrichtenlage` | News briefing from live research that separates confirmed facts, interpretation and unconfirmed claims |
| `skills/news-wirtschafts-briefing` | automatic or `/news-wirtschafts-briefing` | Economics and financial-market briefing: market picture, lead story, economic data, agenda |
| `skills/plan-create` | `/plan-create "<feature>" [sprache]` | Writes a plan document from a feature description: step table and waves first, then design for architecture, data flow, API, schema and caching |
| `skills/plan-review` | `/plan-review <plan> [sprache]` | Checks a plan against the real codebase before anyone implements it |
| `skills/plan-lint` | `/plan-lint <plan> [sprache]` | Cleans up a plan: dead knowledge, inconsistencies, contradictions |
| `skills/plan-coding` | `/plan-coding <plan> <schritt> [sprache]` | Implements exactly one step of the plan, including findings rounds |

All six skills answer in German by default. The plan skills take a different output language as their last, optional argument (`/plan-review my-plan english`, `/plan-create "invoice export as CSV" english` — the free-text description goes in quotes when a language follows); the news skills take the modifier `sprache <language>` in the request. What the language covers and what it leaves alone is described under [Conventions](#conventions).

A skill is named after its directory. Because the skills are loaded as the plugin `skills`, the full name is `/skills:plan-review`; the short form `/plan-review` works as long as no other plugin ships a skill of the same name.

The agent may pull the news skills on its own whenever a request matches their description. The plan skills carry `disable-model-invocation: true` — they only run when you start them explicitly, because they touch files that are not their own.

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

**Choosing a language:** The last argument is optional and names the language of the report, the follow-up questions and everything the skill writes into the plan — as a name or code (`english`, `en`). Without it, German, even for a plan written in English. The skills recognise the history sections in any language and never create them twice; the finding IDs (B/S/O, T/I/W, F) stay untranslated so they remain stable across rounds.

For `/plan-create` the language is the one the plan itself is written in: headings, table labels and prose are translated, while section order, tables, step numbers, the wave labels `W1, W2 …` and the header key `Claude-Session` stay as they are, so the downstream skills can reference them. Because the feature description is free text, put it in quotes when a language follows — arguments are split like a shell command line, so `/plan-create Rechnungsexport als CSV english` would hand the skill `als` as its second argument. Without quotes the skill falls back to the full argument string and treats a trailing bare language name as the language; when that word could just as well belong to the description, it asks instead of guessing.

## The news skills

Both produce prose in the chat, German by default, no file — and both share the same core: the value lies not in the news item but in its classification.

`news-nachrichtenlage` assigns every statement to one of three levels (confirmed / interpretation / unconfirmed) and works through sources from the bottom up: primary sources, news agencies, German quality media across the spectrum, international press, independent media. The real added value sits in the section *Wo die Berichterstattung auseinandergeht* (where the coverage diverges) — it names whether a difference rests on facts, weighting, interpretation or omission.

`news-wirtschafts-briefing` starts with a single overview fetch that delivers prices and editorial weighting at once, deepens the lead story from there, and checks every figure at its primary source — paying particular attention to the reference period, because search results cheerfully mix up months.

Both skills have defaults (language, length, focus) and one empty list each that is waiting for you: **Dauerthemen** (standing topics) in `news-nachrichtenlage`, **Watchlist** in `news-wirtschafts-briefing`. If you want a setting changed permanently, edit it directly in the `SKILL.md` — that is exactly what those blocks are for. For a single run, a modifier in the request is enough: `kurz` (short), `nur <topic>` (only this topic) or `sprache english`. The language of the request alone does not switch the output — a request phrased in English yields a German briefing unless a language is named.

## Installation

The repo is a plugin marketplace and a plugin at the same time: `.claude-plugin/marketplace.json` describes the marketplace, `.claude-plugin/plugin.json` the plugin `skills`, and everything under `skills/` is a skill. Claude Code fetches the plugin straight from GitHub — no clone, no `git pull`, no symlink per skill.

Once per machine:

```bash
claude plugin marketplace add der-audionaut/skills
claude plugin install skills@der-audionaut
```

Then restart Claude Code; `claude plugin list` shows the plugin, `/help` shows the skills. If you previously mounted the skills via symlinks, remove the old links first, otherwise they load twice:

```bash
rm ~/.claude/skills/{news-nachrichtenlage,news-wirtschafts-briefing,plan-create,plan-review,plan-lint,plan-coding}
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
- `argument-hint` / `arguments` — named arguments, mapped positionally and usable in the text as `$plan`, `$schritt`, `$sprache`; an argument that is not passed becomes the empty string, which is why trailing arguments can be optional. The argument string is split like a shell command line, so a value with spaces stays one argument only inside quotes; `$ARGUMENTS` always holds the complete string as typed

The files under `references/` hold the long lists — review criteria, design dimensions, lint categories. They only enter the context when the skill actually reads them via the skill-directory variable (`${CLAUDE_SKILL_DIR}`). That keeps the `SKILL.md` readable and the context small.

## Conventions

- **German by default.** The instructions are written in German, and without a language argument so are report and output. The language can be switched per run — for the plan skills via the last argument, for the news skills via the modifier `sprache <language>`. That is why the language directive is the first line of every skill and is checked once more at the end: a single line in the middle of a German instruction demonstrably is not enough, the model answers in German anyway.
- **The language changes the text, not the structure.** Report formats and histories keep their layout, only the labels are translated; a freshly created plan likewise keeps section order, tables, step numbers and wave labels while headings and prose follow the language. Finding IDs stay untranslated, history sections are recognised in any language and never created twice, code follows the existing codebase rather than the language argument, and the sources and focus of the news skills are untouched.
- **Prose instead of bullet lists.** The skills explain *why* a rule holds; a rule without a reason is the first one to be bent during implementation.
- **Every iterative skill needs a termination criterion.** Without one, an agent invents findings in round three to look busy. Hence the same pattern everywhere: no new optional findings from round 2 on, rejected findings stay rejected, after three rounds without progress the human decides.
- **A section „Vor dem Absenden prüfen" (check before sending) or „Haltung" (stance) at the end.** It states what typically goes wrong — the most common source of error is never the missing rule but the well-known temptation.
