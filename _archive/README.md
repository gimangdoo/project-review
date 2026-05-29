# Archive

폐기된 plan·doctrine 보관. 현재 doctrine 아님 — 의사결정 이력 참고용.

## PLAN_dharness_integration.md

2026-05-29 폐기. project-review를 dharness와 별개 plugin 으로 유지 결정 → 5 Phase plan 중 Phase 3 (caveman builder 7건) 만 진행, 나머지 4 Phase = 영구 폐기.

폐기 근거:
- dharness = 메타(harness 박제), project-review = 도메인(review 박제) — layer 직교, 결합 ROI 낮음
- mirror skill 박제 = 책임 경계 역전 (하류가 상류 doctrine 변경)
- workflow.review enum, /harness:harness-review wrapper = 사용자가 cadence command 직접 호출로 대체 가능
- 16 lane 중 1 lane (L15) 만 dharness 의존 = ROI 6%

대체 doctrine: hub §4 산출물 경로 runtime resolve (env / .review-out / dharness co-location / default). dharness 의존 0.

상세 의사결정: `~/.claude/projects/C--Users-user01-awesome-files-project-review-project/memory/project_review_dharness_split_decision.md`
