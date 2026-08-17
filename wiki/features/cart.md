---
type: moc
tags: [cart]
updated: 2026-08-17
---

# cart (MOC)

`tags: [cart]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 6개.

## decisions

- [[cart-add-product-존재-상태-검증]]
- [[cart-delete-미존재-4xx-entity-경유-삭제]]
- [[cart-path-id-검증-spec을-코드에-맞춤]]
- [[cart-도메인-골격-cartitem-단일-aggregate]]
- [[cart-동시성-낙관락-processor-분리-retry]]

## tradeoffs

- [[cart-회원당-row-상한-미도입]] `open`
