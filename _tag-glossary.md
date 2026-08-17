---
type: meta
updated: 2026-08-17
---

# 태그 사전 (tag glossary)

표준 태그와 동의어를 관리한다. **빈 상태로 시작해 ingest가 자라게 한다.**

- canonical 형태: **소문자·하이픈·단수형** (`Refresh Tokens` → `refresh-token`).
- 동의어는 canonical로 정규화한다 (`React` → `react`).
- `platform` 태그군은 분류 축이라 아래에 별도로 둔다.

## platform 태그군 (분류 축)

| canonical | 동의어 |
|---|---|
| `backend` | server, 서버, api |
| `frontend` | web, 웹, client |
| `infra` | devops, ops |
| `android` | aos |
| `ios` | — |

> `android`(주로 React Native)·`ios`는 예정 플랫폼이다. 어떤 코드 repo가 어떤 platform인지는 각 repo의 `.brain`이 선언한다(이 glossary는 repo 이름에 의존하지 않는다).

## 일반 태그 (성격·기능)

2026-07-14 최초 대규모 ingest로 등재하고 2026-08-17 ingest·lint(2회)에서 갱신했다. canonical = 소문자·하이픈·단수형. 아래 개수는 2026-08-17 기준이며 사용처 다수(5+)인 태그 위주다. 전체 태그는 각 노트 frontmatter가 진실이다. 태그가 **259종**(그중 싱글턴 123종)으로 산발이 있어 **동의어 통합·싱글턴 정리는 lint 후보**다.

### 도메인·기능

| canonical | 비고 |
|---|---|
| `payment` (79) | 결제 도메인. 최상위 관심사 |
| `refund` (30) | 환불. 2026-08 재설계로 독립 aggregate가 됨 |
| `order` (27) | 주문 도메인 |
| `partial-cancel` (18) | 부분취소·부분환불 |
| `stock` (11) / `product` (7) / `cart` (6) | 재고·상품·장바구니 도메인 |
| `auth` (10) / `member` | 인증 도메인. `authentication` 을 흡수 |
| `security` (7) / `access-control` | **접근 제어**. `auth` 와 다른 축 — 결제 노트에서 남의 요청을 막는 뜻으로 쓴다 |
| `naverpay` (19) / `pg-gateway` (15) | 결제사 연동 |
| `reservation` (9) | 결제 예약(RESERVED). 2026-08에 활성 슬롯으로 흡수 |
| `double-payment` (16) | 이중결제·이중환불. `duplicate-payment`·`double-charge` 를 흡수 |
| `payment-attempt` (3) | 옛 결제 시도 이력 aggregate |

### 아키텍처·기법

| canonical | 비고 |
|---|---|
| `idempotency` (32) | 멱등성 |
| `concurrency` (26) | 동시성 |
| `unique-constraint` (21) / `db-unique` | DB unique 제약 |
| `reconciliation` (25) / `escalation` (12) / `unknown-status` (8) / `starvation` | 회수(대사)·에스컬레이션·결과 불명. `reconcile` 을 흡수 |
| `exception-handling` (21) / `error-code` (11) / `exception-strategy` | 예외 처리·에러 코드 체계 |
| `compensation` (15) / `saga` (6) | 보상·사가 |
| `optimistic-lock` (14) / `pessimistic-lock` (8) | 낙관/비관 락 |
| `jpa` (15) / `hibernate` (9) / `flush` (4) / `orm` / `persistence` | 영속성. `flush` 가 `save-and-flush`·`save-flush` 를 흡수 |
| `ddd` (13) / `aggregate` (10) / `adapter` (10) / `layered-architecture` (7) / `hexagonal` / `port-adapter` | DDD·헥사고날 |
| `transaction-boundary` (20) / `outbox` / `after-commit` | 트랜잭션 경계·전파·커밋 시점. `transaction` 을 흡수 |
| `mysql` (13) / `innodb` (5) | DB 엔진 특성 |
| `redis` (12) / `cache` | Redis·캐시 |
| `schema` (11) / `migration` (8) / `flyway` / `cross-aggregate` (5) / `fk` | 스키마·FK→ID 이관. `schema` 가 `schema-design` 을 흡수 |
| `invariant` (8) / `state-model` / `money-model` | 불변식·상태·금액 모델 |
| `append-only` (5) / `ledger` / `event-sourcing` | append-only 원장 |
| `api-contract` (6) / `terminology` | 계약·용어 |

### 프로세스·메타

| canonical | 비고 |
|---|---|
| `convention` (21) / `naming-convention` (8) | 코드/문서 컨벤션. `naming-convention` 이 `naming` 을 흡수 |
| `process` (12) / `workflow` (7) / `harness` (7) | 워크플로·하네스 운영 회고(knowledge) |
| `verification` (8) / `design-review` / `spec-review` | 설계·명세 검증 방법론 |
| `code-review` (7) / `ai-review` | AI·자동 리뷰 |
| `adr` (9) / `documentation` (6) / `task-docs` / `canonical-doc` / `stale-doc` / `immutability` | 문서 정본·불변 |
| `dead-code` (8) / `refactor` (8) / `yagni` (5) / `defensive-programming` | 미사용·과설계 배제·리팩터. `refactor` 가 `refactoring` 을 흡수 |
| `testing` (6) / `integration-test` / `regression-test` / `concurrency-test` | 테스트 |
| `operations` (5) / `observability` / `notification` / `manual-review` | 운영·관측·통지 |
| `measurement` / `cost` / `git` | 실측·비용·이력 추적 |

### 2026-08-17 lint 로 등재 (사용처 3+ 였으나 미등록)

| canonical | 비고 |
|---|---|
| `domain-model` (7) | 도메인 모델 구조 서술(주로 `topics/` 개요 노트) |
| `soft-delete` (5) | 논리 삭제 |
| `database` (4) / `enum` (4) | DB 일반·enum 표현 |
| `jwt` (4) | JWT 토큰 (`auth` 하위) |
| `merchant-pay-key` (4) | 가맹점 결제 키 (`payment` 하위) |
| `dependency-direction` (4) | 의존 방향·순환 회피 (`ddd` 하위) |
| `price-snapshot` (3) | 결제 시점 가격 스냅샷 (`money-model` 하위) |
| `expiration` (3) | 만료·TTL |
| `validation` (3) / `restcontrolleradvice` (3) | 입력 검증·전역 예외 어드바이스 |
| `logging` (3) | 로깅 컨벤션 |
| `archunit` (3) / `spring-batch` (3) | 구조 검사·배치 프레임워크 |

### 동의어 매핑

- 플랫폼: `web`/`웹`/`client` → `frontend`, `server`/`서버`/`api` → `backend` (platform 태그군 참조)
- **2026-08-17 lint 에서 통합 완료(노트 frontmatter 소급 수정 + MOC 재생성):**
  `transaction`→`transaction-boundary` · `refactoring`→`refactor` · `naming`→`naming-convention` ·
  `schema-design`→`schema` · `reconcile`→`reconciliation` · `authentication`→`auth` ·
  `duplicate-payment`·`double-charge`→`double-payment` · `save-and-flush`·`save-flush`→`flush`
- **통합하지 않기로 한 것:** `security`(7)를 `auth`(10)에 합치지 않는다 — 결제 노트에서 `security` 는 **접근 제어**를 뜻하고 인증 도메인과 다른 축이다.
- **2026-08-17 lint 2회차에서 추가 통합:** `infra-adapter`→`adapter` · `pg`→`pg-gateway` · `db-unique`→`unique-constraint` · `cross-aggregate-reference`→`cross-aggregate` · `foreign-key`→`fk` · `unknown-state`→`unknown-status` · `documentation-drift`·`drift`·`spec-drift`→`stale-doc` · `save`→`flush`. **269→259종, 싱글턴 131→123.**
- **통합하지 않기로 한 것 (판단 근거):**
  - `hibernate`(9) / `jpa`(15) — 7개 노트가 **둘 다** 달고 있어 계층 관계다. 상위/하위로 선언만 하고 병합하지 않는다.
  - `batch`/`spring-batch`, `error-category`/`error-code`, `starvation`/`reconciliation` — 하위 개념이 정보를 담는다.
  - `payment-integrity`·`order-number`·`concurrency-test` — 상위 태그의 하위 개념이라 병합하면 정보가 사라진다.
  - `ci`~re**conciliation**, `postprocess-policy`~**process** 같은 문자열 유사 후보는 **부분 문자열 우연**이라 기각했다.
- **폐기한 정리 기준:** "자기 사용처 전부가 더 큰 태그와 함께 달린 태그는 잉여"로 재면 184종이 걸린다. **그 기준은 틀렸다** — `gap-lock` 이 `concurrency` 와 항상 함께 달려도 그 태그는 여전히 정보를 담는다. 싱글턴 123종은 대부분 이런 정당한 구체 태그이므로 일괄 정리 대상이 아니다.

### 상위/하위 관계

- `payment` ⊃ `refund`, `partial-cancel`, `payment-attempt`, `reservation`, `naverpay`, `pg-gateway`, `compensation`, `reconciliation`, `escalation`, `unknown-status`, `double-payment`
- `concurrency` ⊃ `optimistic-lock`, `pessimistic-lock`, `race-condition`, `lost-update`, `gap-lock`, `deadlock`
- `exception-handling` ⊃ `exception-strategy`, `error-category`, `error-code`, `adapter`
- `ddd` ⊃ `hexagonal`, `port-adapter`, `adapter`, `cqrs`, `aggregate`, `layered-architecture`
- `jpa` ⊃ `hibernate`, `flush`, `persistence`, `orm`
- `process` ⊃ `workflow`, `harness`, `design-review`, `spec-review`, `verification`, `code-review`
