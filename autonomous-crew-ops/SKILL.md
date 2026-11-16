---
name: autonomous-crew-ops
description: Use when running kanban-crew tasks or extending herdr-lite.
version: 1.0.0
author: Dalek (Hermes)
license: MIT
metadata:
  hermes:
    tags: [crew, kanban, orchestration, automation]
    related_skills: [fleet-role-platform, kanban-workflows, spy-patch-ready-workflow, external-agent-connectors]
---

# Autonomous crew ops (herdr-lite) — 코드 스프린트 크루 운영

## When to Use

- kanban 카드를 자동 크루(spec→승인→build→검증→병합)로 돌리거나 그 상태를 볼 때
- crew 승인 루프가 멈춘 것처럼 보이거나(락·24h 보류) 루프 HALT를 만났을 때
- 크루의 risk 분류(TierA/B)가 잘못 분류됐거나 새 규칙을 추가해야 할 때
- 코드 스프린트 외 영역(트레이딩 게이트·인프라 생존·장중 대응)에 이 패턴을 끌어가려 할 때 — **범위 금지**, 설계 문서 §1 확인

Kanban 카드를 백로그로 받아 spec→승인→build→validate→merge로 돌리는 자율 크루.
구현체 `D:/fleet-hub/herdr_lite.py`(stdlib) + 상태 `herdr_state.json`. 설계 전문:
볼트 `1-Projects/2026-11-15 herdr-lite 설계 (A안).md`.

## 명령

- `herdr_lite.py` — dry: 백로그 열람 + Tier 분류 + crew 계획만 출력(부작용 0).
- `herdr_lite.py tick [--card t_id]` — 상태머신 1틱. **틱당 1 판정**이 원칙(부작용 최소).
  진행 순서: 활성 crew 진행 먼저 → 없으면 ready 카드의 TierB만 새 투입.
- `herdr_lite.py approve <card>` — 사람 승인 코멘트 기입(실전은 카드/telegram 답신이 정석,
  이 명령은 배선 시험용).
- `herdr_lite.py build-done <card> <sha>` — executor 세션의 커밋을 기록, validate 단계로 전진.
  동일 sha 연속 → 경고 2회 → **HALT**(사람 개입). 스킬이 아니라 사람이 상태를 고쳐야 풀린다.
- `herdr_lite.py merge <card>` — phase=validate에서만 병합. `hermes kanban complete` + 증적.

## 상태머신 규칙 (검증된 것)

1. **카드 코멘트가 진실의 원천이다.** 상태 파일은 캐시일 뿐 — 승인 여부는 `hermes kanban show`
   본문에서 `spec-approved` 유형 마커로 판정하고, 단계 전이는 역시 코멘트로 남긴다. 24h 무응답 =
   `hold`(보류), 사람 코멘트로만 재개.
2. **락은 새 투입만 막는다.** 활성 crew가 있으면 다음 백로그는 못 넣지만(동시 1건), 활성 카드의
   승인 점검·전이는 계속 진행해야 한다 — 진행까지 막으면 승인 루프가 영원히 안 풀린다(실측 버그).
3. **TierA는 자동 투입 금지.** 분류기 키워드 순서가 정의다: database(schema·마이그레이션·pg)
   → money(alpaca·orders·trades·주문) → trade(orb_/gate_/노브/파라미터) → access(.env·config·token·ssh)
   — database를 money 뒤에 두면 "schema migration"이 money로 오독한다(실측 5/6). TierA는 사람
   병합 고정, 카드에 근거·롤백 명령을 코멘트로 남겨 승인 창을 만든다.
4. **Executor는 patch-ready 규율을 따르는 에이전트 세션이다.** diff→게이트(py_compile / JSON 유효성 /
   대상 레포 테스트)→커밋→커밋 sha를 build-done으로 보고. 게이트 미통과는 validate 진행 없이
   카드 코멘트로 실패 사유를 남긴다.
5. **Validator는 실측이 우선이고 모델 심의는 보조다.** 테스트·빌드 실측이 판정 근거이고, 크로스 벤더
   심의는 fleet MCP 게이트(read-only)로 우리 자산을 읽게 한 뒤 의견만 받는다. 같은 벤더 모델이
   같은 계열 코드를 검증하면 눈이 같은 벽을 놓친다 — 벤더를 바꿔 대조하라.

## Kanban CLI 사실 (실측)

- 병합은 `hermes kanban complete <id> --result <증적> --summary <본문>` — `close` 하위명령은
  없고, 증적 없는 complete는 **거부된다**(빈 완료는 증거가 아니다 — 게이트가 강제한다; 무시하고
  성공 라벨만 붙였다가 show로 잡힌 실측 있음. 출력을 믿지 말고 show로 재확인).
- 코멘트는 `hermes kanban comment <id> <text>`(author 기본 default). 목록은 `list`, 단건은 `show`.
- 상태 진이상을 수습할 때는 상태 JSON을 직접 고치지 말고 **가능한 한 코멘트+명령 경로**로 유지하되,
  시험 사고(루프 투입) 같은 예외는 상태를 재기록하고 그 사실을 카드 코멘트로 남긴다 — 기록 없는
  상태 수술은 감사 사슬을 끊는다.

## 운영 상한

동시 crew 1건 / 시간당 1 기동 / 일 최대 6회 / 단일 crew 60분 캡. 백로그 0이면 tick은 부작용 없이
끝나야 한다(비용 억제의 핵심). crew 틱은 agent-run 크론(hermes cron)로 예약하고 실패 알림만 telegram으로:
`--deliver local --failure-deliver telegram`.