---
name: finding-schema
description: Use when reading, writing, or rendering pr-crucible findings — defines the findings.json JSON schema, the findings.md markdown ledger format, and the severity-to-pill presentation map shared by analyze, redteam, and report.
---

# Finding schema

Single source of truth for the pr-crucible output contract. Every command that
touches findings (`analyze`, `redteam`, `report`) reads this skill so the schema
is defined in exactly one place.

## `findings.json` (machine-readable ledger)

```jsonc
{
  "pr": "2497",
  "base_sha": "…",
  "head_sha": "…",
  "diff_range": "origin/staging...HEAD",
  "generated": { "analyze": "<iso8601>", "redteam": "<iso8601|null>" },
  "findings": [
    {
      "id": "regression-1",                 // "<lens>-<n>", unique, stable across stages
      "lens": "regression",                 // emitting lens key
      "title": "Short noun phrase",
      "location": "path/to/file.ext:42",    // file:line
      "claim": "One concrete sentence: what breaks and why.",
      "suspected_severity": "HIGH",          // CRITICAL | HIGH | LATENT | INFO
      "suspected_blocker": true,             // maps to MERGE BLOCKER pill
      "what_to_verify": "The exact probe that would confirm or refute.",
      "status": "proposed",                  // proposed | confirmed | refuted | unverified | unverified (static)
      "static_verdict": null,                // BROKEN | LIKELY-OK | EMPTY | NEEDS-MANUAL
      "evidence": null,                      // filled stage 2 (shape below)
      "repro_artifact": null,                // path to emitted probe/analyzer script
      "refute_reason": null                  // filled if status == refuted
    }
  ]
}
```

### `evidence` shape (filled in stage 2)

```jsonc
{
  "mode": "http | cli | test | custom",
  "base_result": "<captured output from base side>",
  "branch_result": "<captured output from branch side>",
  "captured_output": "<verbatim: status/exit codes, record counts, diffs, error strings>",
  "sweep": "<does the pattern exist elsewhere? scoped result>"
}
```

### Status rules (enforced by redteam)

- `confirmed` requires real before/after evidence (degradation layers 1–2). Never set `confirmed` without `evidence`.
- Static layer only → `status: "unverified (static)"` plus a `static_verdict`.
- Reasoned baseline / explicit skip → `status: "unverified"`.
- Disproved → `status: "refuted"` plus a `refute_reason`. Never delete a refuted finding; keep it with its reason.

## `findings.md` (light markdown ledger)

Skimmable, human-annotatable. One section per finding. `analyze` writes the
pre-evidence version; `redteam` rewrites it with verdicts.

```markdown
# PR <pr> — findings ledger

- **Range:** `<diff_range>`  (base `<base_sha>` … head `<head_sha>`)
- **Generated:** analyze `<iso>` · redteam `<iso|—>`

> To drop a finding before redteam, strike its title or add `KILL:` on its line.

## <pill> <pill> regression-1 — <title>

- **Location:** `path/to/file.ext:42`
- **Claim:** <claim>
- **What to verify:** <what_to_verify>
- **Status:** <status>
- **Evidence:** <captured_output or "—">
- **Refute reason:** <refute_reason or "—">
```

## Severity → pill presentation map (shared with `report`)

| `suspected_severity` | Pill label | HTML class string |
|----------------------|------------|-------------------|
| `CRITICAL` | `SEVERITY: CRITICAL` | `pill sev critical` |
| `HIGH` | `SEVERITY: HIGH` | `pill sev` |
| `LATENT` | `LATENT / DEFENSE-IN-DEPTH` | `pill sev` |
| `INFO` | `INFO` | `pill sev info` |

| `suspected_blocker` | Pill label | HTML class string |
|---------------------|------------|-------------------|
| `true` | `MERGE BLOCKER` | `pill block blocker` |
| `false` | `NOT A MERGE BLOCKER` | `pill block` |

These class strings are the literal `class=` attribute values the `report` template's
inlined CSS defines (`.pill.sev`, `.pill.sev.critical`, `.pill.sev.info`, `.pill.block`,
`.pill.block.blocker`). `report` emits them directly — there is no intermediate alias layer.

`location` renders as a monospace `file:line`. `report` uses this same map so
stage-1 ledger and stage-3 report stay visually consistent without coupling
their generators.
