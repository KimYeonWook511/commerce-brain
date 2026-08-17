---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, order-item, price-snapshot, migration, backfill, money-vo, code-review]
created: 2026-06-04
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-04-pr-204-unit-price-snapshot]]"
---

# OrderItem 결제 시점 단가 snapshot 컬럼 도입 — LEFT JOIN + COALESCE backfill

## 문제 — 항목 단가 휘발

단가(`unitPrice`)는 주문 생성 시 도메인에 흘러 들어왔지만 `Order.totalPrice` 누적 결과로만 남고 항목 단위로는 휘발했다 — `Product.price`가 나중에 바뀌면 영수증·환불·정산·통계에서 결제 당시 가격을 재구성할 수 없었다. e-commerce 표준대로 결제 시점 가격을 항목에 snapshot하기 위해, JPA 연관관계 분리 series(재고 #199 / 주문 #200 / 결제 #202)가 지켰던 "schema 변경 0건" 메타 원칙과 FK 일괄 제거(#203) 트랙이 끝난 직후, 그 series가 미뤄뒀던 첫 schema 변경 트랙으로 이 작업(#201, PR #204)이 분리됐다. 변경 면적은 작고(엔티티 필드·팩토리 인자·집합 루트 호출 1줄·Flyway 1개·테스트), 핵심 지식은 backfill 결정이 리뷰에서 뒤집힌 지점에 있다.

## 컬럼 타입 int(Money VO 미룸)·외부 시그니처 보존

`tbl_order_item.unit_price`를 `INT NOT NULL`로 두고 `OrderItem`에 `int unitPrice` 필드·생성자·`of(...)` 인자를 추가했다. `Order.addOrderItem(productId, quantity, unitPrice)`의 **외부 시그니처는 그대로** 두고 내부에서만 `OrderItem.of(this, productId, quantity, unitPrice)`로 흘려보냈다.

- **타입 int 선택 이유:** 기존 `Order.totalPrice`·`Payment.amount`·`Product.price`가 전부 int라 통일성상 자연스럽다. `Money` VO를 도입하면 응집력을 위해 이 세 필드까지 함께 전환해야 하고, 그러면 Order·Payment 도메인 전반과 `addOrderItem` 외부 시그니처까지 영향이 번져 사실상 별도 series 규모가 된다. 이 task 목적("결제 시점 가격을 엔티티에 보존")은 int 통일로 달성되므로 VO는 별도 트랙으로 남겼다. Payment.amount 등 가격 필드 int 통일 맥락은 [[payment-append-only-원장과-exists-완료판단]].
- **시그니처 보존이 만든 이득:** `addOrderItem(...)` 형태는 앞선 주문 리팩터(PR #200)에서 자리잡아 production·test 호출부가 여럿이었다. 외부 시그니처를 안 건드려 이 PR 변경 영향이 엔티티·migration·단위 테스트 assertion 보강에만 국한됐다(호출부 영향 0).
- 주석 대신 결정으로: PR #200 이후 남아 있던 미해결 주석("가격도 넣어야 하나 / 추후 고려")을 코드에서 제거하고 근거는 task 단위 ADR로 관리하기로 했다.

## backfill INNER→LEFT 전환(리뷰가 결정을 바꿈)

`unit_price`를 NOT NULL로 두려면 기존 row를 먼저 채워야 한다. 결제 시점 가격이 이미 휘발했으니 어떤 값도 정확하진 않지만, 0 일괄보다 "그럴듯한 추정값"인 `tbl_product.price`(현재가)가 후속 통계에서 덜 오해를 부른다.

- **처음 합의:** INNER JOIN으로 `tbl_product.price`를 채운다. "운영 데이터에 hard-delete가 없으니 모든 항목의 product가 존재해 NULL 잔여가 없다"는 데이터 분포 가정이 근거였다.
- **리뷰가 뒤집은 지점:** 리뷰 단계에서, `product_id` FK가 앞선 FK 일괄 제거 트랙(PR #203, migration V4)에서 이미 제거돼 schema가 product hard-delete를 막아주지 않는다는 점이 지적됐다. INNER JOIN이면 product가 사라진 row의 `unit_price`가 NULL로 남아 마지막 `MODIFY ... NOT NULL`에서 migration이 실패할 수 있다. 단순 사실 정정이 아니라 "데이터 분포 가정" 자체를 보강해야 한다는 시각의 전환이었다. FK 제거가 backfill 가정을 깬 공통 근원(#203/V4)은 [[cross-aggregate-fk-to-id-참조-컨벤션]].
- **최종 형태:** 컬럼을 먼저 nullable로 추가 → `UPDATE tbl_order_item oi LEFT JOIN tbl_product p ON oi.product_id=p.id SET oi.unit_price=COALESCE(p.price,0) WHERE oi.unit_price IS NULL` → `MODIFY COLUMN unit_price INT NOT NULL`. product 있는 row는 현재가로, hard-delete된 row는 0으로 fallback해 migration 안정성을 확보했다.

## 0 fallback sentinel의 의미

0은 "product가 존재하지 않아 결제 시점 가격을 재구성 불가"의 sentinel이다. 0은 비현실적 값이라 후속 통계·영수증 사용처에서 이상치로 잡힌다 — NULL은 "값이 없다"라 검사 누락이 흔하지만, 0은 "값이 비현실적"으로 보여 발견될 가능성이 높다. 검토했다 기각한 대안: (A) 전부 0으로 채운다 — 향후 응답 노출 시 기존 주문 단가가 0원으로 보여 오해; (C) NULL 허용 유지 — NOT NULL이 "snapshot 보존"이라는 도메인 invariant를 schema로 표현하는 유일한 수단이라 NULL 허용은 설계 의도를 무력화.

## 응답 DTO 노출은 이 PR 범위 밖

Issue #201 원 범위엔 "unitPrice 노출 응답 DTO 정비"가 있었으나, 현재 코드베이스에 `OrderItem`을 직접 노출하는 응답 DTO가 없었다(`OrderCreateResult`는 orderId/totalPrice/status, `OrderCancelResult`는 orderId/status, `PaymentReadyService`는 `order.getTotalPrice()`만 사용). 사용처 없는 상태에서 선제 추가하면 죽은 필드가 늘어, 실제 소비처(주문 상세·영수증)가 생길 때 별도 PR로 미뤘다. 다만 "단가 컬럼은 DB에 있는데 응답엔 없다"는 partial 변경 상태가 만들어져, 향후 노출 PR에서 분리된 히스토리 탐색 비용이 생긴 셈이다.

## 트레이드오프 — 남는 부정확성

기존 row의 `unit_price`는 "결제 시점"이 아니라 "migration 적용 시점의 product 현재가(또는 product 부재 시 0)"라는 의미다. 통계·영수증 사용처가 생기면 row의 `created_at`·migration 적용 시점·`unit_price=0` sentinel 가능성을 함께 봐야 정확히 해석된다(현재 사용처 없어 이 PR에선 비이슈).

배치 원칙 곁가지 둘: 이 결정은 Order 도메인 한정이라 본문 ADR 신설 없이 task ADR + 전역 색인 표 한 줄로 처리했고(cross-cutting 결정 자리를 침범하지 않으려 의식적으로 피함), 앞선 series의 완료된 task 폴더 회고를 보강해 series 연계를 남기는 안은 [[backend-완료된-task-문서-불변-원칙]]을 위반하므로 기각했다. "리뷰가 결정을 바꾼" 이 케이스는 schema 차원 invariant의 부재를 외부 시각으로 짚어준 것이 LLM 코드 리뷰 도구의 진짜 가치가 드러난 지점이었다([[ai-코드리뷰-종합-층위통합과-심각도-도메인보정]]).

> [!note] 이 스냅샷이 일반 원리로 승격됐다 (2026-07)
> "결제 시점 가격을 항목에 snapshot한다"가 **금액을 성격으로 셋(도출 / 기록 / 스냅샷)으로 가르는 원리**의 한 사례로 정리됐다([[주문-금액-모델-도출·기록·스냅샷-3분류와-청구액-승인액-분리]]). 그 노트가 더한 것은 **스냅샷의 이유가 성능이 아니라 시간이라는 것**과, 배송비·쿠폰이 들어올 때 정책은 각 도메인이 갖고 금액 스냅샷은 주문이 갖는다는 배치다. 이 단가 스냅샷이 부분취소의 잔액 정본을 수량 기준으로 택할 수 있게 한 근거가 되기도 했다([[부분취소-잔액-정본-수량기준-상태무관-누적]]).

## 미해결

- **Money VO 도입 검토:** totalPrice·amount·price·unitPrice가 전부 int. 가격 정책 변경·환불·할인·정산 연산이 추가되면 원시 타입으로 흩어진 가격 개념 부채가 가시화될 수 있고, 도입 시엔 네 필드를 한 series로 일괄 전환(도메인 재설계 규모).
- **응답 DTO 노출**은 주문 상세·영수증 추가 시점에 별도 PR. 그때 기존 row의 "migration 시점 현재가(또는 0 sentinel)" 한계도 사용처에 함께 문서화.
- 운영 DB V5 적용 시 `0` sentinel row가 실제로 발생하는지 추적(발생 시 어떤 product가 hard-delete됐는지 정합성 분석 단서).

## 근거

- [[raw/sessions/backend/2026-06-04-pr-204-unit-price-snapshot]]
