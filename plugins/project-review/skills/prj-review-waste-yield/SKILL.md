---
name: prj-review-waste-yield
description: "만들었으나 안 쓰이는 산출물 — feature, flag, endpoint, skill, agent, branch, file. full-audit '미활용'이 코드 라인 수준이면 이건 '노력 ROI' 수준. 월간 cadence 권장. 트리거: '안 쓰이는 기능', '죽은 feature', 'ROI 점검', 'waste audit', 'unused skill'. should-NOT 트리거: 코드 dead-line (=full-audit), 단일 변수 제거."
---

# Waste / Yield Lane (L16)

만들었으나 사용되지 않는 **거시 단위** 산출물 탐지. ROI 회수 의사결정 입력.

## When to invoke
- monthly cadence
- 분기 회고 직전
- 리팩터링 진입 전 (제거 대상 식별)

## Inputs
- feature flag 정의 + 사용 로그 (가용 시)
- endpoint 정의 + access log
- agent·skill 정의 + invocation log (`~/.claude/projects/.../*.jsonl` 분석)
- git branch 목록 + 마지막 commit 일자
- 파일·디렉토리 마지막 access 일자
- 사용자 분석 도구 (Mixpanel·Amplitude 등, 가용 시)

## Procedure
1. **feature flag 분석**
   - 정의 일자 vs 토글 변경 일자
   - 모든 사용자에게 `true` 인 flag → 제거 후보
   - 모든 사용자에게 `false` 인 flag → 제거 후보 (또는 잊혀진 실험)
2. **endpoint 분석**
   - 정의 vs 30일 access 0회
3. **agent / skill 사용도**
   - 정의 vs invocation log 0회 (30일)
   - 정의 vs invocation 1~2회 (낮은 yield)
4. **branch 분석**
   - 마지막 commit > 90일 + 미머지 brunch
5. **고아 파일**
   - import 0 + git access 0 (>60일)
6. **roadmap 항목 미수확**
   - roadmap에 "완료" 박제됐으나 실사용 신호 0

## Severity 가이드
| 종류 | severity |
|----|----|
| 사용자 노출 endpoint 30일 0회 | medium (정리 후보) |
| 정의 후 사용 0회 agent·skill (60일) | medium |
| feature flag 정리 후보 | low |
| 오래된 branch | low |
| 고아 파일 | low |
| 사용 0 + 유지보수 비용 큼 | high |

## Finding 예시
```yaml
finding:
  id: L16-002
  lane: waste-yield
  severity: medium
  location: plugins/project-review/skills/prj-review-foo/
  problem: skill prj-review-foo 정의 후 60일 invocation 0회
  evidence: |
    invocation log scan: 0 events for skill="prj-review-foo" since 2026-03-28
    SKILL.md last modified: 2026-03-25
  why_it_matters: 유지보수 부담만 발생, hub 카탈로그 노이즈
  fix:
    type: harness
    suggestion: |
      1. trigger 설계 미스인지 책임 부재인지 grilling
      2. 트리거 미스면 description 보강 (/harness:harness-evolve)
      3. 책임 부재면 /harness:harness-remove
  confidence: 80
  tags: [unused, skill]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L16_waste-yield.md`

frontmatter:
```yaml
---
lane: waste-yield
window_days: 30|60|90
flag_candidates: <n>
endpoint_candidates: <n>
unused_skills: <n>
unused_agents: <n>
stale_branches: <n>
orphan_files: <n>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.monthly`
- baseline 참조: invocation log 디렉토리 (Claude projects log)
- 결과는 의사결정 입력:
  - skill·agent unused → `/harness:harness-remove`
  - feature flag → 자체 정리 PR sigil
  - branch → 사용자 결정 (자동 삭제 금지 — destructive)

[[prj-review]] [[prj-review-harness-meta]] [[prj-review-full-audit]]
