---
name: prj-review-adversarial
description: "외부 공격자·악의적 사용자 관점에서 프로젝트를 파괴할 논리를 능동 구성하여 취약점·실패 시나리오 도출. 코드 옹호 금지, 제작자 관점 배제. 산출은 공격 시나리오 + 재현 단계 + 차단 권고. 트리거: '취약점 찾아', '공격 시나리오', 'redteam', '망가뜨릴 방법', '악의적 입력', '파괴 논리'. should-NOT 트리거: 단순 코드 스타일, OWASP 정적 스캔 단독(=/security-review), 단일 함수 unit test."
---

# Adversarial Lane (L02)

코드 옹호자(advocate) 모드를 강제 비활성. **공격자 페르소나로만 사고**. "이게 어떻게 망가질 수 있나" 만 질문.

## When to invoke
- 인시던트 사후
- 릴리스 전 stress check
- 외부 노출 surface (API, 파일 업로드, 사용자 입력, 권한 경계) 변경 직후

## When NOT
- 단순 코드 스타일 리뷰
- 정적 보안 스캔만 필요할 때 (`/security-review` 영역 — adversarial은 **논리 구성 능동**)

## 페르소나 강제
실행 중 다음 4개 페르소나 중 1개 이상 채택, 각 페르소나로 별도 finding 그룹 emit:

| 페르소나 | 동기 | 도구 |
|----|----|----|
| **external attacker** | 데이터 탈취·시스템 침해 | 입력 fuzz, 인증 우회, race, deserialization |
| **malicious user** | 권한 상승·과금 회피·spam | API rate-limit 우회, replay, IDOR |
| **disgruntled insider** | 흔적 없이 sabotage | 로깅 무력화, 백도어 commit, secret 유출 |
| **chaos monkey** | 의도 없는 장애 | 부분 실패, 의존 down, clock skew, disk full |

각 페르소나는 코드를 **부수기 위한** 시나리오 ≥3개 구성. "이건 안전" 결론 금지 — 안전하면 다음 페르소나로 진행.

## Procedure
1. **surface 매핑** — 외부 입력·권한 경계·신뢰 경계·상태 변환 지점 enum
2. **페르소나 라운드** — 각 페르소나 채택 → 시나리오 구성 → 코드에서 재현 경로 추적
3. **공격 가설 → 검증** — 가설마다 actual code path trace, 차단 코드 존재 여부 확인
4. **차단 부재 시 finding emit** — 재현 단계 명시
5. **차단 존재 시도 일부 박제** — `info` severity, "이미 차단됨" 박제 (커버리지 증거)

## Severity 가이드
| 결과 | severity |
|----|----|
| 데이터 탈취·RCE·인증 우회 재현 가능 | critical |
| 권한 상승·DoS 가능 | high |
| 정보 유출·부분 sabotage | medium |
| 이론상 가능하나 prereq 비현실 | low |
| 차단 검증 통과 (커버리지 박제) | info |

## Finding 예시
```yaml
finding:
  id: L02-007
  lane: adversarial
  severity: critical
  location: src/api/upload.ts:34-89
  problem: malicious user 페르소나 — multipart filename에 ../ 포함 시 임의 경로 쓰기 가능
  evidence: |
    재현:
    1. POST /upload, filename="../../etc/passwd"
    2. src/api/upload.ts:56 path.join(uploadDir, filename) 직접
    3. path traversal 차단 코드 부재
  why_it_matters: 임의 파일 덮어쓰기 → 권한 상승·서비스 corruption
  fix:
    type: code
    suggestion: filename = path.basename(filename) 또는 allowlist 정규식 강제, 추가로 chroot/sandbox 디렉토리
  confidence: 95
  tags: [path-traversal, upload, malicious-user]
```

## 출력 박제
`{review_out}/{date}_{slug}/L02_adversarial.md`

frontmatter:
```yaml
---
lane: adversarial
personas_run: [external, malicious, insider, chaos]
scenarios_total: <n>
breaches_found: <n>
critical: <n>
high: <n>
---
```

본문 구조: 페르소나별 섹션 → 시나리오 목록 → finding refs.

## dharness 연동 (future)
- trigger: `workflow.review.on_incident`, `workflow.review.pr_gate` (외부 surface 변경 PR)
- baseline 참조: `_workspace/_baseline/threat_model.md` (있으면) → 가설 우선순위
- critical finding 발견 시 → `/harness:harness-evolve` 로 hardening 우선순위 박제 시그널

## 안티-옹호 게이트
finding 작성 시 다음 문구 검출되면 abort + 재작성:
- "큰 문제는 아니지만"
- "이론상 가능하나"
- "공격자가 굳이"
- "현실적으로는"
근거 약화 표현 금지. 사실 + 재현 + 영향만.

[[prj-review]] [[prj-review-failure-mode]]
