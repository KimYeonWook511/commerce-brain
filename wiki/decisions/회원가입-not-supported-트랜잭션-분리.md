---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [auth, member, transaction, not-supported, redis, after-commit]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-auth-member-domain-overview]]"
---

# 회원가입 트랜잭션 분리 NOT_SUPPORTED — REQUIRED/AFTER_COMMIT/NOT_SUPPORTED 3자 비교

`AuthSignUpService.signUp()`에 `@Transactional(propagation = NOT_SUPPORTED)`을 붙였다. 회원가입 흐름은 [[auth-member-security-도메인-구조-개요]]의 데이터 흐름을 참고.

## 문제 상황(REQUIRED commit 시점 함정)

단순히 `@Transactional`만 붙이면:

- `signUp()`이 외부 트랜잭션을 열고 `register()`가 `REQUIRED`로 합류
- Spring의 commit 시점 규칙: *트랜잭션을 시작한 메서드가 종료될 때 commit*
- 따라서 `register()`가 반환된 시점에도 DB는 *아직 미커밋*
- 그 사이 `issue()`가 Redis 저장 → **DB commit 전 Redis 저장 불일치**
- register 후 어떤 이유로 rollback되면 Redis엔 refresh 살아있는데 DB엔 member 없음 = *유령 토큰*

핵심은 "내부 메서드가 반환돼도 DB는 미커밋"이라 *반환 직후 외부 시스템(Redis)을 다루면 그 시점은 DB commit 이전*이라는 REQUIRED 전파의 함정이다.

## 후보 3자 비교(REQUIRED/제거/AFTER_COMMIT/NOT_SUPPORTED)

| 후보 | 결과 | 채택 안 한 이유 |
|---|---|---|
| `@Transactional` 그대로 | Redis 저장이 DB commit 전 | 위의 불일치·유령 토큰 |
| method-level `@Transactional` 제거 | class-level `readOnly=true` 합류 → Hibernate flush MANUAL | register의 save가 의도와 다르게 동작 |
| `@TransactionalEventListener(AFTER_COMMIT)` | 응답 *반환 후* 이벤트 실행 | 두 문제(아래) → strict 정책과 양립 불가 |
| `NOT_SUPPORTED` ✓ | signUp 트랜잭션 없음 / register 자체 트랜잭션으로 commit / issue는 DB commit 이후 | 행위 정확 + strict 호환 |

## AFTER_COMMIT을 제외한 두 문제

진지하게 검토했다가 제외했다. (1) 응답 반환 *후* 실행이라 Redis 실패를 클라이언트 응답에 반영 못 함 → [[redis-장애-strict-정책-soft-fail-기각]] 위반. (2) 회원가입만 성공하고 Redis 저장이 실패하면 *후속 분기 처리*가 필요 → 코드 복잡도 증가. 이 "부분 실패 분기 부담 회피"는 strict 결정의 "soft fail은 분기 비용도 손해" 통찰과 같은 패턴이다.

## 핵심 통찰(AFTER_COMMIT 적용 조건)

AFTER_COMMIT 패턴은 "성공 보장 X, 실패해도 클라이언트에 안 알려도 OK"인 경우에만 쓸 수 있다. 회원가입은 *Redis 저장 실패를 클라이언트에 즉시 알려야* 하므로(= strict 정책) 같은 흐름 안에서 동기로 처리돼야 한다.

> 대비: [[주문-멱등성-캐싱-after-commit-이벤트-분리]]는 order 쪽에서 AFTER_COMMIT을 채택했는데, 그건 멱등 캐싱이라 "실패해도 안 알려도 OK"인 성질이라 조건에 맞았다. 같은 메커니즘이라도 도메인의 실패 통보 요구가 채택 여부를 가른다.

## 기존 패턴과의 일관성(OrderCreateService)

`OrderCreateService.createOrder()`가 동일하게 `NOT_SUPPORTED` 패턴이다. 코드베이스 안에 *이미 검증된 패턴*이 있었다("외부 시스템 호출과 commit 시점" 공유 지식 — [[order-도메인-구조-개요]]와 공통).

## 트레이드오프(부분 실패는 복구 가능)

member가 DB에 commit됐는데 Redis 저장이 실패하면 클라이언트는 500을 받는다. 하지만 같은 이메일 재시도는 `DUPLICATE_EMAIL`(이미 DB에 있음), 로그인 시도는 비밀번호를 알면 성공한다. 즉 *유령 회원*은 안 만들어지고 사용자 입장에서도 *복구 가능한 상태*다.

## class-level readOnly 패턴의 향후 리팩토링

> [!warning] EVOLUTION — 세션 이후 뒤집힌 부분
> 세션 시점에는 통일성을 위해 class-level `@Transactional(readOnly=true)` + method-level override 패턴을 썼다. 하지만 이 패턴은 *트랜잭션 경계가 한눈에 안 들어온다* — 클래스 단위 애너테이션이 기본값을 숨기고, signUp이 `NOT_SUPPORTED`로 class-level을 override하는 구조 자체가 어색함의 신호였다. 실제로 이 세션 이후 응용 Service의 `@Transactional`을 method-level로만 두는 방향이 컨벤션으로 정해지고, 이어 class-level `@Transactional`은 전 도메인에서 폐지됐다. 지금 코드는 이 방향으로 정리돼 있다(맥락은 [[auth-member-security-도메인-구조-개요]]의 drift 메모).

## 근거

- [[raw/sessions/backend/2026-05-29-auth-member-domain-overview]] — "결정한 것 3. 회원가입 트랜잭션 분리 NOT_SUPPORTED"
