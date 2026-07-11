---
platform: backend
author: KimYeonWook511
created: 2026-06-04
origin:
  - { type: pr, repo: commerce-backend, ref: 204 }
---

# OrderItem 결제 시점 가격 snapshot 컬럼 도입 — 기존 row backfill을 LEFT JOIN + COALESCE 안전망으로

주문 항목(`OrderItem`)에 결제 시점 단가를 영구 보존하는 `unit_price` 컬럼을 신설한 트랙이다. 그동안 단가(`unitPrice`)는 주문 생성 시 도메인에 흘러 들어왔지만 `Order.totalPrice` 누적 결과로만 남고 항목 단위로는 휘발했다 — `Product.price`가 나중에 바뀌면 영수증·환불·정산·통계에서 결제 당시 가격을 재구성할 수 없었다. e-commerce 표준대로 결제 시점 가격을 항목에 snapshot 하기 위해, 앞선 JPA 연관관계 분리 series(재고 #199 / 주문 #200 / 결제 #202)가 지켰던 "schema 변경 0건" 메타 원칙과 FK 일괄 제거(#203) 트랙이 종료된 직후, 그 series가 미뤄뒀던 첫 schema 변경 트랙으로 이 작업(Issue #201, PR #204)이 분리됐다.

변경 면적 자체는 작다 — 엔티티 필드·팩토리 인자 추가, 집합 루트 내부 호출 1줄, Flyway migration 1개, 단위·슬라이스 테스트 보강. 핵심 지식은 코드량이 아니라 backfill 결정이 PR 리뷰에서 뒤집힌 지점에 있다.

## 결정한 것

### 1. 컬럼 타입은 `int`, Money VO 도입은 다시 미룸

`tbl_order_item.unit_price`를 `INT NOT NULL`로 두고, `OrderItem` 엔티티에 `int unitPrice` 필드와 생성자·`of(...)` 팩토리 인자를 추가했다. `Order.addOrderItem(productId, quantity, unitPrice)`의 **외부 시그니처는 그대로 두고** 내부에서만 `OrderItem.of(this, productId, quantity, unitPrice)`로 인자를 흘려보냈다.

- **타입 `int` 선택 이유:** 기존 `Order.totalPrice`·`Payment.amount`·`Product.price`가 전부 `int`라 통일성 측면에서 자연스러운 선택. `Money` VO를 도입하면 응집력을 위해 이 세 필드까지 함께 전환해야 하고, 그러면 Order·Payment 도메인 전반과 `addOrderItem` 같은 외부 시그니처까지 영향이 번져 사실상 별도 series 규모가 된다. 이 task의 목적("결제 시점 가격을 엔티티에 보존")은 `int` 통일로 달성되므로 VO는 별도 트랙으로 남겼다.
- **시그니처 보존이 만든 이득:** `addOrderItem(productId, quantity, unitPrice)` 형태는 이미 앞선 주문 리팩터링(PR #200)에서 자리잡아 production·test 호출부가 여럿이었다. 외부 시그니처를 건드리지 않으니 이 PR의 변경 영향이 엔티티·migration·단위 테스트 assertion 보강에만 국한됐다(호출부 영향 0).
- **주석 대신 결정으로:** PR #200 이후 `OrderItem.java`에 남아 있던 미해결 주석("가격도 넣어야 하나 / 추후 고려")을 코드에서 제거하고, 결정 근거는 task 단위 ADR로 관리하기로 했다.

### 2. 기존 row backfill을 LEFT JOIN + COALESCE(price, 0)로 — 리뷰가 결정을 바꾼 케이스

`unit_price`를 NOT NULL로 두려면 기존 row를 먼저 채워야 한다. 결제 시점 가격이 이미 휘발했으니 어떤 값으로도 정확성은 보장 못 하지만, 0으로 일괄 채우는 것보다 "그럴듯한 추정값"인 `tbl_product.price`(현재가)로 채우는 편이 후속 통계에서 덜 오해를 부른다. 그래서 product를 JOIN 해 현재가로 backfill 하기로 했다.

- **처음 합의:** INNER JOIN으로 `tbl_product.price`를 채운다. "운영 데이터에 hard-delete가 없으니 모든 항목의 product가 존재해 NULL 잔여가 없을 것"이라는 데이터 분포 가정이 근거였다.
- **리뷰가 뒤집은 지점:** PR 코드 리뷰 단계에서, `product_id` FK가 앞선 FK 일괄 제거 트랙(PR #203, migration V4)에서 이미 제거된 상태라 schema가 product hard-delete를 막아주지 않는다는 점이 지적됐다. INNER JOIN이면 product가 사라진 row의 `unit_price`가 NULL로 남고, 마지막 `MODIFY ... NOT NULL` 단계에서 migration이 실패할 수 있다. 이건 단순 사실관계 정정이 아니라 "데이터 분포 가정" 자체를 보강해야 한다는 시각의 전환이었다.
- **최종 형태:** 컬럼을 먼저 nullable로 추가 → `UPDATE tbl_order_item oi LEFT JOIN tbl_product p ON oi.product_id = p.id SET oi.unit_price = COALESCE(p.price, 0) WHERE oi.unit_price IS NULL` → `MODIFY COLUMN unit_price INT NOT NULL`. product가 있는 row는 현재가로, hard-delete된 row는 `0`으로 fallback 해 migration 안정성을 확보했다.
- **`0` fallback의 의미:** "product가 존재하지 않아 결제 시점 가격을 재구성 불가"의 sentinel이다. 0은 비현실적 값이라 후속 통계·영수증 사용처에서 이상치로 잡힌다 — NULL은 "값이 없다"라 검사 누락이 흔하지만 0은 "값이 비현실적"으로 보여 발견될 가능성이 높다.
- **검토했다 기각한 대안:** (A) 전부 0으로 채운다 — 단순하지만 향후 응답에 노출되면 기존 주문 단가가 0원으로 보여 오해를 부른다. (C) NULL 허용을 유지한다 — NOT NULL이 "snapshot 보존"이라는 도메인 invariant를 schema로 표현하는 유일한 수단이라, NULL 허용은 설계 의도를 무력화해 기각.
- **트레이드오프(남는 부정확성):** 기존 row의 `unit_price`는 "결제 시점"이 아니라 "migration 적용 시점의 product 현재가(또는 product 부재 시 0)"라는 의미다. 통계·영수증 사용처가 생기면 row의 `created_at`·migration 적용 시점·`unit_price = 0` sentinel 가능성을 함께 봐야 정확히 해석된다. 현재 사용처가 없어 이 PR에서는 이슈가 되지 않는다.

### 3. 본문 ADR 신설 없이 task ADR + 색인 표 한 줄

프로젝트 ADR 정책은 "코드베이스 전반에 영향을 주는 cross-cutting 결정은 본문 ADR에, 특정 도메인 한정 결정은 task 단위 ADR에 둔다"로 명시돼 있다. `OrderItem.unitPrice` 신설은 Order 도메인 한정 결정이라 task ADR 3개로 분리하고, 전역 ADR 색인 표에 한 줄만 추가했다.

- 본문 ADR을 새로 늘리는 것은 cross-cutting 결정을 담는 자리이니, 도메인 한정 결정으로 그 자리를 침범하지 않으려 의식적으로 피했다. 매번 본문 ADR을 늘리는 대신 색인 표만 늘리는 패턴이 누적 비용을 줄인다.
- 함께 검토했다 기각: 앞선 series의 완료된 task 폴더 회고를 보강해 series 연계를 남기는 안 — 완료된 task 문서 불변 원칙을 위반하므로, series 연계 사실은 이번 회고와 루트 docs에서만 표현.

### 4. 응답 DTO 노출은 이 PR 범위 밖

Issue #201 원래 범위엔 "결제 응답·영수증 등 `unitPrice` 노출이 필요한 응답 DTO가 있으면 함께 정비"가 적혀 있었다. 하지만 현재 코드베이스에 `OrderItem`을 직접 노출하는 응답 DTO가 없었다 — 주문 생성 결과(`OrderCreateResult`)는 `orderId/totalPrice/status`만, 주문 취소 결과(`OrderCancelResult`)는 `orderId/status`만, 결제 준비 서비스(`PaymentReadyService`)는 `order.getTotalPrice()`만 쓴다.

- 사용처 없는 상태에서 DTO를 선제 추가하면 죽은 필드가 늘어난다. 실제 소비처(주문 상세 조회, 영수증 응답)가 생길 때 별도 PR로 다루기로 했다.
- **다만** 큰 그림에선 "단가 컬럼은 DB에 있는데 응답엔 없다"는 partial 변경 상태가 만들어졌다. 향후 노출 PR에서 분리된 히스토리를 추적해야 하는 탐색 비용이 생긴 셈이다. 사용처가 분명해지는 시점(예: 주문 상세 조회 API 기획 확정)에 엔티티·migration·응답 DTO를 한 PR로 완결하는 편이 더 깔끔했을 수 있다.

## 막힌 점

큰 막힘은 없었다. 다만 INNER JOIN backfill의 hard-delete blind spot을 사전 탐색에서 스스로 잡지 못하고 리뷰에서 드러난 게 아쉬운 지점이다. "FK가 V4에서 제거된 상태"와 "운영 데이터에 hard-delete 없음이라는 가정"을 결합해서 봤으면 사전에 잡을 수 있었던 케이스다.

## 배운 것

- **FK 제거의 부산물:** schema가 제공하던 참조 무결성이 빠지면, 그 FK에 기대던 backfill·migration·가정도 다시 봐야 한다. cross-aggregate FK 일괄 제거로 `product_id` FK가 사라진 상태에서 product 참조 무결성을 가정한 INNER JOIN backfill은 잠재 결함이었다.
- **데이터 분포 가정의 fragility:** "운영 데이터에 X 없음"으로 가드를 생략하는 결정은 schema 차원 안전망(FK 등)이 함께 있을 때만 안전하다. 안전망이 없을 땐 LEFT JOIN + COALESCE 같은 방어적 SQL이 cheap insurance다.
- **`0` fallback의 sentinel 가치:** NULL보다 0이 후속 사용처에서 이상치로 잡힐 가능성이 높다. NULL은 "값이 없다"라 검사 누락이 흔하고, 0은 "값이 비현실적"으로 보여 발견된다.
- **cross-cutting vs 도메인 한정 결정의 배치 경계:** ADR 정책이 그 경계를 명시해 둔 덕에, 매번 본문 ADR을 늘리지 않고 색인 표만 늘리는 패턴으로 누적 비용을 줄일 수 있다.
- **리뷰가 결정을 바꾼 케이스의 가치:** 단순 사실 정정이 아니라 "데이터 분포 가정" 자체를 보강해야 한다는 시각의 전환이었다. schema 차원 invariant의 부재를 외부 시각으로 짚어주는 게 LLM 코드 리뷰 도구의 진짜 가치가 드러난 지점이었다.

## 미해결·열린 질문

- **`OrderItem.unitPrice` 응답 DTO 노출:** 주문 상세 조회·영수증 응답이 추가되는 시점에 별도 PR로. 그때 기존 row의 `unit_price`가 "migration 적용 시점 현재가(또는 0 sentinel)"라는 한계도 사용처에 함께 문서화해야 한다.
- **Money VO 도입 검토:** `Order.totalPrice`·`Payment.amount`·`Product.price`·`OrderItem.unitPrice`가 전부 `int`다. 가격 정책 변경, 환불·할인·정산 같은 연산이 추가되면 원시 타입으로 흩어진 가격 개념의 부채가 가시화될 수 있다. 도입 시엔 네 필드를 한 series로 일괄 전환하는 게 적합하다 — 단순 필드 교체가 아니라 도메인 재설계에 가까운 규모.
- **운영 DB에서 V5 적용 시 `0` sentinel row가 실제로 발생하는지 추적:** 발생한다면 어떤 product가 hard-delete됐는지 데이터 정합성 분석의 단서가 된다.
