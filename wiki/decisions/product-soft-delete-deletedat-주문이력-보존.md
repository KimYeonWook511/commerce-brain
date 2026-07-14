---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [product, soft-delete, order, data-integrity, audit]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-product-domain-overview]]"
---

# Product soft delete(deletedAt) — 주문 이력 보존이 결정을 강제

## 컨텍스트·문제(상품 삭제와 주문 이력)

상품 삭제를 hard delete로 할지 soft delete로 할지의 문제. 결론은 `deletedAt = now()` 기반 soft delete이고, 공개·관리자 조회 모두 `deletedAt IS NULL` 필터를 건다. 이 결정은 취향이 아니라 **외부 제약이 강제**했다.

## 선택 근거(외부 제약이 결정)

- 주문 항목(`order_item`)이 어떤 상품을 구매했는지 **상품 id로 참조**한다([[order-도메인-구조-개요]] · cross-aggregate 참조 컨벤션 [[cross-aggregate-fk-to-id-참조-컨벤션]]). 과거 주문은 "그때 산 상품"이 무엇이었는지를 영구히 가져야 한다.
- hard delete를 하면 과거 주문 조회 때 사라진 상품을 가리키는 **dangling 참조나 NULL**이 남아 주문 이력 정합성이 깨진다.
- soft delete면 row는 보존되고 공개 조회만 필터로 제외한다.

> 참고: 상품명·가격을 주문 시점 값으로 얼려두는 스냅샷 부재 부채는 order_item 쪽 별도 결정 [[orderitem-단가-snapshot-컬럼과-backfill-leftjoin-coalesce]]에서 다룬다. soft delete는 "id 참조가 살아 있게" 하는 최소 보장이고, 그때의 값 보존은 그 스냅샷 문제와 직교한다.

## 왜 archive 테이블이 아닌가

- archive 테이블로 분리하면 read path가 이중화(현재+과거 union, 또는 application 분기)돼 비용이 크다.
- 단일 테이블 + `deletedAt` 컬럼은 인덱스 한두 개면 충분히 빠르다.
- MVP 규모에 과한 분리는 "불필요한 추상화를 피한다"는 프로젝트 원칙에 어긋난다.

## 왜 boolean이 아니라 LocalDateTime인가

- **언제** 삭제됐는지 추적이 필요할 수 있다고 봤다. 운영자가 "이 상품이 언제부터 안 보였지"를 물을 수 있다.
- boolean이면 그 시점 정보가 사라진다. **시점 컬럼은 boolean의 상위 호환**이다 — NULL 여부로 boolean 의미(삭제/미삭제)도 그대로 표현된다.

## 보류한 인덱스

- `(status, deleted_at, created_at)` 조합 인덱스는 **후속 검토로 보류**했다.
- 공개 조회는 `where deleted_at is null and status in (...) order by created_at desc` 형태라 product row가 늘면 풀스캔 비용이 커진다. 다만 MVP 데이터 규모에서는 인덱스 없이 시작한다.
- 마이그레이션 도구가 없어 인덱스·컬럼 변경 절차 자체가 미정인 것과 함께 묶여 있다 → [[flyway-도입-ddl-auto-validate-전환]].

## 트레이드오프(row 단조 증가, GDPR 별도 절차)

- 운영 DB 크기가 단순 row count 기준으로 **단조 증가**한다("삭제된" 상품도 row가 남는다).
- 정말로 row를 지워야 하는 GDPR 류 시나리오는 별도 절차가 필요하다(현재 범위 밖).

관련: `STOPPED`(일시 중지)와 `deletedAt`(영구 제거)의 구분은 [[productstatus-3상태-공개노출-정책]].

## 근거

- [[raw/sessions/backend/2026-05-29-product-domain-overview]]
