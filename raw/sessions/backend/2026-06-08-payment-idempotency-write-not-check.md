---
platform: backend
author: KimYeonWook511
created: 2026-06-08
---

## 출발 질문
> "PG 결과를 받고 우리 서버에 반영하는데, 반대로 우리 서버에서 먼저 approve 중복 방지를 위해 `payment.succeed()`를 성공하면 그때 PG에 승인 요청을 보내는 식으로 하면 안 되나? 더 안 좋나?"

이 질문에서 출발해 "멱등을 어떻게 보장하는가"의 패턴을 정리했다.

## 핵심 결론
멱등 보장은 **"SELECT로 확인하고 없으면 생성"이 아니라 "INSERT를 시도하고 unique 제약조건 위반이면 기존 것을 반환"** 이다. 그리고 `succeed()`(=성공 기록)를 PG 호출 *전에* 하는 선점 방식은 틀렸다 — 성공의 의미를 뒤집기 때문. 멱등 성공 응답이 필요한 곳(돈 걸린 승인)은 *위반을 잡아서 만드는* 게 아니라 *진입 사전 체크로 위반 자체를 회피*하고, 극히 드문 동시 race만 보상/에러로 격리한다.

## 정리한 개념

### 1. `succeed()` 선점은 왜 안 되나 (성공의 의미 뒤집힘)
- 제안: `payment.succeed()`(SUCCEEDED 기록) 먼저 → 통과하면 PG에 승인 요청.
- 문제: `succeed()`는 "성공했다"는 사실 기록인데, **PG에 물어보기도 전에 성공으로 단정**하는 것.
  - PG가 실패를 주면? 이미 SUCCEEDED + Order PAID로 적어버려서 되돌려야 함.
  - PG가 타임아웃이면? 거짓 성공이 박제됨(돈 안 빠졌는데 결제 완료 처리 = 매출 손실, 이중결제보다 어떤 의미론 더 나쁨).
- 결론: 결제는 **외부(PG)가 진실의 원천**. `succeed()`는 PG가 성공을 확인해준 *뒤에* 기록하는 것이지, 요청을 보내기 위한 선결 조건이 아니다.
- 단 "처리 중(REQUESTED)" 선점은 정당하고 이미 하고 있다. 선점할 거면 SUCCEEDED가 아니라 REQUESTED를 선점해야 한다(= getOrCreate가 REQUESTED 행을 먼저 만드는 것).

### 2. "확인 후 생성"이 동시성에 뚫리는 이유 (MVCC)
```
T1: findByKey → 없음
T2: findByKey → 없음   (T1 미커밋 + REPEATABLE READ 스냅샷이라 T1의 INSERT 안 보임)
T1: save 성공 / T2: save 성공  → 둘 다 "없음"을 봐서 둘 다 생성
```
- SELECT는 MVCC 스냅샷이라 서로의 미커밋 INSERT를 못 본다. 그래서 "SELECT 확인 후 INSERT"는 race에 뚫린다.
- 반면 **write(INSERT)는 스냅샷이 아니라 현재 커밋 상태와 충돌**한다 → unique 제약조건이 "이미 있음"을 정확히 잡아준다.

### 3. 멱등 getOrCreate의 정석
```java
// find-first + INSERT 폴백 + 위반 안전망
public Payment getOrCreate(String key, ...) {
    return repo.findByKey(key)                 // 1. 흔한 경우(이미 있음) 빠르게 흡수
        .orElseGet(() -> insertOrReread(key)); // 2. 없으면 생성, 3. race면 기존 반환
}
```
- find는 **1차 필터**(흔한 시간차 중복 흡수)일 뿐, **진짜 보장은 unique 제약조건**이다. find가 "없음"을 봤어도 MVCC 때문에 진실이 아닐 수 있으므로 INSERT 위반을 최종 안전망으로 둔다.
- commerce-backend 프로젝트의 `findReusableReserved`(reserve) / `getOrCreate`(approve attempt)가 이 패턴.

### 4. 위반난 트랜잭션에서 재조회하는 것의 함정 (중요)
처음엔 "INSERT 위반나면 catch 안에서 findByKey로 기존 것 반환"을 떠올렸지만 — **같은 트랜잭션 안에서는 못 믿는다.**
- (a) **MVCC**: 위반은 났지만(write는 현재와 충돌) 같은 트랜잭션 SELECT는 시작 시점 스냅샷이라 그 행을 **여전히 못 볼 수 있음** → 재조회가 빈 결과 → `orElseThrow` 터짐. 1번에서 못 본 걸 3번에서도 못 본다(스냅샷 동일).
- (b) **세션 오염**: flush 중 예외가 나면 트랜잭션은 rollback-only로 마킹되고 Hibernate 세션은 "더 쓰지 마라"가 기본. 같은 세션 재조회가 항상 보장되지 않음.
- 그래서 재조회로 수습하려면 **새 트랜잭션**(새 스냅샷)이라야 하고, 곡예가 싫으면 위반을 그냥 던진다.

### 5. 위반의 의미로 응답이 갈린다
- **노이즈성(reserve 따닥, 돈 안 걸림)** → 위반 시 그냥 500. 재시도하면 find가 흡수. 곡예 불필요. (실제로 우리가 택한 방식)
- **돈 걸린 멱등(승인)** → 500을 주면 사용자가 "실패했나" 재시도 위험 → 멱등 성공 응답이 필요.

### 6. 돈 걸린 곳의 멱등 성공 — "위반 처리"가 아니라 "위반 회피 + 안전망"
복잡함의 원인은 "위반난 자리에서 멱등 응답을 만들려는 것". 발상 전환:
```
1차: 진입에서 findApproveSucceeded(merchantPayKey) 사전 체크
     → 있으면 멱등 성공 응답, PG 호출 안 함. 끝.   (시간차 중복=대부분, 위반 자체를 회피)
2차: 그래도 동시 race로 unique 제약조건 위반
     → 도메인 예외 throw → 보상(REQUIRES_NEW로 PG 환불) + PAYMENT_DUPLICATE
     → 사용자는 에러받고 재시도 → 재시도 땐 1차가 멱등 성공으로 흡수
```
- 대부분의 중복은 **시간차**(콜백 재도착, 새로고침)다. 동시(같은 순간) race는 극히 드물다.
- 그래서 시간차는 1차 사전 체크의 if문 하나로 단순하게 잡고, 복잡한 처리(보상/새 트랜잭션)는 **거의 안 타는 2차 경로로 격리**한다.
- 2차에서 멱등 성공을 굳이 안 만들어도 된다 — 보상 + 에러로 두고 재시도가 1차에서 흡수(가장 단순, 우리 task 방식).

## 결정/원칙 (commerce-backend 프로젝트)
- 멱등 = "INSERT 시도 + unique 제약조건 위반 시 기존 반환". SELECT 확인 후 생성 금지(MVCC에 뚫림).
- `succeed()`는 PG 확인 후에만. 선점은 REQUESTED까지.
- find-first는 흔한 경우 흡수용 1차 필터, 최종 보장은 unique 제약조건.
- 위반난 같은 트랜잭션에서 재조회 금지(MVCC 빈 결과 + 세션 오염). 재조회는 새 트랜잭션.
- reserve 따닥(노이즈) → 위반 시 500 + 재시도 흡수. 승인(돈) → 진입 사전 체크로 멱등 성공 + 동시 race만 보상/에러.

## 열린 질문 / 주의
- 동시 race(2차)에서 멱등 성공 응답을 *만들지* 여부는 정책. 안 만들고 재시도에 넘기는 게 단순. 만들려면 새 트랜잭션 재조회.
- 보상(PG 환불)이 또 실패하는 경우 → 배치 재처리 후보(별도 raw: 배치/스케줄러).

## 관련
- [[raw/sessions/backend/2026-06-08-jpa-flush-transaction-exception-boundary]] — 위반을 어디서 잡고 번역하나
- [[raw/sessions/backend/2026-06-07-pr-218-pg-approve-exception-boundary]] — 전송 시점 경계 / UNKNOWN 보존
