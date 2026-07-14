---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [product, domain-model, soft-delete, product-status, stock]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-product-domain-overview]]"
---

# Product 도메인 구조 개요 — 공개조회/관리자관리 분리와 기반 도메인 위치

## 한 줄 정의

주문 생성·재고 생성·상품 공개 조회가 **모두 밟고 가는 기반 도메인**. 상품 마스터(이름·가격·설명·이미지·판매 상태) + 공개 노출 정책 + soft delete를 책임지고, 재고는 별도 도메인으로 분리한다. payment 도메인 전환에 앞서 repository 경계와 서비스 책임을 먼저 DDD 계층으로 정리해둔 위치다.

## 책임 범위와 재고 분리

- **책임**: 상품 마스터 + 공개 노출 정책 + soft delete.
- **재고는 책임을 뗀다** — 재고 수량·동시성·이력은 [[stock-도메인-구조-개요]]가 가진다. `Product : Stock = 1:1`이지만 참조 컬럼(`product_id`)은 stock 쪽이 들고, product는 stock을 알지 않는다. 의존은 application 계층에서 공개 조회 서비스가 재고 조회 포트를 부르는 **한 방향**으로만 생긴다 → [[product-상세조회-stock-의존-재고누락-0-정규화]].

## 엔티티·판매상태 enum·soft delete

- **엔티티**: `Product` 단일.
- **판매 상태 enum 3개**: `ON_SALE` / `SOLD_OUT` / `STOPPED`. 공개 노출은 `PUBLIC_STATUSES = [ON_SALE, SOLD_OUT]`만 — 운영 상태와 공개 노출을 1:1로 두지 않는 정책은 [[productstatus-3상태-공개노출-정책]].
- **soft delete**: `deletedAt`(LocalDateTime). 공개·관리자 조회 모두 `deletedAt IS NULL` 필터를 건다. hard delete가 아닌 이유(주문 이력 보존 강제)는 [[product-soft-delete-deletedat-주문이력-보존]].
- **상세 응답**은 `stockQuantity`를 포함하되 stock row가 없으면 `0`으로 정규화.

## DDD 이관 후 패키지 구조

- `domain/Product`, `domain/ProductStatus`, `domain/repository/ProductRepository`(도메인 포트)
- `application/ProductQueryService`(공개), `application/AdminProductService`(관리자) + command/result DTO
- `infrastructure/JpaProductRepository`(Spring Data JPA + `@Query`) + `ProductRepositoryAdapter`(포트 구현)
- `presentation/ProductController`(공개), `AdminProductController`(관리자)
- `exception/ProductErrorCode` — 현재 `PRODUCT_NOT_FOUND` 하나.

repository 포트는 파생 쿼리명이 아니라 **의도 기반 메서드명**(`findVisibleProduct`·`findNotDeletedProduct`)을 쓰고 실제 JPA 조건은 `@Query`로 분리하며, `JpaRepository`를 포트에 직접 상속하지 않고 adapter로 격리한다. 이 컨벤션의 "왜"와 트레이드오프는 도메인 공통 정본 [[ddd-이관-컨벤션-adapter-command-query-네이밍]]에 있다(product는 그 인스턴스).

## application 의존 방향(product→stock)

사용자 진입점이 상품 상세라 product가 owner, stock은 딸린 정보. 그래서 의존을 `product → stock` 한쪽으로 고정하고 결합은 application 계층(`ProductQueryService`가 재고 포트를 import)에서만 일어나게 했다. 도메인끼리 직접 의존은 없다. 상세한 근거·재고 누락 정규화·목록 미포함(N+1 회피)은 [[product-상세조회-stock-의존-재고누락-0-정규화]].

## 관련 결정 링크

product 도메인을 정리하며 내린 결정들의 허브:

- [[product-공개query-관리자command-서비스-분리]] — 서비스 command/query 분리
- [[productstatus-3상태-공개노출-정책]] — 상태 enum과 공개 노출
- [[product-soft-delete-deletedat-주문이력-보존]] — soft delete
- [[product-상세조회-stock-의존-재고누락-0-정규화]] — stock 의존·재고 0 정규화
- [[product-mvp-범위-imageurl-카테고리-페이지네이션-제외]] — MVP 범위 스코핑

관계 도메인: [[stock-도메인-구조-개요]](1:1, 재고 owner) · [[order-도메인-구조-개요]](order_item이 상품 id를 참조 → soft delete 강제) · [[payment-도메인-구조-개요]](product 정리 뒤에 온 다음 DDD 이관 대상).

## 열린 질문(마이그레이션·인덱스·legacy 잔존)

- **DB 마이그레이션 도구 부재** — `tbl_product.status` 기존 row 기본값·`deleted_at = null` 설정, enum ENUM→VARCHAR ALTER가 운영 문서로만 남아 있다. 도구 도입은 별도 트랙 → [[flyway-도입-ddl-auto-validate-전환]].
- **공개 조회 인덱스** — 데이터가 늘면 `(status, deleted_at, created_at)` 조합 인덱스 검토(현재 MVP 규모라 보류).
- **stock row 부재 vs stock = 0** — 사용자에겐 둘 다 재고 0으로 보이지만 운영자에겐 의미가 다르다(판매 준비 미완료 vs 재고 소진). 관리자 가시성 갭은 후속 운영 점검 대상.
- **legacy 패키지 잔존** — 이관 회고에 남긴 `product.service`·`controller`·`repository` legacy 제거의 완료 여부는 확인 필요.

## 근거

- [[raw/sessions/backend/2026-05-29-product-domain-overview]]
