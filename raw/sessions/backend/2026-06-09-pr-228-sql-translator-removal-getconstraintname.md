---
platform: backend
author: KimYeonWook511
created: 2026-06-09
origin:
  - { type: pr, repo: commerce-backend, ref: 228 }
---

# DB unique 위반 예외 번역 빈 제거 — 이중결제 식별을 제약명 기반으로 전환

결제 승인 저장에서 "한 주문에 성공 결제 하나"를 강제하는 unique 제약 위반을 잡아 이중결제로
가려내는 판단이, 그동안 `SQLException` 메시지 문자열 정규식에 의존하고 있었다. 그 뿌리에는
JDBC `SQLException`을 Spring DAO 예외로 번역해주던 설정 빈이 있었는데, 이 빈의 존재 이유가
이미 사라졌음을 확인하고 제거하면서 식별 방식을 구조적 접근자로 바꾼 세션이다.

## 결정한 것

### 1. DB unique 위반을 Spring DAO 예외로 번역하던 빈을 제거

`JpaConfig`에서 `SQLErrorCodeSQLExceptionTranslator` 빈을 제거했다. 이 빈은 JDBC 드라이버가
던지는 `SQLException`(예: MySQL unique 위반)을 Spring이 `DuplicateKeyException` 같은
`org.springframework.dao.*` 계층 예외로 변환하도록 등록돼 있던 것이다. 빈이 있으면
`HibernateJpaConfiguration`이 그 translator를 감지해 `HibernateJpaDialect`에 주입하고, 없으면
dialect가 기본 `SQLStateSQLExceptionTranslator`를 써서 unique 위반을 `DuplicateKeyException`
대신 `DataIntegrityViolationException`으로 올린다.

- **빈의 원래 목적은 이미 폐기돼 있었다.** 원래는 애플리케이션 계층이 `DuplicateKeyException`을
  직접 catch하려고 등록한 빈이었다. 그러나 그 방식은 "정상 흐름은 사전 `find` 조회로 처리하고,
  실제 동시 충돌만 안전망 500에 위임한다(find-first)"는 패턴으로 이미 대체돼, 애플리케이션
  어디에서도 `DuplicateKeyException`을 더는 catch하지 않는다.
- **남은 유일한 정당화도 검증해보니 무가치였다.** 마지막 정당화는 "운영 로그에서 unique 위반을
  `DuplicateKeyException` 타입으로 구분할 수 있다"뿐이었다. 실제 핸들러·에러코드·cause 체인을
  대조해보니: (1) 빈 유무와 무관하게 미처리 unique 위반은 같은 안전망 핸들러로 가고 같은
  에러코드(무결성 위반 공통 코드, 코드값 `COMMON-500-1`)로 분류된다. (2)
  `Duplicate entry ... for key 'tbl_payment.uk_...'` 형태의 원본 `SQLException` 메시지가 cause
  체인에 똑같이 남는다. 빈이 더하는 건 스택트레이스 최상위 wrapper 클래스명
  (`DuplicateKeyException`) 하나뿐이고 에러코드로 필터도 안 된다 — 로그 구분값으로서도 쓸모가
  없었다.
- **추가 근거(설계 관점):** 영속화 adapter는 어차피 구현체(JPA/Mybatis/JDBC)별로 따로 작성된다.
  JPA 경로에만 translator를 끼워 "정규화"하는 건 추상화 이득이 없다. 제약명을 실제로 소비하는
  `PaymentRepositoryAdapter`(결제 저장 담당 JPA 전용 infra adapter)는 이미 JPA 전용이라 Hibernate
  API(`getConstraintName()`) 직접 의존이 레이어상 오히려 자연스럽다.
- **검토한 대안:** (A) 빈 유지+문서화만 — 실측 결과 `getConstraintName()`이 테이블 prefix를 포함해
  (아래 "막힌 점") 제거로 얻을 단순화 이점이 반감됐다는 게 유지 논거였으나, 정당화 자체가
  무가치라 기각. (B) translator를 모든 adapter에 주입해 전역 정규화 — adapter가 구현체별로 갈려
  추상화 이득 없어 기각.

### 2. 이중결제 식별을 SQLException 메시지 정규식 → 제약명 비교로 전환

이중결제를 가려내는 판단부(`PaymentRepositoryAdapter.isApprovedOrderKeyViolation`)를
`SQLException` 메시지 정규식 매칭에서 Hibernate `ConstraintViolationException.getConstraintName()`
(위반된 제약의 이름) 기반으로 바꿨다.

- **이 판단부의 역할:** 결제 승인 완료를 저장하는 전용 경로(`saveApproved`)에서 `saveAndFlush`의
  조기 flush가 unique 위반을 그 메서드 호출 안에서 즉시 확정한다.
  `catch (DataIntegrityViolationException)`에서 이 판단부가 true면 도메인 예외
  `PaymentException(PAYMENT_DUPLICATE)`로 매핑하고, false면 원 예외를 rethrow한다. 즉 "한 주문에
  결제 하나"를 강제하는 unique 제약(`uk_payment_approved_order_key`) 위반만 이중결제로 가려내고
  다른 무결성 위반은 오매핑하지 않는 판단부다.
- **비교 방식 확정 과정:** 처음엔 prefix형(`tbl_payment.uk_...`)과 bare형(`uk_...`)을 모두
  흡수하려고 "제약명의 마지막 dot 세그먼트를 추출해 상수와 비교"하도록 구현했다. 이후 AI 코드
  리뷰(Gemini 등)가 (1) 대소문자 비구분(`equalsIgnoreCase`), (2) cause 체인 끝까지 순회를
  제안했다. 최종적으로는 dot 세그먼트·상수를 없애고, MySQL이 실제 반환하는 전체 문자열
  `"tbl_payment.uk_payment_approved_order_key"`를 `equalsIgnoreCase`로 직접 비교하며 cause 체인을
  끝까지 순회하는 형태로 단순화했다.
- **트레이드오프:** 식별이 MySQL 반환 형태(테이블 prefix 포함)에 결합된다. dot 세그먼트 비교가
  형식 변동(prefix/bare, dialect·Hibernate 버전 차이)에는 더 견고했지만, 단일 환경(MySQL)이라
  전체 문자열 비교의 단순함을 택했다. 형식이 바뀌면 통합 테스트
  `PaymentRepositoryDuplicatePaymentTest`(MySQL Testcontainers에서
  `uk_payment_approved_order_key` 위반이 이중결제 도메인 에러 `PAYMENT_DUPLICATE`로 매핑되는지
  검증 — 미처리 위반이 공통 안전망 코드로 떨어지는 것과 달리, adapter가 의도적으로 이 한 제약만
  가려내 도메인 예외로 승격하는 처리 경로를 검증한다)가 회귀로 잡는 게 안전망이다.
- 이중결제 식별 **동작 자체는 보존**했다. 바뀐 건 식별 메커니즘(메시지 문자열 → 구조적 접근자)뿐.

## 막힌 점·해결

### `getConstraintName()`이 bare name을 주지 않는다 — 방향 결정 전 실측으로 발견

방향을 정하기 전에 핵심 불확실성을 먼저 실측했다: "빈을 제거하면 `getConstraintName()`이 깔끔한
bare name(`uk_payment_approved_order_key`)을 주는가?" 일회성 통합 테스트로 확인했고, 이때
`EntityManager.flush()`로 Spring Data repository 프록시의 예외 변환 레이어를 우회해 Hibernate
원본 예외를 직접 관찰했다.

- **결과:** MySQL 8 Testcontainers에서 `getConstraintName()`은 테이블 prefix를 포함한
  `tbl_payment.uk_payment_approved_order_key`를 반환했다(bare 아님). 이때 `SQLState=23000`,
  `errorCode=1062`(MySQL duplicate entry).
- **함의:** "제거 = bare name 한 방 비교" 가설이 틀렸다. prefix 처리가 여전히 필요해 단순화
  효과가 반감됐다. 그래도 빈 정당화가 무가치라 제거는 그대로 진행했다. 이 실측 덕에 헛된 단순화
  기대 없이 현실적인 비교 방식을 택했다.

### 예외 타입에 대한 이전 이해가 뒤집힘

- **빈이 있던 시절:** translator 때문에 unique 위반이 `DuplicateKeyException`(cause=JDBC
  `SQLException`)으로 올라왔고, cause 체인에 Hibernate `ConstraintViolationException`이 남지
  않았다. 그래서 `getConstraintName()`으로 제약명을 얻을 수 없어(dead) 메시지 정규식에 의존했다.
- **빈 제거 후:** unique 위반이 `DataIntegrityViolationException`(cause=Hibernate
  `ConstraintViolationException`(cause=`SQLException`))으로 올라와 `getConstraintName()`이
  살아난다 — 단, 테이블 prefix가 붙은 채로.

### auditing 함정 — 빈만 떼고 어노테이션은 남겨야 한다

`JpaConfig`엔 이 빈 외에 `@EnableJpaAuditing`(생성/수정 시각 자동 채움)도 함께 붙어 있다. 빈만
제거하고 이 어노테이션은 유지해야 한다. 안 그러면 auditing이 꺼져 `created_at NOT NULL` 위반이
난다. 실측 초기에 auditing 없는 설정으로 테스트하다 `created_at cannot be null`로 한 번 실패해
이 숨은 결합을 뒤늦게 알아챘다.

## 배운 것

- **Hibernate ORM 6 + MySQL의 `getConstraintName()`은 table-qualified(`table.constraint`)로
  준다.** bare 제약명을 기대하지 말 것 — 제약 식별 코드는 `endsWith`/dot 세그먼트 추출 또는
  table 포함 전체 비교 중 하나를 명시적으로 골라야 한다. 이 prefix 형태는 dialect·버전 의존이다.
- **설정/빈의 "정당화"를 말로만 받지 말고, 실제 핸들러·에러코드·cause 체인을 직접 대조하면
  무가치가 드러난다.** 여기서 그 대조로 "로그 구분값" 정당화가 허구임을 확인했다.
- **방향 결정 전 핵심 불확실성(제약명 prefix 여부)을 일회성 통합 테스트로 먼저 실측하니 가설이
  틀렸음을 결정 전에 발견했다.** 측정이 헛된 단순화 기대를 막았다.

## 미해결·열린 질문

- 제약명 비교가 MySQL 반환 형태(`tbl_payment.` prefix 포함)에 결합돼 있다. MySQL 외 환경으로
  확장하거나 Hibernate 메이저 업그레이드 시 반환 형태가 바뀔 수 있어, 그때는 통합 테스트
  `PaymentRepositoryDuplicatePaymentTest`로 재확인이 필요하다. (지금은 단일 환경 전제에서 전체
  문자열 비교의 단순함을 의도적으로 택한 상태.)
