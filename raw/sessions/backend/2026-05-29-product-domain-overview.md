---
platform: backend
author: KimYeonWook511
created: 2026-05-29
---

# Product 도메인 개요 — 공개 조회/관리자 관리 분리, soft delete, DDD 이관으로 정리한 기반 도메인

commerce의 상품 도메인을 지금 상태로 정리하면서 내린 결정들을 한 자리에 모은 세션이다. product는 주문 생성·재고 생성·상품 공개 조회가 모두 밟고 가는 기반 도메인이라, payment 도메인 전환에 앞서 repository 경계와 서비스 책임부터 DDD 계층으로 정리해뒀다. 담는 건 상품 마스터·공개 노출 정책·soft delete를 어떻게 갈랐고, 왜 재고를 떼어냈으며, DDD 이관에서 무엇을 확정했는가다.

## 도메인 개요

- **책임**: 상품 마스터(이름·가격·설명·이미지·판매 상태) + 공개 노출 정책 + soft delete. **재고는 책임을 분리**해 별도 stock 도메인이 가진다.
- **엔티티**: `Product` 단일. `Product : Stock = 1:1`이지만 참조 컬럼(`product_id`)은 stock 쪽이 들고, product는 stock을 알지 않는다. 의존 방향은 application 계층에서만 생긴다 — 공개 조회 서비스가 `StockRepository`(재고 조회 포트)를 호출하는 한 방향.
- **application 서비스 2개 — 공개 query / 관리자 command 분리**:
  - `ProductQueryService` — 공개 목록·상세 조회. 비로그인 OK.
  - `AdminProductService` — 관리자 등록·수정·soft delete.
- **판매 상태 enum 3개**: `ON_SALE`, `SOLD_OUT`, `STOPPED`. 공개 노출은 `PUBLIC_STATUSES = [ON_SALE, SOLD_OUT]`만.
- **soft delete**: `deletedAt`(LocalDateTime) 컬럼. 공개·관리자 조회 모두 `deletedAt IS NULL` 필터를 건다.
- **상세 응답**은 `stockQuantity`를 포함하되, stock 레코드가 없으면 예외 대신 `0`으로 정규화한다.

DDD 이관 후 패키지 구조:

- `domain/Product`, `domain/ProductStatus`, `domain/repository/ProductRepository`(도메인 포트)
- `application/ProductQueryService`, `application/AdminProductService` + command/result DTO
- `infrastructure/JpaProductRepository`(Spring Data JPA + `@Query`) + `ProductRepositoryAdapter`(포트 구현)
- `presentation/ProductController`(공개), `AdminProductController`(관리자)
- `exception/ProductErrorCode` — 현재 `PRODUCT_NOT_FOUND` 단 하나.

## 결정한 것

### 1. 공개 query 서비스와 관리자 command 서비스를 분리

기존 단일 `ProductService`에 공개 조회와 관리자 등록/수정/삭제가 섞여 있던 구조를, 공개 조회 전용 `ProductQueryService`와 관리자 쓰기 전용 `AdminProductService` 둘로 쪼갰다. command·query 흐름을 한 서비스에 두지 않는다.

- **분리 시점**: DDD 이관 단계에서 결정했다. 단일 `ProductService`를 그대로 두면 다음 문제가 생길 것이라 판단했다.
  - 클래스 비대화.
  - **관리자 기능과 일반 기능이 섞이면 흐름이 안 보임** — 관리자 기능을 찾으려고 서비스 클래스를 하나하나 열어봐야 한다.
  - 클래스명에서부터 의도가 드러나야 호출부 가독성이 산다.
- **부수적 근거(코드·설계에서 자연히 갈리는 축)**:
  - **호출자 성격이 다름** — 공개 query는 비로그인, 관리자 command는 `ROLE_ADMIN`. 권한 경계가 서비스 단위로 정리된다.
  - **트랜잭션 성격이 다름** — query 서비스는 `@Transactional(readOnly = true)`, 관리자 서비스는 쓰기 메서드마다 `@Transactional`.
  - **DTO 모양이 다름** — query는 외부 노출용 최소 정보(목록은 id·이름·가격), command는 관리자 응답.
  - **로깅 정책도 다름** — 단순 조회인 query는 INFO 로그가 필요 없고, 도메인 상태 전환을 일으키는 command는 INFO 로그가 붙는 쪽이라는 게 분리의 명분이었다. (이 로깅 대비는 분리를 정당화하는 설계 논거로 든 것이지, 이관 시점 코드에서 두 서비스의 로그 유무를 실측해 확정한 축은 아니다.)
- **트레이드오프**: 서비스 클래스가 2개가 되고 둘 다 `ProductRepository`에 의존한다. 그 대신 호출부에서 "공개 흐름인가 관리자 흐름인가"가 한눈에 보인다.

### 2. `ProductStatus` 3개(ON_SALE / SOLD_OUT / STOPPED) 설계와 공개 노출 정책

운영 상태 enum과 공개 노출 정책을 1:1로 일치시키지 않는다. 공개 노출에는 `ON_SALE`과 `SOLD_OUT` 둘 다 포함하고, `STOPPED`만 제외한다.

- **왜 `SOLD_OUT`도 공개에 노출하나?**
  - 품절 상품을 화면에서 통째로 지우면 "어제까지 보던 상품이 사라졌다"는 사용자 인지 충격이 크다.
  - 품절 표시 + 재입고 알림 같은 후속 UX 여지를 남기려면, *상태는 노출하되 주문은 막을 수 있게* 분리해야 한다.
  - 실제 주문 가능 여부는 stock 도메인(재고 수량)이 결정하므로, product 상태는 "운영자 의도"만 표현하면 된다.
- **왜 `STOPPED`가 soft delete와 별도로 필요한가?**
  - soft delete(`deletedAt`)는 *영구 제거 의도*(다시 안 돌아옴).
  - `STOPPED`는 *일시 중지 의도*(재판매 가능) — 시즌오프나 사고로 잠시 내려두는 시나리오.
  - 두 시나리오를 명시적으로 염두에 뒀다. 하나의 메커니즘으로 합치면 "삭제된 상품 다시 살리기"라는 변칙 흐름이 생긴다.
- **`SOLD_OUT` 전이 정책 — 운영자 직접 판단**:
  - 재고가 0이 됐다고 `product.status`를 자동으로 `SOLD_OUT`으로 바꾸는 로직은 없다. 관리자가 수정 API로 직접 set 하는 길만 존재한다.
  - *의도적 운영 판단*이다 — "곧 재입고할 거면 `ON_SALE` 유지" 같은 운영자 재량을 허용한다. 재고와 상태의 자동 동기화는 필요해지면 나중에 추가한다.
- **노출 정책을 enum 안에 배치**(`isPubliclyVisible()` + 정적 `publicStatuses()`):
  - 노출 정책을 enum 자신이 들면 application·repository가 "공개 상태 목록"을 직접 알 필요 없이 enum에 위임할 수 있다.
  - 실제로 목록 조회는 `productRepository.findVisibleProducts(ProductStatus.publicStatuses())`로 부른다. 정책이 바뀌면 enum 한 곳만 고치면 된다.
- **트레이드오프**: enum 값을 추가/제거하면 스키마 마이그레이션과 결합된다. `tbl_product.status` 컬럼은 `@Enumerated(EnumType.STRING)`에 `@JdbcTypeCode(SqlTypes.VARCHAR)`를 함께 붙여 매핑한다 — Hibernate 6.x가 STRING enum을 MySQL 네이티브 ENUM 타입으로 매핑하면, 컬럼을 생략한 INSERT에 첫 enum 값이 조용히 채워지는 함정이 있어 이를 VARCHAR로 강제하는 프로젝트 공통 컨벤션이다.

### 3. soft delete(`deletedAt`) — 주문 이력 보존이 결정을 강제

상품 삭제는 hard delete가 아니라 `deletedAt = now()` 기반 soft delete다.

- **선택 근거 — *외부 제약*이 결정했다**:
  - 주문 항목(order_item)이 어떤 상품을 구매했는지 상품 id로 참조한다. 과거 주문은 "그때 산 상품"이 무엇이었는지를 영구히 가져야 한다.
  - hard delete를 하면 과거 주문 조회 때 사라진 상품을 가리키는 dangling 참조나 NULL이 남아 주문 이력 정합성이 깨진다.
  - soft delete면 row는 보존되고, 공개 조회만 `deletedAt IS NULL` 필터로 제외한다.
- **왜 별도 archive 테이블이 아닌가?**
  - archive 테이블로 분리하면 read path가 이중화(현재+과거 union, 또는 application 분기)돼 비용이 크다.
  - 단일 테이블 + `deletedAt` 컬럼은 인덱스 한두 개면 충분히 빠르다. MVP 규모에 과한 분리는 불필요한 추상화를 피한다는 프로젝트 원칙에 어긋난다.
- **왜 `boolean deleted`가 아니라 `LocalDateTime deletedAt`인가** — 명시적 판단:
  - *언제* 삭제됐는지 추적이 필요할 수 있다고 봤다. 운영자가 "이 상품이 언제부터 안 보였지" 같은 질문을 할 수 있다.
  - boolean이면 그 시점 정보가 사라진다. 시점 컬럼은 boolean의 상위 호환이다(NULL 여부로 boolean 의미도 그대로 표현 가능).
- **아직 안 건 인덱스**: `(status, deleted_at, created_at)` 조합 인덱스는 *후속 검토*로 보류했다. 공개 조회는 `where deleted_at is null and status in (...) order by created_at desc` 형태라 product row가 늘면 풀스캔 비용이 커지지만, MVP 데이터 규모에서는 인덱스 없이 시작한다.
- **트레이드오프**:
  - 운영 DB 크기가 단순 row count 기준으로 단조 증가한다. "삭제된" 상품도 row가 남기 때문.
  - 정말로 row를 지워야 하는 GDPR 류 시나리오는 별도 절차가 필요하다(현재 범위 밖).

### 4. 상세 조회에서 stock 의존 + 재고 누락 시 `stockQuantity = 0` 정규화

`ProductQueryService.getProduct(productId)`는 재고 조회 포트 `StockRepository.findByProductId(productId)`를 호출하고, 결과가 없으면 예외 대신 `0`을 채워 반환한다.

- **왜 product → stock 의존인가**(반대 방향이 아니라):
  - 사용자 화면 진입점이 "상품 상세"라 product가 owner다. stock은 "상품에 딸린 정보".
  - stock 도메인이 product를 모르도록 의존 방향을 한쪽으로 고정했다. stock은 productId로 자기 row를 조회당하는 입장.
  - 결합은 application 계층에서만 일어나 도메인끼리 직접 의존은 없다 — `ProductQueryService`만 `StockRepository`를 import 한다.
- **왜 재고 누락이 예외가 아니라 0인가?**
  - 상품 관리 기능은 *상품 생성 시 재고를 함께 만들지 않는다*(의도적 분리 — 재고 수동 조정·초기 재고 생성은 별도 기능). 그래서 "상품은 있는데 stock row가 없는" 상태가 정상 흐름에 존재한다.
  - 사용자 화면에 "재고 정보 누락" 500 vs "재고 0" 200 — UX 관점에서 후자가 부드럽다.
  - 이 결정을 정리한 상품 조회 ADR이 명시한 트레이드오프: "재고 레코드 누락을 운영 이슈로 숨기는 trade-off가 있으므로 테스트로 명시적으로 고정해야 한다."
- **목록 응답은 의도적으로 stock 미포함**:
  - 목록은 탐색용이라 "재고 N개"를 일일이 그릴 필요가 없다.
  - stock join을 목록에 넣으면 N+1 위험 + 응답 사이즈 증가.
  - 상세에서만 stock을 1회 조회 — 비용이 정확히 필요한 시점에만 발생한다.
  - 향후 목록에 품절 표시가 필요하면 `SOLD_OUT` 상태만으로 표시할 계획이고, 목록에 stock 수량 join은 도입하지 않는다(N+1 회피 + 응답 사이즈).
- **트레이드오프 + 운영 관점 미해결**:
  - "상품은 있는데 stock row 없음" 상태가 운영 데이터에 존재할 수 있다 → *사용자 화면*에는 "재고 0"으로 보인다(의도). 이 정규화는 *사용자 UX 기준*의 선택이다.
  - 하지만 *운영자 관점*에서는 두 상태의 의미가 다르다:
    - **stock = 0**: 판매 의도는 있고 재고만 소진(재입고 대상).
    - **row 없음**: 판매 준비 미완료(초기 재고 미생성).
  - 현재 관리자 API도 같은 정규화를 써서 둘을 구분하지 못한다 → 운영 점검 가시성에 갭이 있다. 별도 운영 점검 대상/후속 작업으로 뺐다.

### 5. imageUrl 문자열 저장 + 파일 업로드 미포함

상품 이미지는 URL 문자열 컬럼(`image_url`)으로만 저장한다. 파일 업로드·외부 스토리지 연동은 이 기능 범위 밖이다.

- **근거**:
  - 파일 업로드를 넣으면 외부 저장소(S3 등) 선택, 업로드 정책, presigned URL, 권한 모델까지 딸려와 기능 크기가 폭발한다.
  - 백엔드 책임을 "URL을 저장한다"로 좁히면 클라이언트가 *어디서든* URL을 대주면 되고, CDN 전환·이미지 서버 교체에도 백엔드 변경이 없다.
  - MVP 범위에서 상품 표시 정보를 확장하는 가장 작은 변경 단위다.
- **트레이드오프**:
  - URL 유효성 검증이 없다. `imageUrl`은 아무 제약도 걸지 않은 선택 필드다 — 요청 DTO에도 `@NotBlank` 같은 검증이 없고 엔티티 생성자도 이름·가격·상태만 검증한다. 깨진 URL이나 외부 호스팅 만료를 백엔드가 인지하지 못한다.
  - 이미지 자체의 소유권·보안(만료 URL, 인증 필요 URL) 정책은 백엔드 책임 밖.
- **파일 업로드 도입 시점**: *어드민 화면 작업 시*. 그때 저장 방식도 함께 정한다(S3 vs server storage). server storage는 리스크가 크다고 보고 있어 S3 쪽으로 기울 가능성이 높지만, 구체 결정은 어드민 화면 task에서 진행한다.

### 6. DDD 이관 — repository 포트는 의도 기반 메서드명 + adapter 분리

`ProductRepository`(도메인 포트)는 Spring Data JPA 파생 쿼리명이 아니라 의도 기반 메서드명을 가진다: `findVisibleProduct`, `findNotDeletedProduct`, `findVisibleProducts`. 실제 JPA 조건은 `JpaProductRepository`의 `@Query`로 분리했다.

- **왜 의도 기반 메서드명인가?**
  - `findByDeletedAtIsNullAndStatusIn` 같은 파생 쿼리명은 *영속성 조건이 application 계층으로 새는* 현상이다. application이 "삭제되지 않은" + "공개 상태" 조건을 직접 알아야 한다.
  - 의도 기반 이름(`findVisibleProduct`)이면 application은 "공개 가능한 상품 찾기"만 알면 되고, 무엇이 "visible"인지는 repository가 책임진다.
  - 정책이 바뀌어도(예: 새 상태 추가) application 호출부를 안 건드리고 repository만 고치면 된다.
- **왜 adapter 패턴인가?**(`JpaRepository`를 도메인 포트에 직접 상속시키는 대신):
  - 이관 회고에서 명시했듯, `JpaRepository`와 도메인 repository를 함께 상속하면 `save` 시그니처가 Spring Data JPA의 generic method에 묶인다.
  - `ProductRepositoryAdapter implements ProductRepository`로 두면 도메인 포트는 도메인 객체만 깔끔하게 다루고, JPA 구현 디테일은 infrastructure에 격리된다.
- **도메인 전반 통일**: stock·order 등 다른 도메인도 모두 adapter 방식으로 정착했다.
  - 솔직한 자기 인식: 헥사고날 아키텍처를 정식으로 학습해서 도입한 게 아니다. DDD를 도입하며 개념에 맞게 설계하다 보니 자연스럽게 port-adapter 구조로 수렴했다. 정식 헥사고날 적용이라기보다 *DDD 학습의 자연스러운 결과물*이다.
- **테스트 fixture 예외 규칙**:
  - application 테스트는 도메인 repository를 mock 한다.
  - `@DataJpaTest`, 통합 테스트, 동시성 테스트는 `JpaProductRepository`를 직접 써도 된다 — `saveAll`·`flush` 같은 JPA 전용 메서드가 필요하기 때문. *테스트 편의를 위해 의도적으로 포트를 우회하도록 허용*한다.
- **legacy 삭제를 별도 커밋으로 분리한 이유**:
  - 단순 리뷰 부담 분산이 아니라 **legacy 의존이 남았는지 검증할 시간을 벌기 위함**이다.
  - 한 PR로 묶으면 production 코드와 테스트 fixture 양쪽에서 legacy 참조가 빠졌는지 검증이 충분히 안 된다.
  - 분리하면 DDD 이관 PR 머지 후 안정화 → 검색(`rg "com\.commerce\.product\.(service|controller|repository)"`)으로 잔존 참조 확인 → 별도 PR로 legacy 삭제, 이 순서를 밟을 수 있다.
- **트레이드오프**:
  - 레이어가 한 겹 늘어난다(adapter). 작은 도메인엔 살짝 over-spec처럼 보일 수 있다.
  - 메서드를 추가하면 포트 + adapter + `JpaProductRepository` 3곳을 수정해야 한다.

### 7. MVP에서 의도적으로 제외한 것

상품 관리·조회 기능의 요구사항 문서가 다음을 명시적으로 *제외 범위*로 뒀다: 페이지네이션 / 검색·필터·카테고리 / 파일 업로드 / 상품 생성 시 초기 재고 생성 / 관리자 UI / 목록 응답의 재고 정보.

- **왜 카테고리 제외?**
  - 카테고리는 별도 엔티티/테이블(`tbl_category`, `tbl_product_category` 등) + 다대다 + 트리 구조 결정까지 큰 변경이다.
  - 더 본질적인 이유 — **어떤 카테고리를 도입할지 비즈니스 도메인 자체가 아직 안 정해졌다**. 지금은 큰 "commerce" 컨셉이고 옷/화장품/기타 같은 *세부 카테고리를 확정하지 않은 상태*다. 카테고리 구조부터 만들면 정해지지 않은 분류 체계에 코드를 맞추는 셈. (이 판단은 요구사항 문서에 적힌 제외 사실을 넘어선 원저자의 근거다.)
  - "상품을 찾아본다"는 기본 시나리오에는 카테고리 없이도 목록 + 상세로 충분하다.
- **왜 페이지네이션 제외 + 도입 트리거**:
  - MVP 데이터 규모(수십~수백 row)에서는 전체 조회로 충분하다.
  - 페이지네이션을 도입하면 정렬·필터 조합 정책까지 딸려와 기능 범위가 커진다.
  - "기본 정렬은 `createdAt DESC` 고정"으로 의사결정 부담만 미리 잠가뒀다.
  - **도입 트리거**: 등록 상품을 한 번에 조회하는 게 부담이 되는 시점부터(전체를 한 번에 부르면 UI 구성도 힘들어지는 임계). 구체 row 수 임계는 미정 — 운영하며 체감으로 판단한다.
- **왜 상품 생성 시 초기 재고를 같이 안 만드나?**
  - 재고는 동시성·이력 등 별도 설계 부담이 있어 상품 관리 기능 안에 끼우면 기능 크기가 폭발한다(이관 회고에서 명시).
  - 그래서 관리자 흐름은 "상품 생성 → 별도 API로 재고 초기화" 2단계다. UX 부담은 있지만 도메인 분리를 우선했다.
- **트레이드오프 — 상품 조회 회고의 회귀 사례**:
  - step 범위가 좁아서 `Product.builder()` 사용처(주문·결제·재고 테스트 fixture)가 깨지는 걸 처음엔 못 잡았고, 전체 테스트 보정이 필요했다.
  - *교훈*: 엔티티 생성 계약 변경은 작은 도메인 변경이 아니라 전체 fixture 변경으로 봐야 한다.

## 배운 것

### DDD 이관 회고에서

- application 서비스는 command/query로 쪼갠다. 같은 도메인이라도 단일 service에 둘을 끼우지 않는다.
- 도메인 repository 메서드명은 *의도*(`findVisibleProduct`), JPA 메서드명은 *조건*(`@Query`로 명시).
- legacy 삭제는 DDD 이관 커밋과 분리한다(리뷰 부담·변경량 분산). 후속으로 `refactor: product legacy 패키지를 정리한다` 커밋을 예정.
- 다음 DDD 후보가 `payment`라는 점에서, product는 payment 이관 전에 먼저 정리해두는 *기반 도메인 정리*의 위치에 있다.

### 엔티티 생성 계약 변경의 파급 (상품 관리 회고)

- `ProductStatus`를 필수화하자 `Product.builder()` 사용처(주문·결제·재고 테스트 fixture)가 모두 영향받았다.
- step Acceptance Criteria가 `com.commerce.product.*`만 실행하고 있어 이 파급을 즉시 잡지 못했다.
- 교훈: shared domain 엔티티 계약 변경은 전체 테스트 실행이 AC에 포함돼야 한다.
- 후속 체크리스트: "`rg "Product.builder"` 같은 파급 범위 탐색을 step에 명시".

### step 범위 vs 실제 변경 범위 (상품 조회 회고)

- step 수정 가능 경로에 도메인 패키지(`com/commerce/product/**`)만 적었더니, 공개 API 요구사항 때문에 `JwtAuthenticationFilter`(인증 필터) 수정도 필요해져 "허용 범위 밖 변경"으로 반복 차단됐다.
- 교훈: 공개 API·인증·공통 예외·문서 동기화 같은 *횡단 관심사*는 처음부터 step 경로에 넣어야 한다.
- 하네스 보완 — reviewer diff는 구현 변경 경로 중심으로만 구성하도록 실행기(`execute.py`)를 수정했다.

### 자동 리뷰의 한계 (상품 관리 회고)

- step1에서 controller·request/command/result DTO·service·test를 한 번에 추가하니 diff가 커져 reviewer에게 전달된 diff가 잘렸고, reviewer가 근거 부족으로 blocked를 냈다(구현과 product 범위 테스트는 통과했는데도).
- 교훈: "자동 리뷰를 믿으려면 diff 전달 방식도 review 대상만큼 중요하다." diff 전달 방식 자체가 하네스 설계의 일부다.

## 미해결·열린 질문

- **DB 마이그레이션 도구 부재** — `tbl_product.status` 기존 row 기본값(`ON_SALE`), `deleted_at = null` 설정 절차가 운영 문서로만 남아 있다. 마이그레이션 도구(Flyway/Liquibase) 도입은 별도 트랙이다. enum을 VARCHAR로 매핑하기로 한 결정에도 같은 미해결이 달려 있다 — 운영 DB의 기존 ENUM 컬럼을 VARCHAR로 ALTER 하는 작업.
- **공개 조회 인덱스** — 데이터가 늘면 `(status, deleted_at, created_at)` 조합 인덱스를 검토한다.
- **이미지 파일 업로드** — 지금은 `imageUrl` 문자열만 저장 중. S3 등 외부 스토리지 연동은 어드민 화면 작업 시 함께 도입 예정.
- **카테고리·검색·필터·페이지네이션** — MVP 범위 밖. 카테고리는 비즈니스 도메인이 명확해진 뒤, 페이지네이션은 데이터 규모/UI 부담이 임계에 닿는 시점에 진입.
- **stock row 부재 모니터링** — "상품은 있는데 stock 없음"이 사용자에게는 재고 0으로 보인다 → 운영 점검 대상으로 분리해뒀다.
- **관리자 API에서 stock row 부재 vs stock = 0 구분** — 두 상태는 의미가 다르지만(판매 준비 미완료 vs 재고 소진) 현재 관리자 API는 모두 `stockQuantity = 0`으로 정규화한다. 운영자 가시성을 위해 별도 필드/엔드포인트가 필요하다. (사용자 화면 정규화는 의도된 UX이므로 그대로 둔다.)
- **legacy 패키지 잔존 가능성** — 이관 회고에 "legacy `product.service`·`product.controller`·`product.repository`를 후속에서 production/test 참조 확인 후 제거"라고 적어뒀다. 현재 시점에서 정리 완료 여부는 확인이 필요하다.
