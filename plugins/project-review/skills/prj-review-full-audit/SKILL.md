---
name: prj-review-full-audit
description: "전체 코드베이스 일괄 감사 — 중복, 미활용(dead code), 일관성 결여 패턴, 보안 의심 영역을 한 번에 스캔. lane 분리된 dep-health·coupling·waste-yield 와 달리 '광역 1회 점검' 용도. 트리거: '전체 감사', '코드베이스 전반', '일괄 점검', '전반 리뷰', 'codebase audit'. should-NOT 트리거: 단일 PR 리뷰, 특정 모듈만, 보안 단독 깊이(=/security-review)."
---

# Full Audit Lane (L03)

**광역·1회·중간 깊이**. 코드베이스 전체를 4개 축(중복·dead code·일관성·보안 의심)으로 스캔. 깊이 우선 영역은 전용 lane 위임:
- 결합도·circular → L11
- dep CVE·라이선스 → L12
- 죽은 feature ROI → L16

## When to invoke
- 분기·월간 정기
- 신규 인원 합류 직전 (현 상태 박제)
- 메이저 리팩터링 진입 전 baseline 박제

## When NOT
- 단일 task·PR 단위 리뷰
- 특정 깊이 분석 (해당 전용 lane 호출)

## Inputs
- 전체 코드 트리 (Glob: src/**, lib/**, app/** 등 프로젝트 root 자동 추정)
- `git ls-files` (없으면 Glob)
- 언어 감지 → 언어별 linter 결과 (가능 시)
- `_workspace/_baseline/architecture_profile.md` (있으면 모듈 enum 참조)

## Procedure (4축 병렬)
### A. 중복
1. 함수·클래스 시그니처 유사도 검색 (이름·매개변수·반환 일치)
2. 동일 string literal·magic number 3회 이상 등장
3. 동일 정규식·SQL fragment 중복
→ finding: 중복 위치 enum + 추출 권고

### B. dead code
1. export 됐으나 import 0회 (정적)
2. 정의됐으나 호출 0회 (private 한정 가능)
3. unreachable branch (`return` 후 코드, 항상 false 조건)
4. commented-out 블록 5줄 이상
→ finding: 위치 + 삭제 안전 신뢰도

### C. 일관성 결여
1. 네이밍 컨벤션 혼재 (camelCase vs snake_case 동일 의미)
2. 에러 처리 패턴 혼재 (throw / return null / Result type)
3. import 정렬·경로 alias 혼재
4. 동일 책임 다른 추상화 (e.g. 일부는 class, 일부는 함수)
→ finding: 컨벤션 후보 ≥2개 빈도 + 다수파 추천

### D. 보안 의심 (정적 패턴 한정, 깊이는 /security-review)
1. hardcoded secret 정규식 (api key, token, AWS_*)
2. SQL string concat (`"SELECT ... " + var`)
3. `eval`·`exec`·`Function()` 호출
4. http (https 아님) URL hardcode
5. `// TODO security` 또는 `// FIXME` 인접 민감 코드
→ finding: 위치 + 즉시 fix 가능 여부

## Severity 가이드
| 축·종류 | severity |
|----|----|
| hardcoded secret | critical |
| SQL injection 후보 | critical |
| dead code 비공개 함수 | low |
| dead code 공개 export | medium |
| 중복 3+회 동일 logic | medium |
| 컨벤션 혼재 다수파 < 60% | medium |
| 컨벤션 혼재 다수파 ≥ 60% | low |

## Finding 예시
```yaml
finding:
  id: L03-042
  lane: full-audit
  severity: critical
  location: src/db/query.ts:118
  problem: SQL injection 후보 — 사용자 입력 직접 concat
  evidence: |
    const q = "SELECT * FROM users WHERE name = '" + req.body.name + "'";
  why_it_matters: 무인증 임의 쿼리 실행 가능
  fix:
    type: code
    suggestion: prepared statement (`?` 바인딩) 또는 ORM 매개변수 전환
  confidence: 98
  tags: [security, sql-injection]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L03_full-audit.md`

frontmatter:
```yaml
---
lane: full-audit
axes: [duplication, dead-code, consistency, security-suspect]
files_scanned: <n>
findings_by_axis: {duplication: 8, dead-code: 12, consistency: 5, security: 2}
critical: 2
escalation_to_lane: [L11, L12, L16]   # 깊이 추적 필요 lane 추천
---
```

## 시간 제약 가이드
- 1k 파일 이하 → 풀 스캔
- 1k~10k → 디렉토리별 샘플링 + hot module 우선
- 10k+ → `architecture_profile.md`의 churn_hotspots 우선 + `[partial-scan]` sigil

## dharness 연동 (future)
- trigger: `workflow.review.weekly`, `workflow.review.architecture`
- baseline 박제: 1st run 시 산출물을 `_workspace/_baseline/audit_snapshot.md` 후보로 hub에 보고 (직접 박제 X)
- 다음 run에서 이전 snapshot과 diff → 회귀·개선 시그널 finding 추가

[[prj-review]] [[prj-review-coupling]] [[prj-review-dep-health]] [[prj-review-waste-yield]]
