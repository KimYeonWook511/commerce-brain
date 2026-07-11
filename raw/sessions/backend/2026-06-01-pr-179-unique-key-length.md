---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 179 }
---

# tbl_payment_attempt unique 제약이 스키마에 없던 회귀 복구 — multi-column unique 대상 컬럼에 length 명시

`tbl_payment_attempt`에 "한 결제 시도를 유일하게 만드는" multi-column unique 제약이 엔티티에는 선언돼 있었지만 실제 DB 스키마에는 적용되지 않은 채 운영되고 있었다. 원인은 대상 컬럼 4개가 `@Column(length=...)` 없이 Hibernate 기본 `VARCHAR(255)`로 생성돼 utf8mb4 환경에서 unique index 바이트 합이 InnoDB 한도를 넘었고, Hibernate가 그 제약 생성 실패를 WARN 로그만 남기고 조용히 부팅을 이어갔기 때문이다. 이슈 #176 진단부터 PR #179 작성·리뷰 대응까지 한 세션에서 마무리했고, 진단 과정에서 초기 가설이 완전히 틀렸던 게 이 세션의 핵심이다. 후속으로 태그 차원 재설계(#177)와 cross-entity `merchantPayKey` 길이 일관성(#178)을 별도 이슈로 떼어냈다.

이 결정의 정본 ADR과 태스크 문서(요구·아키텍처·db-schema·회고)는 코드 repo에 있고, 여기에는 내 이해·검토한 대안·트레이드오프·막힌 점만 남긴다.

## 결정한 것

### 1. multi-column unique 대상 컬럼만 `@Column(length=...)`을 명시 — 기존 컨벤션의 좁은 예외로 신설

`PaymentAttempt`의 unique 제약에 들어가는 4개 컬럼에 length를 박았다. 이 프로젝트에는 "enum 컬럼에는 `@Column(length)`을 명시하지 않고 Hibernate 기본값을 쓴다"는 컨벤션이 이미 있었다 — enum 값은 개발자가 정의한 코드 상수만 들어와 외부 입력 길이 제한 같은 의미가 없고, length를 박으면 enum을 추가할 때 동기화 부담만 늘기 때문이다. 이번 사고는 그 컨벤션이 **multi-column unique 대상 컬럼**에서는 InnoDB 바이트 한도를 깨뜨리는 부작용을 드러낸 경우다.

- **핵심:** 그 컨벤션을 통째로 폐기하고 모든 컬럼에 length를 박는 넓은 옵션(옵션 B)은 과하다고 봐 기각했다. 이번 사고의 원인은 "unique index 바이트 합"이지 일반 컬럼 길이가 아니므로, 컨벤션은 일반 영역에서 유지하고 "multi-column unique 대상 컬럼은 length 명시"라는 좁은 예외만 추가하는 게 정합적이라 판단했다.
- **검토한 대안:** (옵션 B) enum length 미명시 컨벤션을 폐기하고 전 컬럼에 length 명시 — 범위가 과해 기각. (옵션 C) `@UniqueConstraint`/`@Index`에 컬럼별 prefix length를 지정하는 SQL 작성 — JPA 표준에 없는 hack에 가깝고 명시성이 떨어져 기각.

### 2. `halt_on_error: true`는 local에만 적용

Hibernate DDL 실행 실패를 부팅 단계에서 즉시 터뜨리는 `hibernate.hbm2ddl.halt_on_error: true`를 `application-local.yml`에만 켰다. 이번처럼 스키마 회귀가 silent하게 묻히는 걸 개발자가 부팅 시점에 바로 인지하게 하려는 안전망이다.

- **test 제외:** Testcontainer로 신선한 MySQL을 띄우는 `dockerTest`는 부팅 시 `ddl-auto: create-drop`의 drop 단계에서 `ALTER TABLE ... DROP FOREIGN KEY ...`를 실행하는데, MySQL은 이 구문에 `IF EXISTS`를 지원하지 않아 빈 컨테이너에서 무해하게 실패한다. `halt_on_error`가 이 진짜 무해한 실패까지 잡아 컨텍스트 로드를 깨뜨리므로 test에는 켜지 않았다.
- **prod 제외:** 아직 운영 미가동이고, 추후 Flyway 도입과 함께 `ddl-auto: validate`로 전환하면 Hibernate가 DDL을 실행하지 않게 돼 `halt_on_error`의 발동 조건 자체가 사라진다. (실제로 이후 Flyway 도입 PR에서 이 설정은 제거됐다.)
- **트레이드오프(인지한 fragility):** local 적용은 "local의 ddl-auto가 update"라는 전제에 묶여 있다. local ddl-auto가 미래에 `create`/`create-drop`으로 바뀌면 같은 ALTER FK DROP 충돌이 재발하므로, ddl-auto를 바꿀 때 `halt_on_error` 적용 여부를 함께 재검토해야 한다. 지금은 local·prod 모두 update를 쓰는 의도(prod 동작 검증)가 명확해 위험이 낮다고 봤다.

### 3. "race가 실제로 났는지" 자체를 검증하는 단언(`anyMatch`)을 제거

동시성 테스트에서 "race가 실제로 발생했다"를 확인하려고 `errors.anyMatch(DataIntegrityViolationException)`로 단언하던 부분을 제거했다.

- **핵심:** race 발생 여부는 환경 의존적이다. 빠른 환경에서는 거의 항상 race가 나지만, CPU 코어가 적거나 CI 부하가 큰 환경에서는 첫 스레드가 커밋한 뒤 나머지가 사전 `find` 분기로 빠져 race가 아예 안 날 수도 있다. 이 단언은 그래서 CI flake의 원천이었다. race 가시화의 가치 < CI 안정성이라 판단해 제거했다.
- **안전망은 유지:** 대신 `countAttempts == 1`이라는 데이터 invariant(이번 사고와 동일한 unique 누락 회귀는 이 단언이 곧장 잡는다)와, 발생한 예외를 분류하는 기존 `assertRaceOrPaymentError` helper 조합으로 남겼다. 잃는 건 "race 발생 자체의 가시화"뿐이고, 결과 상태(count)·발생 예외 분류(helper)는 환경 독립적으로 남는다.

### 4. HikariCP pool을 동시성 테스트 클래스 단위로 inline 명시

`NaverPayServiceConcurrencyTest`에 `@SpringBootTest(properties = {...})`로 `maximum-pool-size=30`, `minimum-idle=10`, `connection-timeout=30000`을 박았다. 20 스레드 + 보상 흐름(REQUIRES_NEW 별도 커넥션)까지 쓰는 이 테스트는 기본 pool size 10으로는 커넥션을 수용하지 못한다. 다른 동시성 테스트(`OrderConcurrencyServiceTest` 등)가 이미 같은 클래스 단위 inline 설정 컨벤션을 쓰고 있어 그에 맞췄다.

### 5. 컬럼 length 표기 순서를 unique key columnNames 순서로 통일

length 값을 문서·검수에서 적을 때 엔티티 선언 순서가 아니라 unique key의 `columnNames` 순서를 기준으로 통일했다.

- 엔티티 선언 순서는 `merchantPayKey(64)`, `paymentId(64)`, `provider(32)`, `type(32)` → **64/64/32/32**.
- unique key `columnNames` 순서는 `merchant_pay_key(64)`, `provider(32)`, `payment_id(64)`, `type(32)` → **64/32/64/32**.
- 두 순서가 달라 문서마다 값이 뒤섞여 혼동을 줬다. 이번 사고의 본질이 "unique index의 바이트 합"이므로 unique key 기준으로 통일하는 게 맞다고 봤다. 합계는 768 bytes로 InnoDB 한도 3072의 25% 수준이라 enum 값이 늘거나 PG가 더 긴 ID를 발급해도 여유가 있다. (`merchantPayKey` 64는 우리 ID 형식 prefix+UUID/ULID를 담기 충분, `paymentId` 64는 네이버페이 발급 ID 20~30자의 2배, provider/type 32는 현재 enum 최대 10자 미만의 3배 여유.)

**지금 다시 본다면:** 정정 결정을 도입할 때, enum length 미명시 컨벤션을 정한 원래 결정 쪽에도 "예외 적용 조건" 한 줄을 *역으로* 달아두는 걸 고려할 만하다. 지금은 새 예외 결정이 원래 컨벤션을 단방향으로 참조할 뿐이라, 신규 도입자가 원래 컨벤션만 보면 예외의 존재를 모른 채 충돌 인식이 안 생긴다.

## 막힌 점·해결

### 1차 — 잘못된 가설로 시간 손실

`order-idempotency-cache-simplification` 작업 중 `dockerTest`에서 `NaverPayServiceConcurrencyTest`가 대량 실패(8케이스 중 7 실패)하는 걸 발견했고, develop HEAD 깨끗한 상태에서도 재현돼 payment 도메인의 기존 결함으로 분리 인지했다. 처음엔 `IncorrectResultSizeDataAccessException: 2 results were returned`(사전 `find`가 2건 반환)를 보고 "race window가 너무 넓다"는 가설로 진단을 시작했다.

사실은 스키마에 unique 제약이 **처음부터 없었다.** 단일 테스트만 분리 실행하고 **Hibernate DDL 로그를 직접 dump**하고 나서야 발견했다:

```
Specified key was too long; max key length is 3072 bytes
```

`VARCHAR(255) × 4컬럼 × utf8mb4(4byte) = 4080 bytes > 3072`. Hibernate 기본 핸들러 `ExceptionHandlerLoggedImpl`이 스키마 생성 실패를 WARN으로만 로그하고 silent하게 부팅을 진행한 게 이 회귀가 운영에 묻힌 경로다.

- **교훈:** 가설을 검증할 때 우선 *로그 가시화*를 했어야 했다. find-first 패턴 분석에 먼저 뛰어들며 빠뜨린 게 reflex가 됐어야 할 단계였다.

### 2차 — latch barrier로 race를 강제하려다 실패 (revert)

외부 AI 코드 리뷰(codex)가 `anyMatch` 단언의 flake 위험을 P2로 지적했다. "race를 latch로 강제하면 `anyMatch`를 유지할 수 있지 않나?"라는 직관을 따라 `@MockitoSpyBean PaymentAttemptRepository`를 두고 `findApproveAttempt`/`save`에 `CountDownLatch` barrier를 걸어봤다.

- **결과:** 여러 케이스가 실패했다. `errors`에 기대한 `DataIntegrityViolationException` 대신 `UndeclaredThrowableException`이 담겼다. 원인은 이중 wrapping이다 — `@Repository` Adapter가 이미 Spring CGLIB proxy인데 Mockito spy로 한 번 더 감싸면, `callRealMethod`의 reflection 경로에서 unchecked 예외가 `InvocationTargetException` → `UndeclaredThrowableException`으로 포장된다. 게다가 barrier 대기(약 5s×2)에 처리 시간까지 더해져 동시 실행 완료 대기(약 10s)를 초과하면서 일부 케이스는 timeout interrupt → `JpaSystemException` 연쇄까지 났다. (실패 케이스 수·타임아웃 초 값은 당시 기억이라 정확 수치는 미검증.)
- **해결:** 실험을 revert하고 리뷰 원래 권고(anyMatch 제거)로 돌아왔다. 결정성을 더 정밀하게 강제하려면 service 레이어에서 직접 hook하거나 adapter를 분리해야 하는데, 그건 이 태스크 scope를 넘어서 미뤘다.

### 3차 — HikariCP pool 부족으로 잠재 flake

`anyMatch` 제거 후에도 중복 결제 보상 취소 경로 케이스(보상 흐름이 REQUIRES_NEW로 별도 커넥션을 더 쓰는 케이스)가 가끔 `CannotCreateTransactionException: Could not open JPA EntityManager`로 깨졌다. 20 스레드 + 보상 흐름이 쓰는 커넥션이 기본 pool 10을 초과한 것이다. 위 결정 4번의 클래스 단위 inline 설정으로 해소했다. 다른 동시성 테스트가 이미 같은 패턴을 쓰고 있었으니 진작 찾아봤어야 했다.

## 배운 것

- **`@Column(length=...)` 미명시는 multi-column unique 제약에서 스키마 회귀의 silent 진입점이다.** utf8mb4 + InnoDB 3072 bytes 한도. 새 unique를 도입할 땐 바이트 합 계산이 인지 부담으로 붙는다 — 산정 가이드가 필요하다.
- **`ALTER TABLE ... DROP FOREIGN KEY`는 MySQL에서 `IF EXISTS`를 지원하지 않는다.** Testcontainer fresh DB의 `ddl-auto: create-drop` 첫 부팅에서 충돌한다. `drop table if exists`는 도는데 외래키 drop은 안 도는 비대칭이라, `halt_on_error`를 켤 때 가장 흔한 false-positive다. (초기 분석은 "drop table if exists로 도니 무해한 drop 실패는 거의 없다"였는데, 이는 일반 테이블 drop에 한정된 사실이었고 외래키 drop을 누락한 오판이었다.)
- **Hibernate `ExceptionHandlerLoggedImpl` 기본 동작은 스키마 에러를 silent 처리한다.** `halt_on_error`가 없으면 스키마 정합성이 깨진 채로도 부팅이 계속된다. 다음 비슷한 사고를 막으려면 부팅 단계에서 실패시키는 게 가장 빠른 발견 경로다.
- **Spring CGLIB proxy + Mockito spy 이중 wrapping은 reflection 경로의 unchecked 예외 unwrapping을 깨뜨린다.** spy로 동시성을 제어하려면 interface가 아니라 service 레이어에서 직접 hook하거나 production 코드에 test-specific 진입점을 두는 게 안전하다.
- **"race가 실제로 났는지 자체"는 단언으로 강제하기 어렵다.** 데이터 invariant(`count == 1`)와 행동 invariant(예외 *분류*)는 환경 독립적이지만, 행동 invariant 중 "race 발생 여부"는 OS 스케줄러 의존이다. 일반화하면, 동시성 테스트의 단언은 *결과 상태*와 *발생 시 정합성*에 집중하고 *발생 자체*는 trade-off로 포기한다.
- **다른 동시성 테스트의 HikariCP pool 컨벤션을 사전에 봤어야 했다.** 새 동시성 테스트를 추가할 땐 기존 컨벤션 search가 reflex여야 한다.
- **이슈 본문의 초기 가설은 틀릴 수 있고, 그대로 두면 다음 작업자가 함정을 반복한다.** 이번엔 진단 정정 노트를 이슈 본문 상단에 명시하고 무효화된 작업 범위를 strikethrough로 표시했다. 가설이 바뀌면 *이슈 본문도 바뀌어야* 한다.
- **외부 리뷰가 본 작업자가 인지 못 한 환경 가정을 잡아준다.** `anyMatch`의 환경 의존성 지적이 그랬다.

## 미해결·열린 질문

- **enum length 미명시 컨벤션과 이 예외의 공존.** 두 결정이 공존하므로 "일반 원칙 vs 좁은 예외"라는 관계를 명시해 충돌 인식을 차단해 뒀지만, 원래 컨벤션 쪽에 역참조 한 줄을 다는 개선은 아직 안 했다(위 "지금 다시 본다면").
- **length 값이 지금의 데이터 흐름 가정에 묶여 있다.** `paymentId` 30자 이하 같은 전제는 새 PG를 붙이거나 enum 값이 늘면 깨질 수 있어, 그 시점에 재검토가 필요하다.
- **cross-entity 길이 일관성(#178).** `Order.merchantPayKey`/`Payment.merchantPayKey`/`Payment.pgPaymentId`의 length를 PaymentAttempt와 맞추는 건 이번 범위 밖이다. 현재 운영 데이터가 없어 단순 엔티티 수정으로 가능하다.
- **다른 엔티티의 multi-column unique가 한도를 넘는지는 자동 검증하지 않는다.** 현재 인벤토리(`Order`·`CartItem`·`ProcessedEvent`의 unique/index)는 한도 안임을 확인했지만, 신규 추가 시 이 결정을 참고해 length를 명시해야 한다.
- **태그 차원 재설계(#177).** `docker`/`sandbox` 같은 환경 요구 태그와 `concurrency`/`batch` 같은 카테고리 태그가 뒤섞인 걸 정리하는 건 별도 이슈로 뺐다. 이번엔 `dockerTest`에 `excludeTags "concurrency"` 한 줄로 같은 클래스 중복 실행만 임시 해소했다.
- **Flyway 도입 시 정리될 것들.** prod 스키마 정합성(unique 누락·컬럼 length) 점검은 Flyway 도입 흐름에서 처리하고, `ddl-auto: validate` 전환 시 `halt_on_error`의 fragile한 전제도 자연스럽게 해소된다.
