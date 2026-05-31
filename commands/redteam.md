---
description: Stage 2 — one refuter agent per surviving finding tries to disprove it with real before/after evidence; rewrite findings.json + findings.md with verdicts and evidence.
allowed-tools: Read, Write, Edit, Bash, Grep, Glob, Task
argument-hint: "<PR> [--model <model>]"
---

# /pr-crucible:redteam

Stage 2 of the review. Turns proposed claims into evidence-backed verdicts.

## Arguments

- `$1` — PR number (locates `<pr_reviews_dir>/<PR>/findings.json`).
- `--model <model>` — override the refuter model (else profile `refuter_model`, else active model).

## Workflow

1. Load `findings.json`. Cross-check `findings.md` for human edits: skip any finding
   the user struck or marked `KILL:`. NEVER re-derive the diff range — reuse the
   pinned `base_sha`/`head_sha` from `findings.json`.
2. Resolve the profile via the `repo-profile` skill (for `pr_reviews_dir`, `refuter_model`,
   `accepted_divergences`). Then resolve the refuter model: `--model` → profile
   `refuter_model` → currently-active model.
3. **Dispatch one `refuter` agent per surviving finding.** Default to LAYER-1
   two-worktree isolation so refuters run IN PARALLEL via the Task tool (truest
   evidence, fastest). Fall back to serial single-worktree (degradation layer 2) only
   when two worktrees aren't possible. Hand each refuter exactly one finding object +
   the pinned range. Each refuter uses the `verification-harness` (the same engine
   `/pr-crucible:verify` wraps).
4. **Merge verdicts back** per finding, obeying the `finding-schema` evidence gate:
   `confirmed` (+`evidence`) | `refuted` (+`refute_reason`, kept not deleted) |
   `unverified` | `unverified (static)` (+`static_verdict`). Record `repro_artifact` paths.
5. **Rewrite** `findings.json` (set `generated.redteam` = now) and `findings.md` (verdicts
   + evidence), per the `finding-schema` skill.
6. **END:** "Verdicts written. Review confirmed survivors in `findings.md`, then run `/pr-crucible:report` for the polished HTML."
