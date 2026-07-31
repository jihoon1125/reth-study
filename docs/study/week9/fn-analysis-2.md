# `block()` 라인 단위 추적 — 블록은 저장돼 있지 않다

> 9주차 월요일 — "블록 번호로 블록 전체를 어떻게 가져오는가".
> 지난주 `basic_account()`가 저장소 한 곳만 읽었다면, 이 함수는 인덱스와 데이터를 번갈아 왕복한다.

---

## 0. 전체 경로

```
DatabaseProvider::block(id)                    provider.rs:1845          진입점
 │
 ├─1 convert_hash_or_number(id)                block_id.rs:32            Hash면 → MDBX  HeaderNumbers   [인덱스]
 │                                                                       Number면 그대로 통과 (조회 없음)
 ├─2 earliest_history_height()                                           → Err(BlockExpired)
 │
 ├─3 header_by_number(number)                  provider.rs:1757          → static file                  [데이터]
 │
 ├─4 transactions_by_block(number)             provider.rs:2059
 │    ├─ block_body_indices(num)               provider.rs:2238          → MDBX  BlockBodyIndices       [인덱스]
 │    └─ transactions_by_tx_range(range)       provider.rs:2095          → static file                  [데이터]
 │
 └─5 storage.reader().read_block_bodies(..)    chain.rs:151              → MDBX  BlockWithdrawals / BlockOmmers
      → Self::Block::new(header, body)
```

**인덱스 → 데이터 → 인덱스 → 데이터**의 교대. 지난주 금요일에 "v2에서 MDBX는 위치 정보를
든다"고 정리한 것의 실물이 이 경로다.

저장소 왕복 횟수는 조회 방식에 따라 다르다.

| 조회 | MDBX | static file |
|---|---|---|
| `block_by_hash` | 3회 (HeaderNumbers, BlockBodyIndices, Withdrawals/Ommers) | 2회 |
| `block_by_number` | 2회 | 2회 |

`convert_hash_or_number`는 `Number`를 받으면 그대로 돌려준다 (`block_id.rs:33-36`).
즉 번호 조회는 MDBX 왕복이 한 번 적다.

---

## 1. 진입점 — `DatabaseProvider::block`

`crates/storage/provider/src/providers/database/provider.rs:1845`

```rust
fn block(&self, id: BlockHashOrNumber) -> ProviderResult<Option<Self::Block>> {
    if let Some(number) = self.convert_hash_or_number(id)? {
        let earliest_available = self.static_file_provider.earliest_history_height();
        if number < earliest_available {
            return Err(ProviderError::BlockExpired { requested: number, earliest_available })
        }

        let Some(header) = self.header_by_number(number)? else { return Ok(None) };

        // If the body indices are not found, this means that the transactions either do not
        // exist in the database yet, or they do exit but are not indexed.
        let Some(transactions) = self.transactions_by_block(number.into())? else {
            return Ok(None)
        };

        let body = self
            .storage
            .reader()
            .read_block_bodies(self, vec![(&header, transactions)])?
            .pop()
            .ok_or(ProviderError::InvalidStorageOutput)?;

        return Ok(Some(Self::Block::new(header, body)))
    }

    Ok(None)
}
```

### ★ `tables::Blocks`는 없다

블록은 **저장돼 있는 게 아니라 매번 조립된다.** 헤더 / 트랜잭션 / ommers / withdrawals가
각기 다른 매체에 흩어져 있고, 이 함수가 모아서 `Block::new`로 합친다.

---

## 2. ★ 인덱스와 데이터가 갈리는 지점

### MDBX가 든 위치 정보의 실물

`crates/storage/db-models/src/blocks.rs:17`

```rust
pub struct StoredBlockBodyIndices {
    /// The number of the first transaction in this block
    pub first_tx_num: TxNumber,
    /// The total number of transactions in the block
    pub tx_count: NumTransactions,
}
```

블록당 **u64 두 개**. `(시작 번호, 개수)`가 전부다.

```rust
// provider.rs:2238
fn block_body_indices(&self, num: u64) -> ProviderResult<Option<StoredBlockBodyIndices>> {
    Ok(self.tx.get::<tables::BlockBodyIndices>(num)?)          // MDBX
}

// provider.rs:2095
fn transactions_by_tx_range(&self, range: impl RangeBounds<TxNumber>) -> ... {
    self.static_file_provider.transactions_by_tx_range(range)  // static file
}
```

### 왜 트랜잭션을 블록별로 묶지 않았나

트랜잭션은 블록 소속이 아니라 **체인 전체 기준 일련번호(`TxNumber`)** 로 일렬 저장된다.
12번 블록의 tx가 3개면 `[147,148,149]`, 13번 블록은 150부터 이어진다.

static file은 인덱스 탐색이 없다. **"N번째 행"을 산술로 계산해서** 읽는다. 그러려면 번호가
빈틈없이 증가해야 하고, 블록 경계로 자르면 그 성질이 깨진다.

지난주 §5의 static file행 조건 3가지에 대입하면:

| 조건 | 트랜잭션 |
|---|---|
| 불변 | ⭕ 과거 블록의 tx는 안 바뀜 |
| 프루닝 불필요 | ⭕ |
| **순차 번호 접근** | ⭕ **`TxNumber` = 행 번호 → 오프셋 계산으로 직행** |

세 번째가 결정적이다. 그래서 **경계 정보만** 랜덤 조회가 되는 MDBX에 남겼다.

### ★ 조건표에 없던 네 번째 기준

여기서 표의 한계가 드러난다. `BlockBodyIndices` 자체도 SF 조건 3개를 **전부 만족한다** —
불변이고, 프루닝 대상이 아니고, 블록 번호 순이다. 그런데 v2에서도 MDBX에 남는다.

이유는 **원자성**이다.

```
tx_writer.append_transaction(...)   → static file : 트랜잭션 개념 없음, 그냥 파일 append
block_indices_cursor.append(...)    → MDBX        : 트랜잭션 안
                                        ↑
                          이게 커밋돼야 "그 tx는 존재한다"가 성립
```

static file에는 커밋이 없다. 원자적 기준점은 MDBX 하나뿐이다(수요일 concurrency 결론).
따라서 **"SF에 데이터가 들어갔는가"를 판정하는 정보가 SF 안에 있으면** 판정 근거와 판정
대상이 같은 매체가 되어 크래시 복구가 불가능해진다.

`provider.rs:3598` 주석이 이 방침을 명시한다.

> Clean up HeaderNumbers for blocks being removed, **we must clear all indexes from MDBX.**

> **조건표 추가**: SF 데이터를 가리키는 인덱스는 SF 조건을 만족하더라도 MDBX에 남는다.
> SF는 트랜잭션이 없어서, 크래시 시 "인덱스는 있는데 데이터가 없음 / 그 반대"를 판정할
> 기준이 MDBX밖에 없기 때문.

이게 `ProviderFactory::new`의 consistency check와 `ProviderError::MustUnwind`로 이어진다
(→ 화요일).

### 빈 블록의 `first_tx_num` — 설계가 아니라 부산물

doc 주석이 이상한 말을 한다.

> Note: If the block is empty, this is the number of the first transaction **in the next non-empty block**.

의도가 있어 보이지만, 쓰기 쪽을 보면 그냥 결과다.

`provider.rs:3558-3588` `append_block_bodies`

```rust
let mut next_tx_num = tx_block_cursor.last()?.map(|(id, _)| id + 1).unwrap_or_default();

for (block_number, body) in &bodies {
    let tx_count = body.as_ref().map(|b| b.transactions().len() as u64).unwrap_or_default();
    let block_indices = StoredBlockBodyIndices { first_tx_num: next_tx_num, tx_count };
    block_indices_cursor.append(*block_number, &block_indices)?;

    let Some(body) = body else { continue };
    if !body.transactions().is_empty() {
        tx_block_cursor.append(block_indices.last_tx_num(), block_number)?;
    }
    for transaction in body.transactions() {
        tx_writer.append_transaction(next_tx_num, transaction)?;
        next_tx_num += 1;                    // tx가 있을 때만 움직인다
    }
}
```

`first_tx_num`을 먼저 박고 tx마다 카운터를 올리므로, tx가 0개면 카운터가 그대로여서 다음
블록이 같은 값을 받는다. **주석은 의도가 아니라 관찰의 기록이다.**

읽기 쪽이 이 값에 의존하지 않는다는 증거 셋:

1. `transactions_by_block`은 `tx_range.is_empty()`면 `first_tx_num`을 보지도 않고 빈 벡터를
   돌려준다 (`provider.rs:2067`).
2. 재시작 시 카운터 복원은 `BlockBodyIndices`가 아니라 **`TransactionBlocks`의 마지막 키+1**
   이다 (`provider.rs:3558`). 그런데 `TransactionBlocks`는 빈 블록에 항목을 만들지 않는다
   (`provider.rs:3577`). 즉 복원조차 빈 블록의 `first_tx_num`을 안 본다.
3. 범위 조회에서 `first_tx_num`을 쓰는 자리는 빈 블록을 **미리 걸러내고** 쓴다
   (`provider.rs:2212`, `non_empty_blocks[0].first_tx_num()`). 의존이 아니라 회피다.

다만 빈 블록도 `next_tx_num()`(= `first_tx_num + 0`)이 올바른 다음 번호를 내놓아야 번호
사슬이 안 끊긴다. 이건 **쓰기 측 불변식**이지 읽기 측 계약이 아니다.

---

## 3. body 조립만 별도 트레이트인 이유

`block()`의 마지막 단계는 튄다. 헤더와 트랜잭션은 직접 읽어놓고, 조립만 `self.storage.reader()`
에 넘긴다.

`crates/storage/storage-api/src/chain.rs:57-73`

```rust
/// Trait that implements how block bodies are read from the storage.
///
/// Note: Within the current abstraction, transactions persistence is handled separately, thus this
/// trait is provided with transactions read beforehand and is expected to construct the block body
/// from those transactions and additional data read from elsewhere.
pub trait BlockBodyReader<Provider> {
    type Block: Block;

    fn read_block_bodies(
        &self,
        provider: &Provider,
        inputs: Vec<ReadBodyInput<'_, Self::Block>>,
    ) -> ProviderResult<Vec<<Self::Block as Block>::Body>>;
}
```

doc 주석이 분리 이유를 직접 말한다: **트랜잭션 저장은 따로 처리되므로, 이 트레이트는 이미
읽어둔 트랜잭션을 받아 나머지를 다른 데서 읽어 body를 조립한다.**

### 왜 하필 body만인가

`chain.rs:151` — 이더리움 구현

```rust
for (header, transactions) in inputs {
    let withdrawals = if chain_spec.is_shanghai_active_at_timestamp(header.timestamp()) {
        withdrawals_cursor.seek_exact(header.number())?.map(|(_, w)| w.withdrawals)
            .unwrap_or_default().into()
    } else {
        None
    };
    let ommers = if chain_spec.is_paris_active_at_block(header.number()) {
        Vec::new()
    } else {
        // Pre-merge: fetch ommers from database using direct database access
        provider.tx_ref().cursor_read::<tables::BlockOmmers<H>>()?
            .seek_exact(header.number())?.map(|(_, o)| o.ommers).unwrap_or_default()
    };
    bodies.push(alloy_consensus::BlockBody { transactions, ommers, withdrawals });
}
```

헤더와 트랜잭션은 **어느 체인이든 형태가 같다.** body에 무엇이 더 붙는지는 체인마다 다르다.
그래서 바로 아래에 대체 구현이 있다.

`chain.rs:193-199`

```rust
/// A noop storage for chains that don't have custom body storage.
pub struct EmptyBodyStorage<T, H>(PhantomData<(T, H)>);
```

즉 `block()`의 5단계 중 **4단계는 공통, 1단계만 체인별 교체점.**
지난주 §4의 "바뀌는 것만 뒤로 뺀다"가 여기서 다시 나타난다.

부수 효과로 **하드포크 조건이 저장 계층에 들어와 있다.** `is_shanghai_active_at_timestamp` /
`is_paris_active_at_block`이 provider 안에서 호출된다. 조립에 필요한 필드가 하드포크마다
다르니 피할 수 없는 결합이지만, `read_block_bodies` 안으로 **가둬** 놓은 형태다.

### 🎓 `PhantomData` — 값은 없는데 타입은 있다

`chain.rs:87`

```rust
pub struct EthStorage<T = TransactionSigned, H = Header>(PhantomData<(T, H)>);
```

`EthStorage`는 필드가 없다. 상태를 아무것도 안 든다. 그런데 제네릭 `T`, `H`는 필요하다 —
"어떤 트랜잭션 타입, 어떤 헤더 타입을 다루는 저장소인가"를 타입으로 구분해야 하니까.

러스트는 **선언하고 안 쓰는 제네릭 파라미터를 컴파일 에러로 막는다.**

```rust
pub struct EthStorage<T, H>;   // ❌ error: parameter `T` is never used
```

`PhantomData<(T, H)>`는 "이 타입을 쓰긴 쓴다(실체는 없지만)"는 표식이다. **런타임 크기 0.**
순수하게 타입 검사용.

지난주 "테이블 = 타입"과 같은 계열이다. 값이 아니라 타입으로 구분하면 잘못된 조합이
컴파일에서 걸린다.

---

## 4. 구현체 4곳 — 인메모리가 먼저

| 파일:줄 | 하는 일 |
|---|---|
| `blockchain_provider.rs:522` | `consistent_provider()?.block(id)` — 위임만 |
| `consistent.rs:703` | 인메모리 먼저, 없으면 DB |
| `database/provider.rs:1845` | 실제 조립 (§1~3) |
| `static_file/manager.rs:2906` | `Ok(None)` — SF 단독으로는 블록을 못 만듦 |

마지막 줄이 §2를 확인해준다. SF에는 헤더도 트랜잭션도 있지만 **둘을 잇는 인덱스가 MDBX에
있으므로** SF 혼자서는 블록을 조립할 수 없다.

`consistent.rs:703`

```rust
fn block(&self, id: BlockHashOrNumber) -> ProviderResult<Option<Self::Block>> {
    self.get_in_memory_or_storage_by_block(
        id,
        |db_provider| db_provider.block(id),
        |block_state| Ok(Some(block_state.block_ref().recovered_block().clone_block())),
    )
}
```

`consistent.rs:428`

```rust
pub(crate) fn get_in_memory_or_storage_by_block<S, M, R>(
    &self,
    id: BlockHashOrNumber,
    fetch_from_db: S,
    fetch_from_block_state: M,
) -> ProviderResult<R>
where
    S: FnOnce(&DatabaseProviderRO<N::DB, N>) -> ProviderResult<R>,
    M: Fn(&BlockState<N::Primitives>) -> ProviderResult<R>,
{
    if let Some(Some(block_state)) = self.head_block.as_ref().map(|b| b.block_on_chain(id)) {
        return fetch_from_block_state(block_state)
    }
    fetch_from_db(&self.storage_provider)
}
```

화요일 `basic_account`에서 본 구조와 **정확히 같다.** 최신 블록은 아직 디스크에 안 내려갔을 수
있으니 인메모리를 먼저 본다. 여기서는 그 분기를 함수 하나로 뽑고, "인메모리에서 꺼내는 법"과
"DB에서 꺼내는 법"을 클로저로 받는다.

> 화요일 §7-2의 답: 다른 구현체들은 **다른 알고리즘이 아니라 다른 소스 순서**다.

### 🎓 `FnOnce` vs `Fn` — 바운드가 다른 이유

| 트레이트 | 호출 | 캡처한 값을 |
|---|---|---|
| `FnOnce` | 한 번만 | 소유권을 가져가 소비 가능 |
| `FnMut` | 여러 번, 가변 | 빌려서 수정 |
| `Fn` | 여러 번, 불변 | 빌려서 읽기만 |

포함 관계는 `Fn` ⊂ `FnMut` ⊂ `FnOnce`. 따라서 **`FnOnce`가 더 느슨한 요구**(아무 클로저나
받음), **`Fn`이 더 빡센 요구**다.

이 함수는 둘 다 한 번씩만 호출하는데 바운드가 다르다. 이유는 **형제 함수**에 있다.

`consistent.rs:143-157`

```rust
fn get_in_memory_or_storage_by_block_range_while<T, F, G, P>(...)
where
    F: FnOnce(&DatabaseProviderRO<N::DB, N>, RangeInclusive<BlockNumber>, &mut P) -> ...,
    G: Fn(&BlockState<N::Primitives>, &mut P) -> Option<T>,
```

`consistent.rs:231-239`

```rust
for (num, block) in in_memory_range.zip(in_memory_chain.into_iter().rev()) {
    if let Some(item) = map_block_state_item(block, &mut predicate) {   // 블록마다 호출
```

비대칭의 구조적 이유:

- **DB 쪽(`F`)** — 범위를 통째로 넘겨 **한 번** 호출. 커서 하나로 스캔하는 게 효율적이니까
- **인메모리 쪽(`G`)** — 블록별로 **N번** 호출. 인메모리 체인은 블록 단위 링크 구조라 범위 API가 없음

단일 블록 버전은 이 형제와 **모양을 맞춘** 것이다. 실제 호출부의 `M` 자리 클로저는 전부
빌려 읽기만 해서 `Fn`이 자동 만족되므로 걸리는 데가 없다.

> **필요해서가 아니라 의도를 문서화한 바운드.** "이 자리는 여러 번 불릴 수 있는 자리다"를
> 타입으로 적어둔 것.

---

## 5. 실패 사유 넷 중 에러는 하나

| 상황 | 결과 | 줄 |
|---|---|---|
| 해시로 번호를 못 찾음 | `Ok(None)` | 1845 |
| **히스토리 범위보다 오래된 블록** | **`Err(BlockExpired)`** | 1849 |
| 헤더 없음 | `Ok(None)` | 1852 |
| body indices 없음 | `Ok(None)` | 1858 |

주석(`provider.rs:1854-1857`)이 뭉개는 이유를 말한다.

> If they exist but are not indexed, we don't have enough information to return the block anyways,
> so we return `None`.

"아직 없음"과 "있는데 인덱스가 없음"을 구분하지 않는다. **구분해도 호출자가 할 수 있는 게
같기 때문.**

`BlockExpired`만 에러인 이유는 다르다. 이건 "없다"가 아니라 **"영원히 없을 것이고, 그 원인이
노드 설정이다"** 이다. 호출자(RPC)가 사용자에게 다른 메시지를 줘야 한다.

→ 화요일 에러 경로의 첫 재료.

---

## 6. 러스트 노트

### let-else

```rust
let Some(header) = self.header_by_number(number)? else { return Ok(None) };
```

벗겨지면 계속, 아니면 나간다. `else` 블록은 반드시 발산해야 한다(`return`/`break`/`panic!`).
성공 경로를 들여쓰기 없이 평평하게 유지하는 장치.

### let-chain

`provider.rs:2063`

```rust
if let Some(block_number) = self.convert_hash_or_number(id)? &&
    let Some(body) = self.block_body_indices(block_number)?
{
```

`if let`을 `&&`로 잇는 문법(Rust 2024 안정화). 일반 `&&`와 달리 **왼쪽에서 벗겨낸
`block_number`를 오른쪽에서 바로 쓸 수 있다** — 순서 의존이 있다.
예전에는 중첩 `if let`으로 써야 했다.

---

## 7. 오늘의 수확

1. **블록은 저장돼 있지 않다.** `tables::Blocks`가 없고, 조회마다 4개 매체에서 모아 조립한다.
2. **트랜잭션은 블록이 아니라 체인 전체 기준으로 번호가 매겨진다.** `TxNumber` = static file
   행 번호. 블록 경계로 자르면 오프셋 계산이라는 SF의 장점이 사라진다.
3. **★ SF 데이터를 가리키는 인덱스는 SF 조건을 만족해도 MDBX에 남는다.** SF에 트랜잭션이
   없어서, 크래시 시 정합성을 판정할 기준이 MDBX뿐이기 때문. 지난주 조건표에 빠졌던 기준.
4. **체인별 차이는 body 조립 한 단계에만 있다.** 5단계 중 4단계는 공통.
5. **클로저 바운드가 설계 의도를 문서화한다.** `FnOnce`/`Fn`의 비대칭은 형제 함수의 호출
   횟수를 반영한 것.
6. **doc 주석이 항상 의도는 아니다.** 빈 블록 `first_tx_num` 주석은 쓰기 루프의 부산물을
   기록한 것이고, 읽기 측은 이 값에 의존하지 않는다.

## 8. 다음으로 넘길 질문

1. `ProviderError::BlockExpired`와 `MustUnwind`는 둘 다 "SF와 MDBX의 관계"에서 나온다.
   전자는 정상 운영 중의 경계이고 후자는 비정상 상태인데, 판정 시점이 어떻게 다른가? (→ 화)
2. `read_block_bodies`가 `Vec`를 받고 `Vec`를 돌려주는 배치 API인데 `block()`은 원소 하나로
   호출하고 `.pop()`한다. 배치를 쓰는 진짜 호출자는 누구인가? (`block_range` 계열로 추정)
3. `.pop()`이 `None`이면 `InvalidStorageOutput`이다. 입력 1개에 출력 0개가 나오는 구현이
   실제로 가능한가, 아니면 트레이트가 개수 보장을 못 하는 데서 오는 방어 코드인가?
4. `EthStorage` 말고 실제로 `EmptyBodyStorage`를 쓰는 체인이 워크스페이스에 있는가?
   (없다면 이 추상화는 아직 "예비" 상태)
5. `block()` 경로에 RocksDB가 없다. v2에서 RocksDB로 가는 건 tx **해시** 조회인데,
   `transaction_by_hash`는 어느 매체를 몇 번 왕복하나? (→ 수요일 인터페이스와 함께)
