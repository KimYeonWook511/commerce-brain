---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 179 }
---

# PR #179 — tbl_payment_attempt unique constraint 미적용 복구

## 한 일

- 이슈 #176 진단부터 해결까지 한 세션에서 마무리. PR #179 작성 + review 대응 + thread resolve.
- 후속 이슈 #177(Tag 차원 재설계), #178(cross-entity `merchantPayKey` 길이 일관성) 등록.

## 결정한 것

정본은 `commerce-backend/docs/ADR.md` ADR-023, `commerce-backend/docs/tasks/payment-attempt-unique-key-length/`. 여기는 내 이해·트레이드오프만.

1. **multi-column unique constraint 대상 컬럼만 `@Column(length=...)` 명시** — ADR-018("enum length 미명시")의 좁은 예외로 신설. ADR-018을 폐기하고 모든 컬럼에 length를 박는 옵션 B는 과함. 원인이 "unique index 바이트 합"이지 일반 컬럼 길이가 아니라서 범위를 좁히는 게 정합적.
2. **`halt_on_error: true`는 local만 적용** — test는 Testcontainer fresh MySQL 부팅 시 `ALTER TABLE ... DROP FOREIGN KEY ...`가 IF EXISTS 없이 실행되어 무해 실패가 발생, halt_on_error와 충돌. prod는 운영 미가동 + 추후 Flyway 도입 시 validate로 가면 의미 소실. local도 ddl-auto가 update이라는 전제에 묶여 있어 fragile하다는 점 인지.
3. **`anyMatch(DataIntegrityViolationException)` 단언 제거** — race가 실제로 발생했는지 자체는 환경 의존적이라 CI flake 위험. race 가시화 가치 < CI 안정성. `countAttempts == 1` 데이터 invariant + `assertRaceOrPaymentError` helper 조합으로 안전망은 그대로 유지.
4. **HikariCP pool 클래스 단위 inline 설정** — 다른 concurrency 테스트(`OrderConcurrencyServiceTest` 등)와 동일 컨벤션. `@SpringBootTest(properties = {...})`로 `maximum-pool-size=30, minimum-idle=10, connection-timeout=30000`. 기본 pool size 10으론 20 thread + 보상 흐름 수용 불가.
5. **컬럼 length 표기 순서를 unique key columnNames 순서로 통일** — entity 선언 순서(`merchantPayKey, paymentId, provider, type` → 64/64/32/32)와 unique key columnNames 순서(`merchant_pay_key, provider, payment_id, type` → 64/32/64/32)가 달라 문서마다 혼동. unique key 기준이 본 사고의 본질이라 그쪽으로 통일.

지금 다시 본다면: 정정 ADR을 도입할 때 ADR-018 본문에 "예외 적용 조건" 한 줄을 *역으로* 추가하는 것도 고려해볼 만. 두 ADR을 따로 두면 신규 도입자가 둘 다 발견해야 충돌 인식이 형성됨. 지금은 ADR-023이 ADR-018을 명시 참조하는 단방향이라 가독성이 약함.

## 막힌 점

### 1차 — 잘못된 가설로 시간 손실

`IncorrectResultSizeDataAccessException: 2 results were returned`를 보고 "race window가 너무 넓다"는 가설로 진단을 시작. 사실은 schema에 unique constraint이 처음부터 없었음. **Hibernate DDL 로그를 직접 dump**해서야 발견:

```
Specified key was too long; max key length is 3072 bytes
```

`VARCHAR(255) × 4 × utf8mb4(4byte) = 4080 bytes > 3072`. Hibernate 기본 핸들러 `ExceptionHandlerLoggedImpl`이 schema 생성 실패를 WARN으로만 로그하고 silent하게 부팅 진행.

가설을 검증할 때 우선 *로그 가시화*를 했어야 했다. find-first 패턴 분석으로 빠진 게 reflex.

### 2차 — latch barrier로 race 강제하려다 실패

codex P2 / Gemini가 anyMatch flake 위험을 지적. "race를 latch로 강제하면 anyMatch 유지 가능하지 않나?"라는 직관에 따라 `@MockitoSpyBean PaymentAttemptRepository` + `findApproveAttempt`/`save`에 `CountDownLatch` barrier 추가.

→ 6개 케이스 실패. `errors`에 `DataIntegrityViolationException` 대신 `UndeclaredThrowableException`. 원인: `@Repository` Adapter가 이미 Spring CGLIB proxy인데 Mockito spy로 한 번 더 wrap → `callRealMethod`의 reflection 경로에서 unchecked exception이 InvocationTargetException → UndeclaredThrowableException으로 wrap. 그리고 barrier 5s × 2 + 처리 시간이 `runConcurrent.doneLatch.await(10s)`를 초과해 일부 케이스는 timeout interrupt → JpaSystemException 연쇄.

revert 후 codex 원래 권고(anyMatch 제거)로 돌아옴. **spy + CGLIB 이중 wrapping은 reflection 경로의 wrapping을 깨뜨릴 위험이 항상 있음.** 더 들어가려면 service 레이어 hook이나 adapter 분리가 필요한데 task scope를 넘음.

### 3차 — HikariCP pool 부족으로 CON-6 잠재 flake

anyMatch 제거 후에도 CON-6이 가끔 `CannotCreateTransactionException: Could not open JPA EntityManager`. 20 thread + 보상 흐름(REQUIRES_NEW까지) 사용 connection이 기본 pool 10을 초과. 클래스 단위 inline 설정 추가로 해소. 다른 concurrency 테스트가 이미 같은 패턴 쓰고 있었음 — 찾아봤어야 했다.

## 배운 것

- **`@Column(length=...)` 미명시는 multi-column unique constraint에서 schema 회귀의 silent 진입점.** utf8mb4 + InnoDB 3072 bytes 한도. 새 unique 도입 시 바이트 합 계산이 인지 부담이 됨. 정본 ADR-023에 산정 가이드.
- **`ALTER TABLE ... DROP FOREIGN KEY`는 MySQL에서 IF EXISTS 미지원.** Testcontainer fresh DB의 ddl-auto: create-drop에서 첫 부팅 시 충돌. `drop table if exists`는 도는데 외래키는 안 도는 비대칭. halt_on_error를 켤 때 가장 흔한 false-positive.
- **Hibernate `ExceptionHandlerLoggedImpl` 기본 동작은 schema 에러 silent.** halt_on_error 없으면 stack trace까지 찍히는 WARN인데 운영 로그에서 묻힘. 다음 비슷한 사고를 막으려면 부팅 단계에서 실패시키는 게 가장 빠른 발견 경로.
- **Spring CGLIB proxy + Mockito spy 이중 wrapping은 reflection 경로의 unchecked exception unwrapping을 깨뜨림.** spy로 동시성을 제어하려면 interface가 아니라 service 레이어에서 직접 hook하거나 production 코드에 test-specific 진입점을 두는 게 안전.
- **"race가 실제로 일어났는지 자체"는 단언으로 강제하기 어렵다.** 데이터 invariant(`count == 1`)와 행동 invariant(예외 *분류*)는 환경 독립적이지만, 행동 invariant 중 "race 발생 여부"는 OS 스케줄러 의존. 일반화: 동시성 테스트의 단언은 *결과 상태*와 *발생 시 정합성*에 집중하고, *발생 자체*는 trade-off로 포기.
- **다른 concurrency 테스트의 HikariCP pool 컨벤션을 사전에 봤어야 했다.** `OrderConcurrencyServiceTest`가 이미 클래스 단위 inline 설정 패턴을 쓰고 있었음. 새 동시성 테스트 추가 시 기존 컨벤션 search가 reflex여야 함.
- **이슈 본문의 초기 가설은 잘못될 수 있고, 그대로 두면 다음 작업자가 함정 반복.** 본 PR에서는 진단 정정 노트를 이슈 본문 상단에 명시 + 작업 범위 strikethrough. 가설이 바뀌면 *이슈 본문도 바뀌어야* 한다.
- **외부 리뷰(codex, Gemini)가 본 작업자가 인지 못 한 환경 가정을 잡아줌.** 같은 P2 코멘트가 둘 다에서 나왔다 — 두 reviewer가 독립적으로 같은 위험을 짚었다는 건 강한 신호.

## 다음 단계

- 이슈 #177 — `@Tag` 차원 정리(`docker`/`sandbox` 환경 요구 vs `concurrency`/`batch` 카테고리). 본 PR에서 `dockerTest`에 `excludeTags "concurrency"` 한 줄로 임시 처리.
- 이슈 #178 — `Order.merchantPayKey`/`Payment.merchantPayKey`/`Payment.pgPaymentId`의 length를 PaymentAttempt와 일치시키는 cross-entity 일관성. 현 시점 운영 데이터 없으므로 단순 entity 수정으로 가능.
- Flyway 도입 시 — prod schema 정합성 점검(unique 누락, 컬럼 length) 그 흐름에서. ddl-auto: validate 전환 시 `halt_on_error`의 fragile dependency도 자연스럽게 해소.
- ADR-018 본문에 "예외 적용 조건" 한 줄 역참조 추가도 고려.

## 참고

- 이슈 #176, #177, #178
- PR #179
- `commerce-backend/docs/ADR.md` ADR-011, ADR-018, ADR-023
- `commerce-backend/docs/tasks/payment-attempt-unique-key-length/`
