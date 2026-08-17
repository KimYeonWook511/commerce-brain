---
type: moc
tags: [exception-handling]
updated: 2026-08-17
---

# exception-handling (MOC)

`tags: [exception-handling]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 21개.

## decisions

- [[auth-redis-장애-도메인예외-매핑-도메인전용-어드바이스]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[persistence-exception-노출-경계-추상수준]]
- [[pg-승인-예외-경계-요청전송시점]]
- [[redis-장애-멱등캐시-fallback-boolean-예외분리]]
- [[sql-translator-빈-제거-제약명-이중결제-식별]]
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]
- [[도메인-예외-httpstatus-제거-errorcategory]]
- [[무결성위반-도메인예외-번역을-제약이름으로-가른다]]
- [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]
- [[예약-동시소비-가드-version-vs-cas]]
- [[인프라-일시장애-503-예외-매핑-판단축]]
- [[재고복구-트랜잭션-맨뒤-배치와-비관락-경합-계측갈래]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]

## topics

- [[jwt-예외-catch-footgun-bare-securityexception]]
- [[서블릿-필터-예외-처리와-에러-디스패치]]

## knowledge

- [[find-first-write-not-check-db-unique-멱등]]
- [[방어-겹은-정합성겹과-최적화겹으로-가른다]]
- [[보상-catch-2차예외-평탄화-원칙]]
- [[예외타입-실측과-테스트가-명세를-넘어-단언하는-것]]
