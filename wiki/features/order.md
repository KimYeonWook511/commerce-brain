---
type: moc
tags: [order]
updated: 2026-08-17
---

# order (MOC)

`tags: [order]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 27개.

## decisions

- [[cart-도메인-골격-cartitem-단일-aggregate]]
- [[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]]
- [[order-concurrency-service-학습코드-격리]]
- [[order-version-낙관락-비관락-공존]]
- [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]]
- [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]
- [[payment-append-only-원장과-exists-완료판단]]
- [[payment-order-facade-결합끊기-tell-dont-ask]]
- [[payment-order-도메인분리와-pg격리]]
- [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]]
- [[payment-낙관적-락-도입-왜-비관-아님]]
- [[product-soft-delete-deletedat-주문이력-보존]]
- [[redis-장애-멱등캐시-fallback-boolean-예외분리]]
- [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]]
- [[돈정합성-우선-경계넘는-트랜잭션-셋-허용과-셀-수-있게]]
- [[부분취소-스코프-배송전-품목수량-기반-금액입력-기각]]
- [[부분취소-잔액-정본-수량기준-상태무관-누적]]
- [[재고복구-동기취소-vs-outbox-비동기만료-비대칭]]
- [[주문-금액-모델-도출·기록·스냅샷-3분류와-청구액-승인액-분리]]
- [[주문-멱등-캐시-inflight-차단-전용]]
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]
- [[주문-멱등성-캐싱-after-commit-이벤트-분리]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]
- [[주문만료-spring-batch-chunk-retry-skip]]
- [[취소접수-트랜잭션경계-구현을-결정에-맞춘다-전파속성-잔재]]
- [[회수배치가-주문상태를-묻지-않는다-확인대신-전제를-지킨다]]

## topics

- [[order-도메인-구조-개요]]
