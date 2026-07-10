---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 180 }
---

## 한 일

- `order-idempotency-cache-simplification` task 진행 → PR #180 (Closes #171/#172/#173)
- harness skill 6 단계 (Explore → Discuss → Step Design → Worktree → File Drafting → Execution) 전체 흐름
- 도중 발견된 NaverPay 동시성 결함을 #176 issue 로 분리 등록 → 별도 PR #179 로 해결 후 본 task rebase 재개
- 정본 ADR: `commerce-backend/docs/tasks/order-idempotency-cache-simplification/adr.md`

## 결정한 것

핵심 통찰은 **"캐시의 본질은 중복 차단, 결과 캐싱은 부수 효과"**. 이걸 잡으면 다른 결정이 줄줄이 따라옴.

- **캐시 책임을 *in-flight 차단 전용* 으로 좁힘**. `COMPLETED` / `FAILED` 캐싱 제거. DB 가 결과의 진실 단일 원천 (ADR-011 find-first 와 정합).
- **`OrderIdempotencyStore` 인터페이스 4 → 2 메서드** (`reserve`, `clear`). `OrderIdempotencyStatus` enum 자체 제거 (marker 한 종류뿐).
- **sealed interface 거부.** enum + nullable field 로 충분한 도메인에 sealed 도입은 과한 추상화. [[feedback-no-sealed-interface]] 로 메모리 저장.
- **listener 제거 + Service finally clear 직접 호출.** `@TransactionalEventListener(AFTER_COMMIT)` 이 *기본 동기 실행* 이라 *latency 격리 효과 0* 이라는 사실이 결정 분기점. publisher 패턴의 부가 비용 (event 클래스, MDC 전파) 만 떠안고 얻는 게 없음.
- **사전 find 를 reserve 뒤에 배치.** reserve false (동시 요청 차단) 시 DB find 자체가 발생 안 함 → 캐시의 *DB 도달 전 차단* 가치 보존. 앞에 두면 매 요청 DB find 가 발생해서 캐시 명분이 약해짐.
- **동시 요청 응답 500 → 409 `ORDER_IDEMPOTENCY_IN_PROGRESS`.** ADR-011 의 안전망 500 위임은 *Redis fallback 후 도달하는 진짜 race* 한 곳에만 남음.
- **PROCESSING TTL 60초** (기존 600초). MySQL `innodb_lock_wait_timeout` (50초) + α. 기준을 ADR 본문에 명시해뒀음.

받아들인 한계:
- 같은 키 재시도 시점에 DB 상태가 바뀌면 *다른 응답* 가능. *완벽한 응답 일관성* 대신 *DB 상태 기준 멱등성*.
- Redis timeout 시 응답 latency 영향. 비동기 listener 도입은 timeout 빈도가 실제 문제로 드러나면 재검토.
- `#173` (listener 비동기 전환) 자동 close — listener 자체가 사라짐.

## 배운 것

- **`@TransactionalEventListener(AFTER_COMMIT)` 은 기본 동기.** *event 로 분리하면 비동기로 격리된다* 는 착각을 경계해야 함. 비동기로 가려면 `@Async` 또는 별도 `TaskExecutor` 명시.
- **캐시의 책임을 *명확히 정의* 하면 인터페이스가 단순해진다.** 부수 효과 (결과 캐싱) 까지 끌어안으면 정합성 책임이 늘어남. *"이 캐시가 진짜 주는 가치가 뭐냐"* 를 한 번 더 묻는 것의 효과.
- **AC 자동 검증에 *negative assertion* 도입은 위험.** `grep` 의 exit code 의미 (매칭 있음=0 / 없음=1) 와 실행기의 *exit 0=통과* 가정이 충돌. negative assertion 은 `!` 반전 또는 *금지사항* 으로 빼는 게 안전.
- **harness step 의 AC 가 *다른 도메인 기존 결함* 까지 끌어들일 수 있다.** step1 의 `./gradlew dockerTest` 가 NaverPay 도메인 결함까지 잡으면서 본 step blocked. 사전 검증 (develop HEAD 에서 동일 실패 재현 여부) 으로 *진짜 본 step 의 책임인지* 빠르게 분리해야 함.
- **영향 범위 grep 을 미리 해두면 step 분해가 명확해진다.** 이번엔 step2 의 *루트 docs sync* 가 처음엔 좁아 보였지만 실제로 16곳 (ADR.md 5군데 + api-spec / architecture / logging / testing-conventions / 기존 task adr 3개) 이었음. grep 으로 *현재 사용처* 부터 확인하고 분해.

## 다음 단계

- PR #180 review + 머지
- 머지 후 pr-merge-cleanup skill 로 worktree 정리
- 본 task 의 후속 작업은 없음. 미래 결정 트리거는 `commerce-backend/docs/tasks/order-idempotency-cache-simplification/retrospective.md` 의 *미래 결정 시점* 섹션에 명시:
  - Redis timeout 잦아지면 → 비동기 listener 재검토
  - 외부 시스템 후처리 (알림, 정산) → outbox 패턴
  - 도메인 이벤트 (PaymentCompleted 등) 도입 → Spring Event 부활 가능
  - 결제 PG 통합으로 트랜잭션 latency 증가 → PROCESSING TTL 재검토
