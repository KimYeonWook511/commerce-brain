---
type: moc
tags: [optimistic-lock]
updated: 2026-07-14
---

# optimistic-lock (MOC)

`tags: [optimistic-lock]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 9개.

## decisions

- [[예약-동시소비-가드-version-vs-cas]]
- [[재고차감-동시성-비관락과-productid-정렬]]
- [[주문만료-spring-batch-chunk-retry-skip]]
- [[cart-동시성-낙관락-processor-분리-retry]]
- [[cart-delete-미존재-4xx-entity-경유-삭제]]
- [[order-version-낙관락-비관락-공존]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[payment-낙관적-락-도입-왜-비관-아님]]
- [[persistence-exception-노출-경계-추상수준]]
