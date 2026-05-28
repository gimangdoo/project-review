---
name: prj-review
description: "프로젝트 리뷰 메타 허브. 16개 lane(plan-adherence, adversarial, full-audit, hallucination, scope-creep, spec-drift, failure-mode, observability, perf-cost, breaking-change, coupling, dep-health, test-strength, dx-onboarding, harness-meta, waste-yield) 중 사용자 발화·cadence·인자에 맞춰 1개 이상 lane을 라우팅 실행. 산출물은 공통 finding 스키마로 통일되어 dharness-project/dharness-rating/review/{lane}/{date}.md에 박제. 트리거: '프로젝트 리뷰', '코드베이스 감사', '아키텍처 점검', '/prj-review', '/prj-after-task', '/prj-after-feature', '/prj-architecture', 'task 끝났으니 리뷰', '기능 통합 리뷰', '전체 점검'. should-NOT 트리거: 단일 diff/PR 라인 리뷰(=/code-review·/review 영역), 보안 단독(=/security-review), 빌드 검증(=/verify), 단일 함수 디버깅."
---

# Project Review Hub (`prj-review`)

프로젝트 단위 다축 리뷰의 단일 진입점. **자신은 분석하지 않는다** — lane 선택 + 컨텍스트 packing + 산출물 통합만 책임.

각 lane = 독립 skill (`prj-review-{lane}`). hub 미경유 직접 호출도 허용.

---

## 1. lane 카탈로그

| ID | lane skill | 책임 한 줄 |
|----|----|----|
| L01 | `prj-review-plan-adherence` | 사전 계획 대비 진행도, do/don't 박제 인식도 |
| L02 | `prj-review-adversarial` | 외부 공격자 관점, 시스템 파괴 논리 구성 |
| L03 | `prj-review-full-audit` | 중복·미활용·일관성 결여·보안 의심 일괄 |
| L04 | `prj-review-hallucination` | LLM 환각 산출물(가짜 import/API/시그니처) |
| L05 | `prj-review-scope-creep` | 미요청 추가 코드·추상화·플래그 |
| L06 | `prj-review-spec-drift` | 문서·주석 vs 코드 의미 불일치 |
| L07 | `prj-review-failure-mode` | 실패 시나리오·rollback·idempotency |
| L08 | `prj-review-observability` | 로그·메트릭·트레이스 사후 디버깅 가능성 |
| L09 | `prj-review-perf-cost` | hot path, N+1, 토큰/비용 회귀 |
| L10 | `prj-review-breaking-change` | 외부 API·스키마 semver 위반 |
| L11 | `prj-review-coupling` | 모듈 경계·결합도·layering 위반 |
| L12 | `prj-review-dep-health` | CVE·deprecated·라이선스·미사용 dep |
| L13 | `prj-review-test-strength` | 테스트 assertion 강도·격리·mutation 감지 |
| L14 | `prj-review-dx-onboarding` | 신규 인원 setup·README 첫 실행 |
| L15 | `prj-review-harness-meta` | dharness/agent/skill 정의 자체 정합 |
| L16 | `prj-review-waste-yield` | 만들었으나 안 쓰이는 산출물 ROI |

---

## 2. 라우팅 규칙

### 2.1 인자 직접 지정
```
/prj-review L01,L05,L15
/prj-review hallucination scope-creep
```
지정된 lane만 순차 실행.

### 2.2 cadence preset
```
/prj-review after-task         → L04, L05  (+ L01 if plan file present)
/prj-review after-feature      → L06, L07, L13  (+ L03 if >5 files changed)
/prj-review architecture       → L08, L11, L15  (+ L16 if monthly tick)
/prj-review pr-gate            → L10
/prj-review weekly             → L12
/prj-review monthly            → L16
/prj-review on-incident        → L02, L07, L08
```
별도 slash command(`/prj-after-task`, `/prj-after-feature`, `/prj-architecture`)도 동일 매핑.

### 2.3 자유 발화 → lane 추정
사용자 발화에 다음 키워드 발견 시 해당 lane 우선 활성:

| 발화 신호 | 활성 lane |
|----|----|
| "계획대로", "사전 계획" | L01 |
| "공격", "취약점 파괴", "redteam" | L02 |
| "전체 감사", "코드베이스 전반" | L03 |
| "AI가 만든", "환각", "가짜 import" | L04 |
| "범위 넘어", "필요 없는 코드" | L05 |
| "문서 안 맞", "주석 stale" | L06 |
| "실패 시", "rollback", "복구" | L07 |
| "디버깅", "로그 부족" | L08 |
| "느림", "비용", "토큰" | L09 |
| "breaking", "semver" | L10 |
| "결합도", "circular", "layering" | L11 |
| "CVE", "deprecated", "라이선스" | L12 |
| "테스트 약함", "mutation" | L13 |
| "onboarding", "신규 setup" | L14 |
| "harness", "agent 정의", "skill 충돌" | L15 |
| "안 쓰이는", "죽은 기능" | L16 |

발화 모호 시 → 사용자에 `AskUserQuestion` 으로 cadence 선택지 제공.

---

## 3. 공통 finding 스키마 (모든 lane 강제 준수)

각 lane은 0개 이상의 finding을 다음 YAML record 로 emit:

```yaml
finding:
  id: {lane-id}-{seq}              # e.g. L04-001
  lane: hallucination
  severity: critical|high|medium|low|info
  location: path/to/file.py:42-58  # 단일 line·범위·"global"·"missing"
  problem: <1줄 사실 진술>
  evidence: |
    <quote 또는 사실 인용>
  why_it_matters: <1줄 영향>
  fix:
    type: code|doc|process|harness|skip
    suggestion: <구체적 행동>
  confidence: 0-100                # adversarial·hallucination lane 필수
  tags: [list, of, tags]           # cross-lane 검색용
```

**severity rubric (공통):**
- `critical` = 데이터 손실·보안 침해·운영 중단 직결
- `high` = 디버깅 곤란·릴리스 차단·계약 위반
- `medium` = 유지보수 부담·일관성 결여
- `low` = 사소·미관·미래 잠재
- `info` = 관찰 메모, 행동 권고 없음

---

## 4. 산출물 박제 위치

```
dharness-project/dharness-rating/review/
├── {YYYY-MM-DD}_{run-slug}/
│   ├── _hub_summary.md            # hub가 작성
│   ├── L01_plan-adherence.md      # lane별 산출물
│   ├── L04_hallucination.md
│   └── ...
└── _index.md                      # 누적 인덱스 (hub append)
```

- `dharness-project/dharness-rating/` 디렉토리 미존재 시 → hub가 생성 + `.gitkeep` 박제
- `run-slug` = cadence 또는 lane 첫 글자 (e.g. `after-task`, `arch-L8L11L15`)
- `_hub_summary.md` 포맷:
  ```yaml
  ---
  run: {slug}
  date: {ISO}
  cadence: after-task|after-feature|architecture|custom
  lanes: [L01, L04]
  findings_total: 12
  by_severity: {critical: 1, high: 3, medium: 5, low: 3}
  ---
  ## TL;DR
  - {sev}: {count} — {lane-id} {one-liner}
  ## blocker (critical/high)
  - {finding.id}: {problem}
  ## next-action
  - {prioritized action list}
  ```

---

## 5. dharness 연동 contract (현재 stub, 추후 wiring)

### 5.1 trigger registry (예정)
```yaml
workflow.review:
  after_task: [L04, L05]
  after_feature: [L06, L07, L13]
  architecture: [L08, L11, L15]
  pr_gate: [L10]
  weekly: [L12]
  monthly: [L16]
  on_incident: [L02, L07, L08]
```
dharness `intent_profile.workflow.review` 필드에 위 매핑 박제 예정.

### 5.2 baseline 의존 (lane이 _workspace/_baseline/* 참조)
- L01: `intent_profile.md`, `roadmap.md` (없으면 skip + info 박제)
- L06: `intent_profile.md` vs README/CLAUDE.md
- L11: `architecture_profile.md` (module boundary)
- L15: `_workspace/_baseline/harness_state.md`, `MEMORY.md`

baseline 부재 시 → lane은 `severity: info` finding `[baseline-missing]` 박제 후 best-effort.

### 5.3 dharness Phase 매핑
| dharness phase | 권장 lane |
|----|----|
| Phase 6 (작업 직후) | L04, L05 |
| Phase 7 (feature 완성) | L06, L07, L13 |
| Phase 10 (관측 기반 적응) | L01, L09, L16 |
| `harness-audit` 보강 | L11, L15 |

추후 `/harness:harness-review` wrapper 가능 — 본 plugin lane을 dharness 내 자동 호출.

### 5.4 baseline 박제 변경 방지
review lane은 **읽기 전용 + dharness-rating 디렉토리 쓰기**만 허용.
- baseline·skill·agent 정의 직접 수정 금지
- 발견된 drift는 finding 으로만 emit, 자동 수정 X
- 수정은 `/harness:harness-evolve` 또는 사용자 결정 단계로 넘김

---

## 6. 실행 워크플로우

### Phase 0: 라우팅
1. 인자·cadence·자유 발화 분석 → lane set 확정
2. lane set 빈 경우 → `AskUserQuestion` 으로 cadence preset 선택

### Phase 1: 컨텍스트 수집
1. 현재 git status·branch·최근 commit 수집 (있을 시)
2. `_workspace/_baseline/*` 존재 여부 확인 → lane에 전달할 컨텍스트 빌드
3. `_workspace/_intent_profile/{latest}` 확인 → 활성 task 추정

### Phase 2: lane 실행
1. 선택된 lane을 **병렬 가능 시 병렬** 호출 (Agent tool 또는 Skill tool)
2. 각 lane은 자기 SKILL.md procedure 따라 finding emit
3. lane 실행 timeout: 기본 lane 당 10분, 사용자 조정 가능

### Phase 3: 산출물 통합
1. lane 산출물을 `dharness-project/dharness-rating/review/{date}_{slug}/` 디렉토리에 박제
2. `_hub_summary.md` 생성: 통계·blocker·next-action
3. `_index.md` append: `{date} | {slug} | {lanes} | {findings_total} | {top_severity}`
4. 사용자에 caveman 요약 출력 (전체 리스트 X, blocker·next-action·산출물 경로)

### Phase 4: 종료
- 출력 마지막 줄: 산출물 디렉토리 절대 경로
- dharness 연동 미설정 시: `[dharness-wire-pending]` sigil 박제하여 추후 자동 hook 진입 가능

---

## 7. 게이트·제약

- **read-only 강제**: lane은 코드·baseline·skill 정의 수정 금지. 위반 lane은 hub가 즉시 abort
- **사용자 confirm 불요**: cadence preset 또는 명시 인자 호출은 즉시 실행
- **모호 호출 confirm**: 자유 발화 + 키워드 매칭 0개 → AskUserQuestion 1회
- **중복 실행 방지**: 같은 cadence run을 1분 내 재호출 시 idempotent (기존 디렉토리 재사용 + diff append)
- **silent skip 금지**: lane 실행 실패 시 `_hub_summary.md` 에 `[lane-failed]` 박제

---

## 8. 호출 예시

```
# user
프로젝트 리뷰 한 번 돌려줘

# hub
→ AskUserQuestion: cadence?
  [after-task / after-feature / architecture / custom]

# user
architecture

# hub
→ lanes=[L08, L11, L15]
→ 병렬 실행
→ 산출물: dharness-project/dharness-rating/review/2026-05-28_architecture/
  _hub_summary.md
  L08_observability.md
  L11_coupling.md
  L15_harness-meta.md
→ 요약 caveman 출력:
  blocker: 2 critical (L11-003 circular import core↔auth, L15-001 dangling [[skill-x]] in MEMORY)
  next: L11 fix → architecture_profile.md baseline 갱신
  full: <path>
```

---

## 9. 버전·확장

- 신규 lane 추가 시: `plugins/project-review/skills/prj-review-{name}/` 생성 + 본 hub §1 카탈로그·§2.3 키워드 매핑·§5.1 trigger registry 갱신
- lane 책임 변경 시 본 hub §1 표 동기 — 사용자 검색은 hub 카탈로그가 ground truth
- dharness wiring 시 본 §5 contract를 dharness skill에 inverse-import

[[project_review_build]] [[project_methodology_advisor_build]]
