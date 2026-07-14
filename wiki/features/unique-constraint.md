---
type: moc
tags: [unique-constraint]
updated: 2026-07-14
---

# unique-constraint (MOC)

`tags: [unique-constraint]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 10개.

## decisions

- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]
- [[대사-확정-검증보상-대칭-재승인없음]]
- [[이중결제보상-완료가드-제거-pgpaymentid-무조건취소]]
- [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]]
- [[multi-column-unique-length-명시-컨벤션]]
- [[payment-동시성-unique-vs-lock-gap-lock회피]]
- [[payment-부분취소-모델만-열고-구현-보류]] _(open)_
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[sql-translator-빈-제거-제약명-이중결제-식별]]
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]
