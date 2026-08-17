---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [database, jpa, hibernate, schema, unique-constraint, mysql, convention]
created: 2026-06-01
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-01-pr-179-unique-key-length]]"
  - { repo: commerce-backend, path: "docs/adr.md#payment-attempt-unique-length" }
  - { repo: commerce-backend, path: "docs/db-schema.md#tbl_payment_attempt" }
---

# multi-column unique 대상 컬럼에 @Column(length) 명시 — enum length 미명시 컨벤션의 좁은 예외

> 이 결정의 정본 ADR·db-schema·회고 문서는 코드 repo(commerce-backend)에 있다. 여기에는 내가 이해한 문제·검토한 대안·트레이드오프·다시 본다면만 남긴다(companion). 스키마 본문·ADR 본문은 복사하지 않는다.

## 내가 이해한 문제 — silent schema drift 진입점

`tbl_payment_attempt`에 "한 결제 시도를 유일하게 만드는" multi-column unique 제약이 엔티티에는 선언돼 있었지만 실제 DB 스키마에는 적용되지 않은 채 운영되고 있었다(#176). 원인은 unique 대상 컬럼 4개가 `@Column(length=...)` 없이 Hibernate 기본 `VARCHAR(255)`로 생성돼, utf8mb4(문자당 4byte) 환경에서 unique index 바이트 합이 `255 × 4컬럼 × 4byte = 4080 bytes`로 InnoDB 한도 3072를 넘긴 것이다. Hibernate 기본 핸들러 `ExceptionHandlerLoggedImpl`이 제약 생성 실패를 `Specified key was too long; max key length is 3072 bytes` WARN 로그만 남기고 조용히 부팅을 이어가, 스키마 정합성이 깨진 채로 운영에 묻혔다.

즉 `utf8mb4 × VARCHAR(255) × multi-column unique`가 이 프로젝트의 대표적 silent-drift 진입 조합이다. 이 패턴 일반화는 [[silent-schema-drift-패턴]]에 정리돼 있고, 이 사건은 그 topic의 대표 사례다.

진단 자체가 한 번 완전히 헛짚었다. 처음엔 `IncorrectResultSizeDataAccessException: 2 results were returned`를 보고 "race window가 너무 넓다"고 판단해 find-first 패턴 분석에 뛰어들었으나, 단일 테스트만 분리 실행하고 Hibernate DDL 로그를 직접 dump하고 나서야 "unique 제약이 처음부터 없었다"를 발견했다. 교훈은 가설 검증 전에 로그 가시화가 reflex여야 했다는 것.

## 검토한 대안

- **옵션 B — enum length 미명시 컨벤션을 폐기하고 전 컬럼에 length 명시.** 이 프로젝트엔 이미 "enum 컬럼에는 `@Column(length)`을 명시하지 않고 Hibernate 기본값을 쓴다"는 컨벤션이 있다(enum 값은 개발자가 정의한 코드 상수만 들어와 외부 입력 길이 제한 의미가 없고, length를 박으면 enum 추가 때마다 동기화 부담만 는다). 이번 사고의 원인은 "일반 컬럼 길이"가 아니라 "unique index의 바이트 합"이므로, 컨벤션을 통째로 뒤집는 건 범위가 과해 기각.
- **옵션 C — `@UniqueConstraint`/`@Index`에 컬럼별 prefix length 지정.** JPA 표준에 없는 hack에 가깝고 명시성이 떨어져 기각.

## 결정 + 이유 — 좁은 예외 신설

기존 enum length 미명시 컨벤션은 일반 영역에서 유지하고, **"multi-column unique 제약에 들어가는 컬럼만 `@Column(length)`을 명시한다"**는 좁은 예외만 추가했다. 사고의 원인이 unique index 바이트 합인 만큼, 컨벤션을 일반 영역에 남기고 예외만 정합적으로 얹는 게 옳다고 봤다.

이 length 명시는 [[flyway-도입-ddl-auto-validate-전환]] 이후에도 1차 방어선으로 남는다 — Flyway `validate`는 컬럼 타입 불일치는 잡아도 "선언은 됐으나 생성 실패한 unique 누락"까지 자동으로 못 잡기 때문이다.

> [!note] lifecycle — halt_on_error 결정은 flyway 노트로 이관
> 같은 PR의 결정2였던 `hibernate.hbm2ddl.halt_on_error: true`(local 한정, silent 스키마 회귀를 부팅 시점에 즉시 터뜨리는 안전망)는 이후 `ddl-auto: validate` 전환과 함께 발동 조건 자체가 사라져 제거됐다. 그 lifecycle은 [[flyway-도입-ddl-auto-validate-전환]]에서 다룬다. test에 켜지 않은 이유는 `ALTER TABLE ... DROP FOREIGN KEY`가 MySQL에서 `IF EXISTS`를 지원하지 않아 Testcontainer fresh DB의 `create-drop` 첫 부팅에서 무해하게 실패하는데, halt_on_error가 이 false-positive까지 잡기 때문이다(`drop table if exists`는 도는데 FK drop은 안 도는 비대칭).

## length 산정·표기 순서 통일 — unique key columnNames 기준

length 값을 문서·검수에서 적을 때 **엔티티 선언 순서가 아니라 unique key의 `columnNames` 순서**로 통일했다. 사고의 본질이 "unique index의 바이트 합"이라 그 순서가 계산의 기준이기 때문이다.

- 엔티티 선언 순서: `merchantPayKey(64)`, `paymentId(64)`, `provider(32)`, `type(32)` → 64/64/32/32
- unique key columnNames 순서: `merchant_pay_key(64)`, `provider(32)`, `payment_id(64)`, `type(32)` → 64/32/64/32

합계는 **768 bytes로 InnoDB 한도 3072의 약 25%**라 enum 값이 늘거나 PG가 더 긴 ID를 발급해도 여유가 있다(`merchantPayKey` 64는 prefix+UUID/ULID를, `paymentId` 64는 네이버페이 발급 ID 20~30자의 2배, provider/type 32는 현재 enum 최대 10자 미만의 3배를 담는다).

## 트레이드오프·리스크

- **length 값이 현재 데이터 흐름 가정에 묶여 있다.** `paymentId` 30자 이하 같은 전제는 새 PG를 붙이거나 enum 값이 늘면 깨질 수 있어, 그 시점의 재검토 트리거가 남는다(합계 768B 여유는 있으나 가정 자체는 재확인 대상).
- **새 unique 도입 시 바이트 합 계산이 인지 부담으로 붙는다.** 산정 가이드가 필요하다.

## 지금 다시 본다면

정정 결정을 도입할 때, enum length 미명시 컨벤션을 정한 **원래 결정 쪽에도 "예외 적용 조건" 한 줄을 역으로** 달아두는 게 낫다. 지금은 이 예외 결정이 원래 컨벤션을 단방향으로만 참조해, 신규 도입자가 원래 컨벤션만 보면 예외의 존재를 모른 채 충돌 인식이 안 생긴다(아래 미해결).

이 사건 자체가 "정책 문서/컨벤션이 코드 현실과 어긋나 재생산되는" 패턴의 한 예다 — 원칙은 [[문서-코드-정합성-개념정본-심볼최소화]]에 정리돼 있다.

## 미해결·후속

- **enum length 미명시 컨벤션과 이 예외의 공존.** "일반 원칙 vs 좁은 예외" 관계는 명시했으나, 원래 컨벤션 쪽 역참조 한 줄은 아직 안 달았다.
- **cross-entity 길이 일관성(#178).** `Order.merchantPayKey`/`Payment.merchantPayKey`/`Payment.pgPaymentId`의 length를 PaymentAttempt와 맞추는 건 이번 범위 밖(운영 데이터 없어 단순 엔티티 수정으로 가능).
- **타 엔티티 multi-column unique 자동 검증 부재.** 현재 인벤토리(`Order`·`CartItem`·`ProcessedEvent`)는 한도 안임을 확인했지만, 신규 추가 시 한도 초과를 자동 검증하는 장치는 없다 — 이 결정을 참고해 length를 명시해야 한다.
- 이 세션에서 CI flake 원천이던 `errors.anyMatch(DataIntegrityViolationException)` 단언 제거(결과 상태 `count == 1` invariant + 예외 분류 helper만 남김)와 HikariCP pool 클래스 단위 inline 설정도 함께 처리했다 — 동시성 테스트 단언 전략은 [[동시성-테스트-작성-규칙과-단언-전략]] 참조. 태그 차원 재설계(#177)는 [[테스트-태그-2축-모델-ci-잡-분리]]로 이어졌다.

> [!note] 인접 결정 — 같은 "DB에 뭘 맡기고 뭘 안 맡기나" 축
> 이 노트가 **스키마에 명시할 것**(복합 unique의 컬럼 length)을 정한 반면, [[enum-db-check-미사용-application-layer-위임]]은 **스키마에 안 맡길 것**(enum 유효성)을 정했다. 둘을 함께 보면 이 저장소가 DB 제약을 어디까지 쓰는지가 드러난다 — **동시성·유일성은 DB에 맡기고 값 유효성은 애플리케이션이 진다.**

## 근거

- [[raw/sessions/backend/2026-06-01-pr-179-unique-key-length]]
