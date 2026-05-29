---
name: prj-review-failure-mode
description: "각 분기·외부 호출·상태 변화의 실패 시나리오 + rollback·idempotency·부분 실패 복구 점검. adversarial(외부 공격)과 직교 — 본 lane은 '정상 운영 중 부분 고장'. 트리거: '실패 시 어떻게', 'rollback', 'idempotency', '부분 실패', '복구 경로', 'failure mode'. should-NOT 트리거: 단순 try-catch 추가, 성능 단독."
---

# Failure Mode Lane (L07)

정상 trafffic 하 부분 고장 가정 → 시스템이 살아남는가, 데이터는 일관한가, 운영자가 복구할 수 있는가.

## When to invoke
- feature-integration cadence
- 릴리스 전
- 외부 호출·DB·큐 관련 변경 직후
- on-incident cadence

## Inputs
- 코드의 외부 호출 지점 (HTTP, DB, 큐, 파일, IPC)
- 트랜잭션 경계 (DB transaction, 분산 lock, saga)
- 마이그레이션 스크립트
- 배포 매니페스트 (Dockerfile, k8s, terraform)

## Procedure
1. **고장 가설 enum**
   - 외부 호출 timeout
   - 외부 호출 5xx 후 자동 retry
   - DB connection drop 중간
   - 큐 deliver 중복
   - 부분 batch 실패
   - 디스크 full / 메모리 부족
   - 클럭 skew
   - 배포 중 traffic split
2. **각 가설 → 실제 코드 경로 trace**
   - retry 정책 있는가? backoff?
   - idempotency key 있는가?
   - 부분 실패 시 트랜잭션 rollback 되는가?
   - 사용자에게 명확한 에러 vs silent skip?
3. **rollback / forward 평가**
   - 마이그레이션 down 스크립트 존재 여부
   - 배포 롤백 가능 여부 (스키마 변경 등 forward-only 종속)
4. **finding emit**

## Severity 가이드
| 종류 | severity |
|----|----|
| 데이터 일관성 파괴 (no rollback, no idempotency) | critical |
| 부분 실패 silent (사용자·로그 모두 침묵) | high |
| retry without backoff → cascade | high |
| forward-only migration 없는 rollback | high |
| timeout 미지정 외부 호출 | medium |
| 명시적 에러 message 누락 | low |

## Finding 예시
```yaml
finding:
  id: L07-002
  lane: failure-mode
  severity: critical
  location: src/jobs/sync.ts:45-78
  problem: 외부 API 호출 retry 시 idempotency key 부재 → 중복 결제 가능
  evidence: |
    for (let i=0; i<3; i++) { await stripe.charges.create({amount,...}); ... }
    idempotency_key 옵션 미사용 + 재시도 사이 부분 성공 식별 불가
  why_it_matters: 네트워크 일시 장애 시 동일 결제 2~3회 → 환불 부담·신뢰 손상
  fix:
    type: code
    suggestion: idempotency_key=`charge:${userId}:${orderId}` 박제, retry 정책에 exponential backoff
  confidence: 95
  tags: [idempotency, retry, payment]
```

## 출력 박제
`{review_out}/{date}_{slug}/L07_failure-mode.md`

frontmatter:
```yaml
---
lane: failure-mode
hypotheses_run: <n>
traced: <n>
gaps_found: <n>
rollback_assessed: yes|no|partial
---
```

## dharness 연동 (future)
- trigger: `workflow.review.after_feature`, `workflow.review.on_incident`
- baseline 참조: `_workspace/_baseline/runbook.md`·`incident_log.md` (있으면 가설 우선순위)
- critical finding → `/harness:harness-evolve` 의 reliability doctrine 박제 sigil

[[prj-review]] [[prj-review-adversarial]] [[prj-review-observability]]
