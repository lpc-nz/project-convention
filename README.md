# project-conventions

**Stop your AI coding agent from inventing a fifth button component.**

A Claude Code plugin that turns a codebase's implicit conventions into explicit guardrails. Run it once per repo; it generates that repo's own architecture skills so future generated code follows your existing patterns instead of generic best practice.

## The problem

You ask an agent for a new settings page. It writes a perfectly good page — with a hardcoded `#3B82F6`, a fresh `<Card>` that's almost but not quite your existing one, a direct database call bypassing your data layer, and a new column added to the migration but not to the TypeScript interface.

Nothing is *wrong*. Everything is *inconsistent*. Do that thirty times and the codebase has no architecture left.

The fix isn't a longer `CLAUDE.md`. It's splitting rules by **when they need to fire**.

## The model

| Layer | Lives in | Fires | Cost |
|---|---|---|---|
| 0. Source of truth | `docs/architecture.md`, design tokens, migrations | never (it's data) | — |
| 1. Non-negotiables | `CLAUDE.md` | every session | always in context — cap at ~30 lines |
| 2. Architect skill | `.claude/skills/` | *before* writing a file | on trigger |
| 3. Reviewer skill + subagent | `.claude/skills/` + `.claude/agents/` | *after* writing, before commit | fresh context |
| 4. Pattern lint | a shell script + a hook | on every save | free, 100% reliable |

**Anything a regex can catch belongs in layer 4, not a prompt.** Reserve the model for judgement calls — "is this a duplicate of an existing helper?"

## Install

```
/plugin marketplace add YOUR-USERNAME/YOUR-REPO
/plugin install project-conventions@lpc-plugins
```

Or without Claude Code plugins — copy `plugins/project-conventions/skills/project-conventions/` into `~/.claude/skills/`.

Or use it with any coding agent at all: [`references/one-shot-prompt.md`](plugins/project-conventions/skills/project-conventions/references/one-shot-prompt.md) is the whole thing as a copy-paste prompt.

## Use

```
> Set up conventions for this repo
```

**Existing codebase** — it investigates first (folder tree, full component inventory, how many places one entity is defined, the best example file per type), then shows you what it found and asks which patterns are intentional rules versus accidents of history. Then it generates.

**Greenfield** — five questions, writes `docs/architecture.md` first, generates the skills pointing at it.

Restart your session afterwards (`.claude/agents/` only loads at startup), then:

```
> add a customer search page          # architect skill places the files
> @architecture-reviewer review        # fresh-context diff review before commit
```

## What makes the reviewer actually work

Three design decisions, and they're the whole point:

**The Precedent Rule.** The reviewer may not report a finding without citing a specific existing file that demonstrates the correct pattern. This forces it to go *read your components folder* before judging your new component, instead of generating an opinion from its training data. It also means "no precedent found" becomes a distinct, useful output — a first-of-its-kind file isn't wrong, it's a new pattern being set, and that deserves a conscious decision rather than a silent one.

**Fresh context.** The reviewer runs as a subagent with an isolated context window. An agent that just wrote 400 lines will approve those 400 lines. One that only sees the diff won't.

**Silence is a valid result.** The reviewer is explicitly instructed that a clean diff produces a one-line "no findings", and that it must never manufacture findings to look useful. A reviewer that emits 40 items gets muted, and a muted reviewer protects nothing.

## Two steps people skip

**Confirming which patterns are real.** Every codebase contains accidents — a component someone wrote at 2am, a service that skips the data layer for a deadline. An extraction pass can't tell those from deliberate design. Skip the confirmation and you permanently enshrine your accidents as law, in a file that then enforces them on everything you write afterwards.

**Testing the guardrail.** Deliberately break one rule, run the reviewer, confirm it catches it *and* cites a precedent. Two minutes. Without it you have decoration.

## Maintenance

When a review surfaces a *decision needed* and you make the call, write that call into `docs/architecture.md`. If you don't, the same question returns next month. The skills are pointers; the docs are the memory.

## Contents

```
plugins/project-conventions/skills/project-conventions/
├── SKILL.md                              the bootstrap workflow
├── templates/
│   ├── architect.template.md             per-repo file-placement skill
│   ├── reviewer.template.md              per-repo consistency reviewer
│   └── agent-and-support.template.md     subagent, CLAUDE.md, lint script
└── references/
    └── one-shot-prompt.md                the whole thing as a paste-in prompt
```

## Licence

MIT — see [LICENSE](LICENSE).
