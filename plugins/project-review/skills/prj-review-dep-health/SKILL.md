---
name: prj-review-dep-health
description: "외부 패키지 위생 — CVE, deprecated, 라이선스 부적합, version skew, 사용 안 하는 dep, transitive bloat 점검. 주간 cadence 권장. 트리거: 'CVE', '라이선스 점검', 'deprecated', '미사용 패키지', 'dep audit'. should-NOT 트리거: 신규 dep 선택 자체, 라이브러리 비교 추천."
---

# Dependency Health Lane (L12)

외부 의존의 위생 검진. 자체 코드의 보안 ≠ 외부 패키지의 보안.

## When to invoke
- 주간 cadence (스케줄 자동)
- dep 추가·업그레이드 PR
- 릴리스 전

## Inputs
- 매니페스트 (package.json/lock, requirements/lock, go.mod, Cargo.lock, etc.)
- 가능 시 SBOM 생성 도구 출력
- 라이선스 정책 (intent_profile.compliance.licenses 또는 LICENSE-POLICY.md)

## Procedure
1. **CVE 스캔**
   - `npm audit`, `pip-audit`, `cargo audit`, `govulncheck` 등 도구 호출
   - severity·fix 가용성 enum
2. **deprecated 패키지**
   - registry 메타 (npm) 또는 GitHub archived 표시
3. **라이선스 검사**
   - 라이선스 enum → 정책 허용 목록 매칭
   - GPL·AGPL 등 제한 라이선스 도입 여부
4. **version skew**
   - 동일 패키지 N개 버전 동시 박제 (transitive)
   - peer dependency 불일치
5. **미사용 dep**
   - 매니페스트엔 있으나 코드에서 import 0
6. **bloat**
   - 단일 사용처 위해 큰 패키지 도입 여부 (size 측정)

## Severity 가이드
| 종류 | severity |
|----|----|
| critical CVE + fix 가용 + 미적용 | critical |
| high CVE | high |
| AGPL/GPL 도입 (정책 거부) | critical (법적) |
| deprecated 핵심 dep | high |
| 미사용 dep | low |
| version skew (런타임 영향 없음) | low |
| bloat (영향 불명확) | info |

## Finding 예시
```yaml
finding:
  id: L12-002
  lane: dep-health
  severity: critical
  location: package.json: "lodash": "4.17.15"
  problem: CVE-2021-23337 (Prototype Pollution), high severity, fix 4.17.21
  evidence: |
    npm audit:
    lodash <4.17.21 — Prototype Pollution
    fix available: npm i lodash@4.17.21
  why_it_matters: 알려진 RCE/escalation 경로, fix 1줄
  fix:
    type: code
    suggestion: npm i lodash@^4.17.21 + lockfile 갱신 + regression test
  confidence: 100
  tags: [cve, lodash, prototype-pollution]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L12_dep-health.md`

frontmatter:
```yaml
---
lane: dep-health
total_deps: <n>
cves: {critical: 1, high: 2, medium: 5}
deprecated: <n>
license_conflicts: <n>
unused: <n>
version_skew: <n>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.weekly`, `workflow.review.pr_gate` (dep PR 한정)
- baseline 참조: `_workspace/_baseline/compliance.md` (라이선스 정책)
- critical CVE → 자동 SBOM 박제 + `/harness:harness-evolve` 보안 doctrine 강화 sigil

[[prj-review]] [[prj-review-breaking-change]] [[prj-review-full-audit]]
