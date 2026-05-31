---
description: One-time per repo — detect build/run/test/migration facts, confirm via AskUserQuestion, and write the pr-crucible profile (.claude/pr-crucible.local.md).
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, AskUserQuestion
---

# /pr-crucible:init

Bootstrap the shared per-repo profile. Idempotent: re-running offers to update the
existing profile, never clobbers silently.

## Workflow

1. Load the `repo-profile` skill. Walk the tree from the repo root and run its
   detection: migration system, build/run/test commands (from manifests),
   contributor docs (`CLAUDE.md`/`AGENTS.md`), candidate ports, and the
   `exercise.mode` by signal precedence.
2. If `.claude/pr-crucible.local.md` already exists, read it and present detected
   values as proposed *updates* (diff against current), not a fresh write.
3. Present detected values via `AskUserQuestion` — let the user confirm / edit / add:
   `setup_cmd`, `build_cmd`, `run_cmd`, `exercise.mode` (+ ports/probes), `fixtures_cmd`,
   `pr_reviews_dir`, `migration_globs`, `authz_terms`, `refuter_model`. When two
   `exercise.mode` signals tie, ask the user to pick (per the repo-profile skill).
4. Write `.claude/pr-crucible.local.md` (YAML frontmatter per the repo-profile skill
   + a free-form notes section). Ensure `pr_reviews_dir` exists (`mkdir -p`).
5. END: "Profile written to `.claude/pr-crucible.local.md`. Run `/pr-crucible:analyze <PR>` to start a review."

## Notes

- Ship zero project facts: every value comes from detection or the user, never hardcoded.
- The profile is local-only (gitignored via `.claude/*.local.md`).
