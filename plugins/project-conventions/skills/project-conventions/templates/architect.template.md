# Template: `<project>-architect` skill

Write to `.claude/skills/<project>-architect/SKILL.md`. Replace every `<...>` placeholder. Delete any section that doesn't apply to this stack — an empty section is worse than a missing one.

---

```markdown
---
name: <project>-architect
description: Keeps the <Project> repo (<one-line stack summary>) on its canonical folder structure and conventions. Use whenever creating, moving, renaming, or organizing any file here — <list the 4-6 concrete file kinds this repo has: new pages, components, services, migrations, shared types, docs>. Trigger when the user asks "where should this file go", "how do I add a new <thing>", mentions <Project> or <its alternate names>, or references <list the canonical doc paths>. Always consult before creating a new top-level folder, a second architecture/schema doc, or <the kind of logic that must not be duplicated>.
---

# <Project> Architect

<Two or three sentences: what the repo is, and — most importantly — the ONE structural rule most worth protecting as it grows. Name the specific drift you're preventing, e.g. "it's easy for a future change to just add the calc inline in this one screen, and start a fork where three callers compute the same number differently.">

## Before creating any file, ask two questions

1. **Does this already exist somewhere?** Search <the shared-logic path> for the logic, `docs/` for the doc, and the target directory's own README for what's planned there. If something close exists, edit it in place — never add a `-v2`, `-new`, or `-final` sibling.
2. **Where does this kind of file live in the map below?** If the task fits no row, flag the ambiguity to the user rather than inventing a new top-level folder.

## Canonical folder map

<A tree of the repo, 2-3 levels deep. Annotate each significant folder with what belongs there. Mark canonical single-source files explicitly — e.g. "THE architecture doc — edit in place, never fork." Omit node_modules/build output. Aim for 20-40 lines.>

## Placement rules by task type

<One bolded entry per kind of thing this repo creates. For each: the exact destination path, what must be registered/updated alongside it, and what NOT to do. Cover at minimum:>

**New <shared business logic>** → <path>. <Why it can't live in the apps.> Export from <barrel file>.

**New <UI page/screen>** → <path pattern>. Registered in <nav/router file>. Data access goes through <the data layer>, not <the thing it must not call directly>.

**New <UI component>** → <path>. Check <design tokens doc> first — <what's locked>.

**New entity or column** → touch all of these in the same change: <list every place an entity is mirrored, with the casing convention for each>. Search the field name across the repo before considering the change done.

**Database change** → <the migration rule, including the "has the first migration been applied yet?" branch if relevant>.

**New documentation** → <route each doc topic to its canonical file, "edit in place">.

## Naming conventions

- <file type>: <casing> (`<example>`)
- <repeat per type: docs, components, variables, SQL, migrations, tests>

## If something doesn't fit

If a task genuinely maps to no row above, don't force it into an existing folder and don't silently create a new top-level one — name the ambiguity to the user and propose where it should live, checking <architecture doc> and <plan doc> first for whether it's already anticipated there.
```

---

## Notes for whoever fills this in

- **The description field is the trigger.** Claude tends to *under*-trigger skills. Be pushy and concrete: list the actual phrases the user says, and the actual filenames they'd mention. Vague descriptions produce a skill that never fires.
- **A folder's README is part of the source of truth.** If the repo has per-folder READMEs, say so in the skill and instruct that they be updated in the same change.
- **Don't restate the design tokens or schema in this file.** Point at them.
- Keep under ~120 lines. This loads on every file-placement decision.
