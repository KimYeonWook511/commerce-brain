---
type: moc
tags: [ddd]
updated: 2026-08-17
---

# ddd (MOC)

`tags: [ddd]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 13개.

## decisions

- [[cart-도메인-골격-cartitem-단일-aggregate]]
- [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]]
- [[cross-aggregate-fk-to-id-참조-컨벤션]]
- [[product-공개query-관리자command-서비스-분리]]
- [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]]
- [[도메인-정책-빈-등록-도메인이-설정을-모르게]]
- [[돈정합성-우선-경계넘는-트랜잭션-셋-허용과-셀-수-있게]]
- [[보상판단-payment-존재-lock-대신-db-unique]]
- [[응용계층-서비스-분할-기준-다른-도메인까지-바꿀-때만]]
- [[인증-패키지-경계-auth-member-security-분리]]
- [[환불-독립-aggregate-한도판정은-결제가-누적액-컬럼]]

## knowledge

- [[ddd-이관-컨벤션-adapter-command-query-네이밍]]
- [[jpa-메커니즘-이점-한계와-ddd-괴리-트레이드오프]]
