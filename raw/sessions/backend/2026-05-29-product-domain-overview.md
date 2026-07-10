---
platform: backend
author: KimYeonWook511
created: 2026-05-29
---

## 도메인 개요

- 책임: 상품 마스터(이름·가격·설명·이미지·판매 상태) + 공개 노출 정책 + soft delete. **재고는 책임 분리** (stock 도메인).
- 엔티티: `Product` 단일. `Product : Stock = 1:1`이지만 stock 쪽이 FK를 가지고 product는 stock을 알지 않는다. (의존 방향은 application 계층 `ProductQueryService` → `StockRepository`)
- application 서비스 2개 — **공개 query / 관리자 command 분리**:
  - `ProductQueryService` — 공개 목록·상세 조회. 비로그인 OK.
  - `AdminProductService` — 관리자 등록·수정·soft delete.
- ProductStatus enum 3개: `ON_SALE`, `SOLD_OUT`, `STOPPED`. 공개 노출은 `PUBLIC_STATUSES = [ON_SALE, SOLD_OUT]` 만.
- soft delete: `deletedAt` 컬럼. 공개·관리자 조회 모두 `deletedAt IS NULL` 필터링.
- 상세 응답은 `stockQuantity` 포함 — stock 레코드 부재 시 예외 대신 `0`으로 정규화.

DDD 이관 후 패키지 구조:
- `domain/Product`, `domain/ProductStatus`, `domain/repository/ProductRepository` (port)
- `application/ProductQueryService`, `application/AdminProductService` + command/result DTO
- `infrastructure/JpaProductRepository` (Spring Data JPA + `@Query`) + `ProductRepositoryAdapter` (port 구현)
- `presentation/ProductController` (공개), `AdminProductController` (관리자)
- `exception/ProductErrorCode` — 현재 `PRODUCT_NOT_FOUND` 단 1개.

## 핵심 결정

### 1. 공개 query vs 관리자 command 서비스 분리 (DDD 이관 회고)

기존 단일 `ProductService`에 공개 조회·관리자 등록/수정/삭제가 섞여 있던 구조를 `ProductQueryService` + `AdminProductService` 두 개로 쪼갰다. command·query 흐름을 한 서비스에 두지 않는다.

**분리 시점**: DDD 이관 단계에 결정. 단일 ProductService 유지 시 다음 문제가 생길 것이라 판단했다:
- 클래스 비대화
- *관리자 기능과 일반 기능이 섞이면 흐름이 안 보임* — 관리자 기능을 찾으려고 서비스 클래스를 하나하나 열어봐야 함
- 클래스명에서부터 의도가 드러나야 호출부 가독성이 살아남

**자료에서 추출한 부수적 근거**:
- 호출자 성격이 다름 — 공개 query는 비로그인, 관리자 command는 `ROLE_ADMIN`. 권한 경계가 서비스 단위로 정리됨.
- 트랜잭션 성격이 다름 — query는 `@Transactional(readOnly = true)`, command는 `@Transactional`.
- DTO 모양이 다름 — query는 외부 노출용 최소 정보, command는 관리자 응답.
- 로깅 정책도 다름 — query 서비스는 단순 조회라 INFO 로그 없음, command 서비스는 도메인 상태 전환이라 INFO 로그.

**트레이드오프**: 서비스 클래스 2개, 둘 다 `ProductRepository` 의존. 다만 호출부에서 "공개 흐름인가 관리자 흐름인가"가 한눈에 보임.

### 2. `ProductStatus` 3개 (ON_SALE / SOLD_OUT / STOPPED) 설계 + 공개 노출 정책 (product-management ADR)

운영 상태 enum과 공개 노출 정책을 일치시키지 않는다. 공개 노출에는 `ON_SALE` + `SOLD_OUT` 둘 다 포함, `STOPPED`만 제외.

**왜 `SOLD_OUT`도 공개 노출에 포함?**
- 품절 상품을 화면에서 제거하면 "어제까지 보던 상품이 사라졌다"는 사용자 인지 충격이 큼.
- 품절 표시 + 재입고 알림 같은 후속 UX 여지를 남기기 위해 *상태는 노출하되 주문은 막을 수 있게* 분리.
- 실제 주문 가능 여부는 stock 도메인(재고 수량)이 결정하므로, product 상태는 "운영자 의도"만 표현하면 됨.

**왜 `STOPPED`가 따로 필요한가** (soft delete 와 분리한 이유):
- soft delete (`deletedAt`) = *영구 제거 의도* (다시 안 돌아옴)
- `STOPPED` = *일시 중지 의도* (재판매 가능) — 시즌오프/사고로 일시 내림 같은 시나리오
- 두 시나리오가 모두 명시적으로 염두에 있었음. 둘을 한 메커니즘으로 합치면 "삭제된 상품 다시 살리기" 라는 변칙 흐름이 생김.

**`SOLD_OUT` 전이 정책 — 운영자 직접 판단**:
- 재고가 0이 되면 `product.status`를 자동으로 `SOLD_OUT`으로 바꾸는 로직은 없다.
- 관리자가 PATCH로 직접 set 하는 길만 존재.
- *의도적 운영 판단*: "곧 재입고할 거면 `ON_SALE` 유지" 같은 운영자 재량을 허용. 자동 동기화는 향후 필요 시 추가.

**enum 안에 `isPubliclyVisible()` + `publicStatuses()` 정적 메서드 배치**:
- 노출 정책을 enum 자체에 두면 application·repository가 "공개 상태 목록"을 알지 않고 enum에게 위임 가능.
- `ProductQueryService.getProducts()` → `productRepository.findVisibleProducts(ProductStatus.publicStatuses())`. 정책 변경 시 enum 한 곳만 수정.

**트레이드오프**: enum 변경(상태 추가/제거)이 schema 마이그레이션과 결합. `tbl_product.status` 컬럼은 `@JdbcTypeCode(SqlTypes.VARCHAR)` 매핑 (ADR-018).

### 3. soft delete (`deletedAt`) — 주문 이력 보존이 결정 (product-management ADR)

상품 삭제는 hard delete가 아니라 `deletedAt = now()` 기반 soft delete.

**선택 근거** — *외부 제약*이 결정:
- `tbl_order_item`이 `tbl_product.id`를 FK로 참조 (`db-schema.md` 171-173). 과거 주문은 "구매한 상품"이 무엇이었는지를 영구히 가져야 함.
- hard delete → 과거 주문 조회 시 dangling FK 또는 NULL. 주문 이력 정합성 깨짐.
- soft delete → row 보존, 공개 조회만 `deletedAt IS NULL` 필터로 제외.

**왜 별도 archive 테이블이 아닌가?**
- archive 테이블 분리는 read path 이중화 비용이 큼 (현재 + 과거 union, 또는 application 분기).
- 단일 테이블 + `deletedAt` 컬럼은 인덱스 한두 개로 충분히 빠름. MVP 규모에 과한 분리는 ADR-009 ("불필요한 추상화") 원칙에 어긋남.

**왜 `boolean deleted`가 아닌 `LocalDateTime deletedAt`인가** — 명시적 판단:
- *언제* 삭제됐는지 추적이 필요할 수 있다고 판단. 운영자가 "이 상품이 언제부터 안 보였지" 같은 질문을 할 수 있음.
- boolean이면 그 시점 정보가 사라짐. 시점 컬럼은 boolean의 상위 호환 (NULL 여부로 boolean 의미도 그대로 표현 가능).

**현재 미적용 인덱스**:
- `tbl_product.status`, `deleted_at`, `created_at` 조합 인덱스는 *후속 검토*로 보류. 공개 조회는 `where deleted_at is null and status in (...) order by created_at desc`인데 product row 수가 늘면 풀스캔 비용이 커짐. MVP 데이터 규모에서는 인덱스 없이 시작.

**트레이드오프**:
- 운영 DB 크기가 단순 row count 기준으로 단조 증가. "삭제된" 상품도 row가 남으므로.
- 정말로 row를 제거해야 하는 GDPR 같은 시나리오에는 별도 절차가 필요 (현재 범위 밖).

### 4. 상세 조회에서 stock 의존 + 재고 누락 시 `stockQuantity=0` 정규화 (product-query ADR)

`ProductQueryService.getProduct(productId)`는 `stockRepository.findByProductId(productId)`를 호출하고, 결과가 없으면 예외 대신 `0`을 반환한다.

**왜 product → stock 의존인가** (반대 방향이 아니라):
- 사용자 화면 진입점이 "상품 상세"라 product가 owner. stock은 "상품에 딸린 정보".
- stock 도메인이 product를 모르도록 의존 방향을 한 방향으로 고정. stock은 productId로 자기 row를 조회당하는 입장.
- application 계층에서 결합하므로 도메인끼리 직접 의존은 없음. `ProductQueryService`만 `StockRepository`를 import.

**왜 재고 누락이 예외가 아니라 0인가?**
- product-management feature는 *상품 생성 시 재고를 함께 생성하지 않음* (의도적 분리, "재고 수동 조정·초기 재고 생성은 별도 feature"). → 상품은 있고 stock row가 없는 상태가 정상 흐름에 존재.
- 사용자 화면에 "재고 정보 누락" 500 에러 vs "재고 0" 200 응답 — UX 관점에서 후자가 부드러움.
- ADR 본문이 명시한 trade-off: "재고 레코드 누락을 운영 이슈로 숨기는 trade-off가 있으므로 테스트로 명시적으로 고정해야 한다."

**목록 응답은 의도적으로 stock 미포함**:
- 목록 = 탐색용. 화면에 "재고 N개" 일일이 그릴 필요 없음.
- stock join을 목록에 넣으면 N+1 위험 + 응답 사이즈 증가.
- 상세에서만 stock 1회 조회 — 비용이 정확히 사용 시점에만 발생.

**향후 목록에 품절 표시** — `SOLD_OUT` 상태만으로 표시 예정. 목록 응답에 stock 수량 join은 도입하지 않음 (N+1 회피 + 응답 사이즈).

**트레이드오프 + 운영 관점 미해결**:
- "상품은 있는데 stock row 없음" 상태가 운영 데이터에 존재 가능 → *사용자 화면*에는 "재고 0"으로 보임 (의도). 이 정규화는 *사용자 UX 기준*의 선택.
- 하지만 *운영자 관점*에서는 두 상태가 의미가 다름:
  - **stock = 0**: 판매 의도 있고 재고만 소진 (재입고 대상)
  - **row 없음**: 판매 준비 미완료 (초기 재고 미생성)
- 현재 admin API도 동일 정규화로 둘을 구분 못 함 → 운영 점검 가시성에 갭. 별도 운영 점검 대상 / 후속 작업으로 분리.

### 5. imageUrl 문자열 저장 + 파일 업로드 미포함 (product-management ADR)

상품 이미지는 URL 문자열 컬럼(`image_url`)으로만 저장. 파일 업로드·외부 스토리지 연동은 feature 범위 밖.

**근거**:
- 파일 업로드를 포함하면 외부 저장소(S3 등) 선택, 업로드 정책, presigned URL, 권한 모델까지 따라붙음 → feature 크기 폭발.
- 백엔드 책임을 "URL을 저장한다"로 좁히면 클라이언트가 *어디든* URL을 제공하면 됨. CDN 전환·이미지 서버 교체에도 백엔드 변경 없음.
- MVP 범위에서 "최소 변경"으로 상품 표시 정보를 확장하는 가장 작은 단위.

**트레이드오프**:
- URL 유효성 검증 없음 (현재 `imageUrl`은 선택 + blank 검증만). 깨진 URL이나 외부 호스팅 만료를 백엔드가 인지 못 함.
- 이미지 자체의 소유권·보안(만료 URL, 인증 필요 URL) 정책은 백엔드 책임 밖.

**파일 업로드 도입 시점**: *어드민 화면 작업 시*. 그 시점에 저장 방식도 결정 (S3 vs server storage). server storage는 리스크가 크다고 인식하고 있어 S3 쪽으로 기울 가능성. 구체 결정은 어드민 화면 task에서 진행 예정.

### 6. DDD 이관 — repository port 의도 기반 메서드명 + adapter 분리 (product-ddd-migration-retrospective)

`ProductRepository` (domain port)는 Spring Data JPA 파생 쿼리명이 아니라 의도 기반 메서드명을 가짐: `findVisibleProduct`, `findNotDeletedProduct`, `findVisibleProducts`. JPA 조건은 `JpaProductRepository`의 `@Query`로 분리.

**왜 의도 기반 메서드명?**
- `findByDeletedAtIsNullAndStatusIn` 같은 파생 쿼리명은 *영속성 조건이 application 계층으로 새는* 현상. application이 "삭제되지 않은" + "공개 상태" 조건을 직접 알아야 함.
- 의도 기반 이름(`findVisibleProduct`)이면 application은 "공개 가능한 상품 찾기"만 알면 되고, 무엇이 "visible"인지는 repository가 책임.
- 정책 변경(예: 새 상태 추가) 시 application 호출부 수정 없이 repository만 바꿀 수 있음.

**왜 adapter 패턴인가? (`JpaRepository` 직접 상속 대신)**
- 회고 명시: `JpaRepository`와 domain repository를 함께 상속하면 `save` 시그니처가 Spring Data JPA의 generic method에 묶임.
- adapter 방식(`ProductRepositoryAdapter implements ProductRepository`)으로 domain port는 깔끔하게 도메인 객체만 다루고, JPA 구현 디테일은 infrastructure에 격리.

**도메인 전반 통일**: stock·order 등 다른 도메인도 모두 adapter 방식으로 정착.
- 솔직한 자기 인식 메모: 헥사고날 아키텍처를 정식으로 학습해서 도입한 게 아님. DDD를 도입하면서 그 개념에 맞게 설계하다 보니 자연스럽게 port-adapter 구조로 수렴. 정식 헥사고날 적용이라기보다 *DDD 학습의 자연스러운 결과물*.

**테스트 fixture 예외 규칙**:
- application 테스트는 domain repository를 mock.
- `@DataJpaTest`, 통합 테스트, 동시성 테스트는 `JpaProductRepository`를 직접 사용 OK — JPA 전용 메서드(`saveAll`, `flush` 등)가 필요하기 때문. *테스트 편의성을 위해 의도적으로 port 우회 허용.*

**legacy 삭제를 별도 커밋으로 분리한 이유**:
- 단순 리뷰 부담 분산이 아니라 **legacy 의존이 남아있는지 검증할 시간을 벌기 위함**.
- 한 PR로 묶으면 production 코드와 테스트 fixture 양쪽에서 legacy 참조 누락 검증이 충분히 안 됨.
- 분리하면 DDD 이관 PR 머지 후 안정화 → 검색 명령(`rg "com\.commerce\.product\.(service|controller|repository)"`)으로 잔존 참조 검증 → 별도 PR로 legacy 삭제.

**트레이드오프**:
- 레이어가 한 겹 추가됨 (adapter). 작은 도메인엔 약간 over-spec처럼 보일 수 있음.
- 메서드 추가 시 port + adapter + JpaRepository 3곳 수정 필요.

### 7. MVP에서 의도적으로 제외한 것들 (product-management PRD, product-query PRD)

PRD는 명시적으로 다음을 *제외 범위*로 둠:
- 페이지네이션
- 검색·필터·카테고리
- 파일 업로드
- 상품 생성 시 초기 재고 생성
- 관리자 UI
- 목록 응답의 재고 정보

**왜 카테고리 제외?**
- 카테고리는 별도 엔티티/테이블(`tbl_category`, `tbl_product_category` 등) + 다대다 + 트리 구조 결정까지 큰 변경.
- 그보다 본질적인 이유 — **어떤 카테고리를 도입할지 비즈니스 도메인 자체가 명확하지 않음**. 현재는 큰 "commerce" 컨셉이고 옷/화장품/기타 등 *세부 카테고리를 정하지 않은 상태*. 카테고리 구조부터 만들면 정해지지 않은 분류 체계에 코드를 맞추는 셈.
- "상품을 찾아본다" 기본 시나리오에는 카테고리 없이도 충분 (목록 + 상세).

**왜 페이지네이션 제외 + 도입 트리거**:
- MVP 데이터 규모(수십~수백 row)에서 전체 조회로 충분.
- 페이지네이션 도입은 정렬·필터 조합 정책까지 따라옴 → feature 범위 확대.
- "기본 정렬은 `createdAt DESC` 고정"으로 의사결정 부담만 미리 잠가둠.
- **도입 트리거**: 등록된 상품을 한 번에 조회하는 데 부담이 되는 시점부터 (전체 정보를 한 번에 불러오면 UI 구성도 힘들어지는 임계). 구체 row 수 임계는 미정 — 운영하면서 체감으로 판단.

**왜 상품 생성 시 초기 재고를 같이 안 만드는가?**
- 회고에서 명시: 재고는 동시성·이력 등 별도 설계 부담이 있어 product-management 안에 끼우면 feature 크기 폭발.
- 그래서 관리자 흐름은 "상품 생성 → 별도 API로 재고 초기화" 2단계. UX 부담은 있지만 도메인 분리 우선.

**트레이드오프 — product-query retrospective의 회귀**:
- step 범위가 좁아서 `Product.builder()` 사용처(주문·결제·재고 테스트 fixture)가 깨지는 것을 처음엔 못 잡음 → 전체 테스트 보정 필요.
- *교훈*: 엔티티 생성 계약 변경은 작은 도메인 변경이 아니라 전체 fixture 변경으로 봐야 한다.

## 도메인 경계에서 배운 것

### DDD 이관 회고에서 (`docs/ddd/product-ddd-migration-retrospective.md`)

- application 서비스는 command/query로 쪼개고, 같은 도메인이라도 단일 service에 둘을 끼우지 않는다.
- domain repository 메서드명은 *의도* (`findVisibleProduct`), JPA 메서드명은 *조건* (`@Query`로 명시).
- legacy 삭제는 DDD 이관 커밋과 분리한다 (리뷰 부담·변경량 분산). 후속 작업으로 `refactor: product legacy 패키지를 정리한다` 커밋.
- 다음 DDD 후보가 `payment`라는 점에서 product가 payment 이관 전 *기반 도메인 정리* 의 위치.

### product-management 회고 — 엔티티 생성 계약 변경의 파급

- `ProductStatus` 필수화로 `Product.builder()` 사용처(주문·결제·재고 테스트 fixture)가 모두 영향받음.
- step Acceptance Criteria가 `com.commerce.product.*`만 잡고 있어 회귀를 즉시 못 잡음.
- 교훈: shared domain 엔티티 계약 변경은 전체 테스트 실행이 AC에 포함되어야 한다.
- 후속 체크리스트: "`rg "Product.builder"` 같은 파급 범위 탐색을 step에 명시"

### product-query 회고 — step 범위 vs 실제 변경 범위

- step 수정 가능 경로에 도메인 패키지(`com/commerce/product/**`)만 적었더니, 공개 API 요구사항 때문에 `JwtAuthenticationFilter` 수정도 필요해져 "허용 범위 밖 변경"으로 차단됨.
- 교훈: 공개 API/인증/공통 예외/문서 동기화 같은 *횡단 관심사*는 처음부터 step 경로에 포함해야 함.
- 하네스 보완 — reviewer diff는 구현 변경 경로 중심으로만 구성하도록 `execute.py` 수정.

### product-management 회고 — 자동 리뷰의 한계

- step1에서 controller·DTO·service·test를 한 번에 추가하니 diff가 잘려서 reviewer가 근거 부족으로 blocked.
- "자동 리뷰를 믿으려면 diff 전달 방식도 review 대상만큼 중요하다."
- → diff 전달 방식 자체가 하네스 설계의 일부.

## 다시 본다면

- (사용자 작성 영역 — 현재 시점에서 product 도메인을 처음부터 짠다면 바꿀 것 / 그대로 갈 것)

## 다음 단계 / 미해결

- **DB 마이그레이션 도구 부재** — `tbl_product.status` 기존 row 기본값, `deleted_at = null` 설정 절차가 운영 문서로만 남아있음. Flyway/Liquibase 도입은 별도 트랙. (ADR-018에도 비슷한 미해결 명시: ENUM → VARCHAR 운영 DB ALTER)
- **공개 조회 인덱스** — 데이터 늘어나면 `(status, deleted_at, created_at)` 조합 인덱스 검토.
- **이미지 파일 업로드** — `imageUrl` 문자열만 저장 중. S3 등 외부 스토리지 연동은 어드민 화면 작업 시 함께 도입 예정.
- **카테고리·검색·필터·페이지네이션** — MVP 범위 밖. 카테고리는 비즈니스 도메인 명확화 후, 페이지네이션은 데이터 규모/UI 부담 시점 진입.
- **stock row 부재 모니터링** — "상품은 있는데 stock 없음"이 사용자에게는 재고 0으로 보임 → 운영 점검 대상으로 분리되어 있음.
- **관리자 API에서 stock row 부재 vs stock=0 구분** — 두 상태는 의미가 다르나(판매 준비 미완료 vs 재고 소진) 현재 admin API는 모두 `stockQuantity=0`으로 정규화. 운영자 가시성 위해 별도 필드/엔드포인트 필요. (사용자 화면 정규화는 의도된 UX이므로 변경 안 함.)
- **legacy 패키지 잔존 가능성** — DDD 이관 회고에 "legacy `product.service`, `product.controller`, `product.repository`를 후속에서 production/test 참조 확인 후 제거"라고 적혀 있음. 현재 시점에서 정리 완료 여부 확인 필요.

## 인용

- `[[commerce-backend/docs/tasks/product-management/prd.md]]` — 관리자 상품 API 요구사항·범위
- `[[commerce-backend/docs/tasks/product-management/adr.md]]` — soft delete + imageUrl 문자열 + ProductStatus 3개 결정
- `[[commerce-backend/docs/tasks/product-management/retrospective.md]]` — 엔티티 계약 변경 파급·diff truncation 교훈
- `[[commerce-backend/docs/tasks/product-query/prd.md]]` — 공개 조회 요구사항
- `[[commerce-backend/docs/tasks/product-query/adr.md]]` — 목록·상세 응답 분리, 재고 0 정규화
- `[[commerce-backend/docs/tasks/product-query/retrospective.md]]` — step 범위 vs 실제 변경 범위 교훈
- `[[commerce-backend/docs/ddd/product-ddd-migration-retrospective.md]]` — query/command 서비스 분리, repository adapter 패턴
- `[[commerce-backend/docs/architecture.md]]` — product 도메인 책임·데이터 흐름
- `[[commerce-backend/docs/db-schema.md#tbl_product]]` — 컬럼·FK 관계
- `[[commerce-backend/docs/ADR.md#ADR-018]]` — Hibernate ENUM 매핑 회피 (ProductStatus 컬럼 적용)
