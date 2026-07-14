---
type: meta
updated: 2026-07-14
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

2026-07-14 최초 대규모 ingest(backend 64 raw → 101 노트)로 등재. canonical = 소문자·하이픈·단수형. 아래는 사용처 다수(3+)인 태그 위주이고, 전체 태그는 각 노트 frontmatter가 진실이다. 태그가 266종으로 산발이 있어 **동의어 통합·싱글턴 정리는 lint 후보**다.

### 도메인·기능

| canonical | 비고 |
|---|---|
| `payment` (38) | 결제 도메인. 최상위 관심사 |
| `order` (21) | 주문 도메인 |
| `product` (7) / `stock` (6) / `cart` (6) | 상품·재고·장바구니 도메인 |
| `auth` (7) / `member` (3) / `security` (2) | 인증 3패키지 |
| `reservation` (6) | 결제 예약(RESERVED) |
| `payment-attempt` (3) | 결제 시도 이력 aggregate |
| `naverpay` (7) / `pg-gateway` (3) | PG 연동 |

### 아키텍처·기법

| canonical | 비고 |
|---|---|
| `idempotency` (20) | 멱등성 |
| `concurrency` (18) | 동시성 |
| `optimistic-lock` (9) / `pessimistic-lock` (6) | 낙관/비관 락 |
| `unique-constraint` (10) / `db-unique` (3) | DB unique 제약 |
| `exception-handling` (17) / `exception-strategy` (3) | 예외 처리·전략 |
| `compensation` (11) / `saga` (5) | 보상·사가 |
| `reconciliation` (9) / `escalation` (7) / `starvation` (3) | 대사·에스컬레이션 |
| `jpa` (11) / `hibernate` (6) / `orm` (2) / `persistence` (2) | 영속성 |
| `ddd` (9) / `hexagonal` (2) / `adapter` (7) / `port-adapter` (3) / `layered-architecture` (3) | DDD·헥사고날 |
| `transaction` (7) / `transaction-boundary` (4) / `outbox` (3) / `after-commit` (2) | 트랜잭션 경계 |
| `redis` (9) / `cache` (3) | Redis·캐시 |
| `migration` (6) / `flyway` (3) / `schema` (6) / `cross-aggregate` (4) / `fk` (3) | 스키마·FK→ID 이관 |
| `error-code` (7) / `error-category` (2) / `http-status` (2) | 에러 코드 체계 |
| `append-only` (3) / `ledger` (1) / `event-sourcing` (1) | append-only 원장 |
| `mysql` (7) / `innodb` (4) | DB 엔진 특성 |

### 프로세스·메타

| canonical | 비고 |
|---|---|
| `convention` (11) | 코드/문서 컨벤션 |
| `code-review` (5) / `ai-review` (1) / `harness` (4) / `workflow` (3) / `process` (4) | AI 리뷰·하네스 운영 회고(knowledge) |
| `dead-code` (6) / `yagni` (4) / `defensive-programming` (2) | 미사용·과설계 배제 |
| `testing` (3) / `integration-test` (2) / `concurrency-test` (1) | 테스트 |
| `adr` (3) / `documentation` (2) / `immutability` (1) | 문서 정본·불변 |

### 동의어 매핑

- 플랫폼: `web`/`웹`/`client` → `frontend`, `server`/`서버`/`api` → `backend` (platform 태그군 참조)
- `refactoring` → `refactor`, `naming` → `naming-convention` (권장)
- **미통합(lint 대상):** `double-payment`·`duplicate-payment`·`double-charge`(→ 하나로), `unknown-status`·`unknown-state`(→ 하나로), `reconcile`→`reconciliation`, `foreign-key`·`cross-aggregate-reference`→`fk`/`cross-aggregate`, `save-and-flush`·`save-flush`. 이번 ingest에선 노트 frontmatter를 소급 수정하지 않고 남겨둠(lint에서 승인 후 통합).

### 상위/하위 관계

- `payment` ⊃ `payment-attempt`, `reservation`, `naverpay`, `compensation`, `reconciliation`, `escalation`
- `concurrency` ⊃ `optimistic-lock`, `pessimistic-lock`, `race-condition`, `lost-update`, `gap-lock`
- `exception-handling` ⊃ `exception-strategy`, `error-category`, `error-code`, `adapter`
- `ddd` ⊃ `hexagonal`, `port-adapter`, `adapter`, `cqrs`, `aggregate`
