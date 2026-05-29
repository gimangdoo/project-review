---
name: prj-review-hallucination
description: "LLM 생성 코드의 환각 산출물 탐지 — 존재하지 않는 import, 가짜 API 호출, 잘못된 시그니처, copy-paste 환각 패턴, 죽은 분기. AI 코딩 환경 고유 리스크. 매 task 직후 권장. 트리거: '환각 점검', '가짜 import', 'AI가 만든 거짓 호출', 'LLM 산출물 검증', 'fake API'. should-NOT 트리거: 일반 런타임 디버깅, type-check 단독."
---

# Hallucination Lane (L04)

LLM이 자신 있게 만든 **존재하지 않는 것**을 사실 대조로 잡는다. linter·type-checker 가 잡는 항목과 겹치되, 의미론 환각까지 추적.

## When to invoke
- AI가 코드 작성 직후 (after-task cadence 의 기본 lane)
- 새 디펜던시 추가 직후
- vendor SDK·API 호출 작성 후

## Inputs
- `git diff` (마지막 commit 또는 HEAD vs base)
- `package.json`/`requirements.txt`/`go.mod` 등 의존 매니페스트
- 가능 시 의존 패키지의 실제 export 목록 (Bash로 `npm ls`, `pip show`, 또는 해당 모듈 Read)

## Procedure
1. **import 환각**
   - diff에서 새로 추가된 import line 수집
   - 매니페스트에 패키지 존재 여부 확인
   - 패키지 존재 시 → 해당 패키지에서 import한 심볼 실제 export 여부 확인
2. **API 호출 환각**
   - 외부 SDK 호출 (e.g. `stripe.charges.create_advanced(...)`) → 해당 메서드 실제 존재 여부
   - 시그니처 환각 — 매개변수 이름·순서·필수 여부 실제와 일치하는지
3. **인접 환각 패턴**
   - "유사 함수 이름" (e.g. `getUser` 호출했으나 실제는 `fetchUser`)
   - "다른 버전 API" (deprecated 시그니처)
   - 거짓 type/interface 참조
4. **죽은 분기 / placeholder**
   - `// TODO`, `pass`, `throw new Error("not implemented")` 남아있는 path
   - 호출되지 않는 helper (방금 생성 + 호출 0)

## Severity 가이드
| 종류 | severity |
|----|----|
| import 환각 (런타임 즉시 fail) | critical |
| API 시그니처 환각 (잘못된 매개변수) | high |
| 잘못된 메서드 이름 (typo 유사) | high |
| placeholder 남음 (호출 path 활성) | high |
| placeholder 남음 (호출 path 비활성) | medium |
| 사용 안 되는 helper | low |

## Finding 예시
```yaml
finding:
  id: L04-001
  lane: hallucination
  severity: critical
  location: src/payment/handler.ts:12
  problem: stripe SDK에 존재하지 않는 메서드 호출 — `stripe.charges.create_v2`
  evidence: |
    diff: + await stripe.charges.create_v2({...})
    실제 stripe 11.x export: create, retrieve, update, capture (create_v2 부재)
  why_it_matters: 호출 시 TypeError → 결제 path 전면 차단
  fix:
    type: code
    suggestion: stripe.charges.create({...}) + idempotency_key 옵션 활용
  confidence: 100
  tags: [hallucination, stripe, api-method]
```

## 출력 박제
`{review_out}/{date}_{slug}/L04_hallucination.md`

frontmatter:
```yaml
---
lane: hallucination
diff_base: <commit-sha>
imports_checked: <n>
api_calls_checked: <n>
hallucinations_found: <n>
critical: <n>
---
```

## 검증 도구
- 가능 시 Bash: `node -e "require('pkg')"` / `python -c "import pkg; print(dir(pkg.x))"`
- 패키지 docs WebFetch (최신 버전 시그니처 확인)
- type-check 명령 (`tsc --noEmit`, `mypy`, `go vet`)

## dharness 연동 (future)
- trigger: `workflow.review.after_task` (필수 lane)
- baseline 참조: `_workspace/_baseline/dep_manifest.md` (있으면 빠른 매핑)
- finding 누적 시 → 해당 SDK·API 영역의 prompt 보강 시그널 (`/harness:harness-evolve`)

[[prj-review]] [[prj-review-scope-creep]] [[prj-review-test-strength]]
