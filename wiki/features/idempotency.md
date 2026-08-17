---
type: moc
tags: [idempotency]
updated: 2026-08-17
---

# idempotency (MOC)

`tags: [idempotency]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 32개.

## decisions

- [[cart-delete-미존재-4xx-entity-경유-삭제]]
- [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]
- [[payment-amount-mismatch-이중검증-409-vs-400-분리]]
- [[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]] `superseded`
- [[payment-완료여부-사실조회-hascompletedpayment-srp]]
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[paymentattempt-상태전이-도메인-검증-defensive]]
- [[pg-승인-예외-경계-요청전송시점]]
- [[redis-장애-멱등캐시-fallback-boolean-예외분리]]
- [[검증-전에-채우는-외부값에-유일제약-금지]]
- [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]]
- [[결과불명-재호출은-같은-멱등키로-새키-결론-뒤집힘]]
- [[결제-escalation-종착통지-escalatedAt-직교필드]] `superseded`
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]
- [[네이버페이-환불-이력-지목근거-보내기전-우리가-정하는-값]]
- [[동시도착은-선점층이-받고-처리중-전용응답]]
- [[멱등키-세-값-분리와-요청멱등키는-호출자가-발급]]
- [[부분취소-동시성-주문행-단일잠금과-캐시겹-미도입]]
- [[이중환불-최종방어선-잔액대조-도입]] `superseded`
- [[인프라-일시장애-503-예외-매핑-판단축]]
- [[잔액대조-옵션-미사용-공급자-권장의-전제-확인]]
- [[조회-또는-생성-메서드-해체-조회·생성·기존요청대조-셋]]
- [[주문-멱등-캐시-inflight-차단-전용]]
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]
- [[주문-멱등성-캐싱-after-commit-이벤트-분리]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]
- [[취소요청키-유일범위-주문단위-와-같은키-다른내용-거부]]

## tradeoffs

- [[payment-부분취소-모델만-열고-구현-보류]]

## topics

- [[order-도메인-구조-개요]]
- [[네이버페이-환불-api-실측-기록]]

## knowledge

- [[find-first-write-not-check-db-unique-멱등]]
- [[방어-겹은-정합성겹과-최적화겹으로-가른다]]
