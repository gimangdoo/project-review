---
name: prj-review-spec-drift
description: "README·CLAUDE.md·주석·docstring·티켓 본문 vs 실제 코드 의미 불일치 탐지. 문서가 코드를 따라잡았는지 평가(후향). plan-adherence는 코드가 계획 따라잡는지(전향) — 방향 반대. 트리거: '문서 stale', '주석 안 맞', 'README 거짓', 'doc drift', 'docstring 불일치'. should-NOT 트리거: 신규 문서 작성, 문법 점검."
---

# Spec/Doc Drift Lane (L06)

문서·주석·명세 vs 코드 사실 대조. 문서가 거짓말하면 신규 인원·LLM 둘 다 오류 유발.

## When to invoke
- feature-integration cadence 기본 lane
- 릴리스 전
- 메이저 시그니처 변경 직후

## Inputs
- README*, CHANGELOG*, docs/**
- CLAUDE.md, AGENTS.md
- 코드 내 docstring·블록 주석 (특히 함수 시그니처 위)
- intent_profile.md (있으면)
- 외부 API 문서 (있으면 WebFetch)

## Procedure
1. **문서 claim 추출**
   - "X 함수는 Y 반환" → 시그니처 대조
   - "config 키 Z 지원" → 실제 config 파싱 코드 grep
   - "다음 환경변수 필수: A, B" → 실제 코드 참조 vs 문서
   - "사용 예제: foo.bar()" → 실제 export 여부
2. **코드 사실 추출** — 동일 함수·config·env에서 실제 동작
3. **mismatch 분류**
   - `doc-says-but-code-doesnt` (문서는 약속, 코드는 없음)
   - `code-does-but-doc-doesnt` (코드는 하나, 문서는 침묵)
   - `stale-comment` (주석이 옛 동작 설명)
   - `example-broken` (사용 예제 실행 불가)
4. **finding emit**

## Severity 가이드
| 종류 | severity |
|----|----|
| 사용 예제 실행 불가 | high |
| 환경변수·config 문서 거짓 | high |
| 함수 시그니처 stale docstring | medium |
| README 일반 설명 stale | medium |
| 사소한 typo·구버전 버전 표기 | low |

## Finding 예시
```yaml
finding:
  id: L06-004
  lane: spec-drift
  severity: high
  location: README.md:88-95 vs src/cli/init.ts:12
  problem: README는 `--project` 플래그 문서화, 실제 CLI는 `--workspace`만 수용
  evidence: |
    README: "`init --project my-app` 으로 시작"
    src/cli/init.ts:12: program.option('-w, --workspace <name>', ...)
    (--project 핸들러 부재)
  why_it_matters: 신규 사용자 첫 명령 즉시 실패 → onboarding 차단
  fix:
    type: doc
    suggestion: README의 --project → --workspace 일괄 치환 + CHANGELOG에 rename 기록
  confidence: 100
  tags: [readme, cli, onboarding]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L06_spec-drift.md`

frontmatter:
```yaml
---
lane: spec-drift
docs_scanned: <n>
claims_checked: <n>
drifts_found: <n>
by_kind: {doc-says: 3, code-does: 2, stale-comment: 8, example-broken: 1}
---
```

## dharness 연동 (future)
- trigger: `workflow.review.after_feature`
- baseline 참조: `intent_profile.md` (의도 vs 코드 drift는 본 lane이 우선)
- harness CLAUDE.md HOW 섹션 drift 발견 시 → `/harness:harness-baseline` 재초안 sigil

[[prj-review]] [[prj-review-plan-adherence]] [[prj-review-dx-onboarding]]
