# project-review

**다중 레인 프로젝트 전반 리뷰**를 실행하는 Claude Code 플러그인 — 16개의 독립 리뷰 레인을 단일 허브가 라우팅하며, 표준화된 finding 스키마로 결과를 통일한다. **독립 실행형(stand-alone)** — 동반 플러그인 없이 어떤 저장소에서도 동작한다.

각 레인은 서로 다른 리뷰 축을 담당한다 (계획 준수, adversarial 레드팀, 전체 감사, LLM 환각, 범위 초과, 스펙 드리프트, 실패 모드, 관측 가능성, 성능/비용, 호환성 파괴, 결합도, 의존성 위생, 테스트 강도, DX, 하네스 메타, 낭비/수율). 레인은 직접 호출하거나 cadence 프리셋(`after-task` / `after-feature` / `architecture`)으로 호출한다.

[`dharness`](https://github.com/gimangdoo/dharness)와의 선택적 통합은 런타임에 자동 감지된다([출력](#출력) 참조) — 하드 의존성 없음.

> English README: [README.md](./README.md)

---

## 구성 요소

| 컴포넌트 | 경로 | 역할 |
|-----------|------|---------|
| 허브 스킬 | `plugins/project-review/skills/prj-review/SKILL.md` | 라우터. cadence 프리셋·자유 발화 트리거·명시 레인 ID로 레인 집합을 선택. 공통 finding 스키마와 출력 위치 규약을 소유. |
| 레인 스킬 × 16 | `plugins/project-review/skills/prj-review-{lane}/SKILL.md` | 리뷰 축당 스킬 1개. 각각 공통 finding YAML 레코드를 emit. |
| Cadence 커맨드 × 3 | `plugins/project-review/commands/prj-{after-task, after-feature, architecture}.md` | 각 cadence를 해당 레인 부분집합에 매핑하는 슬래시 진입점. |

---

## 레인 카탈로그

| ID | 레인 | 탐지 대상 |
|----|------|---------|
| L01 | `prj-review-plan-adherence` | 사전 계획 대비 진행도, 작업 내부 do/don't 인식 여부 |
| L02 | `prj-review-adversarial` | 시스템을 파괴하기 위해 능동 구성한 외부/공격자 논리 |
| L03 | `prj-review-full-audit` | 중복, dead code, 일관성 결여, 보안 의심 영역 (광역 1회 스캔) |
| L04 | `prj-review-hallucination` | LLM이 만든 가짜 import, 가짜 API, 잘못된 시그니처, placeholder |
| L05 | `prj-review-scope-creep` | 요청 범위를 넘어 추가된 코드 — CLAUDE.md §3 강제 |
| L06 | `prj-review-spec-drift` | README / CLAUDE.md / 주석 / docstring vs 실제 코드 |
| L07 | `prj-review-failure-mode` | 실패 경로, rollback, idempotency, 부분 실패 복구 |
| L08 | `prj-review-observability` | 사후 디버깅 가능성 — 로그, 메트릭, 트레이스, 에러 컨텍스트 |
| L09 | `prj-review-perf-cost` | N+1, hot-loop 알고리즘, 메모리 누수 후보, LLM 토큰 비용 회귀 |
| L10 | `prj-review-breaking-change` | 공개 API / 스키마 / CLI / env / 설정 semver 위반 |
| L11 | `prj-review-coupling` | 모듈 경계, circular import, layering 위반, leaky abstraction |
| L12 | `prj-review-dep-health` | CVE, deprecated, 라이선스 충돌, 미사용/transitive bloat |
| L13 | `prj-review-test-strength` | assertion 강도, 격리, 결정성, mutation 감지 (커버리지 아님) |
| L14 | `prj-review-dx-onboarding` | 첫 5분 setup, env var 문서화, 에러 친화성 |
| L15 | `prj-review-harness-meta` | dharness / agent / skill 정합성 — dangling link, stale baseline, 중복 trigger |
| L16 | `prj-review-waste-yield` | 거시 수준 미사용 산출물 — feature, flag, endpoint, agent, skill, branch |

---

## Cadence 프리셋

```
/prj-after-task         → L04 + L05 (plan 파일 존재 시 +L01)
/prj-after-feature      → L06 + L07 + L13 (조건부 +L03/L09/L10)
/prj-architecture       → L08 + L11 + L15 (선택 +L16/L03/L12)
```

허브에 인자로 전달 가능한 추가 cadence:

```
/prj-review pr-gate     → L10
/prj-review weekly      → L12
/prj-review monthly     → L16
/prj-review on-incident → L02 + L07 + L08
```

직접 레인 호출:

```
/prj-review L01,L05,L15
/prj-review hallucination scope-creep
```

---

## 출력

모든 레인은 공유 YAML 스키마로 finding을 emit한다:

```yaml
finding:
  id: L{NN}-{seq}
  lane: {lane-id}
  severity: critical | high | medium | low | info
  location: path/to/file.ext:line[-range]
  problem: <한 줄 사실 진술>
  evidence: |
    <인용 또는 사실 출처>
  why_it_matters: <한 줄 영향>
  fix:
    type: code | doc | process | harness | skip
    suggestion: <구체 조치>
  confidence: 0-100
  tags: [...]
```

finding은 `{review_out}/` 아래에 박제된다 — 허브가 런타임에 우선순위 순으로 해석:

| 우선순위 | 조건 | 해석된 `{review_out}` |
|----|----|----|
| 1 | `PRJ_REVIEW_OUT` env var 설정됨 | `$PRJ_REVIEW_OUT` |
| 2 | `<repo-root>/.review-out/` 디렉터리 존재 | `<repo-root>/.review-out` |
| 3 | `<repo-root>/dharness-project/dharness-rating/` 디렉터리 존재 (dharness 환경 자동 감지) | `dharness-project/dharness-rating/review` |
| 4 | 기본값 | `<cwd>/.review` |

```
{review_out}/
└── {YYYY-MM-DD}_{run-slug}/
    ├── _hub_summary.md          # TL;DR + blocker + next-action
    ├── L{NN}_{lane}.md          # 레인당 파일 1개
    └── ...
```

허브는 누적 추적을 위해 `_index.md`에 한 줄 항목도 append한다.

---

## 설치

이 저장소 자체가 [Claude Code 플러그인 마켓플레이스](https://docs.claude.com/en/docs/claude-code/plugin-marketplaces)다. 추가:

```
/plugin marketplace add gimangdoo/project-review
/plugin install project-review
```

또는 로컬에 clone 후 `./`를 마켓플레이스로 추가.

---

## dharness 통합 (선택, 자동 감지)

dharness와의 긴밀한 결합은 **명시적으로 범위 밖** — 이 플러그인은 독립 도구로 동작한다. dharness 통합은 다음으로 한정된다:

1. **출력 경로 공존** — `<repo-root>/dharness-project/dharness-rating/`가 존재하면 허브는 `.review/` 대신 `dharness-project/dharness-rating/review/` 아래에 finding을 기록한다. dharness 스킬·스키마는 호출하지 않는다.
2. **Baseline 읽기 (L15 레인 한정)** — `prj-review-harness-meta`가 존재 시 `<repo-root>/dharness/_workspace/_baseline/*`를 읽는다. baseline 부재 → `info: [baseline-missing-{file}]` finding + best-effort.

dharness 이벤트 기반 자동 트리거 레인, `/harness:harness-review` 래퍼, intent_profile 스키마 확장은 **추진하지 않는다** — cadence 커맨드(`/prj-after-task`, `/prj-after-feature`, `/prj-architecture`)가 cross-plugin 결합 없이 동일 표면을 커버한다.

---

## 상태

- **v0.2.0** — 독립 실행형 확정. 출력 경로 런타임 해석 (env / `.review-out/` / dharness 공존 / 기본 `.review/`). 샘플 dogfood finding (L04 + L15 패치) 적용.
- **v0.1.0** — 초기 16레인 + 허브 + 3-cadence 드롭.

---

## 라이선스

[MIT](./LICENSE) — © 2026 gimangdoo
