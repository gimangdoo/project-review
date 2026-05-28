---
name: prj-review-perf-cost
description: "성능·비용 회귀 탐지 — N+1, hot loop 알고리즘, 메모리 누수 후보, LLM 토큰/비용 폭증 프롬프트, CI 빌드시간 증가. 정합·보안과 직교. 측정 기반. 트리거: '느림', 'N+1', '토큰 비용', 'perf regression', '빌드 느림', 'cost spike'. should-NOT 트리거: 마이크로 최적화 단독, 벤치마크 작성 자체."
---

# Perf / Cost Lane (L09)

회귀 우선. 절대 성능 튜닝 아님. **이번 변경이 이전보다 나빠진 지점**을 찾는다.

## When to invoke
- feature-integration cadence
- 주간
- Phase 10 관측 적응

## Inputs
- `git diff` (이번 사이클 변경)
- 가능 시 벤치 결과 (before/after)
- LLM 호출 코드 (prompt 길이·model 선택·캐시 사용 여부)
- 빌드 시간 (CI 메타 또는 로컬 시간 측정)
- DB 쿼리 (ORM 사용 시 lazy load 패턴 grep)

## Procedure
1. **N+1 / loop 내 IO**
   - for/forEach 내부에 await (DB·HTTP)
   - ORM relation lazy load 패턴
2. **알고리즘 회귀**
   - O(n²) 새 도입 (nested loop on 사용자 데이터)
   - 큰 컬렉션 sort/copy 반복
3. **메모리**
   - 무한 캐시 (Map without eviction)
   - 클로저로 큰 객체 보유
   - stream 미사용 대용량 read
4. **LLM 비용**
   - prompt 길이 증가 (system + user)
   - cache_control 부재
   - 작업 size 대비 over-spec model 선택
   - retry/loop 시 비용 증폭
5. **빌드/CI**
   - 새 dep 추가의 cold install 시간
   - 새 step의 평균 시간

## Severity 가이드
| 종류 | severity |
|----|----|
| 사용자 요청 path latency 2x | critical |
| 메모리 누수 후보 (장기 leak) | high |
| LLM 비용 5x 증가 prompt 도입 | high |
| 빌드 시간 30%+ 증가 | medium |
| 마이크로 비효율 (영향 < 5%) | low |

## Finding 예시
```yaml
finding:
  id: L09-003
  lane: perf-cost
  severity: critical
  location: src/api/list.ts:23-31
  problem: N+1 — for loop 내 user.posts() ORM lazy load
  evidence: |
    for (const u of users) {
      u.posts = await db.post.findMany({where: {userId: u.id}});
    }
  why_it_matters: users 1k 시 1001 query → 응답 3s 초과
  fix:
    type: code
    suggestion: db.user.findMany({include: {posts: true}}) 단일 쿼리 또는 prisma groupBy 전환
  confidence: 100
  tags: [n+1, orm, latency]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L09_perf-cost.md`

frontmatter:
```yaml
---
lane: perf-cost
diff_base: <commit>
n_plus_one_found: <n>
memory_candidates: <n>
llm_cost_deltas: <n>
build_delta_seconds: <n|null>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.after_feature`, `workflow.review.weekly`, Phase 10
- baseline 참조: `_workspace/_baseline/perf_baseline.md` (latency·cost 기준선)
- 회귀 누적 시 → Phase 10 `harness-adapt` 진입 sigil + perf doctrine 강화

[[prj-review]] [[prj-review-observability]] [[prj-review-failure-mode]]
