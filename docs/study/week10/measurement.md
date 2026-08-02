# 무엇이 달라졌나 — 컴파일러가 고칠 곳을 열거해줬다

> 10주차 금요일. 이번 수정은 성능이 아니라 에러 메시지를 바꾼 것이라 벤치마크할 게 없다.
> 대신 **before/after 응답**과 **변경 범위를 어떻게 확정했는가**를 남긴다.
> 실제 수확은 후자였다.

---

## 1. before / after

바뀐 것은 JSON-RPC 응답 하나다.

```json
// before
{"error": {"code": 4444, "message": "Pruned history unavailable"}}

// after
{"error": {"code": 4444, "message": "pruned history unavailable: requested 5, earliest available 100"}}
```

에러 코드 `4444`(EIP-4444)는 그대로 두고 메시지만 바꿨다. 클라이언트가 코드로 분기하는 로직은
영향을 안 받는다.

테스트가 이걸 고정한다.

```rust
#[test]
fn pruned_history_error_reports_available_range() {
    let err: EthApiError =
        reth_errors::ProviderError::BlockExpired { requested: 5, earliest_available: 100 }.into();
    assert!(matches!(
        err,
        EthApiError::PrunedHistoryUnavailable { requested: 5, earliest_available: 100 }
    ));

    let err: jsonrpsee_types::error::ErrorObject<'static> = err.into();
    assert_eq!(err.code(), 4444);
    assert_eq!(err.message(), "pruned history unavailable: requested 5, earliest available 100");
}
```

`From` 변환 → 코드 → 메시지 세 단계를 한 테스트로 묶었다.

---

## 2. ★ 고칠 곳은 grep이 아니라 컴파일러가 찾았다

변형 정의 한 곳만 바꾸고 빌드하면, **나머지를 컴파일러가 전부 열거한다.**

### 1단계 — 정의만 수정

```rust
-    #[error("Pruned history unavailable")]
-    PrunedHistoryUnavailable,
+    #[error("pruned history unavailable: requested {requested}, earliest available {earliest_available}")]
+    PrunedHistoryUnavailable {
+        requested: u64,
+        earliest_available: u64,
+    },
```

### 2단계 — 같은 크레이트

```
$ cargo check -p reth-rpc-eth-types

error[E0533]: expected unit struct, unit variant or constant,
              found struct variant `EthApiError::PrunedHistoryUnavailable`
   --> crates/rpc/rpc-eth-types/src/error/mod.rs:364:13     ← match 팔
error[E0533]: expected value, found struct variant `Self::PrunedHistoryUnavailable`
   --> crates/rpc/rpc-eth-types/src/error/mod.rs:548:51     ← From impl (본체)
error: could not compile `reth-rpc-eth-types` (lib) due to 2 previous errors
```

### 3단계 — 의존 크레이트

```
$ cargo check -p reth-rpc

error[E0533]: expected value, found struct variant `EthApiError::PrunedHistoryUnavailable`
   --> crates/rpc/rpc/src/eth/filter.rs:495:32
error[E0533]: expected value, found struct variant `EthApiError::PrunedHistoryUnavailable`
   --> crates/rpc/rpc/src/eth/filter.rs:586:32
error[E0533]: expected value, found struct variant `EthApiError::PrunedHistoryUnavailable`
   --> crates/rpc/rpc/src/trace.rs:373:24
error: could not compile `reth-rpc` (lib) due to 3 previous errors
```

**총 5곳. 하나도 빠뜨릴 수 없다.**

`grep`으로 찾았어도 5곳이 나왔겠지만, 그건 "내가 찾은 게 전부인지"를 보증하지 못한다.
컴파일이 통과했다는 사실이 곧 "빠진 곳이 없다"는 증명이다.

### 왜 이렇게 되나

유닛 변형(`PrunedHistoryUnavailable`)과 구조체 변형(`PrunedHistoryUnavailable { .. }`)은
**문법상 쓰는 방법이 다르다.** 유닛 변형은 값 그 자체로 쓰지만, 구조체 변형은 중괄호로 필드를
채워야 값이 된다. 그래서 기존 사용처가 전부 문법 오류(E0533)가 된다.

`.into()`로 감싸져 있든 `return Err(...)` 안에 있든 상관없다. **타입이 아니라 문법 단계에서
걸리기 때문에** 어떤 문맥에 있어도 놓치지 않는다.

---

## 3. ★ 그런데 이 버그를 못 잡은 것도 컴파일러다

여기가 오늘의 진짜 결론이다.

우리가 고친 그 한 줄은 **원래 아무 경고도 안 냈다.**

```rust
ProviderError::BlockExpired { .. } => Self::PrunedHistoryUnavailable,
```

`{ .. }`는 **"이 필드들을 의도적으로 안 쓴다"는 선언**이다. 개발자가 명시적으로 그렇게 적었으므로
컴파일러 입장에서는 아무 문제가 없다. `unused` 계열 경고도 안 붙는다 — 바인딩을 만든 적이
없으니 "안 쓴 변수"가 존재하지 않는다.

| | 컴파일러가 | |
|---|---|---|
| 필드를 **추가**했다 | 기존 사용처 5곳을 전부 잡아줌 | ✅ |
| 필드를 **안 썼다** | 아무 말도 안 함 | ❌ |

> **타입 시스템은 "형태가 안 맞는 것"은 잡지만 "정보를 버리는 것"은 못 잡는다.**

`{ .. }`가 diff에서 눈에 안 띄는 것도 한몫한다. #21270에서 이 줄은 `+` 한 줄짜리 추가였고,
그 커밋의 리뷰 코멘트는 0개였다.

**그래서 이런 종류는 사람이 읽어야 찾는다.** 이번 후보가 9주차에 직접 쓴 노트에서 나온 이유도
같다 — "③은 가능 범위를 필드로 준다"고 적어놓고 그 필드가 어디서 소비되는지 따라가 본 것이
전부다.

---

## 4. 정량 측정은 왜 못 했나

노드를 띄워서 before/after를 재려면 `BlockExpired`가 실제로 발생해야 하는데, **dev 노드에서는
만들 수 없다.** 이유가 코드에 있다.

```
--prune.bodies.distance N
  → Bodies 프루너
  → delete_segment_below_block(StaticFileSegment::Transactions, …)
  → earliest_history_height > 0
  → BlockExpired 발생
```

경로 자체는 살아 있다. 막히는 건 static file의 단위다.

| 제약 | 위치 |
|---|---|
| static file 하나 = **500,000 블록** | `static-file/types/src/lib.rs:35` `DEFAULT_BLOCKS_PER_STATIC_FILE` |
| **가장 높은 파일은 절대 안 지운다** | `static_file/manager.rs:835-838` |
| 노드에 blocks-per-file 오버라이드 없음 | `grep -rn "blocks_per_static_file"` → manifest 명령에만 존재 |

dev 노드는 블록이 수백 개라 Transactions 파일이 **하나뿐**이고, 그 하나가 곧 최고 파일이라
영원히 안 지워진다. 100만 블록을 만들지 않는 한 재현이 안 된다.

그래서 검증은 테스트 레벨로 갔다. 필요해지면 `prune/segments/user/bodies.rs:148`의
`setup_static_file_jars`가 가짜 jar를 만들어 `initialize_index()`를 부르는 패턴을 쓰면
provider 레벨 테스트도 쓸 수 있다.

---

## 5. 변경 규모

```
crates/rpc/rpc-eth-types/src/error/mod.rs | 35 +++++++++++++++++++++++++++----
crates/rpc/rpc/src/eth/filter.rs          | 12 +++++++++--
crates/rpc/rpc/src/trace.rs               |  6 +++++-
3 files changed, 46 insertions(+), 7 deletions(-)
```

35줄 중 18줄이 테스트다. 실제 로직 변경은 정의 7줄 + 사용처 4곳뿐이다.

---

## 6. 오늘의 수확

1. **★ 필드를 추가하면 컴파일러가 체크리스트를 대신 만들어준다.** 정의 한 곳을 바꾸고
   `cargo check`를 두 번 돌린 것이 전부다. 5곳이 정확히 열거됐다.
2. **★ 반대로 "정보를 버리는 코드"는 컴파일러가 못 잡는다.** `{ .. }`는 의도적 무시라
   경고 대상이 아니다. 이번 버그가 7개월간 살아남은 이유다.
3. **모든 수정이 벤치마크 대상은 아니다.** 재현 조건을 만들 수 없는 이유를 코드로 짚는 것이
   이번 건의 측정이었다.
4. **에러 코드는 유지하고 메시지만 바꿨다.** 클라이언트가 코드로 분기하는 계약을 안 깨는 선택.
