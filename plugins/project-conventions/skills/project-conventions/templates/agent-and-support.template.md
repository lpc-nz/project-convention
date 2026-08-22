# Templates: subagent, CLAUDE.md, and pattern-lint script

Three small artifacts. The subagent is nearly stack-agnostic; the other two are short by design.

---

## 1. `.claude/agents/architecture-reviewer.md`

Replace only `<project>` and, if the repo has no git, the diff commands.

```markdown
---
name: architecture-reviewer
description: Reviews written code for consistency with this repo's architecture, design system, and data conventions. Use proactively after any change that adds or edits <the file kinds that drift in this repo>, and before any commit. Also use when the user says "review this", "check this follows our patterns", "is this consistent", or asks for a diff or PR review.
tools: Read, Grep, Glob, Bash, Skill
model: sonnet
---

You are a consistency reviewer for this codebase. You do not write or edit code. You read the change, compare it against what already exists, and report.

## Non-negotiable first step

Invoke the `<project>-reviewer` skill and follow its protocol. It holds the actual rules. If that skill isn't available, say so and fall back to: read `CLAUDE.md`, then the canonical docs it names, then proceed.

You start with a fresh context window — you did not see the conversation that produced this code, and that is deliberate. Never assume intent. Read the files.

## Standing constraints

- **Review the diff, not the repo.** Start with `git diff` (or `git diff main...HEAD` on a branch). Untouched code is out of scope.
- **The Precedent Rule.** No finding without a cited existing file showing the correct pattern. If you can't find a precedent, it's a *decision needed*, not a violation — say that plainly.
- **Verify before asserting.** Every "this already exists" or "we always do X" must come from a grep or read you performed in this session. Never state a convention from memory of similar codebases.
- **Read-only.** Do not edit, stage, or commit. Report and stop. The author decides.
- **Silence is a valid result.** If the diff matches existing patterns, say "No findings" in one line. Never manufacture findings to look useful — a noisy reviewer gets ignored, and an ignored reviewer protects nothing.
- **Skip what tooling catches.** Lint, format, and type errors belong to the linter. Don't spend the report on them.

## Output

Use the report format defined in the reviewer skill. End with the verdict line — no closing summary, no encouragement.
```

**Why `model: sonnet`:** review is a bounded, checklist-shaped task and runs often. Bump to opus only if the repo has genuinely subtle invariants. Note that subagent files are loaded at startup — restart the session after adding one.

---

## 2. `CLAUDE.md` (repo root)

Hard ceiling ~30 lines. This is in context every single session, so it earns its place only if it's needed *every* session. Everything conditional belongs in a skill.

```markdown
# <Project>

<One line: what this is.>

## Canonical documents — edit in place, never fork

| Topic | File |
|---|---|
| Architecture & data model | `docs/architecture.md` |
| Decision history (append-only) | `docs/history.md` |
| Design system | `<tokens file>` |
| Build order / status | `docs/implementation-plan.md` |
| Database schema | `<migration file>` |

Never create a second file on any of these topics. No `-v2`, `-new`, `-final`.

## Non-negotiables

1. <The load-bearing layering rule — one line.>
2. <The data-mirror rule: an entity is defined in N places, all updated in the same change.>
3. <The design-token rule: no hardcoded visual values.>
4. Before adding a file, consult the `<project>-architect` skill. Before committing, run `@architecture-reviewer`.

## Naming

<file type>: <casing> · <repeat, one line total if possible>
```

---

## 3. `scripts/lint-patterns.sh`

Only include checks you verified are real rules. Each is a mechanical rule the LLM reviewer then doesn't have to spend attention on. Adapt the paths and patterns; the shape stays the same.

```bash
#!/usr/bin/env bash
# Mechanical convention checks. Fast, deterministic, no LLM.
# Run on save (PostToolUse hook) and in CI.
set -uo pipefail
fail=0

check() { # check <description> <grep-command...>
  local desc="$1"; shift
  local out
  out=$("$@" 2>/dev/null) || true
  if [[ -n "$out" ]]; then
    echo "✗ $desc"
    echo "$out" | sed 's/^/    /'
    fail=1
  fi
}

# --- Design tokens: no hardcoded colours outside the theme layer
check "hardcoded hex colours outside the theme layer" \
  grep -rEn --include=*.tsx --include=*.ts \
    '#[0-9a-fA-F]{6}\b' <ui source paths> \
    --exclude-dir=<theme dir>

# --- Layering: UI must not reach past the data layer
check "database client imported directly in a page/screen" \
  grep -rn --include=*.tsx 'createClient\|new DbContext\|<the forbidden import>' <pages path>

# --- SQL casing
check "camelCase identifiers in SQL migrations" \
  grep -rEn '[a-z]+[A-Z][a-z]*\s+(text|integer|uuid|numeric|boolean|timestamptz)' <migrations path>

# --- Doc forking
check "forked canonical docs" \
  bash -c 'ls docs/ 2>/dev/null | grep -Ei "(-v[0-9]|-new|-final|-copy|-old)\."'

# --- Migration immutability (enable once the base migration is applied)
# check "modified an applied migration" \
#   bash -c 'git diff --name-only HEAD | grep "<base migration path>"'

if [[ $fail -eq 0 ]]; then
  echo "✓ pattern checks passed"
else
  echo
  echo "Pattern violations found. See CLAUDE.md for the conventions."
fi
exit $fail
```

`chmod +x scripts/lint-patterns.sh`. Wire it up in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Write|Edit",
        "hooks": [ { "type": "command", "command": "./scripts/lint-patterns.sh" } ] }
    ]
  }
}
```

Where a rule can be expressed as a real lint rule instead of a grep — import boundaries via `eslint-plugin-boundaries`, .NET analyzers, `import-linter` for Python — prefer that. A proper lint rule understands the AST; grep doesn't, and will eventually produce a false positive that trains the user to ignore it.
