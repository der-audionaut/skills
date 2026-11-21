---
name: plan-frontend-create
description: Creates a frontend plan document in Markdown from a description of an application's user interface and stores it in the project — component architecture with reuse of existing building blocks, a component catalogue with props design, states, accessibility, responsive behavior and usage calls, and steps that each deliver one component complete with loading, empty and error states and its accessibility check; structurally compatible with plan-review, plan-lint and plan-coding. Takes the description as argument, optionally followed by the output language (default German).
argument-hint: "[feature-description] [language]"
arguments: feature sprache
disable-model-invocation: true
allowed-tools: Read Grep Glob Write Edit
---

**Output language: `$sprache`** — if nothing is given there, or a word that recognizably names no language, German. Everything you address to the human or write into the plan file, you write in this language: plan document, follow-up questions, abort messages, handoff. That these instructions are in English changes nothing about this.

Plan the frontend from this invocation: **$ARGUMENTS**

Think like a senior frontend engineer: reusable components, accessible, production-ready. A component is done when it handles loading, empty, error and edge cases, works on every width the project supports and can be operated without a mouse — not when its happy path renders. Design the component architecture so the second screen reuses what the first one built, then cut the smallest production-ready version into steps. The result is a document, not code: the components are built step by step via `/plan-coding`.

## 1. Resolve language and description

**Language.** The output language is the last, optional argument — a language name or code in any spelling (`englisch`, `english`, `en`). Without it, German stays, even if the description or the codebase is in English; whoever wants the plan in another language specifies it.

The language changes the text, not the structure: layout and order of the template stay, and so do the tables; headings, column titles, labels and prose get translated. The step numbers, the wave codes `W1, W2 …`, the component identifiers `K1, K2 …` and the key `Claude-Session` in the header block stay the same in every language, because `/plan-review`, `/plan-lint` and `/plan-coding` reference them later.

**Description.** The description is free text, but the invocation is split into words like a shell command line. If the description is in quotes, the mapping is unambiguous — `$feature` is the description, `$sprache` the language — and that is how the invocation is meant: `/plan-frontend-create "Kundenliste mit Filter" english`. Without quotes, `$sprache` holds only the second word of the description; then the full invocation above counts:

1. If its last word recognizably names a language, that is the output language and everything before it is the description.
2. Otherwise the whole invocation is the description, and the language stays German.
3. If the last word could also belong to the description ("Login-Seite auf englisch"), don't guess: add the question to the follow-up questions from section 3 and assume German until answered.

## 2. Understand before you plan

First the codebase, then the plan. A frontend plan written without looking into the project invents a framework, a design system and breakpoints the project does not have.

- **Stack.** Manifests and framework configuration: framework and version, language, styling approach, component library, state management, routing, data fetching, i18n, and the breakpoints from tokens, theme or CSS framework configuration. These go under `### Stack` with `file:line`.
- **Building blocks.** The component directory, the design system or tokens, Storybook or a comparable catalogue, layouts and hooks. Grep for every component the description implies before you plan it as new — buttons, fields, tables, dialogs, empty and error states usually exist. What you find goes under `### Vorhandene Bausteine` or, if the interface uses it, into the catalogue as a reused `K`.
- **Neighbors and data.** Does the interface already half exist, or is there a similar screen whose structure it should follow? That screen is the pattern. Its endpoints, fixtures and API limits tell you the realistic data volume.
- **Verification.** Build, test, type-check and lint commands, the test library and whether an automated accessibility check exists. These go under `### Build und Tests`; what is missing stays missing — the plan does not introduce a tool.
- **Decisions and standard.** Read `CLAUDE.md`, `CONTEXT.md`, ADRs, the project's documentation and existing plans once. Terminology and decisions already made are binding. The accessibility standard is WCAG 2.2 level AA unless one of them prescribes another; it goes under `### Maßstab Barrierefreiheit` with its source.
- **Storage location.** Determine the location and file name per section 4 now and check whether the file exists, so that an extend-or-rename question goes into the one message of section 3 instead of a second round.

**Greenfield.** If the project has no frontend yet, there is nothing to read the stack from. Then the stack is a follow-up question from section 3, with your proposal fitting the rest of the project — its language, its backend framework's usual frontend, what the team evidently knows — and the plan proceeds with that proposal as an assumption.

## 3. Ask follow-up questions

Ask the questions that would change the plan — bundled in one message, at most five, each with your proposal and a short justification, so that a "passt" ("fine") suffices as an answer.

Ask about: the stack (only in a greenfield project), target devices and widths if the project does not define them, an accessibility standard other than the default, the data source if no endpoint exists yet, an existing target file, scope and the budget of steps. Do **not** ask about what is in the code — you read that yourself.

If a question remains open, keep planning with your assumption, mark it in the plan as an assumption and enter the question under `## Offene Fragen`. A plan waiting for an answer is worthless; an unmarked assumption is dangerous.

## 4. Determine the storage location

1. Are there already plans in the project (`docs/plans/`, `.claude/plans/`, `plans/`)? Then put it there, in the naming scheme used there.
2. Otherwise create `docs/plans/`.
3. File name `frontend-<titel>.md` in kebab case, the title short and descriptive in the output language: `frontend-kundenliste.md`, with `english` `frontend-customer-list.md` — unless the project prescribes a different naming scheme.
4. If the file already exists → do not overwrite. Name it and ask, in the message of section 3, whether to extend it or use a new name.

## 5. Design the components

Now read `${CLAUDE_SKILL_DIR}/references/frontend-criteria.md` in full before you note the first `K`. It decides what goes into the component split, the props, the states, the responsive concept, the accessibility and the verification; the template decides where it goes.

Derive the component tree from the description and what section 2 found: page or route, feature components, building blocks. Mark each node `wiederverwendet`, `erweitert` or `neu`. Every node becomes a `K<n>` in the catalogue, in the shape the template shows for its status; the usage call is written in the project's syntax.

Then the cross-cutting parts: where the state lives and how loading and errors travel down (`### Zustandshaltung und Datenfluss`), what changes per breakpoint (`### Responsive-Konzept`), and — only with data binding — the endpoints or signatures the interface uses (`## Schnittstellen`).

Whatever the design provides for but this interface does not need yet belongs under non-goals, not in a step.

## 6. Write the plan

The full structure is in `${CLAUDE_SKILL_DIR}/references/frontend-template.md`. Binding are:

- **Header block** with `Frontend`, date, status and `Claude-Session: ${CLAUDE_SESSION_ID}` — so it can later be traced which session the plan came from.
- **Step overview at the very top**, as a table, before any prose. Whoever opens the plan sees the work steps first.
- **Parallel overview** directly below, in waves.
- Then goal, non-goals, current state, component architecture, component catalogue, interfaces if any, the steps in detail, test strategy, rollout, risks, assumptions, open questions.

You write the file once, after the steps, waves and cross-check from sections 7 to 9 are settled. The template is German, because German is the default output language. For a different output language you translate its headings, column titles and header block labels while carrying them over — not only during the cross-check.

## 7. Cut the steps

- A step has **one** concern and a verifiable result. If the description ends at "implement", it is not thought through.
- A component step delivers its component complete: all states from its catalogue entry, its accessibility and its responsive behavior. There is no later step "accessibility", "loading states" or "mobile" — that is the happy path in disguise, and the later step is the one that gets cut.
- Order: building block before feature component before page or route; data binding before the component that consumes it; a shared token or type before its first user.
- Every new or extended `K` is built by exactly one step, which names it in its title or under `Betroffen`. A reused `K` needs no step.
- Each step names result, affected files, dependencies, the `Zustände` bullet as the template defines it, and an acceptance criterion.
- After each step the application builds and the tests run. No broken intermediate state, no route to a page that does not exist yet.
- A step is too big if it would itself need a plan. Too small if its result means nothing on its own — a single props type is not a step.

## 8. Determine parallelism

Two steps may only go into the same wave if **both** hold: no functional dependency, and no files written by both. Changing the same file at the same time is no parallelization gain but a merge conflict with a detour.

Typical collision points in a frontend that never run in parallel: design tokens and theme, route registration, barrel or index files that export components, i18n message files, shared types and API clients, the global store.

Form waves from this: W1 is all steps without a precondition — typically independent building blocks and the data binding —, W2 everything that depends only on W1, and so on. For each wave, name the steps and the files where they could touch. If no real parallelism results, write exactly that — an invented wave costs more later than an honest chain.

## 9. Cross-check

Check the composed text once completely, as if you had not written it yourself:

- Does the sum of the steps cover the goal, and nothing beyond it?
- Do the dependencies in the overview match those in the steps? And the waves?
- Does every new or extended `K` have exactly one step, and does every step that builds a component find it in the catalogue?
- Does every step carry `Zustände`, and do the states it names exist in the catalogue entry of its `K`?
- Is there component code anywhere instead of a usage call?
- Is every invented detail gone — files, components, tokens, breakpoints, endpoints you have not verified?
- Does every assumption stand as an assumption?
- Is the plan in the output language throughout — including the headings and table headers that come from the German template?

Fix what you find and check again. You are done when a pass turns up nothing new. For the thorough review, point to `/plan-lint` and `/plan-review` afterwards — you don't rebuild those here.

## 10. Hand off

Name the path of the file, the number of `K` by status (reused, extended, new), the accessibility standard, the number of steps, the waves in one line and the open questions. No rehashing of the content — the plan is in the file.

## 11. Stance

The temptations of this skill: planning a new component where the project has one; a props interface that grows a boolean per use case instead of composition; accessibility and loading states as a later step; planning only the happy path; inventing breakpoints, tokens or a design system; proposing a new framework or library because it would be nicer; writing the component instead of its usage call; citing a WCAG criterion without checking what it demands. A plan with four components, all of whose states are thought through, is better than one with twelve that only render data.

If the description is too vague to derive a component tree — no screen, no data, no task the user performs —, say so instead of inventing structure.

And before you send: are the answer and the plan file in the output language? The English instructions and the German template otherwise pull toward their own language, even when something else was requested above.
