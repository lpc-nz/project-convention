---
name: project-conventions
description: "Sets up architecture-consistency guardrails for any codebase — generates a project-specific CLAUDE.md, an architect skill (decides where new files go), a reviewer skill and subagent (checks written code against existing patterns), and a deterministic pattern-lint script. Use whenever the user starts a new project, clones or inherits an unfamiliar repo, or says any of these — set up conventions, make Claude follow my architecture, stop generating random components, create a review agent for this repo, how do I keep the AI consistent, add guardrails, onboard this codebase. Also use when the user complains that generated code doesn't match their existing patterns, components, or database schema — that complaint is the trigger."
---

# Project Conventions Bootstrap

One job: turn a codebase's implicit conventions into explicit guardrails that survive across sessions.

The output is five artifacts, generated **per repo**:

| Artifact | Path | Purpose |
|---|---|---|
| `CLAUDE.md` | repo root | ~30 lines of non-negotiables, always in context |
| `<project>-architect` skill | `.claude/skills/` | fires **before** writing — where does this file go? |
| `<project>-reviewer` skill | `.claude/skills/` | fires **after** writing — does this match what exists? |
| `architecture-reviewer` subagent | `.claude/agents/` | runs the reviewer in fresh, unbiased context |
| `scripts/lint-patterns.sh` | repo | deterministic greps for the mechanical rules |

Templates for the middle three live in `templates/`. Read a template only when you're about to write that file.

## The design principles behind all of this

State these to the user if they ask why the setup looks the way it does. They also govern your choices while generating.

1. **Split rules by when they fire.** Everything in one `CLAUDE.md` means everything is ignored once it passes ~100 lines. Placement rules fire before writing; consistency rules fire after; mechanical rules fire on save.
2. **Anything a regex can catch belongs in a script, not a prompt.** "No hex colours outside the theme file" is `grep`. Reserve the LLM for judgement — "is this a duplicate of an existing helper?"
3. **Point at sources of truth; never restate them.** The skills should say "check `design-tokens.md`", not copy the palette. Copied rules go stale and then actively mislead.
4. **The Precedent Rule.** A reviewer may not report a finding without citing an existing file that shows the right way. This is what stops it inventing generic best-practice opinions instead of enforcing *this* repo.
5. **The reviewer must run in fresh context.** An agent that just wrote the code will approve the code. A subagent starts with an isolated context window and only sees the diff.
6. **Silence must be a valid output.** A reviewer producing 40 findings gets muted, and a muted reviewer protects nothing.

## Workflow

### Step 1 — Determine the mode

**Existing codebase** (files present) → go to step 2A. **Greenfield** (empty or near-empty) → step 2B.

Ask the user which, only if genuinely ambiguous. Otherwise just look.

### Step 2A — Extract conventions from the existing repo

Do not interview the user for things you can read. Investigate first, then confirm.

Run this reconnaissance, adapting the commands to the stack:

- **Shape**: `git ls-files | head -200`, plus a directory tree 3 levels deep excluding `node_modules`, `dist`, `bin`, `obj`, `.git`.
- **Existing docs**: any `README.md`, `docs/`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, ADRs. These are gold — the conventions may already be written and just not wired in.
- **The layering**: what imports what? Look for a shared/core/domain package, a data-access layer, and whether UI files reach past it. Note the *intended* direction of dependencies.
- **UI system**: locate the component folder. List every component. Find the theme/tokens/variables file (`tailwind.config`, `theme.ts`, `_variables.scss`, a design-tokens doc). Open 2 components and note their shared shape — prop naming, wrapper treatment, how they handle loading and empty states.
- **Data layer**: migrations folder and its numbering scheme, the ORM/entity/model definitions, validation schemas. Critically: **how many places does one entity appear in?** (e.g. SQL + TS interface + local mirror + validation schema). That count is the most important fact you will extract.
- **Naming**: casing per file type — SQL, components, variables, folders, migrations, test files.
- **Precedent files**: for each kind of thing (page, component, service, repository, migration, test), pick the *best* existing example. These become the "canonical example" pointers.

Then **confirm the load-bearing invariants with the user** before writing anything. Present what you found as a short list and ask which are real rules versus accidents:

> I found these conventions. Which are intentional rules I should enforce?
> 1. All money/tax calculations live in `packages/core` — apps never compute inline
> 2. Pages never call the database directly, always through the data provider
> 3. Every entity appears in 3 places: SQL migration, TS interface, local mirror
> ...

This step matters. A repo full of accidents will otherwise get its accidents enshrined as law.

### Step 2B — Greenfield: decide the conventions with the user

Nothing to extract, so establish. Keep the interview short — 5 questions, not 20:

1. Stack and rough shape (monorepo? separate apps? what talks to what?)
2. Where does shared business logic live, and what's forbidden from having it?
3. UI: which component library or design system, and where do tokens live?
4. Database: what's the migration tool, and how many places does an entity get defined?
5. Naming conventions per file type.

Then write `docs/architecture.md` **first** — the skills must point at a real document. Skills that point at nothing produce reviewers that enforce nothing.

### Step 3 — Generate the artifacts

Read each template in `templates/` immediately before writing its file, and fill it in. Order matters:

1. `docs/architecture.md` — only if it doesn't exist. Short: stack, folder map, layering rules, data model.
2. `docs/history.md` — only if it doesn't exist. A single empty dated-log doc (date, one-line decision, commit hash) that Step 6's maintenance rule appends to over time. Keeping it separate from `architecture.md` means the architecture doc always reads as current state, never a log.
3. `CLAUDE.md` — from `templates/claude-md.template.md`. Hard ceiling of ~30 lines. If a rule isn't worth spending context on every single session, it belongs in a skill instead.
4. `.claude/skills/<project>-architect/SKILL.md` — from `templates/architect.template.md`.
5. `.claude/skills/<project>-reviewer/SKILL.md` — from `templates/reviewer.template.md`.
6. `.claude/agents/architecture-reviewer.md` — from `templates/agent.template.md`. Nearly stack-agnostic; mostly just name the reviewer skill.
7. `scripts/lint-patterns.sh` — from `templates/lint-patterns.template.sh`. Only include checks you verified are real rules in this repo.

### Step 4 — Wire up the deterministic layer

Offer, don't impose:

- Hook the lint script to run after edits, in `.claude/settings.json`:
  ```json
  { "hooks": { "PostToolUse": [ { "matcher": "Write|Edit",
      "hooks": [ { "type": "command", "command": "./scripts/lint-patterns.sh" } ] } ] } }
  ```
- Same script in CI on pull requests.
- Where the rule can be expressed as a lint rule instead of a grep (import boundaries via `eslint-plugin-boundaries`, `dotnet` analyzers, `import-linter` for Python), say so — a real lint rule beats a shell script.

### Step 5 — Test it before declaring victory

Two smoke tests, run them and show the user:

1. **Architect test** — "where would a new `<thing typical for this repo>` go?" The answer should cite the folder map, not guess.
2. **Reviewer test** — deliberately violate one rule in a scratch file (a hardcoded colour, a page calling the DB directly), run the reviewer, confirm it catches it *and cites a precedent file*. Then delete the scratch file.

If the reviewer misses it, the rule was too vague — sharpen it and retest. A guardrail nobody tested is decoration.

### Step 6 — Hand over

Tell the user, briefly:
- Restart the Claude Code session so `.claude/agents/` loads.
- Invoke the reviewer before commits: `@architecture-reviewer review my changes`.
- **The maintenance rule**: when a review surfaces a "decision needed" and they make a call, update `docs/architecture.md` with the new rule *and* append a dated entry to `docs/history.md` (date, one-line decision, commit hash if the call was made alongside a commit) — otherwise the same question comes back every week, and there's no trail of what the rule used to be or why it changed. `architecture.md` stays current-state only; `history.md` is the append-only log, never edited in place. The skills are pointers; the docs are the truth, and only the docs are worth keeping current.

## Naming

Prefix the skills with the project name (`craftstock-architect`, `pacs-reviewer`) rather than generic names. A user with several projects will otherwise have four skills called `architect` and no way to tell which triggered. Keep the subagent name generic (`architecture-reviewer`) since it's project-scoped in `.claude/agents/` anyway.

## Adapting per stack

The categories are constant; the checks are not. Match the reviewer's check sections to what this stack can actually get wrong:

- **.NET / C#** — layer boundaries between API/Application/Domain/Infrastructure, DI registration missed, EF migration vs. model drift, async-all-the-way violations, DTO vs. entity leaking to the controller.
- **React/Next** — reinvented components, off-token styles, client/server component boundary, data fetching in the wrong layer.
- **Mobile (Expo/RN)** — screen registration in the navigator, platform-specific branches, offline/sync path bypassed.
- **Any SQL** — migration numbering and immutability once applied, snake_case, the entity-mirror count from step 2A.
- **Python** — package boundaries, settings/config access, ORM session handling.

Do not include a check for something the repo doesn't do. An unused check is noise that trains the user to skim the report.

## If the repo is a mess

Some repos have no consistent pattern to extract — three different ways to write a page, two competing data layers. Don't paper over it by picking one silently. Name the forks to the user, ask which is canonical going forward, and have the architect skill say explicitly: "the codebase contains both X and Y patterns; X is canonical, Y is legacy and should not be extended." That's more useful than a skill that pretends the repo is tidy, and it makes the drift visible instead of permanent.
