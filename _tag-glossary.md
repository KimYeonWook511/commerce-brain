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

2026-07-14 최초 대규모 ingest로 등재하고 2026-08-17 ingest에서 갱신했다. canonical = 소문자·하이픈·단수형. 아래 개수는 2026-08-17 기준이며 사용처 다수(5+)인 태그 위주다. 전체 태그는 각 노트 frontmatter가 진실이다. 태그가 **279종**으로 산발이 있어 **동의어 통합·싱글턴 정리는 lint 후보**다.

### 도메인·기능

| canonical | 비고 |
|---|---|
| `payment` (79) | 결제 도메인. 최상위 관심사 |
| `refund` (30) | 환불. 2026-08 재설계로 독립 aggregate가 됨 |
| `order` (27) | 주문 도메인 |
| `partial-cancel` (18) | 부분취소·부분환불 |
| `stock` (11) / `product` (7) / `cart` (6) | 재고·상품·장바구니 도메인 |
| `auth` (7) / `security` (7) / `authentication` (5) / `member` | 인증·인가 |
| `naverpay` (19) / `pg-gateway` (15) | 결제사 연동 |
| `reservation` (9) | 결제 예약(RESERVED). 2026-08에 활성 슬롯으로 흡수 |
| `double-payment` (10) | 이중결제·이중환불 |
| `payment-attempt` (3) | 옛 결제 시도 이력 aggregate |

### 아키텍처·기법

| canonical | 비고 |
|---|---|
| `idempotency` (32) | 멱등성 |
| `concurrency` (26) | 동시성 |
| `unique-constraint` (21) / `db-unique` | DB unique 제약 |
| `reconciliation` (23) / `escalation` (12) / `unknown-status` (8) / `starvation` | 회수(대사)·에스컬레이션·결과 불명 |
| `exception-handling` (21) / `error-code` (11) / `exception-strategy` | 예외 처리·에러 코드 체계 |
| `compensation` (15) / `saga` (6) | 보상·사가 |
| `optimistic-lock` (14) / `pessimistic-lock` (8) | 낙관/비관 락 |
| `jpa` (15) / `hibernate` (9) / `flush` / `orm` / `persistence` | 영속성 |
| `ddd` (13) / `aggregate` (10) / `adapter` (10) / `layered-architecture` (7) / `hexagonal` / `port-adapter` | DDD·헥사고날 |
| `transaction-boundary` (11) / `transaction` (11) / `outbox` / `after-commit` | 트랜잭션 경계 |
| `mysql` (13) / `innodb` (5) | DB 엔진 특성 |
| `redis` (12) / `cache` | Redis·캐시 |
| `migration` (8) / `schema` (7) / `flyway` / `cross-aggregate` (5) / `fk` | 스키마·FK→ID 이관 |
| `invariant` (8) / `state-model` / `money-model` | 불변식·상태·금액 모델 |
| `append-only` (5) / `ledger` / `event-sourcing` | append-only 원장 |
| `api-contract` (6) / `access-control` | 계약·접근 제어 |

### 프로세스·메타

| canonical | 비고 |
|---|---|
| `convention` (21) / `naming-convention` (6) | 코드/문서 컨벤션 |
| `process` (12) / `workflow` (7) / `harness` (7) | 워크플로·하네스 운영 회고(knowledge) |
| `verification` (8) / `design-review` / `spec-review` | 설계·명세 검증 방법론 |
| `code-review` (7) / `ai-review` | AI·자동 리뷰 |
| `adr` (9) / `documentation` (6) / `task-docs` / `canonical-doc` / `stale-doc` / `immutability` | 문서 정본·불변 |
| `dead-code` (8) / `refactor` (7) / `yagni` (5) / `defensive-programming` | 미사용·과설계 배제·리팩터 |
| `testing` (6) / `integration-test` / `regression-test` / `concurrency-test` | 테스트 |
| `operations` (5) / `observability` / `notification` / `manual-review` | 운영·관측·통지 |
| `measurement` / `cost` / `git` | 실측·비용·이력 추적 |

### 동의어 매핑

- 플랫폼: `web`/`웹`/`client` → `frontend`, `server`/`서버`/`api` → `backend` (platform 태그군 참조)
- `refactoring` → `refactor`, `naming` → `naming-convention` (권장)
- **미통합(lint 대상):** `double-payment`·`duplicate-payment`·`double-charge`(→ 하나로), `unknown-status`·`unknown-state`(→ 하나로), `transaction`·`transaction-boundary`(→ 하나로), `jpa`·`hibernate`(상위/하위 관계로 정리), `schema`·`schema-design`, `reconcile`→`reconciliation`, `foreign-key`·`cross-aggregate-reference`→`fk`/`cross-aggregate`, `save-and-flush`·`save-flush`, `spec-review`·`design-review`. 노트 frontmatter를 소급 수정하지 않고 남겨둠(lint에서 승인 후 통합).

### 상위/하위 관계

- `payment` ⊃ `refund`, `partial-cancel`, `payment-attempt`, `reservation`, `naverpay`, `pg-gateway`, `compensation`, `reconciliation`, `escalation`, `unknown-status`, `double-payment`
- `concurrency` ⊃ `optimistic-lock`, `pessimistic-lock`, `race-condition`, `lost-update`, `gap-lock`, `deadlock`
- `exception-handling` ⊃ `exception-strategy`, `error-category`, `error-code`, `adapter`
- `ddd` ⊃ `hexagonal`, `port-adapter`, `adapter`, `cqrs`, `aggregate`, `layered-architecture`
- `jpa` ⊃ `hibernate`, `flush`, `persistence`
- `process` ⊃ `workflow`, `harness`, `design-review`, `spec-review`, `verification`, `code-review`
