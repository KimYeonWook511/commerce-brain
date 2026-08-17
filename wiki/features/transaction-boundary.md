---
type: moc
tags: [transaction-boundary]
updated: 2026-08-17
---

# transaction-boundary (MOC)

`tags: [transaction-boundary]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 20개.

## decisions

- [[cart-동시성-낙관락-processor-분리-retry]]
- [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[requires-new-격리-제거-보상판단-트랜잭션정책]]
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]
- [[돈정합성-우선-경계넘는-트랜잭션-셋-허용과-셀-수-있게]]
- [[무결성위반-도메인예외-번역을-제약이름으로-가른다]]
- [[보상판단-payment-존재-lock-대신-db-unique]]
- [[예약-동시소비-가드-version-vs-cas]]
- [[응용계층-서비스-분할-기준-다른-도메인까지-바꿀-때만]]
- [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]]
- [[재고복구-트랜잭션-맨뒤-배치와-비관락-경합-계측갈래]]
- [[주문-멱등-캐시-inflight-차단-전용]]
- [[주문-멱등성-캐싱-after-commit-이벤트-분리]]
- [[취소접수-트랜잭션경계-구현을-결정에-맞춘다-전파속성-잔재]]
- [[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]]
- [[회수배치가-주문상태를-묻지-않는다-확인대신-전제를-지킨다]]
- [[회원가입-not-supported-트랜잭션-분리]]

## knowledge

- [[애너테이션-잔재-판별과-전파변경의-osiv-파급]]
- [[회귀테스트는-판별력있는-단언과-도달지점을-함께-못박는다]]
