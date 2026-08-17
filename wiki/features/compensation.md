---
type: moc
tags: [compensation]
updated: 2026-08-17
---

# compensation (MOC)

`tags: [compensation]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 15개.

## decisions

- [[payment-order-facade-결합끊기-tell-dont-ask]]
- [[payment-status-사실만-분류는-정책계산-manual-review-철회]]
- [[payment-낙관락-충돌처리-3계층-흡수는-트랜잭션-밖]]
- [[payment-완료여부-사실조회-hascompletedpayment-srp]]
- [[requires-new-격리-제거-보상판단-트랜잭션정책]]
- [[결제사건-테이블분리-기각과-유일제약-문자열-단일컬럼-교체]]
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]
- [[대사-확정-검증보상-대칭-재승인없음]]
- [[보상판단-payment-존재-lock-대신-db-unique]]
- [[보상흐름-설계-payment-application-책임-pgcanceller-dispatcher]]
- [[이중결제보상-완료가드-제거-pgpaymentid-무조건취소]]
- [[전용-종결상태-돈이-나갔다-돌아오는-중]]
- [[환불사유-목록값-두-경로-한-사건-회원노출-필터]]

## knowledge

- [[보상-catch-2차예외-평탄화-원칙]]
- [[종류·테이블-분리시-조용한-회귀와-전수조사-대상]]
