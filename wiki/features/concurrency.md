---
type: moc
tags: [concurrency]
updated: 2026-08-17
---

# concurrency (MOC)

`tags: [concurrency]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 26개.

## decisions

- [[cart-동시성-낙관락-processor-분리-retry]]
- [[order-concurrency-service-학습코드-격리]]
- [[order-version-낙관락-비관락-공존]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[payment-낙관적-락-도입-왜-비관-아님]]
- [[payment-동시성-unique-vs-lock-gap-lock회피]]
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]
- [[결제-escalation-종착통지-escalatedAt-직교필드]] `superseded`
- [[동시도착은-선점층이-받고-처리중-전용응답]]
- [[배타점유-슬롯-미리잡기-vs-성공시-감지·되돌리기]]
- [[보상판단-payment-존재-lock-대신-db-unique]]
- [[부분취소-동시성-주문행-단일잠금과-캐시겹-미도입]]
- [[예약-동시소비-가드-version-vs-cas]]
- [[유일슬롯-비우고-같은값-재점유-쓰기순서와-메서드이름-신호]]
- [[자식-환불-자기-낙관락-부모버전은-불변식이-바뀔때만]]
- [[잔여전부취소-별도주소-미신설과-전액취소-명명-기각]]
- [[재고복구-트랜잭션-맨뒤-배치와-비관락-경합-계측갈래]]
- [[재고차감-동시성-비관락과-productid-정렬]]
- [[주문-멱등-캐시-inflight-차단-전용]]
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]

## topics

- [[order-도메인-구조-개요]]
- [[동시성-테스트-작성-규칙과-단언-전략]]

## knowledge

- [[find-first-write-not-check-db-unique-멱등]]
- [[방어-겹은-정합성겹과-최적화겹으로-가른다]]
