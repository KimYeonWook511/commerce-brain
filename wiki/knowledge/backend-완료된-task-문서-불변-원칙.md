---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [documentation, adr, task-docs, operations, immutability]
created: 2026-06-02
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-02-pr-192-payment-completion-fact-query]]"
  - "[[raw/sessions/backend/2026-06-05-pr-210-payment-naming-cleanup]]"
  - "[[raw/sessions/backend/2026-06-04-pr-204-unit-price-snapshot]]"
---

# 완료된 task 문서는 불변 — commerce-backend 문서 운영 원칙

**한 줄 정의.** commerce-backend 는 작업을 `docs/tasks/<task>/` 폴더 단위로 문서화하는데, **PR 머지 시점부터 그 task 폴더 문서는 불변**으로 얼리고, 이후 변경은 살아있는 루트 `docs/`·ADR 후속으로만 표현한다는 문서 운영 원칙.

## 배경 — 머지된 task 문서 사후 수정 의문

이 원칙은 결제 완료 판단 정리(#192) 세션에서 우연히 도출됐다. 이슈 #182 가 이미 머지가 끝난 두 task 의 architecture 문서를 갱신하라고 명시했고, 처음엔 그대로 갱신했다. 그러다 **이미 머지된 task 폴더 문서를 사후에 수정하는 게 맞나**라는 의문이 들면서 운영 원칙 논의로 번졌고, 결국 갱신을 되돌리고 원칙부터 세웠다.

## 다섯 갈래 원칙

"task 폴더 문서는 PR 머지 시점부터 불변"을 뼈대로 한 다섯 갈래다.

1. **완료된 task 폴더는 불변.** 머지된 task 의 문서는 그 시점 결정·의도의 기록으로 얼려 둔다.
2. **진행 중 task 폴더만 자유 수정.** 단 "진행 중"은 현재 worktree 에서 본인이 작업 대상으로 삼은 그 task 하나로 한정. 리뷰 반영·설계 변경·단계 갱신 등 머지 전 변경은 본문에 반영 OK. 다른 worktree/브랜치의 진행 중 task 나 이미 머지된 task 는 손대지 않는다.
3. **완료 후 변경은 루트 `docs/` 문서로 표현.** task 폴더가 아니라 살아있는 루트 정책 문서(예외 처리 전략·아키텍처·API 스펙·DB 스키마)와 ADR 후속 노트로.
4. **결정의 진화는 ADR 본문 갱신 또는 후속 노트.** task 폴더에 후속 노트를 흩뿌리지 않고, 결정 이력을 ADR 한 곳에서 추적할 수 있게 한다.
5. **새 정책 문서는 최소화.** "결정 1개 = 문서 1개"가 아니라 "영역 1개 = 문서 1개". 기존 문서에 흡수하기 어려울 때만 새로 만든다.

## 정본화 위치

이 원칙은 task 문서 운영 가이드(루트 `docs/tasks/README.md`)의 "완료된 tasks 불변 원칙" 섹션에 정본화하고, repo 루트 CLAUDE.md 에는 행동 지침 한 줄("머지된 task 폴더 문서는 이후 수정하지 않고, 이후 변경은 루트 docs 로만 표현")로 걸었다. 즉 정본은 코드 repo 문서 쪽에 있고, 이 wiki 노트는 그 원칙의 배경·사례를 모은 지식 노트다.

## 반복 적용 사례

이 원칙은 한 번 세운 뒤 여러 작업에서 반복 적용됐다 — 원칙이 실제로 살아 있다는 신호라 별도 knowledge 노트로 승격했다.

- **attempt 네이밍 정리(#210).** 옛 `PaymentAttempt` 식별자 잔재를 걷어낼 때, 역사 기록(레거시 단일 ADR 서술, 이미 머지된 task 폴더, flyway migration 파일 등)은 결정 당시 상태를 기록한 것이라 소급 수정하지 않고 보존했다. "역사 기록은 소급 수정 안 함"이 이 원칙의 직접 적용이다. → [[payment-attempt-네이밍-정리와-refactor-경계]]
- **단가 snapshot(#204).** 앞선 JPA 연관관계 분리 series 의 완료된 task 폴더 회고를 보강해 series 연계를 남기자는 안을 검토했다가, 완료된 task 문서 불변 원칙을 위반하므로 기각했다. series 연계 사실은 이번 회고와 루트 docs 에서만 표현하기로 했다. → [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]]

두 사례 모두 "완료된 것을 뒤늦게 고쳐 매끈하게 만들고 싶은 유혹"을 원칙으로 눌렀다는 공통점이 있다. 이는 commerce-brain 자체의 `raw/` 불변 정책(원본은 얼리고 정정은 새 raw 로만)과도 결이 같다 — 완료된 기록을 소급 수정하지 않고 후속으로만 진화시킨다는 메타 원칙이다.

## 관련 링크

- [[작업중-결정을-영구-adr로-승격하는-단위-개정사슬-접기]] — 이 원칙이 만든 관례("본문은 안 고치고 상태 줄에만 정정을 적는다")를 나중에 옮길 때 무슨 일이 생기나. **원본 안에서는 상태 줄을 먼저 읽는 규율로 버티지만 옮겨진 곳에서는 그 규율이 안 따라간다.**
- [[코드-주석에-문서-내부번호를-박지-않는다-재사용이-더-나쁘다]] — 완료된 task 폴더가 보관되면 코드만 남는다는 이 원칙의 귀결. 폴더 로컬 번호를 코드에 박으면 찾아갈 곳이 없거나 엉뚱한 결정에 도착한다.
- [[payment-완료여부-사실조회-hascompletedpayment-srp]] — 이 원칙이 도출된 #192 세션의 본 결정
- [[스코프-규율-한-pr-한-목적-인접부채-별도이슈-분리]] — "이번 PR 범위 밖은 별도 이슈로"와 같은 규율 결
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]] — series 전체를 관통한 메타 원칙·scope 규율
- [[문서-코드-정합성-개념정본-심볼최소화]] — 문서/코드 정본 배치 원칙

## 열린 질문

- **후속 노트의 "임계 졸업" 기준.** 한 결정의 후속 노트가 임계(대략 3~4개)에 이르면 별도 결정 문서로 졸업시킨다는 운영 판단이 있으나, 언제·어떻게 졸업시킬지는 아직 미확정. ([[payment-완료여부-사실조회-hascompletedpayment-srp]]의 미해결 항목과 동일 논의.)

## 근거

- [[raw/sessions/backend/2026-06-02-pr-192-payment-completion-fact-query]] — 원칙이 도출된 세션
- [[raw/sessions/backend/2026-06-05-pr-210-payment-naming-cleanup]] — 역사 기록 소급 수정 안 함(attempt 네이밍)
- [[raw/sessions/backend/2026-06-04-pr-204-unit-price-snapshot]] — 완료 task 폴더 회고 보강 기각
