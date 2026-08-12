# Template: `<project>-reviewer` skill

Write to `.claude/skills/<project>-reviewer/SKILL.md`. Replace every `<...>`. The sections marked **KEEP VERBATIM** carry across every project unchanged — they're what makes a reviewer usable rather than noisy. Only the rule-source table and the three check sections are project-specific.

---

```markdown
---
name: <project>-reviewer
description: Reviews already-written code in the <Project> repo (<stack>) for consistency with the established architecture, design system, and <data layer> conventions. Use after generating or editing any code here, before committing, when the user says "review this", "check this follows our patterns", "is this consistent", "did I do this right", or asks for a PR or diff review. Also use proactively after any change that adds <the 4-5 file kinds that historically drift in this repo>. Trigger even when the user only asks "does this look ok".
---

# <Project> Reviewer

Reviews code that already exists against established patterns. Counterpart to `<project>-architect`: architect = prevention, reviewer = detection.

The job is not "is this good code". Plenty of good code is wrong here because it duplicates <existing helper location>, invents a value that isn't in <the tokens file>, or contradicts <the schema source of truth>. **Consistency with what's already there is the thing being graded.**

## The Precedent Rule    <!-- KEEP VERBATIM -->

> **Do not report a finding unless you can cite a specific existing file, and ideally a line, that demonstrates the correct pattern.**

Before judging any new file, find and read 1–2 sibling files of the same kind. A new page is judged against the other pages in this repo, not against general framework advice.

Three effects: it stops the review inventing rules that aren't this repo's rules; it makes findings actionable ("match `X.tsx`" beats "use consistent styling"); and when **no precedent exists**, that's real signal — say so and escalate as a *decision needed*, not a violation. A first-of-its-kind file isn't wrong, it's a new pattern being set, and it deserves a conscious choice.

## Review protocol

**1. Scope the change.** Get the actual diff — `git diff --stat` then `git diff`, or `git diff main...HEAD` on a branch. If the user pasted code with no repo state, review what they pasted and say clearly you couldn't verify against the live tree. Reviewing the whole repo instead of the change is the fastest way to produce noise nobody reads.

**2. Load only the rules that apply to this diff.** Map each changed path to its rule sources and read them:

| Changed path | Read before reviewing |
|---|---|
| `<ui components path>` | <tokens/design doc> + 2 existing components in the same folder |
| `<pages path>` | 1 existing page + <the data layer file> |
| `<shared logic path>` | the folder's README + <the canonical types file> |
| `<migrations path>` | <the base schema file> + <the doc recording whether it's been applied> |
| any new <calculation / domain rule> | search `<shared logic path>` for a near-duplicate **first** |
| docs | the canonical-doc list in `CLAUDE.md` |

**3. Run the checks** in the sections below.

**4. Verify, don't assume.**   <!-- KEEP VERBATIM --> Every claim about "this already exists" or "we always do X" must be backed by a grep or read you performed in this session. If a check needs a fact you didn't verify, downgrade it to a question rather than stating it as a violation.

**5. Emit the report** in the format at the bottom.

---

## Check: architecture & layering

<The load-bearing rule of this repo, stated in one line.>

- **Duplicated logic.** <What kind of logic must be centralised, where it belongs, and the grep to run before flagging.> Critical.
- **Layer skipping.** <The specific bypasses: e.g. a page calling the DB directly instead of the data layer; a screen with inline queries instead of a repository.> Critical.
- **Wrong-direction imports.** <The dependency direction that must hold.>
- **New top-level folder** not in the architect skill's map → escalate as a decision, not a violation.
- **Registration missed.** <Barrel export, DI container, router — whatever this stack requires alongside a new file.> Minor, but it's how inline duplicates get born a week later.

## Check: UI & design system

<Delete this whole section if the repo has no UI.>

- **Reinvented primitive.** Before accepting any new component, list what's already in <components path>. If something within one prop of it exists, the finding is "extend `<Existing>` with a variant" and must name the file. This is the most common drift in any UI codebase and the check worth running hardest.
- **Off-token values.** Hardcoded colours, arbitrary font sizes, one-off spacing instead of <tokens source>. Quote the token that should have been used.
- **Structural mismatch.** Compare against a sibling: same prop-naming, same wrapper treatment, same handling of loading and empty states? An inconsistent empty state is a real finding, not nitpicking — it's what makes an app feel assembled by five different people.
- **Placement.** Presentational components in <path>; anything with data fetching belongs in <the layer that owns it>.

## Check: database & data model

- **The N-way mirror.** <List every place one entity must be defined, with casing per place.> Grep the new field name across the repo and report exactly which places are missing. Critical.
- **Migration discipline.** <The immutability rule and how to check whether it applies yet.> Never guess which state the project is in; if ambiguous, ask.
- **Naming.** <Casing per layer.> A wrong-cased column is Critical, not Minor — it silently breaks the mapper convention.
- **Type drift.** Nullability and types must agree between <the SQL column>, <the validation schema>, and <the language-level type>.
- **Validation bypass.** New form or mutation not going through <the shared validation schema>. Major — two callers validating the same entity differently is the same class of problem as two callers computing the same number differently.

## Check: conventions & docs

- <Naming conventions per file type.>
- **Doc forking.** A new `architecture-v2.md`, `schema-new.sql`, or second canonical doc is Critical regardless of how good the content is. Canonical docs are edited in place.
- **Stale status.** Code landed that a README or plan file still describes as "not started" → the same change should have updated it. Minor, but these rot fast, and a rotten plan file gets ignored, which removes a layer of the system.

---

## What NOT to flag    <!-- KEEP VERBATIM -->

A reviewer that produces 40 findings gets muted, and then it protects nothing. Stay silent on:

- Personal style preferences the repo hasn't taken a position on.
- Anything the linter, formatter, or type checker already catches — that's the deterministic layer's job.
- Suggested refactors of untouched code outside the diff.
- Deviations that are clearly deliberate and explained in a comment or the plan doc.
- Speculative future-proofing ("you might later want…").

If the diff is clean, say so in one line. "No findings — matches existing patterns" is a valid and valuable output.

## Output format    <!-- KEEP VERBATIM -->

```
## Review: <branch or description>  ·  <N> files

**Verdict:** PASS | PASS WITH NITS | CHANGES NEEDED

### Critical
1. **<one-line title>** — `path/to/file:42`
   What: <what the code does>
   Rule: <the convention, and where it's written down>
   Precedent: `path/to/existing/example` does it this way
   Fix: <concrete change>

### Major
...

### Minor
...

### Decisions needed (no precedent found)
- <thing> — no existing pattern for this. Options: A / B. Recommend <X> because <reason>.
```

Severity:
- **Critical** — forks the architecture, breaks the data mirror, duplicates centralised logic, or edits an immutable migration. Blocks merge.
- **Major** — bypasses an established layer or shared schema; works today, causes divergence later.
- **Minor** — naming, stale docs, missed registration.

End with the verdict line and nothing else. No summary paragraph, no encouragement — the author is going to act on the list.
```

---

## Notes for whoever fills this in

- Only include checks for things **this** repo can actually get wrong. An unused check trains the user to skim the report.
- Each check should be phrased so it's falsifiable — "uses a token from `theme.ts`" is checkable; "follows good design" is not.
- Keep under ~200 lines.
