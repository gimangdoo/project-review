---
name: prj-review-scope-creep
description: "요청 범위를 넘어 추가된 코드·추상화·옵션·플래그·리팩터링 탐지. CLAUDE.md §3 surgical-changes 강제 lane. 매 task 직후. 트리거: '범위 넘어', '필요 없는 추가', 'scope creep', '왜 이것까지 건드렸나', '미요청 리팩터링'. should-NOT 트리거: 신규 기능 자체 리뷰, 단순 버그 fix 한 줄."
---

# Scope Creep Lane (L05)

원본 요청 범위 ↔ 실제 diff 비교. "이 변경 line 이 사용자 요청에 직접 trace 되는가?" 미충족 line을 finding.

## When to invoke
- after-task cadence 기본 lane
- task 종료 시 diff 큰 경우 (>50 LOC 또는 >5 파일)

## Inputs
- 사용자 원본 요청 (대화 컨텍스트 또는 task 메타)
- `git diff` (last task의 base..HEAD)
- CLAUDE.md §3 (surgical-changes 원문)

## Procedure
1. **요청 범위 enum** — 사용자 요청에서 요구된 행동 추출 ("X 추가", "Y 수정", "Z 삭제")
2. **diff line별 trace**
   - 각 변경 line이 요청 enum 중 어디에 trace 되는가?
   - trace 0 line → 후보
3. **후보 분류**
   - `incidental-fix` (요청과 무관한 버그 수정)
   - `unrequested-refactor` (스타일·구조 변경)
   - `speculative-abstraction` (단일 사용처 추상화)
   - `gratuitous-option` (요청 없는 플래그·설정)
   - `cleanup-of-others-mess` (요청 무관 dead code 삭제)
4. **finding emit**

## Severity 가이드
| 종류 | severity |
|----|----|
| speculative-abstraction (재사용 0) | medium |
| gratuitous-option | medium |
| unrequested-refactor (large) | high |
| unrequested-refactor (small, 동일 파일) | low |
| incidental-fix (별 PR 가치) | medium + split 권고 |
| cleanup-of-others-mess | medium (사전 합의 부재 시) |

## Finding 예시
```yaml
finding:
  id: L05-002
  lane: scope-creep
  severity: medium
  location: src/utils/format.ts (신규 파일)
  problem: 요청은 "login 버튼 색 변경", 실제 diff에 formatter helper 추출
  evidence: |
    user: "login 버튼 색을 brand-blue로 변경"
    diff: src/utils/format.ts (+45 lines, 단일 호출처 src/auth/login.tsx)
  why_it_matters: 단일 사용처 추상화 → 유지보수 부담, surgical 원칙 위반
  fix:
    type: code
    suggestion: format.ts 인라인 → login.tsx 직접 박제, 추후 2회+ 사용처 발생 시 추출
  confidence: 85
  tags: [scope-creep, premature-abstraction]
```

## 출력 박제
`{review_out}/{date}_{slug}/L05_scope-creep.md`

frontmatter:
```yaml
---
lane: scope-creep
request_summary: "<원본 요청 한 줄>"
diff_lines: <n>
on_scope_lines: <n>
off_scope_lines: <n>
categories: {abstraction: 2, refactor: 1, ...}
---
```

## dharness 연동 (future)
- trigger: `workflow.review.after_task`
- 누적 시: `feedback_*` 메모리 박제 추천 sigil → `/harness:harness-evolve`
- 해당 agent의 "기본 surgical 강제" doctrine 보강 신호

[[prj-review]] [[prj-review-plan-adherence]] [[prj-review-waste-yield]]
