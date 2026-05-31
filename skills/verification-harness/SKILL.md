---
name: verification-harness
description: Use when a pr-crucible refuter or the verify command needs before/after evidence for a finding — owns the portable verification methodology (exercise modes, layered degradation, idempotency gate, static fallback, repro artifact). Parameterized entirely by the resolved repo profile.
---

# Verification harness

Single source of truth for "how we get before/after evidence." Loaded by the
stage-2 refuters and by `/pr-crucible:verify`. The method ships; all repo-specific
inputs (commands, ports, probes, fixtures, authz terms) come from the resolved
profile via the `repo-profile` skill — no language, framework, or transport assumed.

## Exercise modes

Compare branch behaviour against base using whatever `exercise.mode` fits:

- **http** — stand up base and branch (two worktrees on two ports, or one worktree toggled), replay the same request to both, diff responses.
- **cli** — run the built binary/script for base vs branch on identical inputs, diff stdout/stderr/exit code.
- **test** — run the project's own test invocation against base vs branch, diff pass/fail + output.
- **custom** — run profile-supplied `probes` for each side, diff captured output.

## Methodology (mode-independent)

1. **Auto-discover the affected surface from the diff** — derive the set to exercise (changed routes / commands / symbols) from `git diff <base>...HEAD`, never a hand-written list.
2. **Primary gate vs warn-only** — one designated signal is the pass/fail gate (HTTP status, exit code, test result); softer signals (body shape, stdout formatting) are warn-only and never fail the run alone.
3. **Idempotency gate — never mutate shared state.** Read-only/idempotent ops run live. Mutating ops are listed but NOT executed against shared resources; they route to static analysis.
4. **Static call-graph fallback** for anything not safely executable: trace the changed entry point through its call graph, extract its operations, check them statically (permissions/grants, contracts, types). Emit per-item verdicts: `BROKEN` | `LIKELY-OK` | `EMPTY` | `NEEDS-MANUAL`.
5. **Accepted-divergence allowlist** — known/intended changes (`accepted_divergences`) "pass" only if they diverge in exactly the expected way; re-runs don't re-flag them.
6. **Explicit skip reporting** — anything unresolved (unbootstrappable params, external deps) is reported as skipped with a reason, never silently passing.
7. **Emit a repro artifact** — write the generated probe/analyzer script into the review dir so a human can re-run it and attach output.

## Layered degradation (resolved fallback)

1. **Live before/after** — isolated base vs branch (two worktrees). Truest evidence. → can reach `confirmed`.
2. **Single worktree toggled** by git stash/checkout when two worktrees aren't possible (serial, mutates worktree). → can reach `confirmed`.
3. **Static analysis** for the unsafe/non-runnable set — `BROKEN`/`LIKELY-OK`/`EMPTY`/`NEEDS-MANUAL`; finding tagged `unverified (static)`, plus a repro script.
4. **Reasoned baseline + explicit skip** — last resort; finding tagged `unverified`, never silently dropped.

A finding reaches `confirmed` ONLY via layer 1–2 (real before/after evidence).
Layers 3–4 yield `unverified (static)` / `unverified`, surfaced honestly in the
report's Method & limitations.

## Output

Per checked finding: the `evidence` object (shape defined in the `finding-schema`
skill) plus the resolved layer used and any skip reasons. `verify` prints this;
refuters fold it into `findings.json`.
