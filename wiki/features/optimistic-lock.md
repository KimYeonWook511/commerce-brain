---
type: moc
tags: [optimistic-lock]
updated: 2026-08-17
---

# optimistic-lock (MOC)

`tags: [optimistic-lock]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 14개.

## decisions

- [[cart-delete-미존재-4xx-entity-경유-삭제]]
- [[cart-동시성-낙관락-processor-분리-retry]]
- [[order-version-낙관락-비관락-공존]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[payment-낙관적-락-도입-왜-비관-아님]]
- [[persistence-exception-노출-경계-추상수준]]
- [[결과회수-상한-폐지와-백오프-표-통지-반복]]
- [[예약-동시소비-가드-version-vs-cas]]
- [[외부-호출기록-aggregate-밖-낙관락-없는-쌓기전용]]
- [[자식-환불-자기-낙관락-부모버전은-불변식이-바뀔때만]]
- [[재고차감-동시성-비관락과-productid-정렬]]
- [[주문만료-spring-batch-chunk-retry-skip]]
- [[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]]

## knowledge

- [[애너테이션-잔재-판별과-전파변경의-osiv-파급]]
