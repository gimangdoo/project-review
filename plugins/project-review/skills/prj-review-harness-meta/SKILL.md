---
name: prj-review-harness-meta
description: "dharness/agent/skill 정의 자체의 정합성 점검 — dangling [[link]], stale memory, agent 정의 vs 실제 사용 불일치, skill trigger 중복, baseline drift. /harness:harness-audit 보강 lane. 트리거: 'harness 점검', 'skill 충돌', 'memory drift', 'baseline 신선도', 'dangling reference'. should-NOT 트리거: 신규 agent 추가, 신규 skill 작성."
---

# Harness Meta Lane (L15)

코드 아닌 **하네스 자체** 감사. dharness 사용 환경 특화. 기존 `/harness:harness-audit` 와 책임 분담:
- `/harness:harness-audit` = **LLM 점검** (의미·도덕 정합: 책임 중복·트리거 모호·통신 프로토콜 일관성)
- 본 lane = **결정적 점검** (파일 존재·링크 정합·invocation 카운트·timestamp 비교·grep 매칭)

두 lane은 **동시 실행이 권장 패턴** — `/harness:harness-audit` 가 의미 drift를, 본 lane이 사실 drift를 잡아 상호 보완. 충돌 시 본 lane 결과(결정적 사실)가 우선.

## When to invoke
- architecture cadence
- 주간
- `/harness:harness-baseline` 갱신 직후 (회귀 확인)
- `/harness:harness-add-skill`·`harness-add-agent`·`harness-merge`·`harness-split` 직후

## Inputs
- 하네스 root (`.claude-plugin/`, `plugins/`, `skills/`, `agents/`)
- 메모리 디렉토리 (MEMORY.md + linked .md)
- baseline 디렉토리 (`_workspace/_baseline/`)
- CLAUDE.md, AGENTS.md
- 활성 settings.json (hooks, permissions)

## Procedure
1. **dangling reference**
   - 메모리 `[[name]]` → 실제 파일 존재 여부
   - SKILL.md description 중 트리거에 다른 skill 명 참조 → 존재 여부
   - hub 카탈로그(`prj-review` 같은) lane 참조 → SKILL.md 존재
2. **stale 박제**
   - 메모리 frontmatter 일자 ↔ 마지막 수정 일자
   - baseline 박제 일자 vs 코드 churn (큰 churn 후 baseline 미갱신 → drift)
3. **중복 정의**
   - 동일 트리거 키워드 2개 이상 skill 박제
   - 동일 agent role 2개 이상 정의
4. **agent vs skill 정합**
   - agent description 명시 skill이 실제 존재?
   - skill 명시 agent가 실제 정의?
5. **사용 vs 정의**
   - 정의됐으나 호출 0회 agent·skill (logs 기반, 가용 시)
6. **CLAUDE.md HOW 섹션 정합**
   - baseline 명시 doctrine vs CLAUDE.md 박제 일치
7. **permission/hook drift**
   - settings.json의 permission이 baseline·intent_profile permission_profiles와 일치?

## Severity 가이드
| 종류 | severity |
|----|----|
| dangling [[link]] (메모리 핵심) | high |
| 동일 트리거 중복 skill | high |
| baseline 90일+ + 큰 churn | high |
| agent → 부재 skill 참조 | high |
| skill 정의 vs 호출 0회 (>30일) | medium |
| permission ↔ baseline 미일치 | medium |
| stale frontmatter 일자 (사실은 갱신됨) | low |

## Finding 예시
```yaml
finding:
  id: L15-001
  lane: harness-meta
  severity: high
  location: MEMORY.md L7 → [[old_project_notes]]
  problem: dangling [[link]] — memory/old_project_notes.md 파일 부재
  evidence: |
    MEMORY.md L7: "- [[old_project_notes]] — 초기 메모"
    ls memory/: old_project_notes.md 부재 (rename → current_project_notes.md 추정)
  why_it_matters: 메모리 recall 실패 → 컨텍스트 누락 → 잘못된 의사결정
  fix:
    type: harness
    suggestion: |
      1. memory/ 디렉토리에서 유사 파일명 grep
      2. MEMORY.md 링크를 실제 파일명으로 치환 (또는 파일 rename)
      3. /harness:harness-audit 후속 호출로 추가 dangling 확인
  confidence: 95
  tags: [dangling, memory]
```

## 출력 박제
`{review_out}/{date}_{slug}/L15_harness-meta.md`

frontmatter:
```yaml
---
lane: harness-meta
skills_total: <n>
agents_total: <n>
memory_files: <n>
dangling_refs: <n>
duplicate_triggers: <n>
stale_baselines: <n>
unused_definitions: <n>
permission_drift: <n>
---
```

## dharness 연동 (future, 본 lane 핵심)
- trigger: `workflow.review.weekly`, baseline 갱신 직후 자동
- baseline 참조: `_workspace/_baseline/harness_state.md`
- 결과는 다음 액션 sigil 박제:
  - dangling ↑ → `/harness:harness-audit`
  - 중복 ↑ → `/harness:harness-merge` 또는 `/harness:harness-split`
  - baseline stale → `/harness:harness-baseline`
  - permission drift → `/update-config`

[[prj-review]] [[prj-review-coupling]] [[prj-review-waste-yield]]
