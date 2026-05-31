---
name: repo-profile
description: Use when a pr-crucible command needs to know how to build, run, or exercise the target repo — owns tree detection and profile resolution (profile over auto-detection over defaults), including exercise.mode precedence. Single source for repo facts.
---

# Repo profile

Single source of truth for "what is this repo and how do we drive it." All
pr-crucible commands resolve repo facts through this skill so detection logic
lives in exactly one place. The plugin ships zero project knowledge.

## Profile file

`.claude/pr-crucible.local.md` — official `plugin-settings` pattern (YAML
frontmatter + free-form notes). Every field optional; absent fields are
auto-detected or fall back to defaults.

```yaml
---
setup_cmd: "<commands to make a fresh worktree runnable>"
build_cmd: "<build, if any>"
run_cmd:   "<start the thing under test, if long-running>"
exercise:
  mode: "http | cli | test | custom"   # default auto-detected (precedence below)
  base_port: 0                          # http only
  branch_port: 0                        # http only
  probes: ["<request/command/test invocation>", "…"]
fixtures_cmd: "<command to fetch test data/credentials, if needed>"
pr_reviews_dir: "pr-reviews"            # repo-root-relative
migration_globs: ["<glob>", "…"]
authz_terms:     ["<term>", "…"]
refuter_model:   ""                      # blank = active model
accepted_divergences: ["<id/path>", "…"]
---
Free-form notes the lenses/refuter should know (domain model, seeded data, gotchas).
```

## Resolution order (first hit per source wins)

Ascend from the target worktree:

1. `.claude/pr-crucible.local.md` — explicit profile (authoritative).
2. **Auto-detection by scanning the tree** when a field is absent:
   - migration system: any `migrations/` dir, `*.sql`, ORM/framework schema or migration config;
   - build/run/test commands: manifest scripts — `package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `Procfile`, etc.;
   - conventions: any `CLAUDE.md` / `AGENTS.md` / contributor docs found while ascending.
3. Built-in defaults: `pr_reviews_dir: pr-reviews`; verification mode = branch-only.

Detected-but-unconfigured values are surfaced once (via `AskUserQuestion`) with
an offer to persist them to a profile. Nothing project-specific is baked into
plugin code.

## `exercise.mode` auto-detection — signal precedence (language-independent)

First match wins:

1. HTTP server marker (`Procfile` `web:`, framework run-cmd, bound port in config) → `http`
2. else test runner in a manifest (`scripts.test`, `pytest`, `cargo test`, `go test`, etc.) → `test`
3. else single binary/entrypoint (`bin`, `main`, `cmd/`) → `cli`
4. else → `custom`

When two strong signals tie (e.g. HTTP server **and** test runner), surface the
candidates via `AskUserQuestion` rather than guessing — consistent with the
confirm-step UX. Keep the mapping declarative; bake no ecosystem facts into code.

## What this skill returns to a caller

A resolved profile object: every field above, each tagged with its source
(`profile` | `detected` | `default`) so callers can tell the user what was
assumed and offer to persist detected values.
