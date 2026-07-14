---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [product, stock, dependency-direction, normalization, ux, n-plus-one]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-product-domain-overview]]"
---

# 상품 상세조회 stock 의존 + 재고 누락 시 stockQuantity=0 정규화

## 컨텍스트·문제(상세에 재고 표시)

상품 상세 응답에 재고 수량을 담아야 한다. `ProductQueryService.getProduct(productId)`는 재고 조회 포트 `StockRepository.findByProductId(productId)`를 호출한다. 두 가지가 결정 대상이었다 — (1) 의존 방향을 어느 쪽으로 둘지, (2) stock row가 없을 때 어떻게 처리할지.

## 왜 product→stock 의존인가

- 사용자 화면 진입점이 "상품 상세"라 **product가 owner**, stock은 "상품에 딸린 정보".
- stock 도메인이 product를 모르도록 의존 방향을 **한쪽으로 고정**했다. stock은 productId로 자기 row를 조회당하는 입장이다([[stock-도메인-구조-개요]]).
- 결합은 application 계층에서만 일어나 도메인끼리 직접 의존은 없다 — `ProductQueryService`만 `StockRepository`를 import 한다. 이 단방향 결합은 [[product-도메인-구조-개요]]의 뼈대다.

## 왜 재고 누락이 예외가 아니라 0인가

- 상품 관리 기능은 **상품 생성 시 재고를 함께 만들지 않는다**(의도적 분리 — 초기 재고 생성은 별도 API). 이 스코핑은 [[product-mvp-범위-imageurl-카테고리-페이지네이션-제외]]와 관리자 재고 조작 [[관리자-재고조작-별도api-이력-감사-분리]]에서 왔다. 그래서 "상품은 있는데 stock row가 없는" 상태가 **정상 흐름에 존재**한다.
- 사용자 화면에 "재고 정보 누락" 500 vs "재고 0" 200 — UX 관점에서 후자가 부드럽다. 누락을 예외로 터뜨리지 않고 `0`으로 정규화한다.
- 상품 조회 ADR이 명시한 트레이드오프: "재고 레코드 누락을 운영 이슈로 숨기는 trade-off가 있으므로 **테스트로 명시적으로 고정**해야 한다." 숨김을 의식적 선택으로 못 박고 회귀로 가둔다.

## 목록은 stock 미포함(N+1 회피)

- 목록은 탐색용이라 "재고 N개"를 일일이 그릴 필요가 없다.
- 목록에 stock join을 넣으면 **N+1 위험 + 응답 사이즈 증가**(join 대안·응답 외부 주입 논의는 [[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]]). 상세에서만 stock을 1회 조회 — 비용이 정확히 필요한 시점에만 발생한다.
- 향후 목록에 품절 표시가 필요하면 **`SOLD_OUT` 상태만으로** 표시할 계획이고 stock 수량 join은 도입하지 않는다 → 상태 노출 정책 [[productstatus-3상태-공개노출-정책]].

## 트레이드오프·운영 관점 미해결(stock=0 vs row 없음)

- "상품은 있는데 stock row 없음"은 **사용자 화면**에는 "재고 0"으로 보인다(의도). 이 정규화는 사용자 UX 기준의 선택이다.
- 하지만 **운영자 관점**에서는 두 상태의 의미가 다르다:
  - **stock = 0**: 판매 의도는 있고 재고만 소진(재입고 대상).
  - **row 없음**: 판매 준비 미완료(초기 재고 미생성).
- 현재 관리자 API도 같은 정규화를 써서 둘을 구분하지 못한다 → 운영 점검 가시성에 갭. 관리자용 별도 필드/엔드포인트가 필요하다는 후속 작업으로 뺐다(사용자 화면 정규화는 그대로 유지).

## 근거

- [[raw/sessions/backend/2026-05-29-product-domain-overview]]
