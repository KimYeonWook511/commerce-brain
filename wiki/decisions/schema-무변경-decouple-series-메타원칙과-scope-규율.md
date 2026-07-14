---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [jpa, decouple, schema, flyway, hibernate, sub-pr, pr-scope, fk, migration]
created: 2026-06-03
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-03-pr-199-stock-jpa-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]"
  - "[[raw/sessions/backend/2026-06-03-pr-203-cross-aggregate-fk-cleanup]]"
---

# 코드 association만 해제, DB schema 무변경 — decouple series의 메타원칙과 scope 규율

Stock(#199) / Order(#200) / Payment(#202) 세 sub-PR을 관통하는 실행 메타원칙과, 그것을 지탱한 scope 규율을 한곳에 정리한다. 개별 도메인의 대체 패턴·시그니처 결정은 각 sub-PR 노트로 갈라져 있고, 여기서는 series 전체가 공유한 **"어떻게 변경 없이 끊는가"와 "무엇을 이 PR에 넣지 않는가"**를 다룬다.

## 컨텍스트 — series 정체성

[[cross-aggregate-fk-to-id-마이그레이션-동기-전략]]에서 정한 방향(기존 도메인의 cross-aggregate 객체 참조를 [[cross-aggregate-fk-to-id-참조-컨벤션]]에 맞춰 `Long` ID로 소급 전환) 아래, Issue #195 트랙은 Stock·Order·Payment 세 도메인을 대상으로 실행에 들어갔다. series의 뼈대는 명확했다 — **JPA 매핑에서 `@ManyToOne`/`@OneToOne` 같은 객체 참조만 떼어내고, DB 컬럼·FK 제약·API 응답 계약은 전부 그대로 둔다.** 즉 "schema 변경 0건"으로 코드 차원의 결합만 끊는다.

## schema 변경 0건 — 무엇이 이걸 성립시키나

- **컬럼은 잔류:** `product_id`·`stock_id`·`member_id`·`order_id BIGINT NOT NULL` 컬럼은 이미 있으므로 그대로 쓴다. 매핑에서 연관 애노테이션만 떼고 `@Column` 매핑은 남긴다.
- **FK도 잔류, JPA가 인식만 안 함:** `fk_stock_product_id`·`fk_stock_history_stock_id`·`fk_order_member_id`·`fk_order_item_product_id`·`fk_payment_order_id` FK 제약은 schema에 그대로 두고, Hibernate가 더 이상 FK 정보를 인식하지 않을 뿐이다. DB 차원 referential integrity는 유지된다.
- **핵심 축 — Hibernate `validate`는 FK 존재를 검증하지 않는다:** `validate`는 컬럼 이름·타입·nullable 단위로만 검증하고 **FK 제약의 존재 여부는 검증 대상이 아니다.** 그래서 매핑에서 연관을 떼도 FK가 DB에 남은 상태에서 validate가 통과한다. 이 사실이 "schema 변경 0건" 메타원칙을 코드만으로 완결시키는 축이다. Stock sub-PR에서 실제 MySQL을 Testcontainers로 띄운 `integrationTest`로 통과를 확인했고, Order·Payment에서도 같은 패턴을 반복 검증했다([[flyway-도입-ddl-auto-validate-전환]]로 ddl-auto가 validate로 고정된 위에서 동작).

## 도메인 경계 단위 sub-PR 분할

세 도메인을 한 PR에 묶는 안과 도메인 경계마다 sub-PR을 나누는 안 사이에서 **나누는 쪽**을 택했다.

- **한 PR로 묶는 안의 매력:** 컨텍스트 스위치를 아끼고, 일부 도메인만 바뀐 어중간한 중간 상태를 만들지 않는다.
- **기각 이유 — 각 도메인의 설계 결정이 다르다:** Stock은 fetch join이 없어 JPQL 수정과 ID 필드 교체만으로 끝나는 단순 구조지만, Order는 `join fetch o.member`·`join fetch oi.product`를 어떤 패턴으로 대체할지 새로 정해야 하고([[cross-aggregate-fetch-join-대체-사용처별-분석과-응답-외부주입]]), Payment는 보상 흐름·결제 사실 조회와 얽힌다. 셋을 묶으면 성격이 다른 결정이 한 diff에 뒤엉켜 리뷰 비용이 크게 오른다.
- **커밋 컨벤션·리뷰 한계:** "역할이 다른 변경을 이유 없이 하나로 묶지 않는다"는 [[스코프-규율-한-pr-한-목적-인접부채-별도이슈-분리]]에 어긋나고, 세 도메인 fixture까지 끌려 들어가면 100개 넘는 파일로 추정되는 큰 PR이 된다.
- **결정의 강한 신호:** 이 코드베이스의 PR 평균 단위가 애초에 좁다. 점진 진행이면 Stock 통과 뒤 Order에서 테스트가 깨졌을 때 원인을 Order 변경으로 바로 좁힐 수 있어 회귀 추적도 쉽다.

StockHistory의 `@ManyToOne Stock`도 cross-aggregate로 보고 함께 끊었다 — StockHistory는 audit 도메인이라 Stock의 lifecycle에 종속되지 않는 별도 aggregate로 다루는 게 맞다.

## task 이름 migration → decouple

처음 잡은 작업명에 "migration"이 들어 있었는데, 이 단어가 **schema migration**으로 오해를 부른다는 지적이 나왔다. 실제 작업은 schema를 건드리는 게 아니라 JPA 엔티티의 association을 푸는 것이라, 직설적인 "decouple"로 바꿨다. 후속 prefix도 `order-jpa-association-decouple`·`payment-jpa-association-decouple`로 통일해 series 전체가 "association 해제"임을 이름만으로 읽히게 했다. 작업 이름이 작업 본질을 드러내는 도구라는 사례.

## scope 규율 — 별도 트랙으로 미룬 것들

series를 관통한 규율은 "association 해제와 성격이 다른 정책 결정은 이 PR에 넣지 않는다"이다. 실제로 미룬 항목:

- **echo 응답 계약 정비:** path의 productId를 그대로 되돌려주는 echo 응답을 손질하는 것은 별도 정책 결정이다. 섞으면 PR 메시지가 흐려지고 revert 단위가 얽힌다.
- **단위테스트 근본 회복:** Stock sub-PR 리뷰에서 `StockHistoryTest`가 식별자 없는 transient fixture를 비교하는 알맹이 없는 테스트로 지적됐지만, 연관을 끊고 `stockId(1L)` 상수로 정비해도 **본질 가치는 안 오른다** — StockHistory에 validation도 도메인 메서드도 없어 사실상 Lombok `@Builder`+`@Getter` 검증에 그친다. 진짜 가치는 도메인 validation을 도입하는 시점에 회복되며(그건 scope 밖), [[cart-도메인-골격-cartitem-단일-aggregate]]의 CartItem 테스트(validation+팩토리+경계값·null 테스트)와 대비하면 차이가 분명하다. 리뷰 권장 선에서만 마무리하고 근본 회복은 미뤘다.
- **API 계약 변경 일반:** 응답에서 필드를 제거하거나 신설하는 계약 변경은 refactor 목적이 아니므로 전부 뺐다.

이 scope 규율은 series의 리뷰 부담을 낮추고 revert 단위를 선명하게 유지한 핵심이다.

## 트레이드오프·리스크 — code-schema lag

FK를 sub-PR마다 흩뿌리지 않고 series 끝으로 몰아 [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]] 트랙(PR #203)에서 단일 Flyway로 일괄 제거했다. 이 선택의 대가로 **과도기 lag**가 생겼다 — 코드는 이미 association을 해제했는데 DB FK 5건은 아직 남은 불일치 상태가 약 하루 이어졌다.

- **왜 감수하나:** FK를 도메인별로 부분 제거하면 일부 도메인만 FK 없는 불균형 schema가 되고, V 파일을 sub-PR마다 흩으면 schema 변경의 정책 단위가 분산된다. Issue #195 원문도 "모든 코드 마이그레이션 완료 후 FK 일괄 제거"를 명시했다.
- **lag 기간 기술 위험은 낮다:** `validate`는 FK 존재를 안 보고, FK가 남은 동안 부모 row 삭제 시도는 DB가 거부해 공통 안전망 500으로 위임되는 의도된 동작이었다.
- **또 하나의 침투 면적:** 엔티티 builder 시그니처 한 줄(`product(Product)` → `productId(Long)`) 변경이 자기 도메인 밖 Order/Product 테스트 여덟 개 넘는 fixture로 번졌다. cross-aggregate 객체 참조가 test fixture에서도 도메인 경계를 넘어 침투해 있었다는 방증이고, 영향 면적 산정 시 의존 도메인의 test fixture까지 미리 포함해야 한다는 교훈.

## 미해결·후속

- **lag 허용 기간 표준의 ADR 정립:** 이번 lag는 표본 1건이라 표준화를 보류했다. 다른 cross-aggregate 정리 series에서 반복 확인되면 그때 일반화한다.
- **StockHistory 도메인 validation/팩토리 도입** → 의미 있는 단위 테스트 회복 트랙.
- **FK 실제 제거의 운영 DB 배포 절차:** local/test까지만 완료됐고 운영 배포 시점·무중단·롤백은 [[fk-drop-후-잔류-index-unique-유지와-innodb-비대칭]]에서 별도 축으로 미뤘다.

## 근거

- [[raw/sessions/backend/2026-06-03-pr-199-stock-jpa-decouple]]
- [[raw/sessions/backend/2026-06-03-pr-200-order-jpa-association-decouple]]
- [[raw/sessions/backend/2026-06-03-pr-202-payment-jpa-association-decouple]]
- [[raw/sessions/backend/2026-06-03-pr-203-cross-aggregate-fk-cleanup]]
