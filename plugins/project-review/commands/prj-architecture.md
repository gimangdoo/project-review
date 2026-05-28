---
name: prj-architecture
description: 전체 아키텍처·일관성 전면 리뷰 — observability(L08) + coupling(L11) + harness-meta(L15). 월간 옵션은 waste-yield(L16) 추가.
---

# /prj-architecture

월간·분기 전면 점검. 광역 + 메타 lane 통합.

## 실행 lane
- L08 observability (필수)
- L11 coupling (필수)
- L15 harness-meta (필수, dharness 환경)
- L16 waste-yield (`--monthly` 또는 매월 1일)
- L03 full-audit (광역 한 번 더 — `--full` 옵션)
- L12 dep-health (`--with-deps`)

## 호출
`prj-review` skill을 다음 인자로 호출:
```
cadence=architecture [--monthly] [--full] [--with-deps]
```

## 산출물 요약 우선순위
1. coupling critical (circular, god module)
2. harness-meta critical (dangling, baseline drift)
3. observability critical (PII leak, 결제 path silent)
4. waste-yield (제거 후보 enum)

## dharness 연동 (future)
- `workflow.review.architecture` trigger 매핑
- `/harness:harness-baseline` 갱신 후 자동 후속 호출 후보
- 본 cadence 산출물은 다음 baseline 박제의 hot input
