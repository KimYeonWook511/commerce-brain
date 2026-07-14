---
type: moc
tags: [migration]
updated: 2026-07-14
---

# migration (MOC)

`tags: [migration]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 6개.

## decisions

- [[cross-aggregate-fk-to-id-마이그레이션-동기-전략]]
- [[cross-aggregate-fk-to-id-참조-컨벤션]]
- [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]]
- [[flyway-도입-ddl-auto-validate-전환]]
- [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]]
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]]
