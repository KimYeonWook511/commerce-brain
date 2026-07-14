---
type: decision
status: accepted
platform: backend
author: KimYeonWook511
decided_by: KimYeonWook511
tags: [order, spring-batch, expiration, optimistic-lock, keyset-pagination, scheduler]
created: 2026-05-29
updated: 2026-07-14
superseded_by: null
sources:
  - "[[raw/sessions/backend/2026-05-29-order-domain-overview]]"
---

# 주문 만료 처리 — Spring Batch(chunk 100·retry/skip 차등·keyset 페이징)

## 컨텍스트·문제(고아 주문 정리)

주문과 결제 트랜잭션을 분리한 결과 "주문은 생겼는데 결제가 미완료"인 **고아 주문**이 남는다([[재고복구-동기취소-vs-outbox-비동기만료-비대칭]]의 사슬 1단계). `status = INIT`으로 cutoff(`now - 60분`)보다 오래 남은 주문을 만료 처리(취소 + 재고 복구 outbox 발행)해야 한다.

## 왜 polling이 아니라 batch인가

- **시점 일관성** — cutoff(`now - 60분`) 기준이 명확하다. cutoff는 reader 생성 시 1회 고정한다.
- **운영 가시성** — Spring Batch 메타데이터로 진행·실패·재시작을 추적한다.
- **peak 부하 회피** — 장애 복구 spike가 정상 트래픽에 섞이지 않는다.

비즈니스 유스케이스(`expireOrder`)는 batch 패키지가 아니라 application service에 둬서 scheduler/admin/수동 복구 흐름에서도 재사용 가능하게 했다. batch 패키지(`OrderExpirationBatchConfig`)는 Job/Step/Reader/Writer 구성만 담당한다.

## retry/skip 차등 정책

`faultTolerant` + `retry(OptimisticLockingFailureException, 3)` + `skip(OptimisticLockingFailureException | CustomException, 10)`.

- **OLE(낙관락 충돌)는 retry + skip 둘 다** — 일시 충돌은 재시도하고, 그래도 안 되면 단건 skip 후 다음 chunk로 진행.
- **CustomException은 skip만** — 도메인 위반(주문이 이미 다른 상태로 전이됨 등)은 재시도해도 결과가 같아 skip만 한다.
- **skip 한도 10** — "전체 실패 vs 개별 단건 보류"의 임계.

이 OLE는 만료 batch와 결제 콜백이 같은 Order를 동시에 다룰 때 `@Version`이 잡아내는 충돌이다 — [[order-version-낙관락-비관락-공존]]의 race window가 이 retry/skip 정책으로 처리된다.

## keyset 페이징 Reader

Reader는 `id > lastId` + `chunkSize`로 다음 청크를 읽고 `status = INIT AND createdAt < cutoff` 조건(`findExpiredOrdersAfterId`)을 건다. offset·count 없이 keyset으로 offset 페이징의 누적 cost를 회피한다. Writer는 chunk 안에서 `OrderExpirationService.expireOrder(orderId, requestedAt)`를 단건씩 호출한다.

## 숫자는 기본값(튜닝 미정)

chunk 100 / retry 3 / skip 10 / cron 5분(`0 */5 * * * *`)은 모두 기본값이다 — 부하 테스트나 운영 경험 기반이 아니다. 여기에 멱등 TTL 10분·만료 cutoff 60분까지 더하면 튜닝 대상 숫자가 여럿이다. k6 부하 테스트 후 조정 예정임을 솔직히 남긴다.

## 학습 흔적(Writer lambda vs Chunk 주석, @Profile(!test) 자기비판)

- **Writer lambda vs `Chunk<>` 주석**: 주석 처리된 `Chunk<>` 방식 `ItemWriter`와 현재 사용 중인 lambda 방식이 함께 남아 있다. lambda 표현이 아직 직관적으로 와닿지 않아 자기 스타일로 짠 뒤 lambda를 주석으로 남겼다. 학습이 충분해지면 lambda로 정리 예정.
- **`@Profile("!test")` 자기 비판**: `OrderExpirationJobScheduler`에 `@Profile("!test")`를 붙여 test profile일 때 스케줄러 자동 실행을 차단(테스트 격리)한다. 의식적 패턴이지만, production 코드가 테스트 격리 책임까지 떠안는 구조라 스케줄러가 늘 때마다 같은 애너테이션을 반복해야 하는 코드 스멜이다. 대안: (B) `@TestConfiguration`에서 스케줄러 빈을 mock/no-op override, (C) 별도 `SchedulerConfig`로 등록을 분리해 그 config에만 `@Profile("!test")`를 붙여 스케줄러 코드는 깨끗하게 유지. 통합 테스트의 만료 흐름 검증은 `JobLauncher` 수동 launch로 한다.

## 미해결·후속

- 만료 batch 숫자 튜닝(chunk/retry/skip/cron) — k6 후 조정.
- `@Profile("!test")` 구조 개선(TestConfiguration 또는 SchedulerConfig 분리).
- Writer lambda 정리(학습 완료 후 주석 `Chunk<>` 제거).

## 근거

- [[raw/sessions/backend/2026-05-29-order-domain-overview]]
