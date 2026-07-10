---
platform: backend
author: KimYeonWook511
created: 2026-06-08
origin:
  - { type: pr, repo: commerce-backend, ref: 226 }
---

## 한 일
- 결제 승인 완료 경로(completeVerifiedApproval — PG가 SUCCESS=캡처 완료를 응답한 뒤 호출되어 키·금액 검증 통과 후 결제행을 SUCCEEDED+주문 PAID로 반영)의 보상·예외 처리를 정리. 이슈 #225, PR #226. (이번 세션의 AI·작업 운영 교훈은 같은 PR의 다른 메모 [[raw/sessions/backend/2026-06-08-pr-226-review-and-scope-lessons-ai]]에 분리.)

## 결정한 것
- **완료 우선**: 정상 승인(PG 캡처 성공 + 키·금액 검증 통과) 후 DB 기록만 transient하게 실패(데드락 등)하면, 예전엔 PG 환불 + 결제행 FAILED 박제였는데 이제 보상 없이 예외 전파(500)하고 결제행을 REQUESTED(결과 미확정 흔적 상태 — SUCCEEDED=완료, FAILED=실패 확정, REQUESTED=아직 미확정)로 남긴다. 배치 대사(reconcile)가 나중에 PG 재확인 후 완료시킨다는 모델. 명시적 비정상(키 불일치·금액 불일치)은 틀린 결제라 환불 유지. 정본: docs/tasks/approval-compensation-cleanup/adr.md, docs/adr.md(루트). 내 이해: 실시간 경로는 "완료 또는 흔적(REQUESTED) 남김"까지만 책임지고 결과 확정은 배치로 미룬다. 단 대사 구현(#221/#208)이 아직 없어 그 전까지는 코드 레벨 self-heal 안전망이 없다.
- **이중결제 탐지를 adapter로**: application이 Spring의 DataIntegrityViolationException(DB 무결성 위반 예외)을 직접 catch하던 것을 제거하고, repository adapter의 succeed-approve 전용 저장 메서드(saveApproved)가 uk_payment_approved_order_key(주문당 성공 결제 1개 보장 unique 제약) 위반을 도메인 예외 PaymentException(PAYMENT_DUPLICATE)로 번역하게 바꿈. 인프라 예외 번역은 adapter 책임이라는 원칙(ADR-011: application은 무결성 위반을 직접 catch하지 않고 GlobalExceptionHandler 500 안전망에 위임하나, "도메인 의미가 분명한 제약 위반을 adapter에서 도메인 예외로 번역"하는 건 그 carve-out으로 허용. 정본: docs/adr.md). saveAndFlush(즉시 flush)가 unique 위반을 그 메서드 호출 안에서 확정하는 게 load-bearing — 일반 save로 바꾸면 flush 타이밍이 밀려 매핑이 깨진다.

## 막힌 점 / 핵심 발견
- **예외 변환 경로**: JpaConfig(전역 JPA 설정)가 SQLErrorCodeSQLExceptionTranslator(DB 벤더 에러코드 기반 번역기 — unique 위반을 Spring DuplicateKeyException으로 매핑, cause는 JDBC SQLException) 빈을 등록한다. 이 경로엔 Hibernate ConstraintViolationException이 cause 체인에 없다 → 그 예외의 getConstraintName()(위반 제약 이름을 구조적으로 주는 API)으로 제약명을 확인하는 코드는 이 프로젝트에선 한 번도 안 타는 dead 경로다(MySQL 통합테스트로 확인). 실제 제약명은 SQLException 메시지 문자열로만 얻을 수 있어, 메시지 매칭이 폴백이 아니라 주 경로다. 최종적으로 SQLException 메시지를 단어 경계 정규식(\b...\b, 대소문자 무시)으로 매칭해 uk_payment_approved_order_key_v2 같은(현재 실재하지 않는 가상 예시) prefix 공유 이름의 오탐까지 막았다. (판별 함수 short-circuit 수정 변천: name이 null일 때만 폴백 → 이름이 다른 값이면 false 단정하는 문제 → "일치할 때만 true, 끝까지 못 찾으면 false"인 OR 구조로 수렴.)
- **translator를 빼면?**: 빈을 제거하면 기본 SQLStateSQLExceptionTranslator(SQL state 기반 번역기 — unique 위반을 DataIntegrityViolationException으로 매핑, cause로 Hibernate ConstraintViolationException 보존)가 되어 getConstraintName()이 살아난다 → 식별이 메시지 파싱 없이 깔끔해진다. 하지만 그 빈은 운영 로그에서 unique 위반을 DuplicateKeyException 타입으로 구분하려고 둔 전역 설정이라, 빼면 전역 예외 분류·로깅이 바뀐다. 게다가 원래 등록 목적(application에서 DuplicateKeyException 좁게 catch)은 이미 폐기됐고(find-first 정책 전환) 남은 정당화는 로그 구분뿐이라 "과한 추상화 아니냐"는 의문은 타당. 다만 이중결제 한 메서드 편의로 전역 빈을 빼는 건 영향 범위가 다른 결정이라 별도 이슈 #227로 분리.

## 다음 단계
- #227: translator 재검토. 제거하면 getConstraintName 기반 단순화 가능하나, 전역 예외 분류·로깅 영향 + MySQL의 getConstraintName이 테이블 prefix(tbl_payment.uk_...) 포함 반환하는지 실측 필요.
- reconcile 구현(#221/#208): 완료 우선 모델이 의존하는 self-heal 주체. 그 전까지 REQUESTED 잔여를 회수할 코드가 없다.
