---
name: prj-after-feature
description: 새 기능 통합 직후 리뷰 — spec-drift(L06) + failure-mode(L07) + test-strength(L13). diff 큰 경우 full-audit(L03) 추가.
---

# /prj-after-feature

기능 단위 완성 직후. 일관성·버그·충돌·중복·실패 시나리오 통합 점검.

## 실행 lane
- L06 spec-drift (필수)
- L07 failure-mode (필수)
- L13 test-strength (필수)
- L03 full-audit (diff > 5 파일 또는 > 200 LOC)
- L09 perf-cost (외부 호출·DB·LLM 변경 시)
- L10 breaking-change (public surface 변경 감지 시)

## 호출
`prj-review` skill을 다음 인자로 호출:
```
cadence=after-feature
```

hub가 diff 분석 → lane set 조정.

## 산출물 요약 우선순위
1. critical breaking change (병합 차단 신호)
2. failure-mode gap (rollback·idempotency 미비)
3. test-strength weak path
4. spec drift (README/CLAUDE.md 후속 갱신 필요)

## dharness 연동 (future)
- `workflow.review.after_feature` trigger 매핑
- PR 생성 직후 자동 호출 후보 (gh CLI hook)
