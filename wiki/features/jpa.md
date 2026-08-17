---
type: moc
tags: [jpa]
updated: 2026-08-17
---

# jpa (MOC)

`tags: [jpa]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 15개.

## decisions

- [[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]]
- [[enum-db-check-미사용-application-layer-위임]]
- [[multi-column-unique-length-명시-컨벤션]]
- [[payment-attempt-네이밍-정리와-refactor-경계]]
- [[requires-new-격리-제거-보상판단-트랜잭션정책]]
- [[schema-무변경-decouple-series-메타원칙과-scope-규율]]
- [[unique-위반-예외번역-infra-adapter-경계와-flush-타이밍]]
- [[결제-도메인-orm-선택과-jpa-오염-격리-실용진영]]
- [[도메인-팩토리-long-id-시그니처-전환과-정책-표면화]]
- [[무결성위반-도메인예외-번역을-제약이름으로-가른다]]
- [[유일슬롯-비우고-같은값-재점유-쓰기순서와-메서드이름-신호]]

## topics

- [[silent-schema-drift-패턴]]

## knowledge

- [[jpa-메커니즘-이점-한계와-ddd-괴리-트레이드오프]]
- [[애너테이션-잔재-판별과-전파변경의-osiv-파급]]
- [[예외타입-실측과-테스트가-명세를-넘어-단언하는-것]]
