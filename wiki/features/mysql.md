---
type: moc
tags: [mysql]
updated: 2026-08-17
---

# mysql (MOC)

`tags: [mysql]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 13개.

## decisions

- [[flyway-도입-ddl-auto-validate-전환]]
- [[multi-column-unique-length-명시-컨벤션]]
- [[payment-reserve-예약테이블-분리-a안-b안]] `superseded`
- [[payment-동시성-unique-vs-lock-gap-lock회피]]
- [[payment-이중결제-reserve따닥-mysql-null트릭-unique]]
- [[sql-translator-빈-제거-제약명-이중결제-식별]]
- [[검증-전에-채우는-외부값에-유일제약-금지]]
- [[결제사건-테이블분리-기각과-유일제약-문자열-단일컬럼-교체]]
- [[멱등키-세-값-분리와-요청멱등키는-호출자가-발급]]
- [[무결성위반-도메인예외-번역을-제약이름으로-가른다]]
- [[유일슬롯-비우고-같은값-재점유-쓰기순서와-메서드이름-신호]]

## topics

- [[silent-schema-drift-패턴]]

## knowledge

- [[예외타입-실측과-테스트가-명세를-넘어-단언하는-것]]
