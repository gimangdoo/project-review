# project-review ↔ dharness Integration Plan (v0.1.0 → v0.2.0)

**작성일**: 2026-05-28
**입력**: `dharness-project/dharness-rating/review/2026-05-28_sample/` (dogfood 결과, finding 20건)
**목표**: project-review v0.1.0 의 dharness 연동 stub 을 실제 wiring 으로 전환

## 0. 현재 상태 (sample run 요약)

### 비대칭 contract (해소 대상)
| 항목 | project-review 측 | dharness 측 |
|----|----|----|
| `workflow.review.{cadence}` enum | hub §5.1 박제 | intent_profile schema 미정의 |
| baseline 파일 11개 참조 | lane Inputs § 박제 | schema 미정의 (intent_profile.md / project_profile.md 만 실재) |
| `/harness:harness-review` wrapper | 호출 약속 | skill 미박제 |
| finding 스키마 | hub §3 정의 | dharness-rating 기존 포맷 미합의 |
| Phase 매핑 (P6/P7/P10) | hub §5.3 박제 | doctrine 미박제 |
| invocation log read | L15·L16 가정 | telemetry 디렉토리는 존재 (`_workspace/_telemetry/`), 카운터 도구 부재 |

### blocker (high finding)
- L04-001 baseline 파일 schema 합의 (11 후보)
- L04-002 intent_profile 필드 확장 (5 후보)
- L04-003 `harness-review` skill 박제

---

## 1. Plan 구조 (5 Phase, 사용자 게이트 5회)

```
Phase 1 (schema 합의)         → 게이트 1 → Phase 2 (dharness mirror)
Phase 2 (dharness mirror)     → 게이트 2 → Phase 3 (prj-review 수정)
Phase 3 (prj-review 수정)     → 게이트 3 → Phase 4 (hooks opt-in)
Phase 4 (hooks opt-in)        → 게이트 4 → Phase 5 (telemetry tool)
Phase 5 (telemetry tool)      → 게이트 5 → 회귀 sample run
```

각 Phase 종료 시 사용자 confirm. Phase 간 순서는 dependency 강제 (역순 진입 금지).

---

## 2. Phase 1 — schema 합의 (doctrine-only, behavior change 0)

### 2.1 intent_profile schema 확장

**파일**: `dharness-project/dharness/_workspace/_baseline/intent_profile.md` 스키마 정의처 (dharness 코드 어딘가) + `intent_profile.md` 예시 박제

| 추가 필드 | 타입 | 값 | 출처 finding |
|----|----|----|----|
| `workflow.tdd` | bool | true/false | L04-002 |
| `workflow.review` | object | {after_task: [lanes], after_feature: [lanes], architecture: [lanes], pr_gate: [lanes], weekly: [lanes], monthly: [lanes], on_incident: [lanes]} | L15-005 |
| `versioning` | enum | `semver` / `calver` / `none` | L04-002 |
| `compliance.licenses` | array | 라이선스 allowlist (e.g. `["MIT","Apache-2.0","BSD-3-Clause"]`) | L04-002 |

기존 사용 필드 유지:
- `workflow.methodology` (enum, methodology-advisor가 박제)
- `quality.test_rigor` (enum)
- `constraints.{timeline,compliance,quality_target}`
- `permission_profiles`

### 2.2 baseline 파일 schema enum

dharness `_workspace/_baseline/` 스키마에 다음 파일 enum 정의 (실제 생성 의무 X — 존재 시 lane이 활용):

| 파일 | 역할 | 박제처 |
|----|----|----|
| `intent_profile.md` | (기존) 의도 프로파일 | dharness Phase 2 |
| `project_profile.md` | (기존) 코드 프로파일 | dharness Phase 1 |
| `architecture_profile.md` | 모듈/layer 정의 | Phase 3 또는 후속 baseline-update |
| `harness_state.md` | agent·skill·trigger 현황 | harness-status 산출 |
| `dep_manifest.md` | 의존성 요약 | Phase 1 보조 |
| `public_surface.md` | API·CLI·env·config enum | Phase 7 보조 |
| `threat_model.md` | 공격 surface·가설 | 별도 보안 워크플로 |
| `perf_baseline.md` | latency·cost 기준선 | Phase 10 입력 |
| `runbook.md` | 운영 절차 | 외부 박제, dharness read-only |
| `incident_log.md` | 인시던트 누적 | 외부 박제, dharness read-only |
| `observability_matrix.md` | service × signal 매트릭스 | 별도 워크플로 |
| `audit_snapshot.md` | full-audit 직전 산출물 | project-review L03 자동 박제 (Phase 4 wiring) |
| `compliance.md` | 라이선스·규제 정책 | Phase 2 보조 |

**합의 doctrine**: lane은 파일 부재 시 `info: [baseline-missing-{filename}]` finding 박제 후 best-effort 진행. dharness가 자동 생성 의무는 없음.

### 2.3 finding 스키마 통일

`dharness-project/dharness-rating/` 기존 포맷과 cross-check 필요.

**조사 항목**:
- 기존 dharness-rating finding record 포맷 read (ko01-10 batchA 결과 샘플)
- finding id 네이밍 (L{NN}-{seq} vs 기존 {checklist}-{seq})
- severity rubric (project-review 5 단계 vs 기존 dharness-rating 등급)

**합의 산출**: `dharness-project/dharness-rating/_schema/finding.yml` 박제 → 양쪽 SKILL.md 가 참조

### 2.4 Phase 매핑 doctrine

L04-004 finding 해소 — dharness Phase 번호 직접 매핑 제거, event 명 도입:

| event | trigger 원천 | review lane 호출 |
|----|----|----|
| `event.after_task` | dharness 단일 task 종료 | workflow.review.after_task |
| `event.after_feature` | dharness feature 통합 (Phase 6·7 산출) | workflow.review.after_feature |
| `event.architecture_tick` | weekly/monthly/quarterly cron | workflow.review.architecture |
| `event.pr_gate` | gh PR 생성 | workflow.review.pr_gate |
| `event.incident` | 인시던트 박제 | workflow.review.on_incident |
| `event.baseline_updated` | harness-baseline 실행 후 | workflow.review.architecture (subset) |

### Phase 1 산출물
- `dharness-project/dharness/.../intent_profile_schema.md` (확장된 스키마)
- `dharness-project/dharness/.../baseline_files_catalog.md` (13개 파일 enum)
- `dharness-project/dharness-rating/_schema/finding.yml` (통일 스키마)
- `dharness-project/dharness/_workspace/_plans/prj_review_integration_phase1.md` (doctrine patch)

### Phase 1 acceptance
- 사용자 confirm: 4개 산출물 review + 4개 필드 / 13개 파일 / finding 스키마 / event 명 enum 합의
- dharness intent_profile 기존 사용처에서 새 필드 누락 시 backward compat 확인 (모두 optional)

### Phase 1 risk
- intent_profile 사용처 광범위 — 신규 필드 enum 변경 시 기존 advisor·factory 영향
- finding 스키마 통일 시 기존 dharness-rating 산출물 마이그레이션 필요 (별도 검토)

---

## 3. Phase 2 — dharness mirror 박제

### 3.1 신규 skill `/harness:harness-review`

**위치**: `dharness/plugins/harness/skills/harness-review/SKILL.md`

```yaml
---
name: harness-review
description: "project-review plugin 의 cadence 또는 lane 을 dharness 워크플로 내에서 호출하는 wrapper. workflow.review.{cadence} enum 박제된 경우 자동 호출 가능. 트리거: 'review 돌려', 'cadence 리뷰', '/harness:harness-review', dharness event.{after_task|after_feature|...}."
---
```

**책임**:
1. intent_profile.workflow.review.{cadence} read → lane set 확정
2. `prj-review` skill (project-review plugin) 호출 — cadence 인자 전달
3. 산출물 위치 `dharness-project/dharness-rating/review/{date}_{slug}/` 박제 확인
4. 산출물 메타 (`_hub_summary.md` frontmatter) → `dharness/_workspace/_telemetry/review_runs.jsonl` 에 한 줄 append

**부재 시 동작**:
- project-review plugin 미설치 → user 메시지 ("plugin 설치 안내")
- intent_profile.workflow.review 미박제 → `cadence` 인자 강제 요구

### 3.2 기존 skill doctrine 패치

| skill | 패치 내용 |
|----|----|
| `harness` (factory) | Phase 6 (skill 박제) 후 → `event.after_feature` 자동 박제 doctrine. Phase 7 종료 후 → `event.after_feature` |
| `harness-baseline` | 갱신 완료 후 → `event.baseline_updated` 박제 |
| `harness-adapt` (Phase 10) | telemetry 기반 회귀 감지 시 review_runs.jsonl read → 최근 lane finding 통합 입력 |
| `harness-audit` | description should-NOT 트리거에 "결정적 점검(파일·링크·카운트)" 추가 — prj-review-harness-meta 와 책임 분리 명시 (L15-003) |
| `harness-status` | review_runs.jsonl 메타 추가 표시 (최근 cadence·blocker count) |

### 3.3 신규 command `/harness:cm-review-status`

- 최근 review run 메타 표시 (cm-status 의 review subset)
- blocker (high+) 미해결 enum

### Phase 2 산출물
- 신규 SKILL.md 1개 (`harness-review`)
- 기존 SKILL.md 4개 doctrine 패치
- 신규 command 1개
- `dharness/CHANGELOG.md` 항목

### Phase 2 acceptance
- 사용자 confirm: 신규 skill 박제 + 4 skill doctrine 차분 검토
- dry-run: `/harness:harness-review` 호출 → project-review plugin 라우팅 확인
- 기존 dharness 사용 시나리오 회귀 0

### Phase 2 risk
- 기존 skill 4개 description 변경 → 트리거 매칭 회귀 가능. cm-status 등 비교 검증 필수
- harness-adapt 가 telemetry read 추가 → 부재 시 silent fallback 강제

---

## 4. Phase 3 — project-review 수정 (sample finding 즉시 반영)

### 4.1 caveman builder 즉시 수정 (1-2 파일씩)

| finding | 파일 | 수정 |
|----|----|----|
| L04-009 | prj-review-harness-meta/SKILL.md | L15-001 example 재작성 (자기참조 제거) |
| L15-002 | prj-review/SKILL.md | description should-NOT 트리거에 "PR 리뷰", "diff 리뷰" 명시 추가 |
| L15-003 | prj-review-harness-meta/SKILL.md | §책임 경계 추가 (LLM 점검 vs 결정적 점검) |
| L15-006 | prj-review-spec-drift/SKILL.md + plan-adherence | 경계 doctrine 보강 |
| L04-005 | prj-review-perf-cost/SKILL.md | cache_control → LLM 일반화 |
| L04-006 | prj-review-dx-onboarding/SKILL.md | "sandbox" 구체화 |
| L04-008 | prj-review-coupling/SKILL.md | 외부 도구 fallback 절차 |

### 4.2 schema 합의 반영 (Phase 1 산출 import)

| 변경 위치 | 변경 |
|----|----|
| prj-review/SKILL.md §3 finding 스키마 | `dharness-project/dharness-rating/_schema/finding.yml` 외부 참조로 전환 |
| prj-review/SKILL.md §5.1 trigger registry | intent_profile.workflow.review enum 정의 위치 명시 |
| prj-review/SKILL.md §5.3 Phase 매핑 표 | Phase 번호 → event 명 치환 |
| 모든 lane Inputs § | baseline 파일 참조 시 Phase 1 catalog 링크 |

### 4.3 신규 skill description 보강

`prj-review` hub 의 dharness 연동 trigger 추가:
```
트리거: ..., '/harness:harness-review', dharness event.{after_task|after_feature|architecture_tick|pr_gate|incident|baseline_updated}.
```

### Phase 3 산출물
- prj-review-project 내 SKILL.md ~10개 수정
- `awesome-files/project-review-project/CHANGELOG.md` 신규 박제 (v0.1.0 → v0.2.0)
- 메모리 `project_review_build.md` 갱신 (Phase 1·2·3 완료 박제)

### Phase 3 acceptance
- 사용자 confirm: caveman builder diff 검토 7개 묶음
- 자기 dogfood 재실행 (sample) → L04 high 3건·L15 medium 3건 해소 확인

### Phase 3 risk
- description 변경 → 트리거 매칭 회귀 — 매 변경 전후 트리거 키워드 매트릭스 비교

---

## 5. Phase 4 — hooks opt-in (자동 호출)

### 5.1 settings.json hook 후보

**위치**: `~/.claude/settings.json` 또는 프로젝트 `.claude/settings.json`

| hook event | 매핑 cadence | 추천 default | rationale |
|----|----|----|----|
| `PostToolUse` (Edit/Write 후) | `after_task` | off (opt-in) | 자동 박제 부담 — 사용자 의식적 선택 |
| `Stop` (세션 종료) | `after_task` 또는 `after_feature` | off | 누적 변경 큰 경우만 |
| `UserPromptSubmit` (특정 키워드) | hub 키워드 매칭 | n/a | 기존 skill trigger 활용 |
| cron (외부) | `weekly` / `monthly` | off | OS 스케줄러 |
| gh PR webhook | `pr_gate` | off | gh CLI 통합 (별도 박제) |

### 5.2 opt-in UX

`/update-config` skill 활용 — 사용자가 명시 활성화 시에만 박제:
```
/update-config add-hook PostToolUse "/prj-after-task"
```

### 5.3 안전 게이트
- hook 자동 실행이 무한 루프 유발 가능 → review lane 자체는 hook 호출 금지 (read-only + dharness-rating 박제만)
- review run 직전 1분 내 동일 cadence 호출 시 idempotent (이미 hub §7 박제됨)

### Phase 4 산출물
- `awesome-files/project-review-project/docs/HOOKS.md` 박제 (opt-in 가이드)
- `dharness/CLAUDE.md` HOW 섹션 보강 (review hooks 안내)

### Phase 4 acceptance
- 사용자 confirm: opt-in 가이드 review
- 샘플 hook 활성화 → 1 task 후 자동 lane 실행 확인

### Phase 4 risk
- 자동 실행 부담 → 사용자 toxicity. **default off 엄수**

---

## 6. Phase 5 — telemetry tool (invocation log analyzer)

### 6.1 신규 스크립트 박제

**위치**: `awesome-files/project-review-project/scripts/`

| 스크립트 | 책임 | 입력 | 출력 |
|----|----|----|----|
| `invocation_counter.py` | Claude Code 로그 디렉토리 scan → skill·agent 별 invocation count 집계 | `~/.claude/projects/{slug}/*.jsonl` | `dharness/_workspace/_telemetry/invocation_counts.json` |
| `review_run_indexer.py` | dharness-rating/review/ scan → `_index.md` 재생성 | `dharness-project/dharness-rating/review/**/_hub_summary.md` | `_index.md` |
| `finding_aggregator.py` | 전 lane finding → 누적 트렌드 | `**/L*_*.md` | `dharness/_workspace/_telemetry/finding_trends.json` |

### 6.2 lane 연결

- L15 (harness-meta): `invocation_counts.json` read → "정의 vs 사용 0회" finding 자동 산출
- L16 (waste-yield): 동일 + 30/60/90일 window
- harness-adapt: `finding_trends.json` read → Phase 10 회귀 감지 입력

### 6.3 PowerShell 호환

dharness 환경 = Windows + PowerShell — Python 미설치 가능성. Node 또는 PowerShell 버전도 박제:
- `invocation_counter.ps1` (PowerShell 기본 호환)
- `invocation_counter.py` (선택, python 가용 시)

### Phase 5 산출물
- 3 스크립트 (PowerShell + Python 양쪽)
- L15·L16 SKILL.md "사용 가능 도구" 박제
- `dharness/_workspace/_telemetry/` 디렉토리 사용 doctrine 박제

### Phase 5 acceptance
- 스크립트 실행 → 출력 JSON validation
- L15·L16 재실행 시 invocation count 기반 finding 생성 확인

### Phase 5 risk
- Claude Code jsonl 포맷 변경 시 파서 회귀 — schema 버전 박제 필요
- 로그 권한·privacy — 사용자 환경 한정 (외부 전송 X 명시)

---

## 7. 회귀 검증 (Phase 1~5 종료 후)

### 7.1 sample dogfood 재실행
- target: `awesome-files/project-review-project/` (동일)
- lanes: L04 + L05 + L15
- 위치: `dharness-project/dharness-rating/review/2026-{new-date}_sample-v2/`

### 7.2 acceptance
| 항목 | 목표 |
|----|----|
| L04 high count | 0 (3 → 0) |
| L15 medium count (트리거 충돌) | 0 (2 → 0) |
| dharness contract 비대칭 | 0 (해소) |
| 산출물 스키마 통일 | 100% |
| invocation count gap | 0 (도구 박제) |

### 7.3 추가 sample
- target: `methodology-advisor-project/` (다른 plugin 으로 cross-validation)
- cadence: architecture
- lanes: L08, L11, L15 + 자동 추가

---

## 8. 비용·시간 추정

| Phase | 추정 시간 | 작업 분량 |
|----|----|----|
| Phase 1 (schema) | 2-3 시간 | 4 파일 박제 + 사용자 합의 |
| Phase 2 (dharness mirror) | 3-4 시간 | 1 신규 skill + 4 doctrine 패치 + 1 command |
| Phase 3 (prj-review 수정) | 1-2 시간 | caveman builder 7-10 invocation |
| Phase 4 (hooks) | 1 시간 | docs 박제 + 옵션 활성화 가이드 |
| Phase 5 (telemetry) | 3-5 시간 | 3 스크립트 + 양 언어 버전 + L15·L16 연결 |
| 회귀 sample | 30 분 | 자동 |
| **총합** | **10-15 시간** | 5 phase + 검증 |

병렬 실행 불가 — Phase 의존성 strict.

---

## 9. 사용자 결정 입력

본 plan 시작 전 필요:

1. **Phase 1 schema 합의**: intent_profile 신규 필드 4개·baseline 파일 enum 13개·event 명 6개 — 일괄 합의?
2. **finding 스키마 통일**: 기존 dharness-rating 포맷을 project-review 측에 맞출 것인가, 역으로?
3. **harness-review skill 박제**: dharness plugin 자체에 박제 (`dharness/plugins/harness/skills/harness-review/`)? 또는 project-review 측에 dharness adapter 박제?
4. **default off 정책**: Phase 4 hooks default off — 동의?
5. **telemetry 스크립트 언어**: PowerShell + Python 양쪽 박제 OK? 또는 Node?

---

## 10. 비진입 결정 시 fallback

dharness 패치 진입 보류 시 project-review v0.1.0 사용 시나리오:
- 단독 사용 가능 (모든 lane 작동, baseline 부재 → info finding)
- cadence command 직접 호출 (`/prj-after-task` 등)
- 산출물 박제 위치는 `dharness-project/dharness-rating/review/` 그대로 사용 (dharness 미설치도 가능, 빈 디렉토리 박제)
- L15·L16 = invocation log 부재로 부분 동작 (수동 enum 만)

**비용**: 기능 60-70% 가용. dharness 미통합으로 자동 호출·trigger·schema 합의 0.

---

## 11. 참조

- sample run: `dharness-project/dharness-rating/review/2026-05-28_sample/`
- 메모리: `project_review_build.md`, `project_methodology_advisor_build.md`, `project_dharness_progress.md`
- dharness root: `dharness-project/dharness/` (v0.11.0 시점)
- project-review root: `awesome-files/project-review-project/` (v0.1.0)
