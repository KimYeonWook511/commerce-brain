---
platform: backend
author: KimYeonWook511
created: 2026-06-09
origin:
  - { type: pr, repo: commerce-backend, ref: 228 }
---

## 한 일
- commerce-backend `JpaConfig`에서 `SQLErrorCodeSQLExceptionTranslator` 빈 제거 (Spring이 JDBC `SQLException`을 `DuplicateKeyException` 같은 Spring DAO 예외로 변환하게 하던 빈).
- 이중결제 식별 `PaymentRepositoryAdapter.isApprovedOrderKeyViolation`을 `SQLException` 메시지 정규식 매칭 → Hibernate `ConstraintViolationException.getConstraintName()`(위반된 제약 이름) 기반으로 전환.
  - 호출 맥락: `saveApproved`의 `catch (DataIntegrityViolationException)`에서 이 메서드가 true면 `PaymentException(PAYMENT_DUPLICATE)`로 매핑, 아니면 원 예외 rethrow. 즉 "한 주문 1결제" unique(`uk_payment_approved_order_key`) 위반만 이중결제로 가려내는 판단부.
- 동작 보존. PR #228. 정본 결정: `commerce-backend/docs/adr.md`(translator 빈 제거 ADR), `docs/exception-strategy.md`.

## 결정 — 빈 제거
- 빈의 원래 목적(application이 `DuplicateKeyException`을 직접 catch)은 find-first 전환으로 이미 폐기됐고, 남은 정당화는 "운영 로그에서 unique 위반을 DuplicateKeyException 타입으로 구분"뿐이었다.
- 검증해보니 그 정당화가 무가치: 빈 유무와 무관하게 (1) unique 위반은 같은 핸들러·같은 에러코드(미처리 무결성 위반이 GlobalExceptionHandler 안전망에서 받는 공통 코드 COMMON-500-1)로 분류되고, (2) `Duplicate entry ... for key 'tbl_payment.uk_...'` SQLException 메시지가 cause 체인에 똑같이 남는다. 빈이 더하는 건 최상위 wrapper 클래스명(DuplicateKeyException) 하나뿐 — 에러코드로 필터도 안 됨.
- 추가 근거(내 정리): adapter는 어차피 구현체(JPA/Mybatis/JDBC)별로 작성되니 JPA에만 translator를 끼워 Spring DAO 예외로 "정규화"하는 건 추상화 이득이 없다. 제약명을 쓰는 PaymentRepositoryAdapter는 이미 JPA 전용이라 Hibernate API(getConstraintName) 의존이 오히려 자연스럽다.
- 검토한 대안: (A) 빈 유지+문서화만 — prefix 실측으로 단순화 이점이 반감됐으나 정당화 무가치로 기각. (B) translator를 모든 adapter에 주입 — 추상화 이득 없음으로 기각.

## 핵심 실측 / 막힌 점
- 방향 정하기 전 "빈 제거하면 getConstraintName()이 bare name(`uk_payment_approved_order_key`)을 깔끔히 주나?"를 일회성 통합 테스트로 실측. (`EntityManager.flush()`로 Spring Data repository의 translation 레이어를 우회해 Hibernate 원본 예외를 관찰)
- 결과: MySQL 8 Testcontainers에서 `getConstraintName()`은 테이블 prefix 포함 `tbl_payment.uk_payment_approved_order_key`를 반환. bare 아님 (SQLState 23000, errorCode 1062 = MySQL duplicate entry).
- 함의: "translator 제거 = 깔끔한 bare name equals" 가설이 틀림 → prefix 처리가 여전히 필요해 단순화 효과 반감. 그래도 정당화 무가치라 제거 진행.
- 이전 이해가 뒤집힘: 예전엔 translator 때문에 unique 위반이 `DuplicateKeyException`(cause=JDBC SQLException)이고 cause 체인에 Hibernate `ConstraintViolationException`이 없어 getConstraintName이 dead였다. 제거 후엔 `DataIntegrityViolationException`(cause=Hibernate ConstraintViolationException(cause=SQLException))로 와서 getConstraintName이 살아남 — 단 table-prefixed.
- auditing 함정: JpaConfig에 `@EnableJpaAuditing`도 함께 있어, 빈만 제거하고 어노테이션은 유지해야 함(안 그러면 created_at NOT NULL 위반). 실측 실험 초기에 이걸 빠뜨려 한 번 실패.

## 결정 — 제약명 비교 방식 (PR review 후)
- 처음엔 prefix/bare 양형 흡수용 "마지막 dot-세그먼트 추출 후 비교"로 구현.
- PR review(gemini/codex)에서 (1) equalsIgnoreCase 대소문자 비구분, (2) cause 체인 끝까지 순회 제안.
- 최종(사용자 결정): 전체 문자열 `"tbl_payment.uk_payment_approved_order_key".equalsIgnoreCase(getConstraintName())` 직접 비교 + 체인 순회. 상수/dot-세그먼트 제거.
- trade-off: 식별이 MySQL 반환 형태(테이블 prefix)에 결합. dot-세그먼트가 형식 변동(prefix/bare, dialect·버전)에 더 견고했지만 단일 환경(MySQL)에서 단순함 택함. 형식 바뀌면 통합 테스트 `PaymentRepositoryDuplicatePaymentTest`(MySQL Testcontainers에서 uk_payment_approved_order_key 위반이 이중결제 도메인 에러 `PAYMENT_DUPLICATE`로 매핑되는지 검증 — 위 COMMON-500-1 안전망과 달리 adapter가 의도적으로 가려내는 처리 경로)가 회귀로 잡는 게 안전망.

## 배운 것
- Hibernate ORM 6 + MySQL의 `getConstraintName()`은 table-qualified(`table.constraint`)로 준다. bare 제약명 기대 금지 — 제약 식별 코드는 endsWith/dot-세그먼트 또는 table 포함 전체 비교 중 택해야.
- 설정/빈의 "정당화"를 말로 받지 말고 실제 핸들러·에러코드·cause 체인을 대조해 검증하면 무가치가 드러난다. 특히 방향 결정 전 핵심 불확실성(여기선 prefix 여부)을 일회성 통합테스트로 실측하니 가설이 틀렸음을 결정 전에 발견 — 측정이 헛된 단순화 기대를 막았다.
