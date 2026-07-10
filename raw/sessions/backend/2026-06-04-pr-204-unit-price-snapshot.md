---
platform: backend
author: KimYeonWook511
created: 2026-06-04
origin:
  - { type: pr, repo: commerce-backend, ref: 204 }
---

# OrderItem 결제 시점 가격 snapshot 컬럼 도입 (Issue #201 / PR #204)

## 한 일

- Issue #201 의 후속 트랙. PR #200 series 종료 후 분리됐던 결제 시점 가격 snapshot 작업을 진행
- task 폴더 `docs/tasks/order-item-price-snapshot/` 신규 (prd / architecture / adr / api-spec / db-schema / phases / retrospective)
- V5 migration: `tbl_order_item.unit_price INT NOT NULL` 신설 + 기존 row backfill
- `OrderItem` entity 에 `unitPrice` 필드 / 생성자 / `of(...)` 팩토리 확장. `Order.addOrderItem` 외부 시그니처는 그대로 유지 (호출부 영향 0)
- 단위 / 슬라이스 테스트 보강 — snapshot 보존 시나리오 신규 + JPA round-trip
- harness skill 의 phase `0-snapshot` 을 step 3개 (`add-unit-price-snapshot` / `sync-root-docs` / `write-retrospective`) 로 분해. `execute.py` 가 developer (sonnet) → AC 재검증 → reviewer (opus) → commit (haiku) 흐름으로 step 1~3 을 자동 실행 + phase 종료 시 phase index 까지 chore 커밋으로 묶음
- PR #204 생성
- **Gemini Code Assist review 가 backfill 결정을 바꿈** — INNER JOIN → LEFT JOIN + COALESCE 안전망 추가. 단순 fix 가 아니라 *결정 자체의 보강*. `fix` 커밋으로 반영 후 thread resolve

## 결정한 것

본 raw 는 결정 4개를 다룬다 (1~3 은 정본 `commerce-backend/docs/tasks/order-item-price-snapshot/adr.md` 결정 1~3, 4 는 `retrospective.md` 의 메타 결정). 여기는 "내가 어떻게 이해했는가" 만.

### 1. 타입은 `int`. Money VO 도입은 다시 미룸 (재확인)

`Order.totalPrice` / `Payment.amount` / `Product.price` 가 전부 `int` 라 통일성 측면에서 자연스러운 선택. VO 도입은 4개 필드 일괄 전환이 필요한 별도 series 규모라 본 task 범위 밖. 가격 정책 / 환불 / 할인 같은 연산이 들어오는 시점에 부채가 가시화될 가능성은 인지하고 있어야 한다.

### 2. Backfill 은 LEFT JOIN + COALESCE(p.price, 0). **review 가 결정을 바꾼 케이스**

처음 합의는 "INNER JOIN으로 `tbl_product.price` 채움. 운영 데이터에 hard-delete 가 없으니 안전" 이었다. PR review 단계에서 Gemini 가 `product_id` FK 가 V4 (PR #203) 에서 제거된 상태라 schema 가 hard-delete 를 막아주지 않는다는 점을 지적했다. INNER JOIN 이면 NULL 잔여로 `MODIFY ... NOT NULL` 단계에서 migration 이 실패할 수 있다.

**다시 본다면**: "운영 데이터에 hard-delete 없음" 같은 *데이터 분포 가정* 으로 가드를 생략하는 결정은, schema 차원 안전망 (FK 등) 이 없는 상태에서는 fragile 하다. FK 가 있을 때와 없을 때 backfill 의 보수성 cost 차이가 다르다. cross-aggregate FK 일괄 제거 series (#203) 의 부산물 — schema 가 제공하던 invariant 가 빠지면 migration / backfill 의 가정도 다시 봐야 한다. 이게 진짜 학습 포인트.

`0` fallback 의 의미: "product 가 존재하지 않아 결제 시점 가격을 재구성 불가" 의 sentinel. 0 은 비현실적 값이라 후속 통계 / 영수증 사용처에서 이상치로 잡힌다. NULL 잔여보다 검출 가능성이 높다.

### 3. ADR 본문 신규 X. task adr + 색인 표 한 줄

`docs/ADR.md` 상단 정책: "cross-cutting 결정은 본문 ADR, 도메인-specific 결정은 task adr". OrderItem 한정 결정이라 task adr 만. ADR-026 본문 신규는 cross-cutting 영역을 침범하는 일이라 의식적으로 피함.

### 4. 응답 DTO 노출은 본 PR 범위 밖

현재 OrderItem 을 직접 노출하는 응답 DTO 가 없었다 (`OrderCreateResult` / `OrderCancelResult` / `PaymentReadyService` 모두 `unitPrice` 미사용). 사용처 없는 상태에서 DTO 를 선제 추가하면 죽은 필드가 생긴다. 실제 소비처 (주문 상세 조회 / 영수증) 가 생길 때 별도 PR.

**다만** 큰 그림에서 "단가 컬럼은 DB 에 있는데 응답에 없다" 는 partial 변경 상태가 만들어졌다. 향후 노출 PR 에서 분리된 히스토리를 추적해야 하는 탐색 비용이 생긴 셈.

## 막힌 점

- 큰 막힘은 없었다.
- 다만 INNER JOIN backfill 의 hard-delete blind spot 을 본인이 사전 탐색에서 인지하지 못한 것이 review 에서 드러났다. 사전 탐색에서 "FK 가 V4 에서 제거된 상태" 와 "운영 데이터 가정" 을 결합해서 보면 잡을 수 있었던 케이스.

## 배운 것

- **FK 제거의 부산물**: schema 가 제공하던 referential integrity 가 빠지면, 그 FK 에 기대던 backfill / migration / 가정도 다시 봐야 한다. `product_id` FK 가 V4 에서 제거된 상태에서 product 참조 무결성을 가정한 INNER JOIN backfill 은 잠재 결함.
- **데이터 분포 가정의 fragility**: "운영 데이터에 X 없음" 으로 가드를 생략하는 결정은 schema 차원 안전망이 함께 있을 때만 안전. 안전망이 없을 때는 LEFT JOIN + COALESCE 같은 방어적 SQL 이 cheap insurance.
- **0 fallback 의 sentinel 가치**: NULL 보다 0 이 후속 사용처에서 이상치로 잡힐 가능성이 높다. NULL 은 "값이 없다" 라 검사 누락이 흔하지만, 0 은 "값이 비현실적" 으로 보여 발견된다.
- **task adr vs ADR 본문 분기**: cross-cutting vs 도메인-specific 의 경계가 `docs/ADR.md` 상단 정책으로 명시돼 있다는 점. 매번 ADR-N 본문을 늘리지 않고 색인 표를 늘리는 패턴이 누적 비용을 줄인다.
- **review 가 결정을 바꾼 케이스의 가치**: 단순 사실관계 정정이 아니라 "데이터 분포 가정" 자체를 보강해야 한다는 시각의 전환. 본 case 는 LLM review tool 의 진짜 가치가 어디인지 보여줌 — schema 차원 invariant 부재를 외부 시각으로 짚어줌.

## 다음 단계

지식 가치가 있는 미해결만:

- **Money VO 도입 검토**: `Order.totalPrice` / `Payment.amount` / `Product.price` / `OrderItem.unitPrice` 가 전부 `int`. 가격 정책 변경, 환불 / 할인 / 정산 같은 연산이 추가될 때 부채 가시화 가능성. 도입 시 4개 필드 일괄 전환이 한 series.
- **OrderItem.unitPrice 응답 DTO 노출**: 주문 상세 조회 / 영수증 응답이 추가되는 시점에. 그때 기존 row 의 "migration 시점 현재가 또는 0 sentinel" 한계도 같이 사용처 문서화 필요.
- **운영 DB 에서 V5 적용 시 0 sentinel row 가 실제로 발생하는지 추적**: 만약 발생한다면 데이터 정합성 분석 (어떤 product 가 hard-delete 됐는지) 의 단서가 됨.

## 참고

- 정본 결정 1~3: `commerce-backend/docs/tasks/order-item-price-snapshot/adr.md`
- 회고 (메타 결정 4 포함): `commerce-backend/docs/tasks/order-item-price-snapshot/retrospective.md`
- series 맥락: PR #199 (Stock) / #200 (Order) / #202 (Payment) / #203 (FK cleanup) 의 schema 무변경 원칙 종료 직후 첫 schema 변경 트랙
- ADR-020 (`commerce-backend/docs/ADR.md`): 신규 도메인의 cross-aggregate 참조는 ID
