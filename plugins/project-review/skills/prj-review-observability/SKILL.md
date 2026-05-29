---
name: prj-review-observability
description: "사후 디버깅 가능성 평가 — 로그 누락 hot path, 식별 불가 에러, 컨텍스트 없는 throw, 메트릭/트레이스 빈 영역. full-audit '보안 의심'과 다른 '운영 가시성' 축. 트리거: '로그 부족', '디버깅 어려움', 'observability', '메트릭 빈 곳', '에러 식별 불가'. should-NOT 트리거: 로깅 라이브러리 선택 자체, 성능 단독."
---

# Observability Lane (L08)

장애 발생 후 N분 내 원인 식별 가능성 평가. 코드 + 인프라 + 알람 3축.

## When to invoke
- architecture cadence
- on-incident cadence (재발 방지 신호)
- 신규 서비스 추가 직후

## Inputs
- 코드 전체 (로깅·메트릭 호출 grep)
- 로깅 설정 (log level, format, sink)
- 메트릭 설정 (Prometheus, OpenTelemetry, etc.)
- runbook·dashboard 링크 (있으면)

## Procedure
1. **hot path enum**
   - 요청 진입점 (HTTP handler, queue consumer, cron)
   - 외부 호출
   - 데이터 변환 단계
   - 에러 핸들러
2. **각 hot path의 가시성 점검**
   - 진입·종료 로그?
   - 에러 시 context (user id, request id, trace id)?
   - 메트릭 (latency, count, error rate)?
   - trace span 생성?
3. **에러 throw 지점 분석**
   - `throw new Error("failed")` 같은 정보 없음
   - try-catch 후 silent (log 없이 return null)
   - error message에 dynamic context 부재
4. **로그 노이즈 평가**
   - 의미 없는 debug 로그가 prod log를 채움
   - sensitive 정보 (PII, token) 로그 박제

## Severity 가이드
| 종류 | severity |
|----|----|
| 사용자 영향 path 에러 silent | critical |
| trace/correlation id 부재 | high |
| 결제·인증 등 critical 영역 메트릭 0 | high |
| 일반 hot path 로그 부재 | medium |
| 노이즈 로그 (의미 없는 debug prod 박제) | medium |
| 로그 박제 PII | critical |

## Finding 예시
```yaml
finding:
  id: L08-001
  lane: observability
  severity: high
  location: src/api/checkout.ts:88-110
  problem: 결제 path try-catch 후 console.log("failed") 만 — request id·user id·error message 누락
  evidence: |
    } catch (e) {
      console.log("failed");
      return res.status(500).send();
    }
  why_it_matters: 결제 실패 발생 시 어느 사용자·주문·원인인지 식별 불가 → 인시던트 대응 N시간 지연
  fix:
    type: code
    suggestion: structured logger 사용 (logger.error({reqId, userId, orderId, err: e.message, stack: e.stack}))
  confidence: 95
  tags: [logging, error-handling, payment]
```

## 출력 박제
`{review_out}/{date}_{slug}/L08_observability.md`

frontmatter:
```yaml
---
lane: observability
hot_paths_enum: <n>
covered_logs: <n>
covered_metrics: <n>
covered_traces: <n>
gaps: <n>
pii_leak_candidates: <n>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.architecture`, `workflow.review.on_incident`
- baseline 참조: `_workspace/_baseline/observability_matrix.md` (있으면) — service x signal matrix
- 누적 gap 시 → reliability doctrine 강화 sigil

[[prj-review]] [[prj-review-failure-mode]] [[prj-review-perf-cost]]
