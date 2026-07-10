---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 200 }
---

# pr-200-order-jpa-association-decouple

Issue #195 series 두 번째 sub-PR. Order/OrderItem 의 cross-aggregate JPA association (`@ManyToOne Member`, `@ManyToOne Product`) 을 해제하고 Long ID 로 전환. fetch join 대체 패턴을 사용처별 분석으로 처음 정립. PR review 두 라운드에서 existsById 회수 + unitPrice 후속 트랙 분리 (#201).

정본: `commerce-backend/docs/tasks/order-jpa-association-decouple/{prd,architecture,adr,api-spec,db-schema,retrospective}.md`. raw 는 내 이해 / 다시 본다면 만.

## 한 일

- harness 6단계로 Order/OrderItem JPA association 해제
- `Order.member → Long memberId`, `OrderItem.product → Long productId`. same-aggregate (`Order.orderItems`, `OrderItem.order`) 유지
- `JpaOrderRepository` JPQL 3개에서 cross-aggregate `join fetch` 제거. `findByMerchantPayKeyAndMemberId` 는 `join` 자체 제거 후 컬럼 비교로 단순화
- `PaymentReadyService` 에 batch composition + 외부 주입 (`findAllById` 1회 → productName Map → 응답 DTO `from(order, productNameMap)`)
- `Order.create(Long memberId)`, `addOrderItem(Long productId, int, int)` 시그니처 전환 + fixture 30+ 갱신 (order/payment/cart, concurrency 태그 포함)
- gemini review 1건 처리 (PaymentReadyService:82 NPE 가드 modify)
- subagent 독립 리뷰 → existsById 미사용 + unitPrice 비대칭 발견 → existsById 회수, unitPrice 후속 트랙 #201 분리

## 결정한 것

### fetch join 대체 — 사용처별 분석 (ADR 결정 2)

일반 원칙은 두 줄: **same-aggregate fetch 유지 / cross-aggregate fetch 제거**. 그 아래는 호출처에 맡긴다.

**내 이해**: 같은 메서드여도 호출처별 데이터 양상이 다르면 단일 원칙이 일부 호출처에 비효율을 강제한다. cancel/expiration 은 productId 만 필요한데 productName 까지 끌어오면 의미 없는 매핑 부담. PaymentReady 만 productName 노출이 필요하니 batch + 외부 주입을 거기에만 둔다. 메서드를 둘로 쪼개는 것보다 호출처별 후처리를 분리하는 게 깔끔.

### Long ID 시그니처 (ADR 결정 3)

`Order.create(Member)` → `Order.create(Long memberId)`, `addOrderItem(Product, int)` → `addOrderItem(Long productId, int, int)`. application 의 product 검증은 기존 `findById` / `findAllById` 흐름 그대로 유지. (existsById 신설 시도 → 회수, 막힌 점 참조.)

### PaymentReadyService NPE 가드 (gemini review modify)

정상 흐름에서 NPE 발생 불가 (soft delete 는 Map 에 포함, hard delete 는 코드 + FK 막음). ADR-011 안전망 정책상 reject 가 일관됨. 그러나 `OrderCreateProcessor` 가 똑같은 `findAllById + Map.get` 패턴에 똑같은 가드 사용 중. 본 PR 이 LAZY 자동 안전망을 명시 Map 조회로 바꿨다는 변경 의도와 가드 추가가 맞물려 modify.

**내 이해**: ADR 정책과 코드베이스 다른 곳의 실제 패턴이 충돌하면 후자가 의사 결정에 더 큰 정보를 준다. 단일 ADR 으로 모든 사례를 정당화하지 말고 "코드베이스 다른 곳이 같은 패턴에 어떻게 처리했나" 를 먼저 본다.

### OrderItem unitPrice 비대칭 — #201 분리

본 PR 이 `addOrderItem(productId, qty, unitPrice)` 시그니처를 도입했는데 OrderItem 에 가격 컬럼 없음 → totalPrice 누적에만 쓰이고 휘발. 결제 시점 가격 snapshot 부재. 컬럼 신설은 series "schema 변경 0건" 메타 원칙 충돌이라 #201 로 분리.

**내 이해**: 시그니처 전환 sub-PR 이 새로운 비대칭을 낳을 수 있다. 도메인 메서드 인자가 entity 컬럼과 1:1 매칭 안 되면 그건 빚이고, fixture 들이 `addOrderItem(1L, 1, 999)` 같이 productId/unitPrice 를 임의 분리 가능해져 코드 오용 표면이 넓어진다. ADR "후속 정비 항목" 만으로는 추적 안 됨 — issue 등록이 더 강한 tracking 장치.

## 막힌 점

**existsById 사후 회수**. 초기 사용자 결정 ("findById 말고 exists 같은 메서드를 만들어서 효율을 조금 높여보") 을 사용처 분석 없이 ADR 결정 3 으로 박아 넣음. step1 구현 / step2 sync-root-docs / step3 retrospective 모두 완료 후 review 단계에서 호출처 0건 발견 → 회수. 코드 2 파일 + 본 task 문서 5개 + 루트 ADR 색인까지 정리 (commit C1: refactor, C2: docs 로 분리).

회수 이유: 모든 Order 생성 경로가 `product.getPrice()` 를 `addOrderItem` 의 unitPrice 인자로 넘기므로 객체 로드가 어차피 필요. existsById 로 대체 가능한 사용처 0건. `CLAUDE.md` "사용하지 않는 코드 추가 금지" 와 직접 충돌.

**다시 본다면**: step Design 시점에 `grep "productRepository\." src/main` 한 줄로 끝났을 일. "findById 가 모든 컬럼 SELECT 라서 비효율" 이라는 정적 분석만으로 결정하면 동적 호출 흐름에서 객체 필드가 어차피 필요한 사실을 놓친다. 사용자 결정도 사용처 확인 후 적용.

## 배운 것

- **사용처별 분석을 단일 원칙보다 우선**: fetch join 대체에서 정립. Payment sub-PR 에도 같은 정신.
- **신설 API 의 정당성은 사용처가 입증**: 사용처 0건 = 신설 보류. 정적 분석만으로 결정 금지. grep 한 줄이 ADR/구현/회고 사후 회수보다 싸다.
- **review = self-review 보완**: gemini 의 일반 패턴 지적과 subagent 의 코드베이스 일관성 분석이 서로 다른 발견을 줌. 두 라운드 review 가 single PR 의 빚을 명시적으로 끌어올림.
- **defensive 가드 vs ADR 정책의 균형**: 단일 답 없음. 코드베이스 다른 곳의 실제 패턴이 단일 ADR 보다 의사 결정에 더 큰 정보를 줄 수 있다.

## 다음 단계

- **`payment-jpa-association-decouple`**: 본 PR 의 일반 원칙 (same-aggregate 유지 / cross-aggregate 제거) 적용. Payment 도메인 fetch join 사용처별 양상 분석 필요.
- **#201 OrderItem.unitPrice snapshot**: 컬럼 신설 vs Money VO 도입 결정, backfill 정책, 응답 DTO 영향 면적 분석.
- **FK 일괄 제거 트랙**: series "schema 변경 0건" 메타 원칙의 마무리 — Flyway 1개로 일괄.
- **미해결 도메인 질문**: Order 생성 시 Product soft delete 체크 부재. 결정으로 둔 건지 빠진 건지 별개 검토 (사용자가 "그런 제약은 따로 정해야지만 존재" 라는 언급으로 후자 가능성 시사).
