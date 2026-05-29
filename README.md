# project-review

A Claude Code plugin that runs **multi-lane project-wide review** — 16 independent review lanes routed by a single hub, with standardized findings. **Stand-alone** — runs in any repo, no companion plugin required.

Each lane targets a distinct review dimension (plan adherence, adversarial red-team, full audit, LLM hallucination, scope creep, spec drift, failure mode, observability, perf/cost, breaking change, coupling, dependency health, test strength, DX, harness meta, waste/yield). Lanes are invoked directly or via cadence presets (`after-task` / `after-feature` / `architecture`).

Optional integration with [`dharness`](https://github.com/gimangdoo/dharness) is auto-detected at runtime (see [Output](#output)) — no hard dependency.

---

## What's inside

| Component | Path | Purpose |
|-----------|------|---------|
| Hub skill | `plugins/project-review/skills/prj-review/SKILL.md` | Router. Picks lane set from cadence preset, free-form trigger words, or explicit lane IDs. Owns the shared finding schema and output location convention. |
| Lane skills × 16 | `plugins/project-review/skills/prj-review-{lane}/SKILL.md` | One skill per review dimension. Each emits the common finding YAML record. |
| Cadence commands × 3 | `plugins/project-review/commands/prj-{after-task, after-feature, architecture}.md` | Slash entry points mapping each cadence to its lane subset. |

---

## Lane catalog

| ID | Lane | Catches |
|----|------|---------|
| L01 | `prj-review-plan-adherence` | Pre-plan vs progress, do/don't recognition inside the work |
| L02 | `prj-review-adversarial` | Outsider/attacker logic constructed to break the system |
| L03 | `prj-review-full-audit` | Duplication, dead code, inconsistency, security suspects (broad sweep) |
| L04 | `prj-review-hallucination` | LLM-fabricated imports, fake APIs, wrong signatures, placeholders |
| L05 | `prj-review-scope-creep` | Code added beyond the user's request — CLAUDE.md §3 enforcer |
| L06 | `prj-review-spec-drift` | README / CLAUDE.md / comments / docstrings vs actual code |
| L07 | `prj-review-failure-mode` | Failure paths, rollback, idempotency, partial-failure recovery |
| L08 | `prj-review-observability` | Post-incident debuggability — logs, metrics, traces, error context |
| L09 | `prj-review-perf-cost` | N+1, hot-loop algorithms, memory leak candidates, LLM token spend regression |
| L10 | `prj-review-breaking-change` | Public API / schema / CLI / env / config semver violations |
| L11 | `prj-review-coupling` | Module boundaries, circular imports, layering violations, leaky abstractions |
| L12 | `prj-review-dep-health` | CVE, deprecated, license conflicts, unused/transitive bloat |
| L13 | `prj-review-test-strength` | Assertion strength, isolation, determinism, mutation detection (not coverage) |
| L14 | `prj-review-dx-onboarding` | First-5-min setup, env var docs, error friendliness |
| L15 | `prj-review-harness-meta` | dharness / agent / skill self-consistency — dangling links, stale baseline, duplicate triggers |
| L16 | `prj-review-waste-yield` | Macro-level unused output — features, flags, endpoints, agents, skills, branches |

---

## Cadence presets

```
/prj-after-task         → L04 + L05 (+L01 if plan file present)
/prj-after-feature      → L06 + L07 + L13 (+L03/L09/L10 conditional)
/prj-architecture       → L08 + L11 + L15 (+L16/L03/L12 optional)
```

Additional cadences accepted as arguments to the hub:

```
/prj-review pr-gate     → L10
/prj-review weekly      → L12
/prj-review monthly     → L16
/prj-review on-incident → L02 + L07 + L08
```

Direct lane invocation:

```
/prj-review L01,L05,L15
/prj-review hallucination scope-creep
```

---

## Output

Every lane emits findings into a shared YAML schema:

```yaml
finding:
  id: L{NN}-{seq}
  lane: {lane-id}
  severity: critical | high | medium | low | info
  location: path/to/file.ext:line[-range]
  problem: <one-line factual statement>
  evidence: |
    <quote or fact citation>
  why_it_matters: <one-line impact>
  fix:
    type: code | doc | process | harness | skip
    suggestion: <concrete action>
  confidence: 0-100
  tags: [...]
```

Findings are persisted under `{review_out}/` — the hub resolves this at runtime in priority order:

| Priority | Condition | Resolved `{review_out}` |
|----|----|----|
| 1 | `PRJ_REVIEW_OUT` env var set | `$PRJ_REVIEW_OUT` |
| 2 | `<repo-root>/.review-out/` directory exists | `<repo-root>/.review-out` |
| 3 | `<repo-root>/dharness-project/dharness-rating/` directory exists (dharness env auto-detect) | `dharness-project/dharness-rating/review` |
| 4 | default | `<cwd>/.review` |

```
{review_out}/
└── {YYYY-MM-DD}_{run-slug}/
    ├── _hub_summary.md          # TL;DR + blockers + next-action
    ├── L{NN}_{lane}.md          # one file per lane
    └── ...
```

The hub also appends a one-line entry to `_index.md` for cumulative tracking.

---

## Install

This repo is itself a [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces). Add it:

```
/plugin marketplace add gimangdoo/project-review
/plugin install project-review
```

Or clone locally and add `./` as a marketplace.

---

## dharness integration (optional, auto-detected)

Tight coupling with dharness is **explicitly out of scope** — this plugin operates as a standalone tool. dharness integration is limited to:

1. **Output path co-location** — if `<repo-root>/dharness-project/dharness-rating/` exists, the hub writes findings under `dharness-project/dharness-rating/review/` instead of `.review/`. No dharness skill or schema is invoked.
2. **Baseline read (L15 lane only)** — `prj-review-harness-meta` reads `<repo-root>/dharness/_workspace/_baseline/*` when present. Absent baseline → `info: [baseline-missing-{file}]` finding + best-effort.

Auto-triggered lanes on dharness events, `/harness:harness-review` wrappers, and intent_profile schema extensions are **not pursued** — the cadence commands (`/prj-after-task`, `/prj-after-feature`, `/prj-architecture`) cover the same surface without cross-plugin coupling.

---

## Status

- **v0.2.0** — Stand-alone confirmed. Output path resolved at runtime (env / `.review-out/` / dharness co-location / default `.review/`). Sample dogfood findings (L04 + L15 patches) applied.
- **v0.1.0** — Initial 16-lane + hub + 3-cadence drop.

---

## License

[MIT](./LICENSE) — © 2026 gimangdoo
