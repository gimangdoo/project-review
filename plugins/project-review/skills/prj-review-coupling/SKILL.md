---
name: prj-review-coupling
description: "모듈 경계·결합도·layering 위반·circular import·leaky abstraction 탐지. full-audit의 '일관성'과 다른 '구조 결합도' 축. 트리거: '결합도', 'circular', 'layering', 'leaky abstraction', '모듈 경계', '아키텍처 점검'. should-NOT 트리거: 단일 함수 리팩터링, 코드 스타일."
---

# Module Coupling Lane (L11)

코드 구조의 응집·결합 평가. 1회 분석으로 "이 모듈이 왜 저거를 알고 있나" 식 위반 박제.

## When to invoke
- architecture cadence (월간·분기)
- 메이저 리팩터링 직후
- 신규 모듈 추가 직후

## Inputs
- 전체 import 그래프 (정적 분석)
- `_workspace/_baseline/architecture_profile.md` (있으면 layer 정의)
- 모듈 트리 (디렉토리 구조)

## Procedure
1. **import 그래프 빌드**
   - 언어별 도구 우선 호출 — JS/TS: `madge` 또는 `dependency-cruiser`, Python: `pydeps`, Go: `go list -m all`, Rust: `cargo modules`
   - 도구 부재 fallback (필수):
     1. `which <tool>` 또는 `npx <tool> --version` 으로 가용 점검
     2. 미설치 시 grep 기반 import statement 정적 분석 — JS/TS: `^import .* from`, Python: `^(import|from) `, Go: `^import (`, Rust: `^use `
     3. 그래프는 `{module: [imported_modules]}` dict로 인메모리 빌드 (외부 binary 0 의존)
     4. finding frontmatter에 `tool_used: grep-fallback` 박제 — 정밀도 ↓ 명시
2. **circular detection**
   - cycle 검출 → 모든 cycle enum
3. **layer 위반 detection**
   - layer 정의 있는 경우: 하위 → 상위 import 금지 규칙
   - 정의 부재 시: 디렉토리 깊이 기반 휴리스틱 (e.g. `core/` 가 `ui/` 참조 시 의심)
4. **god module detection**
   - 한 파일이 N(>20) 모듈에서 import 됨 (god of import)
   - 한 파일이 N(>20) 모듈을 import (god of dependency)
5. **leaky abstraction detection**
   - 상위 모듈이 하위 모듈의 internal symbol 직접 사용
   - `_internal`·`private` 컨벤션 위반
6. **boundary smell detection**
   - 동일 도메인 개념이 N개 모듈에 다른 이름으로 박제
   - cross-cutting concern이 도메인 모듈에 박제 (logging in business logic, etc.)

## Severity 가이드
| 종류 | severity |
|----|----|
| circular import (런타임 영향) | high |
| layer 위반 (정의 명시) | high |
| god module (양방향) | high |
| leaky abstraction | medium |
| boundary smell | medium |
| 디렉토리 깊이 휴리스틱만 위반 | low |

## Finding 예시
```yaml
finding:
  id: L11-003
  lane: coupling
  severity: high
  location: src/core/db.ts ↔ src/auth/session.ts
  problem: circular import — core/db.ts → auth/session.ts → core/db.ts
  evidence: |
    core/db.ts:5  import {currentUser} from '../auth/session'
    auth/session.ts:3 import {db} from '../core/db'
  why_it_matters: 빌드 도구 변경 시 초기화 순서 깨짐, 단위 테스트 격리 불가
  fix:
    type: code
    suggestion: session에서 db 의존을 함수 인자로 inversion (DI) — db.ts는 session 모름
  confidence: 100
  tags: [circular, core, auth]
```

## 출력 박제
`{review_out}/{date}_{slug}/L11_coupling.md`

frontmatter:
```yaml
---
lane: coupling
modules: <n>
edges: <n>
cycles: <n>
layer_violations: <n>
god_modules: <n>
leaky: <n>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.architecture`
- baseline 참조: `architecture_profile.md` (layer 정의 박제처)
- 결과는 다음 baseline 갱신의 hot input → `/harness:harness-baseline` 후속

[[prj-review]] [[prj-review-full-audit]] [[prj-review-harness-meta]]
