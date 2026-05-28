---
name: prj-review-plan-adherence
description: "사전 계획 대비 진행도 + 작업 내부의 do/don't 박제 인식 여부 평가. plan 파일·roadmap·intent_profile vs 실제 코드/commit/branch 상태를 대조하여 (1)해야 할 것 누락 (2)하지 말아야 할 것 위반 (3)계획 외 추가 작업을 finding 으로 emit. 트리거: '계획대로 가는지', '사전 계획 준수', '진행도 점검', 'plan adherence', 'roadmap 진척'. should-NOT 트리거: 단일 함수 리뷰, 코드 품질 단독, 신규 plan 수립."
---

# Plan Adherence Lane (L01)

사전 계획·도덕(do/don't)을 작업이 얼마나 충실히 따랐는지 평가.

## When to invoke
- task 종료 직후 (`after-task` cadence 의 plan 파일 존재 시)
- feature 완성 직후 (`after-feature`)
- Phase 10 관측 적응 진입 전

## When NOT
- plan 파일·roadmap·intent_profile 전부 부재 → `info: [baseline-missing]` emit 후 종료
- 단일 라인 리뷰 (=/code-review 영역)

## Inputs
- `_workspace/_baseline/intent_profile.md` (필수)
- `_workspace/_baseline/roadmap.md` 또는 `PLAN.md` (있으면 우선)
- 활성 task 식별자 (없으면 최근 commit message N개)
- `git diff <baseline_commit>..HEAD` (있으면)
- CLAUDE.md (do/don't 명시 영역)

## Procedure
1. **do 목록 추출** — plan·roadmap·intent_profile에서 "해야 할" 항목 enum화
2. **don't 목록 추출** — CLAUDE.md §2·§3, intent_profile.constraints, 메모리 feedback_* 항목
3. **현 상태 매핑**
   - do 항목 → 코드/commit에서 증거 탐색
   - don't 항목 → 위반 패턴 탐색 (e.g. "no premature abstraction" → 추상화 추가 코드 grep)
4. **delta 분류**
   - `done` (계획 ∩ 실행)
   - `missing` (계획 - 실행)
   - `violated` (don't ∩ 실행)
   - `extra` (실행 - 계획) → scope-creep lane 위임 시그널
5. **finding emit** — missing/violated/extra 각각 record

## Severity 가이드 (lane 특화)
| 분류 | 기본 severity |
|----|----|
| critical do 누락 (e.g. 보안·결제 필수) | high |
| 일반 do 누락 | medium |
| don't 위반 (명시 금지) | high |
| extra (계획 외) | low → scope-creep 위임 |
| info | done 통계 1건 |

## Finding 예시
```yaml
finding:
  id: L01-003
  lane: plan-adherence
  severity: high
  location: src/auth/login.ts:1-200
  problem: intent_profile.workflow.tdd=true 박제됐으나 테스트 파일 없이 구현 commit
  evidence: |
    plan: "auth feature는 TDD 워크플로"
    git log: 2026-05-25 feat(auth): login impl (test file 없음)
  why_it_matters: 박제된 방법론 위반 → 회귀 시 안전망 부재
  fix:
    type: process
    suggestion: src/auth/__tests__/login.test.ts 선행 작성 → red → impl 재실행
  confidence: 90
  tags: [tdd, methodology, missing-test]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L01_plan-adherence.md`

상단 frontmatter:
```yaml
---
lane: plan-adherence
plan_source: <파일 경로>
do_items: <n>
dont_items: <n>
done: <n>
missing: <n>
violated: <n>
extra: <n>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.after_task`, `workflow.review.after_feature`
- baseline 참조: `intent_profile.md`, `roadmap.md`
- 산출물이 `intent_profile.workflow` 미일치 다수 발견 시 → `/harness:harness-adapt` 진입 추천 sigil 박제

[[prj-review]] [[prj-review-scope-creep]]
