---
name: prj-review-dx-onboarding
description: "신규 개발자·운영자가 N분 내 setup·실행 가능한지 평가. README 첫 5줄 실행 가능? 빠진 setup 단계? 모호한 에러? 트리거: 'onboarding', '신규 setup', 'DX', 'README 실행', 'getting started 검증'. should-NOT 트리거: 단순 문서 추가, 라이선스 텍스트 점검."
---

# DX / Onboarding Lane (L14)

신규 인원의 0→첫 commit 경로 평가. 문서 신뢰성·setup 자동화·에러 친화도 3축.

## When to invoke
- 분기 cadence
- 신규 인원 합류 직후 (사후 회고)
- onboarding 이슈 발생 시
- 메이저 toolchain 변경 후

## Inputs
- README.md, CONTRIBUTING.md, docs/setup
- setup 자동화 스크립트 (Makefile, install.sh, package.json scripts)
- 필수 env var 목록 / .env.example
- CI 설정 (CI에서 동일 setup 작동 = 가장 신뢰할 만한 사양)

## Procedure
1. **README 첫 5분 시나리오 시뮬레이션**
   - "clone → install → run" 명령 enum
   - 각 명령 실제 실행 가능 여부 (가능 시 sandbox에서)
   - 출력 메시지 친화도
2. **prereq 점검**
   - 필요 도구 버전 명시? (node v?, python v?, docker?)
   - OS·shell 의존성 명시?
3. **env / config 점검**
   - .env.example 존재?
   - 필수 키 누락 시 명확한 에러?
4. **에러 친화도**
   - 자주 발생할 setup 실패의 에러 메시지 — 다음 단계 안내 있는가?
   - "permission denied"·"port in use"·"missing X" 등
5. **CI vs 로컬 일치도**
   - CI가 사용하는 setup 단계와 README의 단계 일치하는가?

## Severity 가이드
| 종류 | severity |
|----|----|
| README 첫 명령 실행 불가 | critical |
| 필수 env var 미문서화 | high |
| 도구 버전 미명시 (호환 깨짐) | medium |
| 에러 메시지 모호 | medium |
| CI ↔ README 단계 불일치 | medium |
| 친절한 trouble-shooting 부재 | low |

## Finding 예시
```yaml
finding:
  id: L14-001
  lane: dx-onboarding
  severity: critical
  location: README.md:12 vs package.json scripts
  problem: README "npm run start"는 실행되지 않음 — package.json에 start 스크립트 부재
  evidence: |
    README: "npm install && npm run start"
    package.json scripts: { "dev": "...", "build": "..." } (start 없음)
  why_it_matters: 신규 인원 첫 명령 즉시 실패 → 첫인상·신뢰 손상
  fix:
    type: doc
    suggestion: README의 "npm run start" → "npm run dev" 치환, 또는 package.json에 "start" alias 추가
  confidence: 100
  tags: [readme, scripts]
```

## 출력 박제
`dharness-project/dharness-rating/review/{date}_{slug}/L14_dx-onboarding.md`

frontmatter:
```yaml
---
lane: dx-onboarding
setup_commands_checked: <n>
working: <n>
broken: <n>
env_vars_undocumented: <n>
ci_readme_mismatch: yes|no
estimated_first_run_minutes: <n|unknown>
---
```

## dharness 연동 (future)
- trigger: `workflow.review.quarterly`
- baseline 참조: 없음 (intrinsic 측정)
- critical finding → CLAUDE.md HOW 섹션 setup 단계 보강 sigil

[[prj-review]] [[prj-review-spec-drift]]
