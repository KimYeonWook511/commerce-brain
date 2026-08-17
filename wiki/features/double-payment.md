---
type: moc
tags: [double-payment]
updated: 2026-08-17
---

# double-payment (MOC)

`tags: [double-payment]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 16개.

## decisions

- [[payment-unknown-결과불명-처리와-예외분류]]
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[pg-승인-예외-경계-요청전송시점]]
- [[sql-translator-빈-제거-제약명-이중결제-식별]]
- [[결과불명-unknown-보존-alreadycomplete-cancel-경로확장]]
- [[결과불명-재호출은-같은-멱등키로-새키-결론-뒤집힘]]
- [[결제승인완료-보상-완료우선-이중결제-adapter매핑]]
- [[대사-확정-검증보상-대칭-재승인없음]] `superseded`
- [[배타점유-슬롯-미리잡기-vs-성공시-감지·되돌리기]]
- [[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]]
- [[이중결제보상-완료가드-제거-pgpaymentid-무조건취소]]
- [[이중환불-최종방어선-잔액대조-도입]] `superseded`
- [[잔액대조-옵션-미사용-공급자-권장의-전제-확인]]
- [[주문-이중결제-앞단-진입차단-예약조회-단일화]]

## tradeoffs

- [[부분취소-외부취소와-우리-기록-사이-대응관계-부재]]

## topics

- [[결제사-간편결제-구분과-세-층-역할-결과불명-재시도-모델]]
