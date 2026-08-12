# The one-shot prompt

For when you can't or don't want to install the skill — a different tool, a colleague's machine, a quick trial on an unfamiliar repo. Paste this into a coding agent sitting in the repo root. It does the same job as the skill, in one message.

---

## Prompt: bootstrap conventions for this repo

```
Set up architecture-consistency guardrails for this repo. Work in three phases and
stop for my confirmation between each.

PHASE 1 — INVESTIGATE (read only, no files written yet)

Explore the codebase and report what you find. Do not ask me things you can read:

1. Structure: directory tree 3 levels deep, excluding build output and dependencies.
2. Existing docs: README, docs/, ARCHITECTURE.md, CONTRIBUTING.md, ADRs.
3. Layering: what imports what? Is there a shared/core/domain layer? Does the UI
   reach past the data layer anywhere?
4. UI system: list every component in the component folder. Find the theme/tokens
   file. Open 2 components and describe their shared shape — prop naming, wrapper
   treatment, how they handle loading and empty states.
5. Data layer: migration folder and numbering scheme, entity/model definitions,
   validation schemas. Count how many separate places ONE entity is defined in.
6. Naming conventions per file type: SQL, components, variables, folders, migrations.
7. For each kind of thing (page, component, service, migration, test), name the single
   BEST existing example — these become canonical reference files.

Then give me a numbered list of the conventions you believe are intentional rules,
and ask me which are real versus accidents of history. Flag anywhere the repo
contains two competing patterns rather than picking one silently.

PHASE 2 — GENERATE (after I confirm the rules)

Write these files:

a) docs/architecture.md — only if it doesn't exist. Short: stack, folder map,
   layering rules, data model.

b) CLAUDE.md at the root — MAXIMUM 30 LINES. A table of canonical documents
   ("edit in place, never fork"), 4 non-negotiable rules, naming conventions.
   Anything that isn't needed EVERY session goes in a skill instead, not here.

c) .claude/skills/<project>-architect/SKILL.md — fires BEFORE writing a file.
   Contains: a canonical folder map with each folder annotated; placement rules
   by task type (one entry per kind of thing this repo creates, each naming the
   exact destination and what must be registered alongside it); naming conventions;
   and an "if it doesn't fit, ask rather than invent a new top-level folder" rule.
   The frontmatter description must be pushy and concrete — list the actual phrases
   I'd say and the actual filenames I'd mention. Vague descriptions never trigger.

d) .claude/skills/<project>-reviewer/SKILL.md — fires AFTER writing, before commit.
   It MUST contain all of these:
   - THE PRECEDENT RULE: no finding may be reported without citing a specific
     existing file that demonstrates the correct pattern. If no precedent exists,
     it's a "decision needed", not a violation.
   - Diff-first protocol: review `git diff`, never the whole repo.
   - A table mapping each changed path to the docs and precedent files to read first.
   - Check sections for: architecture/layering, UI/design system, data model,
     naming/docs. Only include checks THIS repo can actually get wrong.
   - A "what NOT to flag" list: style preferences, anything the linter catches,
     refactors outside the diff, speculative future-proofing. State that a clean
     diff should produce a one-line "no findings" and that a noisy reviewer
     gets muted and then protects nothing.
   - An output format with Critical / Major / Minor / Decisions-needed, where every
     finding has: file:line, the rule, the precedent file, and a concrete fix.

e) .claude/agents/architecture-reviewer.md — a read-only subagent that invokes the
   reviewer skill, reviews only the diff, cites precedents, never edits or commits,
   and does not restate rules from memory.

f) scripts/lint-patterns.sh — deterministic grep checks for the MECHANICAL rules
   only (hardcoded colours outside the theme, forbidden imports in UI files, wrong
   casing in SQL, forked docs). Anything a regex can catch belongs here, not in a
   prompt. Make it executable and exit non-zero on violations.

PHASE 3 — TEST

1. Ask yourself "where would a new <typical thing for this repo> go?" and confirm the
   architect skill answers it from the folder map rather than guessing.
2. Write a scratch file that deliberately violates one rule. Run the reviewer.
   Confirm it catches the violation AND cites a precedent file. Delete the scratch file.
3. If the reviewer missed it, the rule was too vague — sharpen it and retest.

Then tell me: what to restart, how to invoke the reviewer, and the maintenance rule
(when a review surfaces a "decision needed" and I make a call, that decision goes
into docs/architecture.md — otherwise the same question returns every week).
```

---

## Shorter variant: reviewer only

If the repo already has decent docs and you just want the review layer:

```
Create a code-consistency reviewer for this repo as .claude/agents/architecture-reviewer.md
plus a matching skill.

First, read the repo's docs and identify: the layering rule, where the design tokens
live, and how many places one database entity is defined in. Name the best existing
example file for each kind of thing (page, component, service, migration).

Then write a reviewer whose core constraint is THE PRECEDENT RULE: it may not report
any finding without citing a specific existing file that shows the correct pattern.
No precedent found = "decision needed", not a violation.

It reviews `git diff` only, never the whole repo. It is read-only — reports and stops.
It skips anything the linter already catches. A clean diff produces a one-line
"no findings"; it must never manufacture findings to look useful.

Output format: Critical / Major / Minor / Decisions-needed, each finding carrying
file:line, the rule, the precedent file, and a concrete fix. Ends with a verdict line
and nothing else.

Then test it by deliberately violating one rule and confirming it catches it.
```

---

## Why the phases matter

The single biggest failure mode is letting the agent write the skills before you've confirmed which conventions are real. Every codebase contains accidents — a component someone wrote at 2am, a service that skips the data layer because of a deadline. An agent extracting patterns can't distinguish those from deliberate design, and if you skip the confirmation step you get your accidents enshrined as law, permanently, in a file that then enforces them on all future code.

The second failure mode is skipping Phase 3. A guardrail nobody tested is decoration. Two minutes of deliberately breaking a rule tells you whether you built something real.
