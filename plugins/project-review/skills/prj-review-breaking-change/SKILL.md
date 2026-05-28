---
name: prj-review-breaking-change
description: "외부 API·스키마·CLI·env·설정의 semver 위반 변경 탐지. PR gate 권장. 트리거: 'breaking', 'semver', '하위 호환', 'API 변경', '스키마 변경'. should-NOT 트리거: 내부 함수 시그니처 변경, 미공개 export."
---

# Breaking Change Lane (L10)

외부 노출 surface의 비호환 변경 탐지 + 마이그레이션 가이드 누락 점검.

## When to invoke
- pr-gate cadence (병합 전 강제)
- 릴리스 전
- 메이저 의존 업그레이드 후

## Inputs
- `git diff` (PR 또는 base..HEAD)
- 공개 surface 정의: 명시적 `export`, OpenAPI/gRPC 스키마, DB 마이그레이션, CLI 정의, env var 목록, config 키
- CHANGELOG.md, MIGRATION.md (있으면)
- semver 정책 (package.json·intent_profile.versioning)

## Procedure
1. **public surface 매핑**
   - 패키지 export
   - REST/GraphQL/gRPC 스키마
   - DB 컬럼·인덱스·제약
   - CLI 명령·플래그
   - env var
   - config 스키마
2. **각 surface 별 변경 분류**
   - `add` (호환)
   - `rename` (breaking)
   - `remove` (breaking)
   - `type-narrow` (breaking)
   - `default-change` (잠재 breaking)
   - `behavior-change` (사양은 동일, 의미 변경 — 잠재 breaking)
3. **마이그레이션 자산 점검**
   - CHANGELOG 항목 존재?
   - codemod·migration 스크립트?
   - deprecation 단계 거쳤는가?
4. **버전 점프 권고**
   - breaking 검출 시 semver MAJOR 권고
   - additive only → MINOR

## Severity 가이드
| 종류 | severity |
|----|----|
| public API remove/rename + CHANGELOG 부재 | critical |
| DB forward-only migration | high |
| env var rename 없이 제거 | high |
| default 값 변경 (사용자 무인지) | high |
| 호환 add | info |
| deprecation 단계 거친 remove | low |

## Finding 예시
```yaml
finding:
  id: L10-001
  lane: breaking-change
  severity: critical
  location: src/cli/index.ts:34 vs CHANGELOG.md
  problem: --workspace 플래그 제거, --project로 rename, CHANGELOG·MIGRATION 모두 미박제
  evidence: |
    diff: -program.option('-w, --workspace ...') +program.option('-p, --project ...')
    CHANGELOG.md: 변경 없음
  why_it_matters: 기존 사용자 스크립트 전부 실패, MAJOR 점프 없이 PATCH 릴리스 시 신뢰 손상
  fix:
    type: doc + code
    suggestion: |
      1. --workspace alias 유지 + deprecation 경고
      2. CHANGELOG에 BREAKING 섹션
      3. version: MAJOR 점프
  confidence: 100
  tags: [cli, breaking, semver]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L10_breaking-change.md`

frontmatter:
```yaml
---
lane: breaking-change
surfaces_scanned: [package-export, rest-api, db, cli, env, config]
breaking_count: <n>
additive_count: <n>
changelog_gaps: <n>
recommended_bump: major|minor|patch
---
```

## dharness 연동 (future)
- trigger: `workflow.review.pr_gate`
- baseline 참조: `_workspace/_baseline/public_surface.md` (있으면 diff 정확도 ↑)
- critical finding 시 → PR merge block sigil (CI hook 후속)

[[prj-review]] [[prj-review-dep-health]] [[prj-review-spec-drift]]
