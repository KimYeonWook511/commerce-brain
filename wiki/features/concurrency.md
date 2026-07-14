---
type: moc
tags: [concurrency]
updated: 2026-07-14
---

# concurrency (MOC)

`tags: [concurrency]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 18개.

## decisions

- [[결제-escalation-종착통지-escalatedAt-직교필드]]
- [[보상판단-payment-존재-lock-대신-db-unique]]
- [[예약-동시소비-가드-version-vs-cas]]
- [[재고차감-동시성-비관락과-productid-정렬]]
- [[주문-멱등-캐시-inflight-차단-전용]]
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]
- [[cart-동시성-낙관락-processor-분리-retry]]
- [[order-concurrency-service-학습코드-격리]]
- [[order-version-낙관락-비관락-공존]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[payment-낙관적-락-도입-왜-비관-아님]]
- [[payment-동시성-unique-vs-lock-gap-lock회피]]
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]

## topics

- [[동시성-테스트-작성-규칙과-단언-전략]]
- [[order-도메인-구조-개요]]

## knowledge

- [[find-first-write-not-check-db-unique-멱등]]
