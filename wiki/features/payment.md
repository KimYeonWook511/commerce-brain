---
type: moc
tags: [payment]
updated: 2026-08-17
---

# payment (MOC)

`tags: [payment]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 80개 — 평탄 나열이 목차 기능을 못 하는 규모라 **하위 축 인덱스**로 재구성했다. 각 축은 그 태그의 MOC 가 정본 목록을 갖는다.

개념 지도(현재 구조)는 [[payment-도메인-현재-구조-2026-08]], 2026-05 스냅샷은 [[payment-도메인-구조-개요]].

## 하위 축

- **환불·부분취소** — [[refund]] (31) · [[partial-cancel]] (19)
- **멱등·유일제약** — [[idempotency]] (23) · [[unique-constraint]] (20)
- **동시성·락** — [[concurrency]] (16) · [[optimistic-lock]] (7) · [[pessimistic-lock]] (5) · [[transaction-boundary]] (13)
- **회수·에스컬레이션·결과불명** — [[reconciliation]] (25) · [[escalation]] (12) · [[unknown-status]] (11)
- **보상·이중결제** — [[compensation]] (13) · [[double-payment]] (16) · [[saga]] (5)
- **결제사 연동** — [[naverpay]] (20) · [[pg-gateway]] (16)
- **예약·슬롯** — [[reservation]] (9)
- **구조·경계** — [[aggregate]] (7) · [[ddd]] (7) · [[adapter]] (7)
- **영속성·스키마** — [[jpa]] (7) · [[mysql]] (9) · [[schema]] (4)
- **예외·에러** — [[exception-handling]] (10) · [[error-code]] (5)

## 어느 축에도 걸리지 않는 노트 (2)

- [[payment-append-only-원장과-exists-완료판단]]
- [[paymentattempt-호출정책-문서-javadoc-archunit-보류]]
