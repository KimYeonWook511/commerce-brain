---
platform: backend
author: KimYeonWook511
created: 2026-07-11
origin:
  - { type: pr, repo: commerce-backend, ref: 282 }
---

# 도메인 예외를 전송 계층에서 떼기 — HttpStatus 대신 의미 분류(ErrorCategory)

도메인 예외가 HTTP 상태코드를 직접 들고 있던 것을 걷어내, 도메인이 전송 계층(HTTP)을 모르게 만든 작업이다. 앞선 [[raw/sessions/backend/2026-07-11-pr-281-exception-exposure-boundary-abstraction]](예외 노출 경계 재정의)에서 미뤄둔 부분을 완결했다.

## 결정한 것

**문제.** 이 백엔드는 각 도메인마다 에러 코드 enum(주문·결제·인증·재고 등의 `*ErrorCode`)을 두고, 도메인 예외가 그 에러 코드를 든다. 그런데 에러 코드 인터페이스가 `getStatus()`로 `HttpStatus`를 직접 반환했다 → 도메인별 에러 코드 enum들이 Spring의 HTTP 타입(`org.springframework.http.HttpStatus`)에 의존했다. 즉 도메인이 전송 계층(HTTP)을 아는 상태. 추후 도메인을 별도 모듈로 떼면 web 의존이 함께 새어 나간다.

**결정 1 — 에러 코드는 HttpStatus 대신 의미 분류(`ErrorCategory`)를 든다.** 인터페이스를 `getStatus()`(HttpStatus 반환) → `getCategory()`(의미 분류 반환)로 바꾼다. 도메인 예외는 상태코드를 모른다. 카테고리 → HttpStatus 매핑은 **HTTP를 아는 경계**가 소유한다(별도 매핑 유틸 `ErrorCategoryHttpStatus.of(category)`). 그 경계는 전역 예외 핸들러 + 인증 필터 + 인가 인터셉터 + 도메인별 예외 advice.

**결정 2 — 카테고리 집합은 현재 외부 응답 상태코드를 정확히 보존하도록 정한다(외부 계약 변경 0).** 이 서비스가 실제로 쓰는 상태는 400·401·403·404·409·500·502(503은 안 씀). 이를 각각 의미 카테고리로 1:1 매핑:

| ErrorCategory | HttpStatus | 의미 |
|---|---|---|
| INVALID | 400 | 요청 형식·값 오류 |
| UNAUTHORIZED | 401 | 인증 필요/실패 |
| FORBIDDEN | 403 | 권한 없음 |
| NOT_FOUND | 404 | 없음 |
| CONFLICT | 409 | 동시성·상태 충돌 |
| UPSTREAM_ERROR | 502 | 결제 PG 등 상류 시스템 장애 |
| INTERNAL | 500 | 내부 오류 |

각 카테고리는 "없음/금지/상류 장애/충돌" 등 진짜 의미 분류라, HttpStatus에 그냥 다른 이름을 붙인 별칭이 아니다.

**검토한 대안과 트레이드오프.**
- (기각) 매핑을 전역 예외 핸들러의 private 메서드로 둔다 — **불가.** 매핑을 쓰는 경계가 4곳인데 그중 3곳(서블릿 필터, 핸들러 인터셉터, 별도의 도메인 예외 advice)이 전역 핸들러 밖에서 응답 상태를 직접 쓴다. 서블릿 필터/인터셉터는 Spring MVC 예외 처리(@RestControllerAdvice) 밖에서 동작하므로 전역 핸들러의 private 메서드에 접근할 수 없다. 각자 switch를 복붙하면 카테고리 추가 시 여러 곳을 다 고쳐야 한다.
- (기각) 매핑을 `ErrorCategory` enum의 메서드로 둔다(`category.toHttpStatus()`) — **불가.** 그러면 `ErrorCategory`가 `HttpStatus`에 의존하는데, `ErrorCategory`는 도메인 에러 코드들이 참조한다 → 도메인이 다시 HTTP에 전이 의존하게 되어, 없애려던 바로 그 의존이 되살아난다.
- (채택) 경계만 쓰는 작은 매핑 유틸(static `of` + switch). `ErrorCategory`는 HTTP를 모르게 유지하고, 매핑은 여러 경계가 공유한다 — 두 요구를 동시에 만족하는 최소 지점.
- (기각) 카테고리를 더 적은 5개 의미로 좁히고 일부 상태를 재매핑(404/403을 400으로 흡수, 502를 503으로) — 외부 응답 상태코드가 바뀌는 **계약 변경**이라 프론트엔드 등 클라이언트에 영향. 리팩터의 전제가 "외부 계약 불변"이라 기각. 같은 이유로 저장소 장애 응답도 현행 500을 유지(503으로 바꾸지 않음).

**매핑 누락 방어는 런타임이 아니라 컴파일 타임.** `ErrorCategoryHttpStatus.of`는 **default 없는 switch expression**으로 짠다. 새 카테고리를 추가하면 매핑을 반드시 채워야 컴파일된다 — 이게 진짜 위험(카테고리 누락)을 막는 안전장치다. 반대로 `category == null` 같은 런타임 null 방어는 **넣지 않기로** 했다: 인자는 항상 enum의 non-null 카테고리 필드이고, 도메인 예외 객체의 생성자가 `super(errorCode.getMessage())`를 호출하므로 null 에러 코드로는 예외 객체 자체가 생성될 수 없다(생성 시점에 먼저 터진다). 즉 그 null은 도달 불가한 상태라, 방어 코드는 사용처 없는 dead code가 된다.

**상태 보존 실증.** 8개 에러 코드 enum의 총 65개 항목 전부 "원래 HttpStatus → 새 카테고리 → 매핑된 HttpStatus"가 1:1로 보존됨을 전수 대조. 특정 HTTP 상태를 단언하는 기존 슬라이스/web 테스트가 그대로 통과. ArchUnit에는 domain의 `org.springframework.http`·`org.springframework.web` 의존 금지 규칙을 추가(에러 코드 enum들이 HttpStatus 의존을 뗀 뒤라야 통과).

## 배운 것

- **도메인을 전송 계층에서 떼는 일반 레시피**: 도메인 예외/에러 코드는 **의미 분류만** 들고, "의미 → 전송(HTTP 상태)" 매핑은 전송을 아는 경계가 소유한다. 매핑을 의미 enum 자체에 넣으면 도메인이 다시 전송에 의존하니, 매핑은 **반드시 경계 쪽 별도 지점**에 둔다.
- **매핑 유틸이 한 핸들러의 private 메서드면 안 되는 이유**: 진입 경계가 하나가 아니다(MVC 예외 advice 외에 서블릿 필터·인터셉터도 응답 상태를 직접 씀). 여러 경계가 공유하는 매핑은 공유 유틸로 빼야 하고, 그래야 카테고리 추가 시 한 곳만 고친다.
- **default 없는 switch가 런타임 null 방어보다 낫다**: 진짜 위험은 "새 분류를 추가했는데 매핑을 빠뜨림"이고, 그건 컴파일 타임에 막힌다. 도달 불가한 입력에 대한 런타임 방어는 dead code라 오히려 코드를 흐린다(불변식으로 null이 못 들어옴을 먼저 확인).
- **"외부 계약 불변"이 전제인 리팩터**에선 의미 분류 집합을 현재 쓰는 상태에 1:1로 맞춰 재매핑을 0으로 만든다. 의미 분류를 과하게 뭉뚱그리면(coarse) 상태가 뭉개져 계약이 바뀐다 — 분류의 세밀도는 "지금 실제로 쓰는 응답 상태를 다 표현하는가"로 정한다.

## 미해결·열린 질문

- 특별히 열려 있는 쟁점은 없다. 앞으로 새 응답 상태/의미가 필요해지면 default 없는 switch가 매핑 누락을 컴파일 타임에 강제하므로 자연히 드러난다.
