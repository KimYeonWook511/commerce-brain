---
platform: backend
author: KimYeonWook511
created: 2026-06-01
origin:
  - { type: pr, repo: commerce-backend, ref: 180 }
---

## 한 일

- PR #180 의 코드 리뷰 (Gemini high `r3331866213` + Codex P2) 대응. 같은 결함을 두 리뷰어가 독립적으로 지적: `RedisOrderIdempotencyStore.reserve()` 의 `DataAccessException` catch 가 `return false` → Redis 장애 시 단독 주문도 409 차단 (시스템 마비).
- 두 리뷰어가 제안한 `return true` 한 줄 패치 거부. 도메인 예외 매핑 + application fallback 분기 방향으로 진행.
- 추가 commit 3개: `2ee5989` (fix, 도메인 예외 + service catch + 헬퍼 추출), `40e701d` (docs, 7개 문서 동기화), `17f4657` (test, `verify never clear` 결박).
- 별도 issue 1개 (#181, Auth 패턴 통일 검토 — Auth 가 여전히 application 직접 catch 패턴이라 일관성 부담).
- PR 본문 갱신 (코드 리뷰 반영 섹션 신설), Gemini 두 thread 답글 + resolve.
- Codex CLI 의 추가 지적 (TTL 60초 초과 시 finally clear 가 다른 요청 marker 삭제) 검토 → trigger 조건 (결제 PG 외부 I/O 트랜잭션 진입) 없음 + ADR-2 가 이미 인지한 trade-off → 무수정.

## 결정한 것

정본 ADR: [[commerce-backend/docs/tasks/order-idempotency-cache-simplification/adr.md]] ADR-1 결정 내용 6번 + [[commerce-backend/docs/exception-strategy.md]] *Redis 캐시 장애 처리* 신규 섹션.

내가 어떻게 이해했는가:

- **boolean 시맨틱 정직성이 핵심 통찰**. *예약됨* 과 *저장소 사용 불가* 를 boolean false 한 신호로 합치면 application 이 *정상 차단(409)* 과 *시스템 사정(fallback)* 을 구별 못 한다. 의미가 두 가지면 *예외로 분리* 가 정직. return type 자체가 추상화 선택이다.
- **infra 매핑 vs application 직접 catch 의 분기점은 *fallback 가능성***. Order 는 DB unique 안전망이 있어 fallback 가능 → infra 매핑이 port 추상화 강화. Auth 는 fallback 불가 (refresh token 없으면 인증 자체 불가) → application 직접 catch 가 단순함. 이 차이를 *현재 코드 패턴 차이* 가 아니라 *fallback 가능성 차이* 로 설명할 수 있어야 정합.
- **`RuntimeException` 직접 상속 vs `CustomException` 상속의 선택은 *catch 흐름 강제 의도***. `CustomException` 상속하면 `GlobalExceptionHandler.handleCustomException` 가 자동 매핑해서 application catch 의도 우회. Spring `@ExceptionHandler` 자동 매핑은 *편의* 와 *catch 우회 위험* 의 양면.
- **예외 위치는 기존 컨벤션** (`com.commerce.<domain>.exception/`). application/exception 같은 layer 별 분리는 이 프로젝트는 안 함 — *cross-cutting 카탈로그* 컨벤션. 단일 예외 1개를 위해 새 패턴 도입하지 않음.

지금 다시 본다면:

- 1차 PR ([[raw/sessions/backend/2026-06-01-pr-180-idempotency-cache-simplification]]) 작업에서 *캐시 책임 단순화* 에 집중하느라 *실패 표현 시맨틱* 을 따로 챙겨야 한다는 점을 놓쳤다. *단일 boolean 시맨틱* 이 OK 라고 본 게 가장 큰 누락.
- 자동 코드 리뷰 (Gemini high) 가 *증상* 만 지적하고 *근본 원인 (시맨틱 합쳐짐)* 은 안 짚었지만, 사용자와 대화하며 *더 정직한 방향* 으로 다시 잡을 수 있었던 흐름이 가치 있었음. *제안 patch 그대로 받기 vs 근본까지 파고들기* 의 판단이 리뷰 수용의 본질.

## 배운 것

- **return type 의 시맨틱이 *정상* 과 *시스템 사정* 을 동시에 표현하면 거짓말이 된다.** 의미 두 가지면 *예외로 분리* — 시그니처가 정직해진다. caller 가 분기 의도를 *코드 흐름으로* 표현 가능해짐.
- **port/adapter 패턴에서 fallback 정책의 결정 주체는 application**. infra 는 *기술적 사실* (어떤 예외) 만 알면 되고, *어떻게 대응할지* 는 application. 이걸 *port 시그니처에 Spring 예외 노출 없음* 으로 강제하면 추상화가 보존됨.
- **TTL race + finally clear 의 메커니즘 — `clear` 가 key 기반 *무조건 delete* 면 *늦게 끝난 자기 자신* 이 *남의 marker* 를 삭제 가능**. owner-token + compare-delete (Lua) 가 해결책. 다만 *trigger 조건 (TTL 초과 가능 latency)* 이 없으면 *메커니즘 명시 + 해결책 사전 파악* 만 해두고 받아들임. *낮은 빈도 race* 의 대응 패턴.
- **자동 코드 리뷰 (Gemini / Codex) 의 한계와 가치**. *증상은 정확* 하지만 *해결책은 단순* (return true 한 줄). *근본 원인까지 파고들어 더 나은 방향* 으로 갈 수 있는 판단은 사람 몫. 리뷰 채택 = 제안 patch 그대로 받기 가 아니라 *지적 본질에 정직한 해결*.
- **`@MockitoSpyBean` 의 자동 reset** (Spring Boot 3.4+, `MockReset.AFTER` 기본) — *테스트 간 stub 누수 없음*. 별도 `@AfterEach reset` 필요 없음. 다른 agent 가 보수적으로 *명시되지 않음* 이라고 경고했는데, 공식 기본 동작 확인이 더 정확.

## 다음 단계

- PR #180 머지 (사용자)
- #181 (Auth 패턴 통일 검토) — 후속 별도 PR. application 직접 catch → infra 매핑으로. 동작 보존, port 추상화만 강화.
- 결제 PG 통합 *계획 없음* 으로 확인됨 → TTL race owner-token 도입 issue 는 불필요.
