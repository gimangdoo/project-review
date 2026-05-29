---
name: prj-after-task
description: 매 task 직후 즉시 리뷰 — hallucination(L04) + scope-creep(L05) lane 실행. plan 파일 존재 시 plan-adherence(L01) 추가.
---

# /prj-after-task

매 task 완료 직후 호출. **빠른 즉시 점검** 용도.

## 실행 lane
- L04 hallucination (필수)
- L05 scope-creep (필수)
- L01 plan-adherence (`_workspace/_baseline/intent_profile.md` 또는 `PLAN.md` 존재 시)

## 호출
`prj-review` skill을 다음 인자로 호출:
```
cadence=after-task
```

hub가 lane set 확정 → 병렬 실행 → `{review_out}/{date}_after-task/` 박제.

## 산출물 요약 우선순위
1. critical/high finding (즉시 fix 권고)
2. scope-creep 비율 (off-scope / total)
3. hallucination 건수 (0 이면 OK 박제)

## dharness 연동 (future)
- `workflow.review.after_task` trigger 매핑
- hooks: PostToolUse 또는 SubagentStop 후 자동 호출 후보 (사용자 opt-in)
