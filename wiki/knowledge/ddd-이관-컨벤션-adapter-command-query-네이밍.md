---
type: topic
status: stable
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [ddd, hexagonal, adapter, repository, naming-convention, cqrs, refactoring]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-payment-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-product-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-stock-domain-overview]]"
  - "[[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]"
---

# DDD 이관 공통 컨벤션 — adapter·command/query·의도기반 메서드명·legacy 분리

## 한 줄 정의

order·payment·product·stock·auth 다섯 도메인의 DDD 이관 회고에서 반복 확정된 공통 컨벤션을 한자리에 모은 노트. 개별 도메인 노트가 위임하는 "왜 이렇게 구조를 잡았나"의 근거 정본이다.

## command/query·유스케이스 단위 분리

- application 서비스는 command/query(또는 유스케이스 책임 단위)로 쪼갠다. 같은 도메인이라도 단일 service에 둘을 끼우지 않는다.
- product: 공개 조회 `ProductQueryService`(readOnly, 비로그인) / 관리자 쓰기 `AdminProductService`(ROLE_ADMIN, 쓰기 트랜잭션)로 분리 — 호출자 성격·트랜잭션·DTO·로깅 정책이 다르다([[product-공개query-관리자command-서비스-분리]]).
- order: `OrderCommandService` 한 덩어리 대신 생성/취소/만료/조회/동시성 단위로 분리([[order-도메인-구조-개요]]).
- payment: APPROVE/CANCEL 흐름별 attempt service 분리. 이 흐름 분리가 도메인 메서드의 type 가드를 자연 강제해 mark 4개를 succeed/fail 2개로 통합하게 만든 진화도 같은 원리다.

## 의도 기반 repository 메서드명 + Adapter 패턴

- domain repository 포트는 의도 기반 메서드명을 갖는다: `findVisibleProduct`, `findNotDeletedProduct`. `findByDeletedAtIsNullAndStatusIn` 같은 파생 쿼리명은 영속성 조건이 application으로 새는 현상이다. JPA 조건은 infrastructure의 `@Query`로 분리한다.
- 포트를 `JpaRepository`에 직접 상속시키지 않고 `Adapter(implements)`로 둔다. `JpaRepository`를 함께 상속하면 `save` 시그니처가 Spring Data generic method에 묶인다. `ProductRepositoryAdapter implements ProductRepository`로 두면 도메인 포트는 도메인 객체만 다루고 JPA 디테일은 infrastructure에 격리된다.
- 트레이드오프: 레이어가 한 겹 늘고, 메서드 추가 시 포트 + adapter + `Jpa*Repository` 3곳을 수정한다. 테스트 fixture는 예외 — `@DataJpaTest`·통합·동시성 테스트는 `saveAll`/`flush`가 필요해 `Jpa*Repository`를 직접 써도 되게 허용한다.

## port-adapter는 DDD 학습의 자연 수렴

솔직한 자기 인식: 헥사고날 아키텍처를 정식 학습해 도입한 게 아니다. DDD 개념에 맞게 설계하다 보니 port-adapter 구조로 자연 수렴했다. 정식 헥사고날 적용이라기보다 DDD 학습의 자연스러운 결과물이다. stock·order·payment·product 모두 같은 adapter 방식으로 정착했다.

## DDD 이관 ≠ legacy 삭제(커밋 분리)

- DDD 구조 추가와 legacy 삭제를 같은 커밋/PR에 섞지 않는다. 이유는 단순 리뷰 부담 분산을 넘어 **legacy 참조가 남았는지 검증할 시간을 벌기 위함**이다.
- 한 PR로 묶으면 production 코드와 테스트 fixture 양쪽에서 legacy 참조가 빠졌는지 검증이 충분치 않다. 분리하면 이관 PR 머지 → 안정화 → 검색(`rg "com\.commerce\.product\.(service|controller|repository)"`)으로 잔존 참조 확인 → 별도 PR로 legacy 삭제, 순서를 밟는다.
- legacy controller는 bean 등록만 끊고 코드는 일단 둔 뒤, 별도 작업으로 production/test 참조를 정리한다.

## 책임 중심 네이밍·계층 접미사 회피

- 클래스명은 책임 중심: `OrderStockService`(주문에서 호출된다는 관점)보다 `StockInventoryService`(재고 책임)를 택한다.
- 구현 전략은 public method 이름에 노출하지 않는다: `decreaseWithPessimisticLock` 금지 → `decrease`. 비관락은 구현 세부라 내부 주석으로 남기고 호출부가 그 세부에 묶이지 않게 한다. (단, 실험·비교가 목적인 `OrderConcurrencyService`는 strategy를 이름에 드러내는 게 오히려 명확해 의도적 예외를 둔다 — [[order-concurrency-service-학습코드-격리]]. 이 노트의 금지 원칙에 대한 정당한 반례다.)
- application 계층 클래스명에 `ApplicationService` 같은 계층 접미사를 반복하지 않는다. 패키지 경로가 이미 계층을 표현한다.
- 테스트는 application service 책임 단위로 분리(Admin/Inventory/Concurrency). 어느 서비스의 계약이 깨졌는지 빨리 파악되고 legacy 삭제 시 영향 범위도 좁아진다.
- 도메인 간 협력은 application service끼리 한다 — infrastructure의 repository를 가로질러 호출하지 않는다(예: `auth → member.application`, [[인증-패키지-경계-auth-member-security-분리]]).

## 엔티티 생성 계약 변경의 파급

`Product.builder()` 회귀 교훈: `ProductStatus`를 필수화하자 주문·결제·재고 테스트 fixture가 모두 영향받았는데, step Acceptance Criteria가 `com.commerce.product.*`만 실행해 즉시 못 잡았다. 교훈 — 엔티티 생성 계약 변경은 작은 도메인 변경이 아니라 전체 fixture 변경으로 봐야 한다. shared domain 엔티티 계약 변경은 전체 테스트 실행이 AC에 포함돼야 하고, `rg "Product.builder"` 같은 파급 범위 탐색을 step에 명시한다.

## 관련 도메인 링크

- [[order-도메인-구조-개요]]
- [[payment-도메인-구조-개요]]
- [[product-도메인-구조-개요]]
- [[stock-도메인-구조-개요]]
- [[auth-member-security-도메인-구조-개요]]

## 열린 질문

- ArchUnit 등 아키텍처 테스트로 컨벤션(포트 직접 호출 차단 등)을 CI 강제할지 — 학습 단계라 현재는 문서·JavaDoc·단일 호출처로 대신하고, 여러 도메인 아키텍처 테스트와 일관되게 도입하는 시점을 후속으로 미룸.

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
- [[raw/sessions/backend/2026-05-29-payment-domain-overview]]
- [[raw/sessions/backend/2026-05-29-product-domain-overview]]
- [[raw/sessions/backend/2026-05-29-stock-domain-overview]]
- [[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]
