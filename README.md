# project-review

A Claude Code plugin that runs **multi-lane project-wide review** — 16 independent review lanes routed by a single hub, with standardized findings designed to bolt into the [`dharness`](https://github.com/gimangdoo/dharness) workflow.

Each lane targets a distinct review dimension (plan adherence, adversarial red-team, full audit, LLM hallucination, scope creep, spec drift, failure mode, observability, perf/cost, breaking change, coupling, dependency health, test strength, DX, harness meta, waste/yield). Lanes are invoked directly or via cadence presets (`after-task` / `after-feature` / `architecture`).

---

## What's inside

| Component | Path | Purpose |
|-----------|------|---------|
| Hub skill | `plugins/project-review/skills/prj-review/SKILL.md` | Router. Picks lane set from cadence preset, free-form trigger words, or explicit lane IDs. Owns the shared finding schema and output location convention. |
| Lane skills × 16 | `plugins/project-review/skills/prj-review-{lane}/SKILL.md` | One skill per review dimension. Each emits the common finding YAML record. |
| Cadence commands × 3 | `plugins/project-review/commands/prj-{after-task, after-feature, architecture}.md` | Slash entry points mapping each cadence to its lane subset. |
| Plan | `PLAN_dharness_integration.md` | 5-phase wiring plan for connecting this plugin to `dharness` (schema agreement, mirror skills, hooks, telemetry). |

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

Findings are persisted under:

```
dharness-project/dharness-rating/review/
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

## dharness integration

This plugin is **stand-alone usable** — no dharness required. Lanes degrade gracefully when `_workspace/_baseline/*` files are absent (they emit `info: [baseline-missing-{file}]` findings and proceed best-effort).

When dharness is present, the plugin auto-discovers:

- `_workspace/_baseline/intent_profile.md` → L01, L06, L13, L14 use it
- `_workspace/_baseline/project_profile.md` → L03, L11 use it
- `dharness-project/dharness-rating/` → all findings are persisted here

Full bi-directional wiring (auto-triggered lanes on dharness events, `/harness:harness-review` wrapper, telemetry-driven L15/L16) is staged in [`PLAN_dharness_integration.md`](./PLAN_dharness_integration.md) as a 5-phase rollout:

1. **Schema agreement** — extend `intent_profile` (`workflow.review`, `workflow.tdd`, `versioning`, `compliance.licenses`) + enumerate 13 baseline files + unify finding schema with `dharness-rating/`.
2. **dharness mirror** — new `/harness:harness-review` skill, doctrine patches to `harness`/`harness-baseline`/`harness-adapt`/`harness-audit`/`harness-status`.
3. **prj-review fixes** — apply the dogfood sample findings (caveman builder, ~7 file edits).
4. **Hooks opt-in** — `PostToolUse` / `Stop` / cron / gh-PR webhooks (default off).
5. **Telemetry tooling** — `invocation_counter`, `review_run_indexer`, `finding_aggregator` (PowerShell + Python).

---

## Status

- **v0.1.0** — Initial 16-lane + hub + 3-cadence drop. Stand-alone usable. dharness wiring stubbed only.
- Sample dogfood run on this repo itself: 20 findings (3 high — all resolved by the integration plan above).
- Next: Phase 1 of `PLAN_dharness_integration.md`, pending user agreement on schema items.

---

## License

[MIT](./LICENSE) — © 2026 gimangdoo
