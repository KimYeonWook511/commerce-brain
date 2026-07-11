---
platform: backend
author: KimYeonWook511
created: 2026-06-08
origin:
  - { type: pr, repo: commerce-backend, ref: 226 }
---

# 결제 승인 완료 경로의 보상·예외 처리 정리 — 완료 우선 + 이중결제 탐지를 adapter 매핑으로

결제 승인 완료 경로(`completeVerifiedApproval` — PG가 SUCCESS=캡처 완료를 응답한 뒤 호출되어,
응답의 키·금액이 우리 결제와 일치하는지 검증을 통과하면 결제행을 SUCCEEDED로, 주문을 PAID로 반영하는
메서드)의 보상·예외 처리를 정리한 세션이다. 두 가지를 손봤다. (1) 승인이 실제로 성공한 뒤 DB 기록만
일시적으로 실패했을 때 환불로 되돌리던 걸 멈추고 흔적만 남긴다. (2) 이중결제(같은 주문의 성공 결제 중복)
탐지를 application 계층의 인프라 예외 catch에서 repository adapter의 도메인 예외 매핑으로 옮긴다.
이슈 #225, PR #226. (이번 세션에서 얻은 AI 코드리뷰·작업 범위 운영 교훈은 같은 PR의 별도 메모로 분리했다.)

## 결정한 것

### 1. 완료 우선 — 정상 승인 뒤 transient 기록 실패는 환불하지 않고 흔적만 남긴다

정상 승인(PG 캡처 성공 + 키·금액 검증 통과)이 끝난 뒤 DB 기록만 일시적으로 실패(데드락 등 transient)하면,
예전엔 PG에 환불을 걸고 결제행을 FAILED(`APPROVE_PROCESS_FAILED`)로 박제했다. 이제는 보상 없이 예외를
그대로 전파(500)하고 결제행을 REQUESTED로 남긴다.

- **상태 모델의 뜻:** 결제행 상태는 SUCCEEDED(완료 확정) / FAILED(실패 확정) / REQUESTED(아직 미확정,
  결과가 안 정해진 흔적) 셋으로 갈린다. 예전 처리는 "완료가 맞음 / FAILED가 맞음 / 진짜 버그"를 한
  status(FAILED)로 싸잡았다. 완료 우선은 실시간 경로의 책임을 "완료시키거나, 아니면 REQUESTED 흔적을
  남기기"까지로 좁히고, 결과 확정·복구는 나중에 배치 대사(reconcile)가 PG를 재확인해서 완료시키도록 미룬다.
- **왜:** 금전 정합이 우선이다. PG가 이미 캡처에 성공했고 키·금액 검증까지 통과한 *맞는 결제*를,
  DB 기록이 잠깐 미끄러졌다는 이유로 환불해 정상 매출을 취소하고 싶지 않다. 경합 확률이 낮아도 돈 문제는
  안전 우선으로 둔다.
- **명시적 비정상은 예외:** 응답 키 불일치(`MERCHANT_KEY_MISMATCH`)·금액 불일치(`AMOUNT_MISMATCH`)는
  *틀린 결제*라 판별이 확실하므로 기존대로 환불(보상)을 유지한다. 완료 우선은 어디까지나 "정상인데
  기록만 미끄러진" transient 실패에만 적용된다. 진짜 예상 못 한 버그도 500으로 가시화한다(예외 전파).
- **구현:** `completeVerifiedApproval`에서 unmapped 예외에 대한 보상 호출(`compensateUnexpected`)을
  제거했다. `PaymentException`의 default 분기·`CustomException`·일반 `Exception` catch에서 모두 보상 없이
  rethrow하도록 바꿨고, 쓰이지 않게 된 `compensateUnexpected` 메서드 자체도 삭제했다.
- **한계(내가 이해한 것):** 이 모델은 REQUESTED 잔여를 회수해줄 배치 대사에 전적으로 의존한다. 대사는
  `APPROVE_RECONCILE` 대상을 골라 PG에서 승인 확정(`PG_APPROVED`) 여부를 재확인한 뒤 완료시키는 흐름인데,
  그 구현(이슈 #221 / Epic #208)이 아직 없다. **그 전까지는 코드 레벨 self-heal 안전망이 없다** — 즉
  지금 REQUESTED로 남는 결제는 회수 주체가 없는 상태다. 이건 감수하고 진행한 부채다.

### 2. 이중결제 탐지를 application catch에서 adapter 도메인 예외 매핑으로

이중결제(같은 주문에 성공 결제가 둘 이상 생기는 것)를 잡던 방식을, application이 Spring의
`DataIntegrityViolationException`(DB 무결성 위반 예외)을 직접 catch하던 것에서, repository adapter가
도메인 예외로 번역해 올려주는 것으로 바꿨다.

- **바뀐 구조:** repository adapter에 승인 완료 저장 전용 메서드 `saveApproved`를 새로 뒀다. 이 메서드가
  "한 주문에 성공 결제 하나"를 강제하는 unique 제약 `uk_payment_approved_order_key` 위반을 만나면 도메인
  예외 `PaymentException(PAYMENT_DUPLICATE)`로 번역하고, 그 외 무결성 위반은 원 예외를 그대로 전파한다.
  application(`NaverPayApprovalService`)에서는 raw `catch(DataIntegrityViolationException)` 블록을 없애고
  (관련 import도 제거), 이제 위로 올라오는 도메인 예외의 `case PAYMENT_DUPLICATE` 분기가 fail-first 보상
  (`compensateDuplicatePayment`)을 수행한다. 승인 완료 저장 호출부(`succeedApproval`)도 일반 `save`에서
  `saveApproved`로 교체했다.
- **왜 adapter가 맞나:** 기존 예외 처리 정책은 "DB unique 위반은 정상 흐름에서 사전 `find` 조회로 처리하고,
  실제 동시 충돌만 안전망 500에 위임한다(find-first)"였고, application·adapter 어디서도 인프라 예외
  타입에 직접 의존하지 않기로 했다. 그 정책은 동시에 "도메인 의미가 분명한 제약 위반을 adapter에서
  도메인 예외로 번역하는" try-save-catch를 좁은 carve-out으로 허용한다. 이중결제는 보상이 필요한 case라
  find-first+500만으로는 부족하고, 이 carve-out에 정확히 들어맞는다. 반대로 예전처럼 application이 raw
  DAO 예외를 catch하던 건 이 정책을 위반하는 부채였다.
- **fail-first로 통일:** 예전 이중결제 보상은 두 갈래였다 — 실제로 타던 cancel-first
  (`compensateDuplicateApproval`: PG를 먼저 취소)와, 호출될 수 없던 dead 경로의 fail-first. cancel-first는
  크래시하면 "approve는 REQUESTED인데 cancel은 나간" 잔여를 만들었다. 이번에 cancel-first 경로를 제거하고
  fail-first 단일 경로(`compensateDuplicatePayment`)로 통일해, 남는 잔여는 별도 보상 정책이 정리하도록 했다.
- **`saveAndFlush`가 load-bearing:** `saveApproved`는 내부에서 `saveAndFlush`(즉시 flush)를 쓴다. 이 조기
  flush가 unique 위반을 그 메서드 호출 *안에서* 확정하는 게 매핑 성립의 핵심 의존성이다. 일반 `save`로
  바꾸거나 flush를 트랜잭션 커밋까지 미루면, 위반이 adapter catch 밖에서 터져 도메인 예외 매핑이 깨진다.
- **오매핑 방지:** 매핑을 `saveApproved` 전용 경로 + `uk_payment_approved_order_key` 제약명 일치로 한정해,
  FK·NOT NULL·다른 unique 위반이 이중결제로 잘못 번역되지 않게 했다. 범용 `save`는 매핑하지 않는다.

## 막힌 점 / 핵심 발견

### 제약명을 구조적 API로 못 얻는다 — `getConstraintName()`이 이 프로젝트에선 dead 경로

`uk_payment_approved_order_key` 위반을 확인하려고 처음엔 Hibernate `ConstraintViolationException`의
`getConstraintName()`(위반된 제약 이름을 구조적으로 주는 API)을 1차로 보고, SQLException 메시지 문자열
매칭을 2차 폴백으로 뒀다. 리뷰 중 "폴백은 어차피 도달 못 하는 보험이니 빼고 `getConstraintName()`만
남기자"고 판단해 폴백을 제거했더니 **MySQL 통합 테스트가 깨졌다.**

- **원인:** 전역 JPA 설정(`JpaConfig`)이 `SQLErrorCodeSQLExceptionTranslator`(DB 벤더 에러코드 기반
  번역기 — unique 위반을 Spring `DuplicateKeyException`으로 매핑하고 cause를 JDBC `SQLException`으로 둔다)
  빈을 등록한다. 이 경로엔 Hibernate `ConstraintViolationException`이 cause 체인에 **아예 없다.** 그래서
  `getConstraintName()` 분기는 이 프로젝트에서 처음부터 한 번도 안 타는 dead 경로였고, 실제 매핑은 줄곧
  SQLException 메시지 폴백이 담당하고 있었다. 결국 메시지 매칭이 폴백이 아니라 *주 경로*다.
- **최종 형태:** dead인 `getConstraintName()` 분기를 걷어내고, cause 체인을 끝까지 순회하며 SQLException
  메시지를 단어 경계 정규식(`\b...\b`, 대소문자 무시)으로 매칭하는 형태로 단일화했다. 단어 경계는
  `contains` 방식이 낼 법한 오탐 — 예컨대 `uk_payment_approved_order_key_v2`처럼 이름을 prefix로 공유하는
  다른 제약(현재 실재하지 않는 가상 예시)에 걸리는 것 — 과 대소문자 문제를 함께 막는다. 돈 관련 식별이라
  저확률 오탐도 안전장치로 처리했다.
- **판별 함수 short-circuit 변천(세 번 만에 수렴):** ① 1차는 `getConstraintName()==null`이면 즉시 `false`
  반환 → 같은 cause 체인의 SQLException 폴백에 도달 못 함. ② 2차는 `name==null`일 때만 폴백을 열었으나,
  `name`이 non-null이지만 다른 값일 때 여전히 `false`를 단정 → `getConstraintName()`이 부정확한 환경에서
  미탐. ③ 최종은 어떤 분기도 `false`를 조기 단정하지 않고 "일치할 때만 `true`, 끝까지 못 찾으면 `false`"인
  OR 구조로 정리. 그 뒤 위 테스트 실패로 `getConstraintName` 경로 자체가 dead임이 드러나 메시지 매칭으로
  단일화됐다.

### translator를 빼면 깔끔해지지만, 영향 범위가 다른 결정이라 분리했다

그 번역기 빈을 제거하면 기본 `SQLStateSQLExceptionTranslator`(SQL state 기반 번역기 — unique 위반을
`DataIntegrityViolationException`으로 매핑하고 cause로 Hibernate `ConstraintViolationException`을 보존)로
바뀌어 `getConstraintName()`이 살아난다 → 식별이 메시지 파싱 없이 구조적 API로 깔끔해진다.

- **그런데 왜 지금 안 뺐나:** 그 빈은 운영 로그에서 unique 위반을 `DuplicateKeyException` 타입으로
  구분하려고 둔 *전역* 설정이라, 빼면 전역 예외 분류·로깅이 바뀐다. 게다가 원래 등록 목적(application이
  `DuplicateKeyException`을 좁게 catch하려던 것)은 find-first 정책 전환으로 이미 폐기됐고, 남은 정당화는
  로그 구분뿐이라 "이젠 과한 추상화 아니냐"는 의문은 타당하다. 다만 이중결제 한 메서드 편의로 전역 빈을
  빼는 건 영향 범위가 전혀 다른 결정이다.
- **분리 기준:** "그걸 안 고치면 지금 작업이 *틀리거나 못 끝나나?*" → 아니오. 현재 설정 그대로 두고
  SQLException 메시지 매칭으로 올바르게 완결되고, 나중에 되돌림 없이 위에 얹을 수 있다. 그래서 이번 #225는
  현 제약 안에서 마무리하고, translator 재검토는 별도 이슈 #227로 분리했다. 발견한 근본 문제를 진행 중인
  작업에 곁다리로 끼우면 묻혀버린다는 판단이다.

## 배운 것

- **"이 코드는 안 탈 것"이라는 추론을 코드만 보고 확신하지 말 것.** `getConstraintName()` 폴백이 dead인지
  아닌지를 소스만 읽고 단정했다가 통합 테스트에서 뒤집혔다. 실제 런타임의 예외 변환 경로(어떤 translator가
  cause 체인을 어떻게 만드는지)는 통합 테스트가 드러낸다. 특히 돈 관련 경로에서는 죽은 줄 알고 지우기 전에
  정말 죽었는지 테스트로 확인해야 한다.
- **인프라 예외 번역은 adapter 책임으로 되돌리면 중복이 준다.** `compensateUnexpected`·
  `compensateDuplicateApproval` 두 보상 경로를 없애고 fail-first로 단일화하면서, application이 raw DAO 예외에
  의존하던 부채를 함께 해소했다.

## 미해결·열린 질문

- **#227 — translator 재검토:** 폐기 목적만 남은 `SQLErrorCodeSQLExceptionTranslator` 빈을 제거하면
  `getConstraintName()` 경로가 되살아나 제약명 식별을 메시지 파싱에서 구조적 API로 단순화할 수 있다. 단
  전역 예외 분류·로깅 영향을 함께 따져야 하고, 무엇보다 **MySQL의 `getConstraintName()`이 테이블
  prefix(`tbl_payment.uk_...`)를 포함해 반환하는지 실측이 필요하다** — bare name이 아니면 단순화 효과가
  반감된다.
- **reconcile 구현(#221 / #208):** 완료 우선 모델이 의존하는 self-heal 주체. 그 구현 전까지는 REQUESTED로
  남는 결제를 회수할 코드가 없다.
