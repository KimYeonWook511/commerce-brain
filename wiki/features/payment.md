---
type: moc
tags: [payment]
updated: 2026-07-14
---

# payment (MOC)

`tags: [payment]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 38개.

## decisions

- [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]]
- [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]]
- [[결제-후처리-대상식별-status중심-재설계]]
- [[결제-escalation-종착통지-escalatedAt-직교필드]]
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]
- [[대사-확정-검증보상-대칭-재승인없음]]
- [[대사-keep-waiting-backoff-next-reconcile-at]]
- [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]]
- [[미확정차단-대사스캔-정합성-starvation-escalation]]
- [[보상판단-payment-존재-lock-대신-db-unique]]
- [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]
- [[예약-동시소비-가드-version-vs-cas]]
- [[이중결제보상-완료가드-제거-pgpaymentid-무조건취소]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]
- [[paid-order-취소환불-단일tx-의도와-standalone-cancel-대사]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[payment-낙관적-락-도입-왜-비관-아님]]
- [[payment-동시성-unique-vs-lock-gap-lock회피]]
- [[payment-부분취소-모델만-열고-구현-보류]] _(open)_
- [[payment-완료여부-사실조회-hascompletedpayment-srp]]
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[payment-amount-mismatch-이중검증-409-vs-400-분리]]
- [[payment-append-only-원장과-exists-완료판단]]
- [[payment-attempt-네이밍-정리와-refactor-경계]]
- [[payment-order-도메인분리와-pg격리]]
- [[payment-order-트랜잭션-경계-cross-aggregate-단일tx]]
- [[payment-order-facade-결합끊기-tell-dont-ask]]
- [[payment-reserve-예약테이블-분리-a안-b안]]
- [[payment-reserve-ready-흐름-재설계-expiresat-재사용만료]]
- [[payment-status-사실만-분류는-정책계산-manual-review-철회]]
- [[payment-unknown-결과불명-처리와-예외분류]]
- [[paymentattempt-상태전이-도메인-검증-defensive]]
- [[paymentattempt-호출정책-문서-javadoc-archunit-보류]]
- [[pg-승인-예외-경계-요청전송시점]]
- [[requires-new-격리-제거-보상판단-트랜잭션정책]]
- [[sql-translator-빈-제거-제약명-이중결제-식별]]
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]

## topics

- [[payment-도메인-구조-개요]] _(draft)_
