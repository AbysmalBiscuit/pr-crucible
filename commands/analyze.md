---
description: Stage 1 — pin the PR diff range, route review lenses, dispatch them in parallel, and write the findings ledger (findings.json + findings.md).
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Task, AskUserQuestion
argument-hint: "<PR> [--base <ref>] [--head <ref>] [--lenses a,b,c]"
---

# /pr-crucible:analyze

Stage 1 of the review. Produces the pre-evidence findings ledger.

## Arguments

- `$1` — PR number (or branch ref).
- `--base <ref>` / `--head <ref>` — explicit range overrides (win over `gh`). Use on repos without a GitHub remote.
- `--lenses a,b,c` — explicit lens set (wins over auto-detection).

## Workflow

1. **Pin the diff range.** Resolve in this precedence:
   - `--base`/`--head` given → use `<base>...<head>` (default head = `HEAD` if only `--base` given).
   - else `gh pr diff <PR>` (derive base from the PR's merge target).
   - else `origin/<base>...HEAD`.
   Record `base_sha`, `head_sha`, and the `diff_range` string. These are authoritative
   for the whole review — every lens receives them and must NOT re-derive them.
2. **Resolve the profile** via the `repo-profile` skill (for `pr_reviews_dir`,
   `migration_globs`, `authz_terms`).
3. **Route lenses:**
   - `--lenses` flag present → use exactly those.
   - else inspect changed paths against each lens's auto-trigger; core lenses
     (`regression`, `impl-redteam`, `test-coverage`) are always pre-checked;
     `migration`/`authz`/`api-contract` pre-checked when their trigger matches.
   - Present the full lens list via `AskUserQuestion` (multiSelect, auto-detected
     pre-checked) so the user enables/disables.
4. **Dispatch** the confirmed lens agents IN PARALLEL via the Task tool, each in
   isolated context, each handed the pinned range. Pass each its matching
   `agents/lens-<key>.md` persona.
5. **Collect** all finding sets. NEVER merge across lenses — divergence is preserved.
   Assign stable ids (`<lens>-<n>`).
6. **Write artifacts** under `<pr_reviews_dir>/<PR>/`, per the `finding-schema` skill:
   - `findings.json` — machine ledger, `generated.analyze` = now, `generated.redteam` = null, every finding `status: "proposed"`.
   - `findings.md` — light markdown ledger to skim/annotate.
7. **END:** "Wrote `<pr_reviews_dir>/<PR>/findings.md`. Review it, annotate or mark findings to drop, then run `/pr-crucible:redteam <PR>`."
