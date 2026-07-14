---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [product, product-status, enum, domain-model, soft-delete, jdbc-type]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-product-domain-overview]]"
---

# ProductStatus 3상태(ON_SALE/SOLD_OUT/STOPPED)와 공개 노출 정책

## 컨텍스트·문제(운영 상태 vs 공개 노출)

`ProductStatus`는 `ON_SALE` / `SOLD_OUT` / `STOPPED` 3개다. 핵심 문제는 **운영 상태 enum과 공개 노출 정책을 1:1로 일치시킬 것인가**였다. 결정은 "아니다" — 공개 노출에는 `ON_SALE`과 `SOLD_OUT` 둘 다 포함하고 `STOPPED`만 제외한다(`PUBLIC_STATUSES = [ON_SALE, SOLD_OUT]`).

## 왜 SOLD_OUT을 노출하나

- 품절 상품을 화면에서 통째로 지우면 "어제까지 보던 상품이 사라졌다"는 **사용자 인지 충격**이 크다.
- 품절 표시 + 재입고 알림 같은 후속 UX 여지를 남기려면 *상태는 노출하되 주문은 막을 수 있게* 분리해야 한다.
- 실제 주문 가능 여부는 재고 수량이 결정한다([[stock-도메인-구조-개요]]). 그래서 product 상태는 "운영자 의도"만 표현하면 되고, "정말 지금 살 수 있나"는 stock에 맡긴다 → 상세 조회의 stock 의존은 [[product-상세조회-stock-의존-재고누락-0-정규화]].

## 왜 STOPPED가 soft delete와 별도인가

- soft delete(`deletedAt`)는 **영구 제거 의도**(다시 안 돌아옴) → [[product-soft-delete-deletedat-주문이력-보존]].
- `STOPPED`는 **일시 중지 의도**(재판매 가능) — 시즌오프나 사고로 잠시 내려두는 시나리오.
- 두 시나리오를 하나의 메커니즘으로 합치면 "삭제된 상품 다시 살리기"라는 변칙 흐름이 생긴다. 그래서 명시적으로 갈랐다.

## SOLD_OUT 전이는 운영자 직접 판단

- 재고가 0이 됐다고 `product.status`를 자동으로 `SOLD_OUT`으로 바꾸는 로직은 **없다**. 관리자가 수정 API로 직접 set 하는 길만 존재한다.
- 의도적 운영 판단이다 — "곧 재입고할 거면 `ON_SALE` 유지" 같은 운영자 재량을 허용한다. 재고와 상태의 자동 동기화는 필요해지면 나중에 추가한다.

## 노출 정책을 enum 안에 배치

- 노출 정책을 enum 자신이 들게 했다: `isPubliclyVisible()` + 정적 `publicStatuses()`.
- application·repository가 "공개 상태 목록"을 직접 알 필요 없이 enum에 위임한다. 목록 조회는 `productRepository.findVisibleProducts(ProductStatus.publicStatuses())`로 부른다.
- 정책이 바뀌면 **enum 한 곳만** 고치면 된다.

## 트레이드오프(마이그레이션 결합, VARCHAR 강제 컨벤션)

- enum 값을 추가/제거하면 스키마 마이그레이션과 결합된다(마이그레이션 도구 부재는 [[flyway-도입-ddl-auto-validate-전환]]에 연결된 미해결).
- `tbl_product.status`는 `@Enumerated(EnumType.STRING)`에 `@JdbcTypeCode(SqlTypes.VARCHAR)`를 함께 붙여 매핑한다. Hibernate 6.x가 STRING enum을 MySQL 네이티브 ENUM 타입으로 매핑하면, 컬럼을 생략한 INSERT에 **첫 enum 값이 조용히 채워지는 함정**이 있어 이를 VARCHAR로 강제한다.
- 이 VARCHAR 강제는 product만이 아니라 `tbl_member.role` 등 enum 컬럼 전반에 적용되는 **프로젝트 공통 컨벤션**이다(enum 제약을 DB에 박지 않고 application 계층에 두는 흐름은 [[enum-db-check-미사용-application-layer-위임]]과 결이 같다). cross-domain 지식으로 승격 후보.

## 근거

- [[raw/sessions/backend/2026-05-29-product-domain-overview]]
