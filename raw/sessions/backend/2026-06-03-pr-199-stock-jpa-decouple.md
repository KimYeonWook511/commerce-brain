---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 199 }
---

# Stock·StockHistory JPA 연관관계를 ID 참조로 분리 — schema 무변경 첫 sub-PR

직전에 "신규 도메인은 다른 aggregate 를 객체가 아니라 `Long` ID 로만 참조한다"는 원칙을
cart 도메인을 기점으로 세웠는데(cross-aggregate ID 참조 원칙), 그때 기존 도메인은 호환성
부담이 커 별도 트랙으로 미뤄뒀다. 이 세션은 그 별도 트랙(Issue #195)의 **첫 sub-PR
(#199)** 로, Stock·StockHistory 두 aggregate 를 대상으로 삼는다. Stock 의
`@OneToOne Product product` 와 StockHistory 의 `@ManyToOne Stock stock` 객체 참조를 각각
`Long productId` / `Long stockId` 필드로 바꾸고, 그로 인해 깨지는 application 응답 조립과
test fixture 를 함께 정리했다. **DB schema 도, API 응답 계약도 바꾸지 않고** JPA 매핑
차원에서만 연관관계를 끊는 것이 이번 series 의 뼈대다. 작업 명은 처음엔 "migration" 계열로
잡았다가 "decouple" 로 고쳤다.

## 결정한 것

### 1. 도메인 경계 단위로 sub-PR 을 쪼갠다 (한 PR 에 몰지 않고)

Stock / Order / Payment 세 도메인을 한 PR 에 묶는 안과, 도메인 경계마다 sub-PR 을 나누는
안 사이에서 **나누는 쪽**을 택했다.

- **한 번에 가는 안의 매력:** 컨텍스트 스위치를 아끼고, 일부 도메인만 바뀐 어중간한 중간
  상태를 안 만든다는 점이 매력적이었다.
- **기각 이유(각 도메인의 설계 결정이 다르다):** Stock 은 fetch join 이 없어 JPQL 수정과
  ID 필드 교체만으로 끝나는 단순한 구조지만, Order 는 `join fetch o.member`·
  `join fetch oi.product` 두 쿼리를 어떤 패턴으로 대체할지 **새로 정해야** 하고, Payment 는
  보상 흐름과 얽혀 정책 단위가 섞인다. 셋을 하나로 묶으면 서로 다른 성격의 결정이 같은
  diff 에 뒤엉켜 리뷰 비용이 크게 오른다.
- **커밋 컨벤션·리뷰 한계:** 묶으면 "역할이 다른 변경을 이유 없이 하나로 묶지 않는다"는
  커밋 컨벤션에 어긋나고, 세 도메인 fixture 까지 끌려 들어가면 리뷰가 감당하기 어려운 큰
  PR(대략 100개 넘는 파일로 추정)이 된다.
- **결정의 강한 신호:** 이 코드베이스의 PR 평균 단위가 애초에 좁다. 좁게 가는 게 이
  코드베이스의 결이다. 점진 진행이면 Stock 통과 뒤 Order 에서 테스트가 깨졌을 때 원인을
  Order 변경으로 바로 좁힐 수 있어 회귀 추적도 쉽다.

StockHistory 의 `@ManyToOne Stock` 도 cross-aggregate 참조로 보고 함께 끊었다 — StockHistory
는 audit 도메인이라 Stock 의 lifecycle 에 종속되지 않는 별도 aggregate 로 다루는 게 맞다.

### 2. StockHistory 의 productId 는 application 에서 외부 주입 (세 안 중 B)

연관관계를 끊으면 응답의 `productId` 를 채우던 `history.getStock().getProduct().getId()`
객체 traversal 이 불가능해진다. 이 응답 필드를 어떻게 살릴지 세 안을 놓고 골랐다.

- **(A) 응답에서 `productId` 필드 자체를 제거:** 가장 단순해 보이지만 **API 응답 계약 변경**을
  동반한다. 이번 PR 의 목적은 "연관관계 해제"이지 "응답 계약 정비(path productId 를 그대로
  되돌려주는 echo 응답 손질)"가 아니다. 그건 별도 정책 결정이라 섞으면 PR 메시지가 흐려지고
  되돌리기 단위도 얽힌다. 기각.
- **(C) `StockHistory.productId` 컬럼을 신설:** Flyway migration 이 딸려 오고, "schema 변경
  0건"이라는 이번 series 메타 원칙을 깬다. audit 의 본질이 아닌 정보를 컬럼으로 박는 부담도
  있다. 기각.
- **(B) `from(history, productId)` 로 application 이 path productId 를 외부 주입 — 채택.**
  StockHistory aggregate 의 본질은 "어떤 stock 에 어떤 변경이 일어났는가"이고 `stockId` 가
  그 invariant 다. productId 는 audit row 의 본질이 아니라 현재 조회 endpoint 의 path
  컨텍스트일 뿐이다. application 계층이 **audit row(엔티티) + path 컨텍스트(productId)** 를
  명시적으로 조립한다는 의도가 `from(history, productId)` 시그니처에 그대로 드러난다. 이는
  cross-aggregate ID 참조 원칙이 지목한 통증 중 하나 — 객체 그래프를 편하게 타고 들어가는
  탐색(`getStock().getProduct().getId()`)을 오용하던 패턴 — 을 코드 표면에서 없애는 것과도
  결이 맞는다.

실제 구현에서 `StockHistoryResult.from(StockHistory history)` 는 `from(history, productId)`
로 바뀌었고, 조회 서비스(`AdminStockService.getHistoriesByProductId`)가 path 로 들어온
productId 를 매핑 시점에 넣어준다. 반면 `AdminStockResult.from(Stock)` 은 Stock 엔티티가
`product_id` 컬럼을 직접 갖고 있으므로 시그니처를 그대로 두고 내부에서
`stock.getProduct().getId()` → `stock.getProductId()` 직접 접근으로만 바꿨다.

### 3. schema 변경 0건 — FK 제거는 series 전체가 끝난 뒤 일괄

이번 series 의 정체성은 "코드 차원 연관관계 해제만, DB schema 는 손대지 않는다"이다.

- **컬럼은 그대로:** `product_id BIGINT NOT NULL` / `stock_id BIGINT NOT NULL` 컬럼은 이미
  있으니 그대로 쓴다. 매핑에서 `@OneToOne`/`@ManyToOne` 만 떼고 `@Column` 매핑은 남긴다.
- **FK 는 남겨둔다:** `fk_stock_product_id`·`fk_stock_history_stock_id` FK 제약은 schema 에
  그대로 두고, JPA 가 더 이상 인식하지 않을 뿐이다. 도메인별 sub-PR 마다 FK 제거 Flyway V
  파일을 흩뿌리면 schema 변경의 정책 단위가 도메인별로 쪼개져 분산된다. Issue #195 원문도
  "모든 코드 마이그레이션 완료 후 FK 일괄 제거"를 명시했다. 그래서 FK 제거는 Stock/Order/
  Payment sub-PR 이 전부 머지된 뒤 **별도 트랙에서 Flyway migration 하나**로 몰아 처리한다.
- **이걸 가능하게 하는 핵심:** Hibernate `validate` 는 컬럼 이름·타입·nullable 단위로 검증하고
  **FK 제약의 존재 여부는 검증 대상이 아니다.** 그래서 매핑에서 연관관계를 떼도 FK 가 DB 에
  남아 있는 상태에서 validate 가 통과한다. 실제 MySQL 을 Testcontainers 로 띄우는 통합
  테스트로 통과를 확인했다.

### 4. task 이름을 "migration" → "decouple" 로 (오해를 이름에서 차단)

처음 잡은 작업 명에 "migration" 이 들어 있었는데, 이 단어가 **schema migration** 으로 오해를
부른다는 지적이 나왔다. 실제 작업은 schema 를 건드리는 게 아니라 JPA 엔티티의 association 을
푸는 것이라, 직설적인 "decouple" 로 바꿨다. 후속 sub-PR prefix 도 `order-jpa-association-decouple`,
`payment-jpa-association-decouple` 로 같은 결을 유지해 series 전체가 "association 해제"임을
이름만으로 읽히게 했다.

### 5. fetch join 대체 패턴은 이 PR 에서 선언하지 않는다

후속 Order sub-PR 에서 `join fetch o.member`·`join fetch oi.product` 두 쿼리가 깨질 예정이고,
그 대체 방안 후보는 (JPQL 명시 join + DTO projection) / (batch composition, cart 에서 쓴
패턴) / (read 전용 QueryService 분리) 셋이다. 하지만 **Stock/StockHistory 에는 fetch join
사용처가 없으므로** 이 PR 에서 대체 패턴을 미리 정하지 않았다. 일반 원칙("hot path 는
projection, 그 외는 batch composition" 류)을 지금 못 박으면, Order sub-PR 에서 사용처별로
분석한 결과가 원칙과 어긋날 때 억지로 따르거나 원칙을 소급 수정하는 비용이 생긴다. 실제
사용처를 보고 정하는 게 합리라 후속 PR 로 미뤘다.

## 막힌 점·해결

### 의미 없어진 단위 테스트 — 상수로 땜질 vs 근본 회복

PR 리뷰(AI 코드 리뷰)에서 `StockHistoryTest` 가 지적됐다. 기존 테스트는 id 가 없는 transient
`Stock` fixture(`createStock()`, `getId()` 가 null)를 만들어 `StockHistory` 에 넣고
`getStock()` 이 그 fixture 와 같은지 확인하는 식이었는데, 식별자 없는 객체를 비교하는 셈이라
검증으로서 알맹이가 없다는 지적이었다.

- **사용자 통찰(핵심):** 연관관계를 끊고 `getStock()` 을 `getStockId()` 로, fixture 를 `1L`
  상수로 바꿔도 **테스트의 본질적 가치는 안 올라간다.** StockHistory 에는 validation 도
  도메인 메서드도 없어서, 이 단위 테스트는 사실상 Lombok `@Builder` + `@Getter` 가 값을 그대로
  넣고 꺼내는지를 확인하는 데 그친다. 값을 넣고 그 값이 나오는지 보는 것 이상이 아니다.
- **결정:** 리뷰 권장대로 `stockId(1L)` 을 명시하고 쓸모없어진 `createStock()` fixture 를
  제거하는 선에서 마무리했다. 도메인 validation·팩토리를 도입해 테스트에 진짜 의미를 주는
  일은 이 PR 의 scope(association 해제) 밖이라 **별도 트랙으로 미뤘다.** 알맹이 없는 테스트를
  그대로 두는 게 마음에 걸렸지만, scope 를 지키는 가치가 더 크다고 봤다.

### 한 줄 시그니처 변경이 다른 도메인 test fixture 로 번짐

`Stock` builder 의 `product(Product)` 를 `productId(Long)` 로 바꾼 **한 줄**이, stock 도메인
바깥의 fixture 호출부까지 줄줄이 컴파일 오류를 냈다. Order 도메인 테스트 여덟 개
(`OrderCreateServiceConcurrencyTest`, `OrderConcurrencyServiceTest` 등)와 Product 도메인
테스트(`ProductQueryServiceTest`)까지 `Stock.builder().product(product)` →
`.productId(product.getId())` 로 전수 갱신해야 했다. 컴파일 오류를 따라가며 다 고쳤다.

## 배운 것

- **JPA association 해제와 schema 변경은 독립이다.** `product_id`·`stock_id` 컬럼은 그대로 둔
  채 객체 매핑만 뗄 수 있다. 이걸 성립시키는 축이 "Hibernate `validate` 가 FK 제약 존재를
  검증하지 않는다"는 점 — 덕분에 "schema 변경 0건" 메타 원칙이 코드만으로 완결된다.
- **작업 이름이 작업 본질을 드러내는 도구다.** "decouple" 같은 직설적 단어가 "schema
  migration" 이라는 오해를 이름 단계에서 차단한다. series 라면 일관된 prefix 까지 같이 본다.
- **API 계약 변경은 PR scope 와 분리한다.** echo 응답 정리 같은 정책 결정은 이 refactor 의
  목적이 아니므로 별도 트랙으로 뺀다. 한 PR 에 성격 다른 정책 목적을 섞으면 리뷰 부담이 커지고
  revert 단위가 흐려진다.
- **단위 테스트 가치의 진실.** transient `getId()` null 비교를 상수로 정비해도, 본질이
  Lombok getter/builder 검증에 그치면 그건 표면 정비일 뿐이다. cart 도메인의 CartItem
  테스트(validation + 팩토리 + 경계값·null 테스트)와 비교해 보면 차이가 분명해진다 — 진짜
  가치는 도메인 validation 을 도입하는 시점에 회복된다.
- **엔티티 시그니처 한 줄 변경의 fixture 침투 면적.** builder 시그니처를 바꾼 한 줄이 자기
  도메인 밖 Order/Product 테스트 여덟 개 넘는 fixture 로 번졌다. cross-aggregate 객체 참조가
  프로덕션 코드뿐 아니라 **test fixture 에서도 도메인 경계를 넘어 침투**해 있었다는 방증이고,
  ID 참조로 강제하려던 원칙이 오히려 그 의존 면적을 역으로 드러낸 셈이다. 후속 Order/Payment
  sub-PR 도 같은 양상이 예상되니, 영향 면적을 잡을 땐 자기 도메인 test 뿐 아니라 **의존 도메인
  test fixture 까지 미리 포함**하는 게 합리적이다.

## 미해결·열린 질문

- **StockHistory 도메인 validation/팩토리 도입** → 의미 있는 단위 테스트 회복 트랙. 이번 PR
  에서 scope 보호를 위해 의도적으로 미룬 부분.
- **Order/OrderItem sub-PR** — fetch join(`join fetch o.member`·`join fetch oi.product`)
  대체 패턴을 사용처별로 분석해(projection / batch composition / QueryService 분리 중) 정하고
  그 PR 에서 처음으로 명문화한다.
- **Payment sub-PR** — 보상 흐름·결제 사실 조회와 얽힌 부분을 별도로 검토해야 한다.
- **DB FK 일괄 제거** — 모든 sub-PR 머지 후 별도 트랙에서 Flyway migration 하나
  (`fk_stock_product_id`·`fk_stock_history_stock_id` 등)로 몰아 정리한다.
