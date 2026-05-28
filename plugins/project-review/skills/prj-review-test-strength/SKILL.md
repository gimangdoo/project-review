---
name: prj-review-test-strength
description: "테스트 커버리지 아닌 assertion 강도·격리·결정성·mutation 감지 평가. 'expect(x).toBeDefined()' 만 같은 약한 단언, happy-path만, flaky, 격리 안 됨 영역 finding. 트리거: '테스트 약함', 'mutation', 'flaky', 'happy path만', 'test quality'. should-NOT 트리거: 커버리지 % 단독 보고, 신규 테스트 작성 자체."
---

# Test Strength Lane (L13)

테스트가 **존재한다 ≠ 의미 있다**. 의미·격리·결정성을 평가.

## When to invoke
- feature-integration cadence
- 릴리스 전
- 회귀 발생 후 (재발 방지 평가)

## Inputs
- 테스트 디렉토리 (`__tests__`, `tests/`, `*_test.go`, `*.spec.ts`)
- 커버리지 리포트 (있으면 — 어느 line이 커버되는지보다 **무엇을 단언하는지**가 중점)
- CI flakiness 로그 (있으면)
- mutation testing 도구 (Stryker, mutmut 등) 가용 시 1회 실행

## Procedure
1. **assertion 강도 평가**
   - 단언이 `toBeDefined`·`toBeTruthy`·`not.toThrow` 만? → 약함
   - 정확한 값·구조 비교 있는가?
   - error message·side-effect 단언 있는가?
2. **happy-path bias 평가**
   - 각 함수의 테스트 → 입력 케이스 분류 (valid / boundary / invalid / failure)
   - failure path 단언 없는 함수 enum
3. **격리 평가**
   - 외부 시스템 호출 — mock 였는가, real이었는가, ICE인가
   - 테스트 간 순서 의존 (전역 state, DB cleanup 부재)
   - 시간/난수 의존
4. **결정성 평가**
   - sleep·timeout 의존
   - 환경 변수·OS·로케일 의존
   - CI에서만 실패 가능성
5. **mutation 감지** (가능 시)
   - mutation kill rate 측정
   - survived mutant 위치 → 단언 약점

## Severity 가이드
| 종류 | severity |
|----|----|
| critical path failure-test 부재 | high |
| toBeDefined 만으로 verify (10+ 곳) | medium |
| flaky 패턴 (sleep, 시간 의존) | high |
| 격리 미흡 (테스트 순서 의존) | medium |
| mutation kill rate < 50% | medium |
| 케이스 풍부, 단언 강함 | info |

## Finding 예시
```yaml
finding:
  id: L13-005
  lane: test-strength
  severity: high
  location: src/payment/__tests__/charge.test.ts:42
  problem: 결제 실패 path 미테스트 — 모든 케이스가 happy path
  evidence: |
    test('charge success', ...)         # 단언: 응답 객체 존재
    test('charge with amount=0', ...)   # 단언: not.toThrow
    (실패 path: card declined, network timeout, idempotency conflict 없음)
  why_it_matters: 결제 실패 회귀 시 테스트 통과 → 운영 노출
  fix:
    type: code
    suggestion: |
      describe('failure modes', ...) 추가:
      - 'card declined' → 401 + 에러 message
      - 'network timeout' → retry 후 실패 응답
      - 'idempotency conflict' → 동일 키 2회 → 1회 charge
  confidence: 90
  tags: [happy-path-bias, payment]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L13_test-strength.md`

frontmatter:
```yaml
---
lane: test-strength
tests_scanned: <n>
weak_assertions: <n>
missing_failure_paths: <n>
flaky_candidates: <n>
isolation_issues: <n>
mutation_kill_rate: <0-100|null>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.after_feature`, `workflow.review.pr_gate`
- baseline 참조: `intent_profile.quality.test_rigor` enum
- test_rigor=`high` 인데 weak assertion 다수 → intent drift 신호 → spec-drift lane 위임 sigil

[[prj-review]] [[prj-review-failure-mode]] [[prj-review-spec-drift]]
