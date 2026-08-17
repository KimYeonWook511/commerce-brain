---
type: moc
tags: [redis]
updated: 2026-08-17
---

# redis (MOC)

`tags: [redis]` 를 단 노트의 생성된 뷰. 진실은 각 노트의 frontmatter `tags` 에 있고, 이 목록은 그 거울이다(덮어쓰기 재생성).

총 12개.

## decisions

- [[auth-redis-장애-도메인예외-매핑-도메인전용-어드바이스]]
- [[jwt-redis-하이브리드-rtr-ttl-근거]]
- [[redis-장애-strict-정책-soft-fail-기각]]
- [[redis-장애-멱등캐시-fallback-boolean-예외분리]]
- [[동시도착은-선점층이-받고-처리중-전용응답]]
- [[부분취소-동시성-주문행-단일잠금과-캐시겹-미도입]]
- [[주문-멱등-캐시-inflight-차단-전용]]
- [[주문-멱등성-redis-setnx-plus-rdb-unique-이중방어]]
- [[주문-멱등성-캐싱-after-commit-이벤트-분리]]
- [[회원가입-not-supported-트랜잭션-분리]]

## topics

- [[auth-member-security-도메인-구조-개요]]

## knowledge

- [[방어-겹은-정합성겹과-최적화겹으로-가른다]]
