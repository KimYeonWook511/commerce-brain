---
type: moc
tags: [schema]
updated: 2026-08-17
---

# schema (MOC)

`tags: [schema]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 11개.

## decisions

- [[enum-db-check-미사용-application-layer-위임]]
- [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]]
- [[flyway-도입-ddl-auto-validate-전환]]
- [[multi-column-unique-length-명시-컨벤션]]
- [[payment-reserve-예약테이블-분리-a안-b안]] `superseded`
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]]
- [[결제-부분환불-도입-현행한계-4가지와-테이블분리]] `superseded`
- [[결제사건-테이블분리-기각과-유일제약-문자열-단일컬럼-교체]]
- [[예약테이블-폐지-결제행-활성슬롯-단일화와-사라지는-방어]]

## topics

- [[silent-schema-drift-패턴]]

## knowledge

- [[종류·테이블-분리시-조용한-회귀와-전수조사-대상]]
