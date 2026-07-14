---
type: moc
tags: [idempotency]
updated: 2026-07-14
---

# idempotency (MOC)

`tags: [idempotency]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 20개.

## decisions

- [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]]
- [[결제-escalation-종착통지-escalatedAt-직교필드]]
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]
- [[인프라-일시장애-503-예외-매핑-판단축]]
- [[주문-멱등-캐시-inflight-차단-전용]]
- [[주문-멱등성-캐싱-after-commit-이벤트-분리]]
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]
- [[cart-delete-미존재-4xx-entity-경유-삭제]]
- [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]
- [[payment-부분취소-모델만-열고-구현-보류]] _(open)_
- [[payment-완료여부-사실조회-hascompletedpayment-srp]]
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[payment-amount-mismatch-이중검증-409-vs-400-분리]]
- [[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]]
- [[paymentattempt-상태전이-도메인-검증-defensive]]
- [[pg-승인-예외-경계-요청전송시점]]
- [[redis-장애-멱등캐시-fallback-boolean-예외분리]]

## topics

- [[order-도메인-구조-개요]]

## knowledge

- [[find-first-write-not-check-db-unique-멱등]]
