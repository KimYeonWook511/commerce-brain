---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [exception-handling, payment, duplicate-payment, unique-constraint, constraint-name, exception-translator, hibernate, mysql, adapter, spring-dao, integration-test]
created: 2026-06-09
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-06-09-pr-228-sql-translator-removal-getconstraintname]]"
---

# DB unique 위반 예외 번역 빈 제거 — 이중결제 식별을 제약명 기반으로 전환

결제 승인 저장에서 "한 주문에 성공 결제 하나"를 강제하는 unique 제약 위반을 잡아 이중결제로 가려내는 판단이, 그동안 `SQLException` 메시지 문자열 정규식에 의존하고 있었다. 그 뿌리에 JDBC `SQLException`을 Spring DAO 예외로 번역하던 설정 빈이 있었는데, 존재 이유가 이미 사라졌음을 확인하고 제거하면서 식별 방식을 구조적 접근자로 바꾼 세션이다(#227, PR #228). 이 작업은 [[결제승인완료-보상-완료우선-이중결제-adapter매핑]](#225)이 스코프 밖으로 분리한 후속이다 — 같은 `saveApproved`/`saveAndFlush`/제약명 매핑, 같은 이중결제 도메인 에러 `PAYMENT_DUPLICATE`.

## 컨텍스트 — 이중결제 식별이 SQLException 메시지 정규식에 의존

빈이 있던 시절에는 `JpaConfig`의 `SQLErrorCodeSQLExceptionTranslator`(DB 벤더 에러코드 기반 번역기)가 unique 위반을 Spring `DuplicateKeyException`(cause=JDBC `SQLException`)으로 올렸다. 이 경로엔 Hibernate `ConstraintViolationException`이 cause 체인에 아예 없어 `getConstraintName()`이 dead였고, 그래서 식별이 `SQLException` 메시지 정규식에 의존했다. (org.springframework.dao 추상 예외 계층 자체의 노출 경계는 [[persistence-exception-노출-경계-추상수준]].)

## 결정 1 — SQLErrorCodeSQLExceptionTranslator 빈 제거 (정당화 실측 대조로 무가치 확인)

`JpaConfig`에서 그 빈을 제거했다. 빈이 있으면 `HibernateJpaConfiguration`이 translator를 감지해 `HibernateJpaDialect`에 주입하고, 없으면 dialect가 기본 `SQLStateSQLExceptionTranslator`를 써 unique 위반을 `DuplicateKeyException` 대신 `DataIntegrityViolationException`으로 올린다.

- **빈의 원래 목적은 이미 폐기돼 있었다.** 원래 application이 `DuplicateKeyException`을 직접 catch하려고 등록했으나, 그 방식은 [[find-first-write-not-check-db-unique-멱등]] 패턴으로 대체돼 어디서도 더는 catch하지 않는다.
- **남은 유일한 정당화도 실측 대조로 무가치 확인.** 마지막 정당화는 "운영 로그에서 unique 위반을 `DuplicateKeyException` 타입으로 구분"뿐이었는데, 실제 핸들러·에러코드·cause 체인을 대조하니: (1) 빈 유무와 무관하게 미처리 unique 위반은 같은 안전망 핸들러로 가 같은 에러코드 `COMMON-500-1`로 분류된다. (2) `Duplicate entry ... for key 'tbl_payment.uk_...'` 형태 원본 `SQLException` 메시지가 cause 체인에 똑같이 남는다. 빈이 더하는 건 스택트레이스 최상위 wrapper 클래스명(`DuplicateKeyException`) 하나뿐이고 에러코드로 필터도 안 돼 로그 구분값으로도 쓸모없었다.
- **설계 관점 추가 근거:** 영속화 adapter는 어차피 구현체(JPA/Mybatis/JDBC)별로 따로 작성된다. JPA 경로에만 translator를 끼워 "정규화"하는 건 추상화 이득이 없다. 제약명을 소비하는 `PaymentRepositoryAdapter`는 이미 JPA 전용이라 Hibernate API 직접 의존이 레이어상 오히려 자연스럽다.
- **검토한 대안:** (A) 빈 유지+문서화만 — `getConstraintName()`이 table prefix를 포함해(아래) 제거 이점이 반감된다는 게 유지 논거였으나 정당화 자체가 무가치라 기각. (B) translator를 모든 adapter에 주입해 전역 정규화 — adapter가 구현체별로 갈려 추상화 이득 없어 기각.

## 결정 2 — 식별을 getConstraintName() 제약명 비교로 전환

판단부(`PaymentRepositoryAdapter.isApprovedOrderKeyViolation`)를 `SQLException` 메시지 정규식에서 Hibernate `ConstraintViolationException.getConstraintName()` 기반으로 바꿨다. `catch (DataIntegrityViolationException)`에서 이 판단부가 true면 `PaymentException(PAYMENT_DUPLICATE)`로 매핑, false면 원 예외 rethrow — `uk_payment_approved_order_key` 위반만 이중결제로 가려내고 다른 무결성 위반은 오매핑하지 않는다.

- **비교 방식 확정 과정:** 처음엔 prefix형(`tbl_payment.uk_...`)/bare형(`uk_...`)을 모두 흡수하려고 "마지막 dot 세그먼트 추출 후 상수 비교"로 구현. AI 코드 리뷰(Gemini 등)가 (1) 대소문자 비구분(`equalsIgnoreCase`), (2) cause 체인 끝까지 순회를 제안. 최종은 dot 세그먼트·상수를 없애고 MySQL이 실제 반환하는 전체 문자열 `"tbl_payment.uk_payment_approved_order_key"`를 `equalsIgnoreCase`로 직접 비교하며 cause 체인을 끝까지 순회하는 형태로 단순화.
- 이중결제 식별 **동작 자체는 보존**했다. 바뀐 건 식별 메커니즘(메시지 문자열 → 구조적 접근자)뿐.

## 실측 — getConstraintName()이 table-qualified(prefix 포함) 반환, 가설 뒤집힘

방향을 정하기 전에 핵심 불확실성을 먼저 실측했다: "빈을 제거하면 `getConstraintName()`이 깔끔한 bare name을 주는가?" 일회성 통합 테스트로 확인(`EntityManager.flush()`로 Spring Data repository 프록시의 예외 변환 레이어를 우회해 Hibernate 원본 예외를 직접 관찰).

- **결과:** MySQL 8 Testcontainers에서 `getConstraintName()`은 table prefix를 포함한 `tbl_payment.uk_payment_approved_order_key`를 반환했다(bare 아님). `SQLState=23000`, `errorCode=1062`.
- **함의:** "제거 = bare name 한 방 비교" 가설이 틀렸다. prefix 처리가 여전히 필요해 단순화 효과가 반감됐다. 그래도 빈 정당화가 무가치라 제거는 그대로 진행. **방향 결정 전 핵심 불확실성을 먼저 실측하니 가설이 틀렸음을 결정 전에 발견해 헛된 단순화 기대를 막았다.**

## auditing 함정 — 빈만 떼고 @EnableJpaAuditing은 유지

`JpaConfig`엔 이 빈 외에 `@EnableJpaAuditing`(생성/수정 시각 자동 채움)도 함께 붙어 있다. 빈만 제거하고 이 어노테이션은 유지해야 한다 — 안 그러면 auditing이 꺼져 `created_at NOT NULL` 위반이 난다. 실측 초기에 auditing 없는 설정으로 테스트하다 `created_at cannot be null`로 한 번 실패해 이 숨은 결합을 뒤늦게 알아챘다.

## 트레이드오프 — MySQL 반환 형태 결합, 통합 테스트가 안전망

식별이 MySQL 반환 형태(table prefix 포함)에 결합된다. dot 세그먼트 비교가 형식 변동(prefix/bare, dialect·Hibernate 버전 차이)에는 더 견고했지만, 단일 환경(MySQL) 전제라 전체 문자열 비교의 단순함을 의도적으로 택했다. 형식이 바뀌면 통합 테스트 `PaymentRepositoryDuplicatePaymentTest`(MySQL Testcontainers에서 `uk_payment_approved_order_key` 위반이 `PAYMENT_DUPLICATE`로 매핑되는지 검증 — 미처리 위반이 공통 안전망 코드로 떨어지는 것과 달리 adapter가 의도적으로 이 한 제약만 도메인 예외로 승격하는 경로를 검증)가 회귀로 잡는 게 안전망이다.

## 미해결 — MySQL 외/Hibernate 업그레이드 시 재확인

제약명 비교가 MySQL 반환 형태(`tbl_payment.` prefix)에 결합돼 있어, MySQL 외 환경으로 확장하거나 Hibernate 메이저 업그레이드 시 반환 형태가 바뀔 수 있다 — 그때는 위 통합 테스트로 재확인이 필요하다.

배운 것(재사용 지식 후보): **Hibernate ORM 6 + MySQL의 `getConstraintName()`은 table-qualified(`table.constraint`)로 준다** — bare 기대 금지, `endsWith`/세그먼트 추출/table 포함 전체 비교 중 하나를 명시적으로 골라야 하며 이 형태는 dialect·버전 의존. 그리고 **설정/빈의 "정당화"는 말로 받지 말고 실제 핸들러·에러코드·cause 체인을 직접 대조해 검증하면 무가치가 드러난다.**

## 근거

- [[raw/sessions/backend/2026-06-09-pr-228-sql-translator-removal-getconstraintname]]
