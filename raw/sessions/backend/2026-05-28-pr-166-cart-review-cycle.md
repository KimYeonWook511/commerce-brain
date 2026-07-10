---
platform: backend
author: KimYeonWook511
created: 2026-05-28
origin:
  - { type: pr, repo: commerce-backend, ref: 166 }
---

# PR #166 — cart phase와 review cycle

Phase 2 장바구니 도메인을 도입하고, codex/gemini 외부 review 4차 + 내부 agent 검토 2차를 누적 대응한 세션. 결정 누적과 review 흐름이 본 세션의 핵심.

본 phase의 결정 정본은 `commerce-backend/docs/tasks/cart/adr.md`. 여기서는 정본을 인용하면서 "내가 어떻게 이해했고 다시 본다면" 관점만 남긴다.

## 한 일

- Phase 2 cart 도메인 신설 — `CartItem`-only 단일 entity aggregate, 다른 aggregate를 객체 참조 대신 `Long` ID로 식별, 동시성 정책(낙관락 + Processor 분리 + retry), 주문 트랜잭션 내 cart 제거 연동
- 4차 외부 review 대응 — codex 1차(동시성 결함), gemini(`CartItem.create` null 검증), codex 2차(path productId 검증), Claude Code 자체 호출 2회(코드/문서 식별자 정합, 보안/edge case/예외 처리)
- 사용자 직접 코드 제안으로 `RemoveCartItemService`를 bulk delete에서 entity 통한 delete로 리팩터 → JPA `@Modifying` 옵션 부착 권고 + bulk delete의 race window silent log + 관련 ADR 단락이 한 번에 자연 해소
- 신규 ADR — ADR-020(신규 도메인의 cross-aggregate 참조는 `Long` ID로), ADR-021(응용 Service `@Transactional`은 method-level), ADR-022(응용 계층에서 `repository.save(entity)` 명시 호출), Task ADR 색인. 모두 `commerce-backend/docs/ADR.md`에 누적
- PR #166: 38 commits, squash merge로 main 1 commit. 머지는 사용자가 직접 진행 예정
- issue #169 발급 — `.claude/settings.json` PreToolUse hook의 cwd-relative path 깨짐 문제(별도 chore PR 트랙)

## 결정한 것

### 동시성 — 낙관락 + Processor 분리 + retry

정본: `commerce-backend/docs/tasks/cart/adr.md` 결정 8.

cart의 race를 두 종류로 분리해서 본 게 핵심.
- **update race**: 같은 (memberId, productId) row에 동시 add 요청 → 두 트랜잭션이 같은 quantity를 fetch 후 각자 increase → 마지막 commit만 반영. 합산 손실 + 99 상한 silent 우회 가능
- **insert race**: cart에 없는 productId에 동시 add 첫 요청 → 두 트랜잭션 모두 `Optional.empty()`에서 insert → UNIQUE 위반으로 한쪽 500

4안을 비교해 낙관락 채택:
- (A) 비관락 — cart contention이 거의 0이라 평상시 lock overhead가 합당하지 않음
- (B) 낙관락 + retry + Processor 분리 — 채택
- (C) atomic UPDATE 쿼리 — invariant가 SQL과 도메인에 이중 표현 + `@Modifying`이 `updated_at` 자동 갱신과 충돌
- (D) insert-first → UNIQUE catch → addQuantity fallback — Hibernate-specific 의존 + rollback-only 마킹 + (B) 보호 추가 필요

Processor 분리는 self-invocation 함정 회피용. outer Service는 retry loop만 담당하고 실제 트랜잭션은 별도 빈(`AddCartItemProcessor`, `UpdateCartItemQuantityProcessor`)의 method-level `@Transactional`이 책임진다. retry attempt마다 빈 경계를 넘어 새 트랜잭션·새 persistence context로 진입.

`repository.save` 명시 호출(ADR-022)도 같은 맥락 — dirty checking 묵시 의존을 끊고 영속화 의도를 코드 표면에 보존.

**다시 본다면**: 재고 도메인의 비관락 정책(`commerce-backend/docs/ADR.md` ADR-003)의 trade-off를 cart에 무비판적으로 끌어붙이려다 사용자 지적으로 객관 평가로 돌아왔다. contention이 매우 낮은 도메인은 처음부터 낙관락으로 빨리 가는 게 맞다.

### DELETE 미존재 4xx + entity 통한 delete

정본: `commerce-backend/docs/tasks/cart/adr.md` 결정 6-4.

초기 구현: 미존재 시 200 + null 반환 멱등 정책 → silent log miss 회귀 발생. PATCH는 4xx throw인데 DELETE만 200이라 정책 비대칭. → DELETE도 미존재 시 `CART_ITEM_NOT_FOUND` 4xx로 통일.

처음에는 bulk `@Modifying DELETE`로 짰고, find→delete 사이 race window를 ADR "범위 제한" 단락으로 명문화했다. 사용자가 "find해왔으면 해당 entity를 delete에 넘기면 되잖아 왜 벌크쿼리로 하는거야?"라고 지적 → entity 통한 delete로 리팩터.

효과:
- `@Version` 체크가 적용되어 race 시 `ObjectOptimisticLockingFailureException`으로 surface → silent log race 자연 해소
- bulk delete가 사라져 JPA `@Modifying(clearAutomatically=true)` 부착 권고(검토에서 제기됨)도 해소
- ADR 6-4 단락이 "범위 제한"에서 "race 처리"로 단순화

**다시 본다면**: bulk delete를 채택한 이유가 "find 결과를 안 쓸 거니까 효율적"이었는데, 같은 쿼리 수에 indirection만 하나 더 늘어남. 처음부터 entity delete였어야.

### Product 존재·상태 검증

정본: `commerce-backend/docs/tasks/cart/adr.md` 결정 6-5.

codex 지적: POST /cart/items가 `@Positive`만 검증하고 실제 Product 존재를 안 보는 결함. ADR-020(FK 미사용)에서 데이터 정합성은 application 검증이 유일 게이트.

매핑:
- 미존재 / soft-deleted → `CART-404-2` (`CART_ITEM_PRODUCT_NOT_FOUND`)
- STOPPED → `CART-409` (`CART_ITEM_PRODUCT_UNAVAILABLE`)
- PATCH는 검증 안 함 — 이미 담아둔 항목의 상태 변화는 결정 6-1(`unavailable=true` 마킹)으로 운영 가시성 확보됨

### path productId — spec을 코드에 맞춤

codex 지적: PATCH/DELETE의 `@PathVariable Long productId`에 양수 검증이 없어 0/음수가 service까지 내려가 404로 응답. spec 문구("양수, 필수")와 어긋남.

처음에 `@Validated + @Positive + ConstraintViolationException 핸들러 신설`을 제안했다가, 사용자가 "다른 controller는 어떻게 하고 있어?"로 물어서 재조사. 결과: 코드베이스 모든 controller가 path id 검증을 안 하고 0/음수가 내려와도 not-found로 일관 처리. cart만 다른 패턴 도입은 부분적 일관성이라 부담.

→ spec 문구를 "코드베이스 path id는 미존재 항목으로 4xx 응답되는 일관 정책"으로 완화. spec/코드 정합성 회복은 양방향 가능하다는 사례.

### 회원당 cart row 상한 미설정

정본: `commerce-backend/docs/tasks/cart/adr.md` 결정 9.

수량은 99 상한이지만 회원당 cart row 개수는 무가드. agent 외부 검토에서 abuse 시나리오 지적. 본 phase에서는 상한 두지 않고 트랙으로 명시 — abuse 관측 시 (a) row 상한 / (b) 페이지네이션 / (c) rate limiting 중 선택.

정상 사용자 부담 0 + 상한 도입은 "어디서 거부? 자동 제거?" 같은 추가 결정을 누적시켜 phase scope 초과.

## 시행착오와 학습

### review 누적과 PR scope의 관계

38 commits, 일반 PR(5~20) 기준 outlier. 원인 분해:
- 초기 cart phase 구현 (harness 6 step) — 14
- review 대응 4차 — 10
- 검토 정리 (5 + 2) — 7

근본 원인은 commit 분리 정책이 아니라 **PR scope가 컸음** + **review 4차 누적**. 본 PR scope:
1. cart 도메인 (코드)
2. order 결합 (코드)
3. 응용 계층 컨벤션 ADR-021/022 (cross-cutting)
4. Task ADR 색인 (메타)
5. concurrencyTest gradle 태스크 (인프라)

5가지 책임이 한 PR. squash merge로 main history는 영향 없지만, PR 페이지 검토 비용은 누적됨.

**다음 phase 적용**: PR scope를 작게(도메인 PR / cross-cutting ADR PR 분리), Draft PR로 출발 → 1차 review 받고 ready.

### Agent 외부 검토 패턴

같은 세션이 만든 결정을 같은 세션이 검토하면 편향이 있다. 본 PR에서 두 차례 외부 agent 검토 호출:
- 1차: 코드 식별자 stale 11건, ADR 4중 표현 drift, unreachable 분기, 광범위 단언 발견 → 5 commits 정리
- 2차: 보안/edge case/예외 처리. Must 0건, Should 3건(cart row 상한, race 시 409 노출, PR 본문 stale)

agent에게 "이미 검토된 영역은 재검토 말라"고 명시하면 비용 작고 stale 발견에 강함. 자체 검토는 빠뜨리는 게 많다.

### 사용자 도메인 인사이트의 가치 — entity delete 사례

가장 강력한 개선이 사용자의 한 마디로 시작:
> "find해왔으면 해당 entity를 delete에 넘기면 되잖아 왜 벌크쿼리로 하는거야?"

이 한 줄이 JPA `@Modifying` 옵션 부착 권고 + bulk delete의 race silent log + 관련 ADR 단락을 한 번에 정리. LLM은 옵션 비교(예: "`clearAutomatically` 부착할까")로 접근하지만 사용자는 "왜 굳이 그렇게 짰지?"라는 도메인 질문으로 접근. 추상적 검토보다 도메인 맥락 인사이트가 강력할 수 있다.

### Sycophancy 회피

사용자가 "낙관적 락이 베스트같구만" 하자 즉시 "맞습니다" 모드로 진입. 사용자 지적: "또 내가 그렇게 말했따고 그게 정답이라는 듯이 말하네." 이후 비관락/낙관락/atomic UPDATE/insert-first 4안 객관 비교로 같은 결론에 도달.

같은 결론이라도 즉시 동조 vs 객관 분석은 다른 산출물. 두 번째는 trade-off가 ADR로 남는다. 사용자 의견도 검증 대상으로 두고 객관 분석을 먼저 한다.

## 다음 단계

- PR #166 머지 (사용자가 직접, squash merge — `gh pr merge 166 --squash`)
- 머지 후 `/pr-merge-cleanup`으로 worktree 정리
- issue #169 (PreToolUse hook cwd 독립화) 별도 chore PR로 처리
- 다음 phase는 PR scope를 작게 시도 — 도메인 PR과 cross-cutting ADR PR 분리 실험
