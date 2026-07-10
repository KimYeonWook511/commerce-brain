---
platform: backend
author: KimYeonWook511
created: 2026-06-03
origin:
  - { type: pr, repo: commerce-backend, ref: 199 }
---

## 한 일

- Issue #195 의 첫 sub-PR (#199 `refactor: Stock·StockHistory JPA 연관관계 분리`) 진행·머지
- task: `stock-jpa-association-decouple`, branch `refactor/stock-jpa-association-decouple`
- 본질: JPA `@OneToOne`/`@ManyToOne` 객체 참조 → `Long` ID 필드. application 응답 조립을 외부 주입 패턴으로
- DB schema·API 응답 계약은 그대로. FK 제약 유지

## 결정한 것

정본: `commerce-backend/docs/tasks/stock-jpa-association-decouple/adr.md`, `commerce-backend/docs/tasks/stock-jpa-association-decouple/retrospective.md`. 아래는 내 이해·다시 본다면.

- **도메인별 sub-PR 분리 vs 한 번에**: 한 번에 가는 게 컨텍스트 스위치 절약·중간 상태 어색 해소 측면에서 매력적이었지만, 각 도메인이 별도 설계 결정을 가짐 (Order fetch join, Payment 보상). 묶으면 commit-conventions ("역할이 다른 변경 묶지 않기") 위반 + 100+ 파일 PR 의 review 한계. 코드베이스 PR 평균 단위가 좁다는 게 결정의 강한 신호였음.
- **StockHistory productId 외부 주입 (B)**: 응답 필드 제거 (A) 는 API 계약 변경이라 echo 응답 정리라는 별도 정책. 컬럼 신설 (C) 는 schema 변경 동반이라 PR series 메타 원칙 위반. 결국 (B) — application 조립 패턴이 데이터 소스 (audit row + path 컨텍스트) 가 어떻게 응답에 합쳐지는지 코드 표면에 드러내는 게 ADR-020 통증 #1 ("편한 탐색 오용") 정신에 맞음.
- **schema 변경 0건 메타 원칙**: 이번 series 의 정체성. Flyway V 파일이 도메인별로 양산되는 패턴 자체를 회피. FK 제거는 모든 sub-PR 끝난 뒤 별도 Flyway migration 으로 일괄. Hibernate `validate` 가 FK 제약 검증 안 한다는 점이 이걸 가능하게 함.
- **task name "migration" → "decouple"**: 사용자가 짚은 통찰 — "migration" 단어가 schema migration 으로 오해 부름. 실제 작업은 JPA Entity association 해제. 후속 PR prefix 도 `order-jpa-association-decouple`, `payment-jpa-association-decouple` 로 일관.
- **fetch join 대체 패턴 본 PR 에 미선언**: Stock 도메인엔 fetch join 없음. Order/OrderItem 의 `join fetch o.member`, `join fetch oi.product` 두 쿼리 대체는 Order sub-PR 에서 사용처별 (P1 DTO projection / P2 batch composition cart 패턴 / P3 read QueryService) 결정. 미리 박으면 후속 PR 자유도 좁아짐.

## 막힌 점

- PR review (Gemini) — `StockHistoryTest` 의 transient `Stock.getId()` = null 로 인한 null vs null assertion 지적
- 사용자 통찰: 1L 상수로 교체해도 본질적 가치 안 올라감. StockHistory 자체에 validation/도메인 메서드 없음 → 단위 테스트가 사실상 Lombok `@Builder` + `@Getter` 동작 검증
- 결정: review 권장 그대로 1L 명시 + `createStock()` fixture 제거. 도메인 validation 도입은 scope 이탈이라 별도 트랙으로 미룸. 의미 없는 테스트를 그대로 두는 게 마음에 걸리지만 scope 보호가 더 큰 가치

## 배운 것

- **JPA association 해제와 schema 변경은 독립**. 컬럼 (`product_id BIGINT NOT NULL`) 은 그대로 두고 객체 매핑만 해제 가능. Hibernate `validate` 가 FK 제약을 검증하지 않는다는 점이 핵심 — 이 점이 "schema 변경 0건" 메타 원칙을 성립시킴
- **task name 이 작업 본질을 드러내는 도구**. 직설적 단어 ("decouple") 가 오해 (schema migration) 를 차단. PR series 일관 prefix 도 같이 고려
- **API 계약 변경을 PR scope 와 분리**. echo 응답 정리 같은 정책 결정은 본 refactor 정책 목적이 아니므로 별도 트랙. 한 PR 에 다른 정책 목적 섞으면 review 부담·revert 단위 흐림
- **단위 테스트 가치의 진실**. transient getId() null assertion 정비도 본질이 Lombok 검증에 그치면 표면 정비에 불과. CartItem 패턴 (validation + 팩토리 + 경계값/null 테스트) 비교로 명확해짐 — 가치 회복은 도메인 validation 도입 시점
- **한 도메인 entity 시그니처 변경의 fixture 침투**. Stock builder `product(Product)` → `productId(Long)` 한 줄 변경이 Order/Product 도메인 테스트 8+개의 fixture 호출부 갱신으로 번짐. ADR-020 의 cross-aggregate ID 참조 강화가 오히려 fixture 의존 면적을 드러낸 역설. 후속 Order/Payment sub-PR 도 동일 패턴 예상 — 영향 면적 산정 시 자기 도메인 test 외에 의존 도메인 test fixture 까지 미리 포함하는 게 합리

## 다음 단계

- StockHistory 도메인 validation/팩토리 도입 → 의미 있는 단위 테스트 회복 트랙. 이번 PR 에서 의도적으로 미룬 부분
- Order/OrderItem sub-PR — fetch join 대체 패턴 사용처별 분석 + ADR 정립 (P1/P2/P3 결정)
- Payment sub-PR — 보상 흐름·결제 사실 조회와 얽힌 부분 검토
- 모든 sub-PR 머지 후 DB FK 일괄 제거 별도 트랙 (Flyway V 파일 1개)
