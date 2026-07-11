---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 202 }
---

# Payment 의 Order 연관관계를 Long ID 로 해제 — cross-aggregate JPA 분리 series 마무리

기존 도메인들의 cross-aggregate JPA 연관관계를 걷어내고 상대 aggregate 를 `Long` ID 로만 참조하도록
바꾸는 작업(Issue #195)의 **세 번째이자 마지막 sub-PR**. 앞선 Stock(#199) / Order(#200) 두 sub-PR
에서 이미 메타 원칙(코드 association 만 해제, DB schema 무변경)과 Long ID 시그니처 패턴이 정립돼
있어서, Payment 에서 새로 결정할 것은 사실상 하나였다. `Payment` 가 갖던 유일한 cross-aggregate
연관관계인 `@OneToOne(LAZY) Order` 를 `Long orderId` 로 전환하고, 그 여파로 결제 완료 엔티티를 만드는
팩토리 메서드 시그니처만 정리했다. 이 PR 머지로 Issue #195 를 close 하고 series 를 끝냈다.

이 세션의 성격은 "새 결정을 내리는" 것보다 "선행 두 sub-PR 에서 세운 원칙이 Payment 도메인을 어디까지
덮는지 검증하는" 쪽에 가까웠다. 그래서 아래 결정 대부분은 "이건 이미 정해진 원칙으로 커버되니 이번엔
안 건드린다" 형태다.

## 결정한 것

### 1. `Payment.order (@OneToOne LAZY Order)` → `Long orderId`

Payment 엔티티가 Order 를 객체로 물고 있던 유일한 cross-aggregate 참조를 값 필드로 바꿨다. 매핑상
`@OneToOne(fetch = LAZY)` + `@JoinColumn(name = "order_id", nullable = false, foreignKey = fk_payment_order_id)`
이던 것을 `@Column(name = "order_id", nullable = false) Long orderId` 로 교체했다(코드 확인함).

- **왜 이 참조가 대상이었나:** Order 와 Payment 는 단방향 `@OneToOne`, cascade 없음, lifecycle 결합이
  약하다 — Order 가 만료/취소돼도 Payment 이력은 남아야 하고 그 반대도 마찬가지다. 그래서 둘은 별
  aggregate 이고, "같은 aggregate 안의 root-child 객체 참조는 허용"이라는 예외에도 해당하지 않는다.
  ID 참조로 바꾸는 게 자연스러운 유일한 후보였다.
- **schema 는 건드리지 않는다:** `order_id BIGINT NOT NULL` 컬럼은 이미 존재하고, DB FK 제약
  `fk_payment_order_id` 도 그대로 남긴다. JPA 매핑에서 `@OneToOne` 만 떼면 Hibernate 는 더 이상 FK
  정보를 인식하지 않지만, DB 차원 referential integrity 는 유지된다. 이건 series 전체를 관통하는 메타
  원칙(코드 association 해제만, schema 변경 0건)을 그대로 따른 것이고, 앞선 Stock/Order sub-PR 에서
  Hibernate `validate` 통과를 Testcontainers MySQL 통합 테스트로 이미 검증한 패턴이다.

### 2. `Payment.createCompleted` 시그니처를 Long ID + `amount` 명시로 전환 — 정책이 표면으로 드러난 게 진짜 소득

결제 승인 완료 엔티티를 만드는 정적 팩토리 `Payment.createCompleted` 를, Order 객체를 받던 것에서
`(Long orderId, int amount, PaymentProvider provider, String merchantPayKey, String pgPaymentId, LocalDateTime approvedAt)`
로 바꿨다(코드 확인함).

- **검토한 대안 두 가지:**
  - (A) 시그니처 유지 — application 이 Order 를 로드해서 객체째 도메인에 넘기고, 도메인 안에서
    `order.getTotalPrice()` 를 호출. `@OneToOne` 을 뗀 뒤에도 도메인이 Order 를 받아서 하는 일이
    `getTotalPrice()` 호출 하나뿐이라, 외부 객체 의존 제거라는 Long ID 전환의 취지를 절반만 달성한다.
  - (B, 택함) Long ID 시그니처 — application 이 `order.getId()` / `order.getTotalPrice()` 를 추출해서
    ID + 단순 값만 넘긴다. 도메인은 외부 객체 의존 0.
- **B 를 택한 이유:** (1) 호출처(`PaymentApprovalService.completeApprovedPayment`, 결제 승인 완료를
  저장하는 application 메서드)가 어차피 같은 트랜잭션에서 Order 를 `findByMerchantPayKeyForUpdate` 로
  잠금 로드해 `order.completePayment()` 를 호출하고 `getTotalPrice()` 를 읽는다. Order 객체는 이미
  손에 있으니 추가 조회 비용이 0이다. (2) 도메인이 ID + `int amount` 만 받으면 unit test fixture 에서
  Order 객체를 조립할 필요가 없어 부담이 준다.
- **의도하지 않았던, 그러나 더 의미 있던 부가 효과:** 이전엔 `Payment.createCompleted(order, ...)` 내부
  에서 `amount = order.getTotalPrice()` 가 호출됐다 — 즉 "Order 의 totalPrice 를 결제 시점 amount 로
  쓴다"는 **정책이 도메인 안에 숨어** 있었다. `amount` 를 호출자가 명시적으로 넘기게 바꾸니(코드상
  실제로 `Payment.createCompleted(order.getId(), order.getTotalPrice(), ...)` 로 호출됨) 이 정책이
  application 호출 라인에 드러난다. 본래 의도는 Long ID 일관성이었는데, 이 표면화 효과가 덤으로 더
  값졌다.

### 3. 존재 검증용 리포지토리 메서드는 신설하지 않는다 — 앞선 회수 학습을 그대로 계승

`OrderRepository` 에 Order 존재 검증용 신규 메서드(`existsById` 류)를 추가하지 않았다(PR 에서
`OrderRepository` 변경 0건 확인함).

- **왜:** 호출처가 같은 트랜잭션에서 Order 의 다른 필드(`completePayment()`, `getTotalPrice()`,
  `getId()`)를 함께 쓰므로, Order 객체 로드는 어차피 필요하다. `findByMerchantPayKeyForUpdate` 가
  없으면 `ORDER_NOT_FOUND` 를 던져 존재 검증이 이미 포함된다. 별도 존재 검증 메서드를 신설하면 그
  사용처가 0건이 된다.
- **패턴 반복:** 앞선 Order sub-PR 에서도 존재 검증을 위해 `existsById` 류 메서드를 신설했다가 같은
  이유(호출처가 이미 객체 필드를 함께 씀 → 신설 메서드 사용처 0)로 회수한 적이 있다. "방어 검증을
  위해 메서드를 신설"하는 함정이 두 번 연속 같은 모양으로 나왔다. 프로젝트 컨벤션으로 합의된 "과한
  추상화(예: 불필요한 sealed interface) 금지"와 같은 결의 판단이다.

### 4. Gemini 자동 코드 리뷰의 방어 검증 제안을 reject

Gemini Code Assist 가 `createCompleted` 의 `orderId` / `amount` 에 null·범위 방어 검증을 두라고
권했으나 기각했다. *(외부 리뷰 도구와의 상호작용 내역이라 git 으로 대조 불가 — 원저자 기록 보존. PR
에서 실제로 해당 가드가 추가되지 않은 것은 확인함.)*

- **기각 근거:** (1) 이 코드베이스의 정적 팩토리들(`Order`, `Stock`, `StockHistory`)은 null/range
  guard 로 `IllegalArgumentException` 류를 던지지 않는 게 일관된 컨벤션이다. (2) 호출처가 한 곳이고
  거기서 영속된 Order 엔티티의 필드를 전달하므로 애초에 발생 불가능한 시나리오다 — 시스템 경계가
  아니라 내부 호출이다. (3) 프로젝트 컨벤션 문서의 "불필요한 추상화를 피한다" 원칙 + 시스템 원칙
  "발생할 수 없는 시나리오에 검증을 넣지 말고, 검증은 시스템 경계에서만 한다".
- 이 세 축(다른 도메인 엔티티의 일관 컨벤션 확인 → 호출처가 신뢰 가능한 내부인지 경계인지 판단 →
  프로젝트/시스템 원칙 인용)으로 묶으면 이런 일반론적으로 옳아 보이는 자동 리뷰 제안에 일관되게
  답할 수 있다.

### 5. 나머지는 선행 원칙으로 커버되니 이번엔 안 건드린다 — 탐색으로 조기 확정

탐색 단계에서 사용자가 처음 지목한 7개 정밀 조사 영역(cross-aggregate 변경 면적 / 보상 흐름 /
concurrency / PG / 응답 DTO / aggregate 경계 / fixture 면적)을 grep 으로 정량 확인했더니, 6개가 "변경
없음"으로 떨어지고 결정해야 할 항목이 위의 시그니처 1건으로 수렴했다.

- **fetch join 대체 정책 — 이번엔 새로 안 정한다:** `JpaPaymentRepository` 에 fetch join JPQL 이
  0건이고 derived query(`findByMerchantPayKey`, `existsByMerchantPayKeyAndStatus`)만 있다(코드 확인함).
  앞선 Order sub-PR 이 네 사용처를 분석해 세운 일반 원칙("same-aggregate fetch 는 유지, cross-aggregate
  fetch 는 제거, 데이터 양상별로 batch composition 또는 컬럼 직접 사용")을 인용만 하면 된다. 그
  원칙은 Order 태스크 문서에서 단일 관리한다.
- **PaymentAttempt 는 범위 밖:** `PaymentAttempt` 는 이미 `merchantPayKey` / `pgPaymentId` / `provider`
  식별자 기반이고 Payment 와 객체 참조가 없다 — 해제할 JPA association 자체가 없다. 이번 sub-PR 의
  정책 목적(cross-aggregate association 해제)과 무관하다.
- **응답 echo 정리도 별도 트랙:** Payment 응답(`NaverPayApproveResponse`)은 Payment 자신의 필드
  (`pgPaymentId`, `status`)만으로 조립된다. Stock sub-PR 에서 시작해 Order sub-PR 에서 확장된 "응답
  DTO 에 외부 컨텍스트를 application 이 명시 주입" 패턴을 새로 적용할 사용처가 없다. 앞선 두 sub-PR
  도 "응답 echo 정리는 별도 트랙"으로 뒀고 그대로 계승했다 — association 해제와 응답 계약 정비를 한
  diff 에 섞으면 PR 의 메시지가 흐려진다.

변경 면적은 `Payment` 엔티티 1개 + `PaymentApprovalService.completeApprovedPayment` 호출부 1개 +
test fixture 5개 파일로 좁게 유지됐다(PR stat 으로 확인: 도메인/서비스 각 1 파일, 테스트 5 파일).

## 배운 것

- **도메인 시그니처 전환의 부가 효과는 의도 밖에서 온다.** Long ID 일관성을 노리고 시그니처를 바꿨더니
  "결제 시점 amount = Order.totalPrice" 정책이 도메인 내부에서 application 표면으로 드러나는 효과가
  따라왔다. 이런 효과는 설계 단계에서 의도로 미리 적기보다 회고에서 사후 정리하는 게 더 정직하다.
  도메인 무관한 일반 원칙.
- **일반론적으로 옳아 보이는 자동 리뷰 제안에 답하는 정형화된 방법.** (1) 다른 도메인 엔티티의 일관
  컨벤션을 확인하고, (2) 호출처가 시스템 경계인지 내부인지 판단하고, (3) 프로젝트/시스템 원칙을
  인용한다. 세 축을 묶으면 일관되게 reject/accept 를 정당화할 수 있다.
- **series 의 마지막 PR 은 새 결정을 내리는 단계가 아니라 "선행 원칙이 어디까지 커버하나"를 검증하는
  단계에 가깝다.** 탐색 초입에 변경 면적을 grep 수치로 정량화하니 결정 비용이 크게 줄었다. 다음
  series 의 마지막 sub-PR 은 "선행 PR 에서 정립된 원칙이 커버하는 영역"을 먼저 grep 으로 빼내는 식으로
  탐색을 더 빨리 좁힐 수 있다.

## 미해결·열린 질문

- **Payment ↔ PaymentAttempt 의 aggregate 경계가 명시된 적 없다.** 둘은 `merchantPayKey` 를 공유 키로
  결합돼 있는데, 이 결합이 어디에도 명시적으로 표현돼 있지 않다. 같은 aggregate 인지 별 aggregate
  인지, `merchantPayKey` 가 도메인 개념인지 단순 식별자인지 미정. 현재는 application 계층이
  `merchantPayKey` 를 키로 두 엔티티를 함께 다루는 방식으로만 표현된다. 이번 series 의 정책 목적과
  무관해 별도 트랙으로 미뤘고, 후속 정비 트랙은 아직 형식화되지 않았다.
- **결제 시점 가격 snapshot 미해결 (Issue #201).** Order sub-PR 에서 `addOrderItem` 의 `unitPrice`
  인자가 `OrderItem` 컬럼에 저장되지 않는 문제가 남았다. 이번 PR 에서 `amount = order.getTotalPrice()`
  를 명시 전달하게 됐지만, 그 `totalPrice` 가 OrderItem 단가의 누적인지 결제 시점 스냅샷인지는 여전히
  모호하다. e-commerce 표준(결제 시점 단가 snapshot 필수)과 어긋난다.
- **코드-schema 과도기 lag 를 언제까지 허용할지에 대한 정책 부재.** 이 series 로 코드 association 은
  걷어냈지만 다섯 FK(`fk_stock_product_id`, `fk_stock_history_stock_id`, `fk_order_member_id`,
  `fk_order_item_product_id`, `fk_payment_order_id`)가 DB schema 에 남아 JPA 가 인식하지 않는 과도기
  상태다. 이 FK 일괄 제거는 별도 issue/PR 로 발행할 예정이지만(Flyway migration 일괄 발행은 실행
  작업), "코드 해제와 schema 정리 사이 lag 를 얼마나 허용할지"의 표준은 어디에도 명시돼 있지 않다.
  다른 series 에서도 반복 등장할 결정 항목이다.
