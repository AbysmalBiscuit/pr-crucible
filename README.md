# pr-crucible

A Claude Code plugin for staged, evidence-backed, adversarial PR review. Repo-,
language-, and framework-agnostic — it ships zero project facts and auto-detects or
reads everything from a per-repo profile.

## Pipeline

| Command | Stage | Does |
|---------|-------|------|
| `/pr-crucible:init` | setup (once per repo) | Detect build/run/test/migration facts, confirm, write `.claude/pr-crucible.local.md`. |
| `/pr-crucible:analyze <PR>` | stage 1 | Pin the diff range, route lenses, dispatch in parallel, write `findings.json` + `findings.md`. |
| `/pr-crucible:redteam <PR>` | stage 2 | One refuter per finding tries to disprove it with real before/after evidence; write verdicts. |
| `/pr-crucible:report <PR>` | stage 3 | Render a self-contained HTML report from the evidenced ledger. |
| `/pr-crucible:verify <PR> [id\|--claim]` | helper | Run one before/after check standalone. |

Two human checkpoints: after `analyze` (review/annotate/kill findings) and after
`redteam` (review confirmed survivors).

## Lenses (v1)

Regression and impl-redteam always run; migration, authz, test-coverage, and
api-contract auto-trigger from the changed paths (and are user-toggleable).

## Install

This repo is also a Claude Code plugin marketplace. Add it, then install:

```
/plugin marketplace add AbysmalBiscuit/pr-crucible
/plugin install pr-crucible@pr-crucible
```

(Replace `AbysmalBiscuit/pr-crucible` with the repo's `owner/name` if you forked it.
You can also `add` a local path instead of the GitHub slug.)

## Profile

`/pr-crucible:init` writes `.claude/pr-crucible.local.md` (local-only, gitignored).
All fields are optional; absent fields are auto-detected. See the `repo-profile`
skill for the full field list.

## License

GPL-3.0-or-later.
