---
type: moc
tags: [migration]
updated: 2026-08-17
---

# migration (MOC)

`tags: [migration]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 8개.

## decisions

- [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]]
- [[cross-aggregate-fk-to-id-참조-컨벤션]]
- [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]]
- [[flyway-도입-ddl-auto-validate-전환]]
- [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]]
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]]
- [[회수배치가-주문상태를-묻지-않는다-확인대신-전제를-지킨다]]

## knowledge

- [[종류·테이블-분리시-조용한-회귀와-전수조사-대상]]
