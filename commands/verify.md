---
description: Helper — run one before/after verification (a single finding id, or an ad-hoc --claim) via the verification harness and print the captured output.
allowed-tools: Read, Write, Edit, Bash, Grep, Glob
argument-hint: "<PR> [finding-id | --claim \"…\"]"
---

# /pr-crucible:verify

A thin, standalone wrapper around the `verification-harness` skill. Used by the
stage-2 refuters and runnable by hand to re-check one finding or probe an ad-hoc claim.

## Arguments

- `$1` — PR number (locates `<pr_reviews_dir>/<PR>/findings.json`).
- `$2` — a finding id (e.g. `regression-1`) → verify that finding's `what_to_verify`;
  or `--claim "…"` → verify an ad-hoc claim with no ledger entry.

## Workflow

1. Resolve the profile via the `repo-profile` skill (commands, ports, probes, fixtures).
2. Load the `verification-harness` skill.
3. Resolve the target: a finding id loads its `what_to_verify` from `findings.json`;
   `--claim` uses the supplied text.
4. Run ONE before/after check at the strongest available degradation layer for the
   resolved `exercise.mode`. Honor the idempotency gate — never mutate shared state.
5. Print the captured output verbatim (status/exit codes, record counts, diffs, error
   strings) and which layer was used. Write the repro artifact into the review dir.
6. Do NOT modify `findings.json` — this command only reports. (The `redteam` command
   is what writes verdicts back.)
