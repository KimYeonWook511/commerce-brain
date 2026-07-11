---
platform: backend
author: KimYeonWook511
created: 2026-05-28
origin:
  - { type: pr, repo: commerce-backend, ref: 166 }
---

# 장바구니 도메인 도입 — 낙관락 동시성과 4차 외부 리뷰 누적 대응

장바구니(cart)를 신규 도메인으로 도입한 세션이다. `CartItem` 단일 entity aggregate로 모델링하고,
다른 aggregate를 객체 참조 대신 `Long` ID로만 식별하며, 동시 add/update의 경쟁 조건을 낙관적 락 +
retry + Processor 분리로 처리했다. 초기 구현 뒤 외부 AI 코드 리뷰(codex·gemini)를 여러 차례 받고
자체 검토를 두 번 더 거치면서, 동시성 결함·Product 미검증·경로 파라미터 검증·삭제 정책 비대칭 등을
누적으로 손봤다. 결정 누적과 리뷰 흐름 자체가 이 세션의 핵심이다.

## 결정한 것

### 1. cart 도메인 골격 — CartItem 단일 aggregate + cross-aggregate는 ID 참조

- **CartItem-only 단일 entity aggregate.** Cart(root)-CartItem(N) 구조 대신 `CartItem(memberId,
  productId, quantity)` 하나만 두고, root를 CartItem 자신으로 했다. "사용자의 cart"는 `memberId`
  필터 조회 결과 리스트다. cart 자체에 붙는 메타데이터(쿠폰 슬롯·메모·상태·만료)가 없고 사용자당
  cart가 1개라, Cart entity를 만들어도 `(id, memberId, createdAt, updatedAt)`뿐인 빈 컨테이너가 된다.
  기존 코드베이스의 `StockHistory`·`RefreshToken` 같은 단일 entity aggregate와 같은 패턴. 나중에
  cart 레벨 메타데이터가 필요해지면 Cart aggregate를 추가하고 `cartId`를 붙이는 마이그레이션으로
  확장 가능하다.
- **다른 aggregate는 `Long` ID로만 참조.** `CartItem`은 `memberId`·`productId`를 원시 `Long`으로
  저장하고 `@ManyToOne`/`@JoinColumn` FK를 쓰지 않는다. 기존 도메인(`Order.member`,
  `OrderItem.product`, `Stock.product` 등)은 객체 참조를 광범위하게 쓰지만 application 계층은 대부분
  ID로 흐름을 다뤄, 도메인과 인터페이스 사이에 이중 표현·N+1·fetch join 부담이 누적돼 있었다.
  DDD의 "다른 aggregate는 identity로만 참조" 원칙을 신설 도메인부터 기본값으로 삼았다. cart 조회 시
  `productRepository.findAllById(productIds)`로 Product를 한 번 더 조회해 응답을 조립하는데, PK 인덱스
  조회라 비용은 무시 가능하다. 대신 DB 참조 무결성을 FK가 보장하지 않으므로, 정합성은 application
  검증·UNIQUE 제약·삭제 순서 정책이 책임진다(이게 아래 "장바구니 추가 시 Product 존재·상태 검증"의 전제가 된다). 이 결정은 이 phase에
  국한하지 않고 이후 신설 도메인 전체에 적용하는 컨벤션으로 명문화했고, 기존 도메인의
  ManyToOne→ID 마이그레이션은 별도 트랙으로 분리했다.
- **가격은 cart에 저장하지 않는다.** `CartItem`은 `productId`·`quantity`만 갖고, 조회 시점에 Product를
  다시 읽어 최신 `price`·`name`·`imageUrl`·`status`를 응답에 조립한다. 주문 흐름이 어차피 주문 시점에
  Product 가격을 재검증하므로 cart에 박아둔 가격은 신뢰할 수 없다.
- **주문 성공 시 cart 제거는 주문 트랜잭션 안에서.** `OrderCreateProcessor`의 주문 저장 트랜잭션
  안에서 `CartItemRemover.removeByMemberAndProducts(memberId, productIds)`를 호출한다(내부적으로
  `deleteByMemberIdAndProductIdIn`). cart와 order가 같은 RDB라 커밋-후 이벤트로 분리할 이유가 없고,
  "주문은 성공했는데 cart는 그대로"인 일시 불일치를 안 만든다. 주문 요청 productId가 cart에 없어도
  0 row 삭제로 자연 처리되므로, 주문에 cart 존재 검증을 걸지 않는다 — "지금 구매"·재시도 같은 정상
  흐름을 막지 않기 위해서다. 멱등 응답 경로는 `OrderCreateProcessor.execute`를 타지 않아 두 번
  제거되지 않는다(별도 가드 불필요).

### 2. 동시성 — 낙관락 + Processor 분리 + retry

cart 항목 추가·수정의 경쟁 조건이 리뷰에서 지적됐다. 경쟁을 두 종류로 나눠 본 게 핵심이다.

- **update race:** 같은 (memberId, productId) row에 동시 add 요청 두 개 → 두 트랜잭션이 같은
  quantity를 fetch한 뒤 각자 `increaseQuantity` → 마지막 commit만 반영. 합산 손실 + 98+1+1 같은
  상황에서 99 상한이 silent로 우회될 수 있다.
- **insert race:** cart에 없는 productId에 동시 첫 add 요청 두 개 → 둘 다 `Optional.empty()` 분기에서
  `save` 시도 → `uk_cart_item_member_product` UNIQUE 위반으로 한쪽 500.
- 같은 양상이 `UpdateCartItemQuantityService`(`changeQuantity`)에도 있다.

4안을 비교해 낙관락을 택했다.

- **(A) 비관락 `@Lock(PESSIMISTIC_WRITE)`** — 재고 차감이 쓰는 패턴과 일관되지만, cart는
  (memberId, productId) 단위라 contention이 사실상 0이라 평상시 매 요청 row lock overhead가 합당하지
  않다. 재고에서 비관락을 기본으로 택하며 감수한 트레이드오프(높은 경쟁 상황의 락 대기·DB 부담)가
  cart에는 이득 없이 단점만 남는다. 게다가 update race만 풀고 insert race는 안전망에 별도 위임해야
  한다. 기각.
- **(B) 낙관락 `@Version` + retry (채택)** — entity에 `@Version` 추가, UPDATE 충돌이
  `ObjectOptimisticLockingFailureException`으로 전파, 충돌 시에만 retry. contention이 0에 가까워
  평균 attempt는 1로 수렴하고 평상시 lock overhead가 없다. 도메인 메서드(`increaseQuantity`,
  `changeQuantity`, `validateQuantity`)를 그대로 써서 invariant가 도메인에 단일 표현된다. Order·Stock이
  이미 같은 `@Version` + retry 패턴(`StockConcurrencyService.decreaseWithOptimisticLock`)을 써 코드베이스
  일관성도 있다.
- **(C) atomic UPDATE 쿼리**(`SET quantity = quantity + :delta WHERE ... AND quantity + :delta <= 99`)
  — 단일 SQL로 race-safe에 가장 가볍지만, 상한(`<= 99`) invariant가 SQL과 도메인에 이중 표현된다.
  `@Modifying`이 dirty checking을 우회해 `updated_at` 자동 갱신이 깨지고, affected rows=0 분기에서
  후속 SELECT가 필요하며 `clearAutomatically=true`로 persistence context stale도 막아야 한다. 기각.
- **(D) insert-first → UNIQUE catch → addQuantity fallback** — 새 사용자의 find 호출을 아끼지만,
  UNIQUE 식별이 Hibernate `ConstraintViolationException.getConstraintName()` 검사 등 구현체-specific에
  의존해 fragile하다. `@Transactional` 안에서 unchecked 예외가 던져지면 트랜잭션이 rollback-only로
  마킹돼 같은 트랜잭션에서 두 번째 작업이 불가하다. `REQUIRES_NEW`나 빈 분리가 필요해 결국 (B)와
  비슷한 구조가 되고, 두 번째 분기도 여전히 update race-prone이라 (B)의 보호가 또 필요하다. (B) 대비
  장점 없이 함정만 추가. 기각.

- **Processor 분리는 self-invocation 함정 회피용이다.** outer Service(`AddCartItemService`,
  `UpdateCartItemQuantityService`)는 어노테이션 없이 retry loop만 담당(`MAX_RETRY = 3`)하고, 실제
  트랜잭션 경계는 별도 빈(`AddCartItemProcessor`, `UpdateCartItemQuantityProcessor`)의 method-level
  `@Transactional`이 책임진다. retry attempt마다 빈 경계를 넘어 새 트랜잭션·새 persistence context로
  진입한다. 같은 세션에서 응용 Service의 `@Transactional`을 class-level 기본값 대신 method-level에만
  붙이는 컨벤션을 새로 명문화했는데, Processor 패턴이 그 컨벤션의 실물이다.
- **retry catch 대상은 `ObjectOptimisticLockingFailureException`만.** insert race의
  `DataIntegrityViolationException`은 retry에 넣지 않고 안전망 500에 위임한다 — DB unique 위반은
  안전망 500에 맡기고 정상 흐름은 사전 find로 거른다는 기존 find-first 정책을 그대로 유지한 것이다.
  UNIQUE 외 다른 무결성 위반(향후 제약 추가 시)이 retry로 silent하게 묻히는 걸 막기 위해서다. cart
  insert race는 같은 사용자가 같은 productId를 처음 담는 순간 ms 단위로 두 번 요청해야 나는 극히
  드문 경우이고, 한 번 row가 생기면 이후는 update race 경로(B로 흡수)로 분기하므로 안전망 위임이
  실용적으로 충분하다.
- Processor 안에서는 dirty checking 묵시 의존을 끊고 `repository.save(entity)`를 명시 호출해 영속화
  의도를 코드 표면에 남기는 컨벤션도 함께 적용했다.

### 3. 장바구니 추가 시 Product 존재·상태 검증

외부 리뷰(codex)가 `POST /cart/items`가 `productId`의 `@Positive`만 보고 실제 Product 존재를 확인하지
않는다고 지적했다 — 임의의 미존재 productId로도 201을 받고 orphan cart row가 생긴다. cart는 FK를
안 두기로 한 결정(cross-aggregate를 ID로만 참조) 때문에 데이터 정합성의 유일한 게이트가 application 검증이다.

- **매핑:** `AddCartItemProcessor`가 row를 만들기 전에 `productRepository.findById`로 검증한다.
  미존재 또는 soft-deleted면 `CART_ITEM_PRODUCT_NOT_FOUND`(`CART-404-2`, 404), `STOPPED`면
  `CART_ITEM_PRODUCT_UNAVAILABLE`(`CART-409`, 409). 결함(미존재·삭제됨)과 일시 차단(STOPPED)을 코드로
  분리해 운영에서 따로 추적할 수 있게 했다.
- **"담은 뒤 상태 변화"와는 다른 문제로 봤다.** 이미 담아둔 항목이 STOPPED/soft-deleted가 되는 건
  응답에 `unavailable=true`로 마킹해 보존·표시하는 별도 정책이 있다. "처음부터 못 사는 상품을 새로
  담는 동작"은 add 시점에 차단하는 게 자연스럽다.
- **PATCH는 product 검증을 두지 않는다.** 기존 cart row의 수량 변경이라, 상태 변화는 이미
  `unavailable` 응답으로 운영 가시성이 확보돼 있다.

### 4. DELETE 미존재 4xx + entity를 통한 delete

초기 구현은 `DELETE /cart/items/{productId}`가 미존재 시 200 + null 반환(멱등 정책)이었다. 그런데
`RemoveCartItemService`가 0 row여도 `log.info("장바구니 항목 삭제 ...")`를 silent로 찍어 운영 로그가
부정확해지는 회귀가 있었다. 게다가 PATCH는 미존재 시 4xx throw인데 DELETE만 200이라 정책이
비대칭이었다. → DELETE도 미존재 시 `CART_ITEM_NOT_FOUND`(`CART-404-1`) 4xx로 통일했다. cart DELETE는
사용자 명시 액션이라 동일 DELETE 재요청 같은 멱등 시나리오가 거의 없어, REST DELETE의 멱등 성질을
약하게 깨더라도 명확한 피드백·운영 가시성을 우선했다.

- **구현은 처음 bulk `@Modifying DELETE`로 짰다.** find→delete 사이 race window를 결정 문서의 "범위
  제한" 단락으로 명문화하는 방식이었다. 그런데 사용자가 "find해왔으면 해당 entity를 delete에
  넘기면 되잖아 왜 벌크쿼리로 하는거야?"라고 지적 → `findByMemberIdAndProductId`로 managed entity를
  로드한 뒤 `cartItemRepository.delete(cartItem)`로 리팩터했다.
- **이 한 번의 리팩터로 세 가지가 동시에 풀렸다.** (1) entity를 통한 delete라 `@Version` 체크가
  적용돼, 동시 DELETE race(다른 트랜잭션이 먼저 삭제했거나 add로 version을 올린 경우)에서 두 번째
  트랜잭션이 `ObjectOptimisticLockingFailureException` → 409(`COMMON-409-1`)로 surface된다. race 시점의
  silent log 회귀도 자연 차단. (2) bulk delete가 사라져, 리뷰에서 나온 `@Modifying(clearAutomatically=
  true)` 부착 권고(persistence context stale 회피)도 불필요해졌다. (3) 결정 문서의 관련 단락이 "범위
  제한"에서 "race 처리"로 단순화됐다.
- `RemoveCartItemService`는 add/update 경로(낙관락 + retry)와 달리 retry loop가 없어 409가 클라이언트에 그대로 노출된다.
  retry 책임은 클라이언트에 위임한다(재요청 시 보통 404 또는 정상 성공으로 정리됨).

### 5. path productId — spec을 코드에 맞춤

외부 리뷰(codex)가 PATCH/DELETE의 `@PathVariable Long productId`에 양수 검증이 없어 0/음수가 service
까지 내려가 404로 응답한다고 지적했다. spec 문구("양수, 필수")와 어긋난다.

- 처음엔 `@Validated + @Positive + ConstraintViolationException 핸들러 신설`을 제안했다가, 사용자가
  "다른 controller는 어떻게 하고 있어?"로 물어 재조사했다. 결과: 코드베이스의 모든 controller가 path
  id 검증을 안 하고 0/음수가 내려와도 not-found로 일관 처리하고 있었다. cart만 다른 패턴을 도입하면
  부분적 일관성이라 오히려 부담.
- → 코드를 spec에 맞추는 대신 spec 문구를 "코드베이스 path id는 미존재 항목으로 4xx 응답하는 일관
  정책"으로 완화했다. spec/코드 정합성 회복은 양방향 가능하다는 사례로 남겼다.

### 6. 회원당 cart row 개수 상한은 두지 않는다

항목당 수량은 MIN=1, MAX=99로 도메인이 강제하지만, 회원당 cart row 개수(서로 다른 productId 수)는
무가드다. 외부 agent 검토에서 abuse 시나리오(자동화로 수천 productId add → row 누적 +
`findAllById(IN ...)` 부담)가 지적됐다.

- 이 phase에서는 상한을 두지 않고 트랙으로만 명시했다. 정상 사용자의 cart 항목 수는 보통 수십
  단위라 부담이 없고, 상한을 도입하면 "어디(도메인/application/DTO)에 둘지, 초과 시 어떤 행위를
  거부할지(가장 오래된 항목 자동 제거? 4xx?), invariant를 어떻게 표현할지" 같은 추가 결정이 누적돼
  phase scope를 넘는다. abuse는 application보다 인증·rate limiting 등 상위 게이트의 책임에 가깝다.
- 운영에서 abuse가 관측되거나 IN 절 binding 한도가 임박하면 (a) 회원당 row 상한 (b) GET /cart
  페이지네이션 (c) rate limiting 게이트 강화 중 하나를 택하기로 미뤘다.

## 배운 것

### 사용자의 도메인 인사이트가 추상적 리뷰보다 강할 수 있다

이 세션에서 가장 강력한 개선(DELETE를 bulk 쿼리에서 entity 경유 삭제로 바꾼 것)이 사용자의 한 마디에서 시작됐다.

> "find해왔으면 해당 entity를 delete에 넘기면 되잖아 왜 벌크쿼리로 하는거야?"

이 한 줄이 `@Modifying` 옵션 부착 권고 + bulk delete의 race silent log + 관련 결정 단락을 한 번에
정리했다. LLM은 옵션 비교(예: "`clearAutomatically`를 붙일까")로 접근하는데, 사용자는 "왜 굳이 그렇게
짰지?"라는 도메인 질문으로 접근한다. 추상적 검토보다 도메인 맥락 인사이트가 더 강력할 수 있다.

### 즉시 동조(sycophancy)를 눌러야 트레이드오프가 남는다

사용자가 "낙관적 락이 베스트같구만" 하자 즉시 "맞습니다" 모드로 진입했다. 사용자가 "또 내가
그렇게 말했따고 그게 정답이라는 듯이 말하네"라고 지적했고, 그 뒤 비관락/낙관락/atomic UPDATE/
insert-first 4안을 객관 비교해 같은 결론(낙관락)에 도달했다. 같은 결론이라도 즉시 동조와 객관
분석은 산출물이 다르다 — 두 번째만 4안의 트레이드오프가 결정 문서로 남는다. 사용자 의견도 검증
대상으로 두고 객관 분석을 먼저 한다.

### 외부 agent 검토가 자체 검토보다 stale 발견에 강하다

같은 세션이 만든 결정을 같은 세션이 검토하면 편향이 있다. 이 세션은 자체 검토를 두 번 더 호출했다:
1차는 코드/문서 식별자 정합(stale identifier 다수, 결정 문서와 코드의 중복 표현 drift, unreachable
분기, 광범위 단언)을 발견해 정리했고, 2차는 보안/edge case/예외 처리를 봤다(Must 0건, Should 3건 —
cart row 상한, race 시 409 노출, PR 본문 stale). agent에게 "이미 검토된 영역은 재검토 말라"고 명시하면
비용은 작고 stale 발견에 강하다. 자체 검토는 빠뜨리는 게 많다.

### 리뷰 누적은 commit 분리 정책이 아니라 PR scope에서 온다

이 PR은 커밋 수가 일반 PR(5~20) 기준 outlier(세션 기록상 38 commits, 초기 구현 14 + 외부 리뷰 대응
10 + 자체 검토 정리 7로 분해됨)였다. 근본 원인은 commit 분리 방식이 아니라 **PR scope가
컸고** + **리뷰가 여러 차례 누적**된 것이었다. 한 PR에 (1) cart 도메인 코드 (2) order 결합 코드
(3) 응용 계층 컨벤션(method-level 트랜잭션·명시적 영속화 호출) (4) 결정 문서 색인 (5) concurrency
테스트 gradle 태스크 — 다섯 가지 책임이 섞였다. squash merge라 main history엔 영향이 없지만, PR
페이지 검토 비용은 그대로 누적된다. → 다음 phase는 PR scope를 작게(도메인 PR / cross-cutting 컨벤션
PR 분리), Draft PR로 출발해 1차 리뷰를 받고 ready로 전환하기로 했다.

## 미해결·열린 질문

- **cart insert race 멱등 흡수 재방문 여지.** 지금은 insert race를 안전망 500에 위임한다. cart UX상
  "두 번째 클릭도 조용히 흡수"가 더 자연스럽다는 의견이 향후 강화되면, 동시성 처리 방식(낙관락 + 안전망 위임)을 재방문해
  (C) atomic UPDATE 또는 `ConstraintViolationException.getConstraintName()` 기반 정확한 UNIQUE 식별 +
  retry로 전환할 수 있다.
- **회원당 cart row 상한.** 위 "회원당 cart row 개수 상한은 두지 않는다"에서 정리한 대로 이 phase에서는 의도적으로 미뤘다. abuse 관측·IN 절 한도
  임박이 트리거.
- **다음 phase의 PR 분리 실험.** 도메인 PR과 cross-cutting 컨벤션 PR을 나누는 방식을 아직 실제로
  적용해보지 않았다.
- **hook cwd 의존 버그(별도 트랙).** 이 세션 도중 `.claude/settings.json`의 PreToolUse hook이
  cwd-relative path에서 깨지는 문제를 발견해 issue #169로 발급했다. cart 도메인과 무관한 chore라
  별도 PR 트랙으로 분리했다.

> 참고: 위 커밋 수 outlier·자체 검토 회차별 발견 건수는 세션 당시의 기록으로, squash merge된 현재
> history에서는 개별 커밋 단위로 재확인되지 않는다.
