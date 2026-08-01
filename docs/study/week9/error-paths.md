# 에러 처리 경로 — 없음의 세 가지 표현과 unwind

> 9주차 화요일 — "실패했을 때 뭘 하는가".
> 어제 `block()`에서 실패 사유 넷 중 하나만 `Err`였던 이유를 여기서 끝까지 판다.

---

## 1. `ProviderError` 지형도

`crates/storage/errors/src/provider.rs:20` — 변형이 50개다. 하나씩 읽을 게 아니라 분류한다.

| 그룹 | 대표 변형 | 의미 |
|---|---|---|
| ① 하위 에러 감싸기 | `Database`, `Pruning`, `Rlp`, `Other` | 아래 계층 에러를 통과시킴 |
| ② 못 찾음 | `HeaderNotFound`, `TransactionNotFound`, `BlockHashNotFound`, `BestBlockNotFound` … 14개 | "있어야 하는데 없다" |
| ③ 정책상 없음 | `StateAtBlockPruned`, `BlockExpired`, `BlockNotExecuted` | **왜** 없는지를 담는다 |
| ④ static file 물리 문제 | `MissingStaticFileBlock`, `UnexpectedStaticFileTxNumber`, `CorruptedChangeSetStaticFile` | SF 특유 |
| ⑤ 정합성 깨짐 | `MustUnwind`, `StateRootMismatch`, `InsufficientChangesets`, `InvalidStorageOutput` | **노드 상태가 잘못됐다** |
| ⑥ 미지원 | `UnsupportedProvider`, `UnboundedStartUnsupported` | |

### ★ ②와 ③의 차이

둘 다 "없다"인데 필드 구성이 다르다.

```rust
// ② — 없다는 사실만
HeaderNotFound(BlockHashOrNumber),

// ③ — 왜 없는지 + 대신 뭐가 가능한지
BlockExpired {
    requested: BlockNumber,
    earliest_available: BlockNumber,   // "여기부터는 있다"
},
BlockNotExecuted {
    requested: BlockNumber,
    executed: BlockNumber,             // "여기까진 됐다"
},
```

②는 버그이거나 일시적 상태다. ③은 **정상 동작인데 요청이 범위 밖**이다.
그래서 ③만 "가능한 범위"를 필드로 같이 준다 — 호출자(RPC)가 사용자에게 대안을 알려줄 수 있게.

어제 `block()`에서 `BlockExpired`만 유일하게 `Err`였던 이유가 이것이다.
나머지 셋("해시로 번호 못 찾음", "헤더 없음", "body indices 없음")은 전부 ②의 성격인데,
`block()`은 그것들을 에러로 올리지 않고 `Ok(None)`으로 돌려준다 (→ §2).

### 🎓 `Box<RootMismatch>` — enum 크기

`provider.rs:59-61`에 러스트 관련 주석이 있다.

```rust
// NOTE: This is a Box only because otherwise this variant is 16 bytes larger than the
// second largest (which uses `BlockHashOrNumber`).
storage_key: Box<B256>,
```

**enum의 크기 = 가장 큰 변형의 크기.** 50개 중 하나가 크면 50개 전부가 그만큼 커진다.
`ProviderResult<T>`는 거의 모든 provider 함수의 반환 타입이라 그 크기가 곧 스택 복사 비용이다.

`Box<T>`로 감싸면 그 변형은 **포인터 8바이트**만 차지하고 실물은 힙으로 간다.
에러는 드물게 발생하므로 힙 할당을 감수하고 **평상시 크기를 줄이는** 트레이드오프.
`StateRootMismatch(Box<RootMismatch>)`도 같은 이유.

---

## 2. `block()`이 `Ok(None)`인 이유 — JSON-RPC 규격

이더리움 JSON-RPC 규격은 **없는 블록에 `null`을 돌려주라고** 정해놨다. 그래서 `eth_getBlockByNumber`
경로 전체가 `Option`을 유지한다.

| 단계 | 코드 | 값 |
|---|---|---|
| 1 | `provider.rs:1859` `block()` | `Ok(None)` |
| 2 | `rpc/rpc-eth-api/src/helpers/block.rs:63` `rpc_block()` | `Ok(None)` |
| 3 | `rpc/rpc-eth-api/src/core.rs:520` `eth_getBlockByNumber` | `RpcResult<Option<RpcBlock>>` |
| 4 | JSON 응답 | `{"result": null}` |

`block()`이 `Err`를 내면 4단계가 `{"error": ...}`가 되어 규격 위반이다. 그래서 `Err`를 낼 수 없다.

`history_by_block_hash`는 RPC 응답용이 아니라 내부에서 `StateProviderBox`를 만드는 함수라
반환 타입에 `Option`이 없고, 없으면 `.ok_or(BlockHashNotFound)`로 끝낸다
(`database/mod.rs:484`).

---

## 3. unwind이란 무엇인가

reth 특유 용어. **체인을 뒤로 되감는 것.**

`crates/stages/api/src/pipeline/mod.rs:157-170`

```rust
match target {
    PipelineTarget::Sync(tip) => self.set_tip(tip),
    PipelineTarget::Unwind(target) => {
        if let Err(err) = self.move_to_static_files() { ... }
        if let Err(err) = self.unwind(target, None) { ... }
        self.progress.update(target);
        return (self, Ok(ControlFlow::Continue { block_number: target }))
    }
}
```

파이프라인의 목표는 두 종류뿐이다 — **앞으로(`Sync`) 아니면 뒤로(`Unwind`).**

되감기가 필요한 상황은 둘이다.

1. **리오르그** — 따라가던 체인이 다수파가 아니었을 때 (정상 운영)
2. **정합성 깨짐** — 크래시로 저장소들이 서로 다른 지점에 멈췄을 때 (→ §4)

각 stage가 `execute`와 함께 `unwind`를 구현해야 하는 이유가 이것이다.
되감기는 예외 처리가 아니라 **파이프라인의 정규 동작 모드**다.

---

## 4. 정합성 검사 — 기동 시 한 번

`crates/storage/provider/src/providers/database/mod.rs:519-549`

```rust
pub fn check_consistency(&self) -> ProviderResult<(Option<u64>, Option<u64>)> {
    let provider_ro = self.database_provider_ro()?
        // Healing can run long-lived read transactions (e.g., iterating changesets
        // over millions of blocks). Disable the default timeout so MDBX doesn't
        // kill the transaction mid-heal, which causes a crash loop on startup.
        .disable_long_read_transaction_safety();

    // Step 1: heal file-level inconsistencies (no pruning)
    self.static_file_provider().check_file_consistency(&provider_ro)?;

    // Step 2: RocksDB consistency check (needs static files tx data)
    let rocksdb_unwind = self.rocksdb_provider().check_consistency(&provider_ro)?;

    // Step 3: Static file checkpoint consistency (may prune)
    let static_file_unwind = self.static_file_provider().check_consistency(&provider_ro)?...;

    // Step 4: Heal finalized/safe block numbers ...
    if rocksdb_unwind.is_none() && static_file_unwind.is_none() {
        self.heal_chain_state_block_numbers(&provider_ro)?;
    }

    Ok((rocksdb_unwind, static_file_unwind))
}
```

### ★ heal과 unwind는 다르다

| 용어 | 뜻 | 언제 |
|---|---|---|
| **heal** | 저장소 자체를 고친다 (SF 파일 끝을 잘라내기 등) | `check_consistency` 안에서 즉시 |
| **unwind** | MDBX를 그 지점까지 되감는다 | 파이프라인이 나중에 |

**SF는 heal, MDBX는 unwind.** 이 비대칭의 원인은 어제 §2의 결론 그대로다.

> SF는 append-only라 **끝을 잘라내는 것만** 가능하고, MDBX는 트랜잭션이 있어서
> **stage 로직으로 되감을 수 있다.**

`disable_long_read_transaction_safety()`도 눈여겨볼 것 — 평소에는 장시간 읽기 트랜잭션을
MDBX가 죽이지만, heal 중에는 그러면 **기동 크래시 루프**가 된다. 예외를 명시적으로 연다.

### 왜 `min`인가

`crates/node/builder/src/launch/common.rs:546-549`

```rust
let (rocksdb_unwind, static_file_unwind) = factory.check_consistency()?;

// Take the minimum block number to ensure all storage layers are consistent.
let unwind_target = [rocksdb_unwind, static_file_unwind].into_iter().flatten().min();
```

저장소 셋이 각기 다른 지점에 멈췄을 수 있으니 **가장 뒤처진 곳**까지 전부 맞춘다.

`max`였다면: RocksDB가 100까지, SF가 80까지 있을 때 100을 택하면 **80~100 구간은 SF에 없는데
있다고 믿는 상태**가 된다. `min`(80)은 RocksDB의 80~100을 버리지만 **남은 건 전부 진짜다.**

> 데이터를 버릴지언정 없는 걸 있다고 하지 않는다.

같은 논리가 세그먼트 레벨에서도 반복된다 (`static_file/manager.rs:1297`).

```rust
let mut unwind_target: Option<BlockNumber> = None;

let mut update_unwind_target = |new_target| {
    unwind_target = unwind_target.map(|current| current.min(new_target)).or(Some(new_target));
};
```

### 🎓 어제 배운 클로저 분류가 여기 나온다

`update_unwind_target`은 바깥 `unwind_target`을 **수정**한다 → `FnMut`.
그래서 `let mut`으로 선언해야 한다 (호출할 때마다 캡처한 것을 가변으로 빌리므로).

---

## 5. ★ `MustUnwind`는 왜 노드가 안 쓰나

같은 검사를 쓰는 API가 두 벌이다.

| API | 반환 | 위치 |
|---|---|---|
| `check_consistency()` | `(Option<u64>, Option<u64>)` — **값** | `database/mod.rs:519` |
| `assert_consistent()` / `new_checked()` | `Err(MustUnwind)` — **에러** | `database/mod.rs:495`, `:176` |

둘째는 첫째를 감싼 얇은 껍데기이고, **워크스페이스 안에 호출자가 0개다.**

```
$ grep -rn "new_checked" --include=*.rs .
crates/storage/provider/src/providers/database/mod.rs:176:    pub fn new_checked(
```

### 이유는 정보량이 아니다

`MustUnwind`도 `unwind_to`와 `data_source`를 들고 있어서 정보를 잃지 않는다.
진짜 이유는 **`Err`가 표현하는 것**에 있다.

> **`Err`는 "포기한다"는 신호인데, 노드는 포기하지 않고 고친다.**

`crates/node/builder/src/launch/common.rs:551-590`

```rust
if let Some(unwind_block) = unwind_target {
    assert_ne!(unwind_block, 0, "A {} inconsistency ...", inconsistency_source);

    let unwind_target = PipelineTarget::Unwind(unwind_block);
    info!(target: "reth::cli", %unwind_target, %inconsistency_source,
          "Executing unwind after consistency check.");

    let pipeline = PipelineBuilder::default()      // unwind 전용 파이프라인을 만들어서
        .add_stages(...)
        .build(...);                               // 되감고 나서 기동을 계속한다
```

**감지 → 복구 → 계속.** 이 흐름은 `Err`로 표현되지 않는다. `?`로 던지면 기동이 중단된다.

### 언제 생겼나 — `git log -S`로 확인

`new_checked` / `assert_consistent` / `MustUnwind` **셋 다 한 커밋에서 동시에 생겼다.**

```
543c77a374 refactor: spanning and misc improvements to consistency check code (#20961)
           2026-02-04
```

그 커밋이 한 일은 `common.rs`에 흩어져 있던 3단계 검사 로직을 `ProviderFactory::check_consistency()`
로 **옮긴 리팩터링**이다. 노드 경로는 그대로 튜플을 받게 두고, 에러 버전 래퍼를 같이 추가했다.
그리고 지금까지 아무도 안 쓴다.

즉 "외부 사용자를 위해 의도적으로 열어둔 API"라는 증거는 없다. 확인되는 건 둘뿐이다.

1. 리팩터링 중 편의 래퍼로 함께 추가됐고, 이후 배선된 적이 없다
2. **노드는 애초에 못 쓴다** — unwind 목표값을 받아 파이프라인을 만들어야 하는데
   `Err`로 오면 `?`에서 기동이 중단된다

`MustUnwind`가 담은 정보(`data_source`, `unwind_to`)만으로 받는 쪽이 할 수 있는 일이 없다는
것이 핵심이다. 고칠 수단이 없는 호출자에게는 "이 팩토리는 못 쓴다"는 신호로 쓸 수 있겠지만,
그런 호출자가 실재하는지는 확인되지 않았다.

### `assert_ne!`는 별개 장치다

`assert_ne!`는 러스트 표준 매크로다(두 값이 **같으면** 패닉). `MustUnwind`를 안 쓰는 이유와는
무관하고, **0까지 되감는 것을 막는** 별도의 안전장치다.

> Highly unlikely to happen, and given its destructive nature, it's better to panic instead.
> Unwinding to 0 would leave MDBX with a huge free list size.

0까지 되감으면 MDBX free list가 거대해진다. **고쳐서 계속하는 게 오히려 해로운 지점이 있고,
거기서는 죽는 쪽을 택한다.** 복구 정책에도 한계선이 있다는 뜻이다.

---

## 6. 오늘의 수확

1. **에러 변형 50개는 6그룹으로 나뉜다.** 특히 ②"못 찾음"과 ③"정책상 없음"의 차이 —
   ③만 `earliest_available` 같은 **가능 범위**를 필드로 동봉한다.
2. **`block()`이 `Ok(None)`인 건 JSON-RPC 규격 때문이다.** 없는 블록은 `null`이어야 하므로
   `Option`이 provider부터 JSON까지 4단계를 그대로 통과한다.
3. **unwind은 예외 처리가 아니라 파이프라인의 정규 모드다.** 목표는 `Sync` 아니면 `Unwind` 둘뿐.
4. **SF는 heal, MDBX는 unwind.** 어제 "SF엔 트랜잭션이 없다"의 직접적 귀결.
5. **정합성 목표는 `min`.** 데이터를 버릴지언정 없는 걸 있다고 하지 않는다.
6. **`MustUnwind`는 호출자가 0이다.** 리팩터링(#20961) 중 편의 래퍼로 추가됐고 배선된 적이 없다.
   노드는 unwind 목표값이 필요해서 `Err` 버전을 쓸 수 없다.
7. **복구 정책에도 한계선이 있다.** 0 unwind은 고치느니 죽는다.

## 7. 다음으로 넘길 질문

1. `check_file_consistency`(Step 1)와 `check_consistency`(Step 3)는 왜 나뉘어 있나?
   전자는 "pruning 없음", 후자는 "may prune"이라고 주석에 적혀 있는데, 그 경계가 무엇인가
2. RocksDB 검사가 **SF의 tx 데이터를 필요로 한다**(Step 2 주석). 어떤 대조를 하는 건가
3. `heal_chain_state_block_numbers`가 "<=1.10.2에서 온 노드"를 위한 것이라고 되어 있다.
   버전별 마이그레이션 코드가 정합성 검사 안에 들어와 있는데, 이런 게 얼마나 쌓여 있나
4. 리오르그 unwind와 정합성 unwind는 같은 `PipelineTarget::Unwind`를 쓴다. 둘의 진입 경로가
   다른데 stage 입장에서 구분이 필요 없는 이유는?
5. `MustUnwind`처럼 "워크스페이스 내 호출자가 없는 pub API"가 provider에 또 있나?
   (→ 수요일 인터페이스에서 소비자를 훑을 때 같이 확인)
