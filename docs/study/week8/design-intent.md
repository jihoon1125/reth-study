# 설계 의도 — 왜 이렇게 나눴나

> 8주차 금요일. 이번 주에 쌓인 가설을 **공식 문서와 대조**하는 날.
> 코드가 아니라 문서를 읽고, 문서가 답하지 않는 건 관찰로 채운다.

---

## 0. 문서 신뢰도부터 (중요)

읽기 전에 **각 문서가 언제 것인지** 확인해야 한다. reth는 지금 v1→v2 전환 중이라 문서마다 세대가 다르다.

| 문서 | 최종 수정 | 신뢰 범위 |
|---|---|---|
| `docs/vocs/docs/pages/run/storage.mdx` | 2026-05-19 | ✅ 현재 라우팅의 **정본** |
| `docs/design/database.md` | 2026-01-19 (오타 수정) | ⚠️ **§Abstractions만** 유효 |
| `docs/crates/db.md` | 2026-01-22 | ⚠️ 반년 됨. 세부는 코드로 확인 |

### ⚠️ `database.md`의 §Table layout은 v1 유물이다

`database.md:24-146`의 ER 다이어그램은 **storage v2가 나오기 전 세계**다. 목요일 결과와 정면으로 충돌한다.

| `database.md`의 다이어그램 | 실제 (목요일 확인) |
|---|---|
| `AccountsHistory { B256 Account "PK" }` | Key는 `ShardedKey<Address>`, v2에선 **RocksDB** |
| `PlainAccountState`가 정본 상태 | v2는 `HashedAccounts` |
| static file 언급 **0회** | 헤더/트랜잭션/receipts/senders/changesets가 전부 여기 |
| RocksDB 언급 **0회** | 인덱스 3개가 여기 |
| *"needs benchmark once `reth` is closer to prod"* | 1.0 이전 문장 |

**§Table layout을 현재 지도로 쓰면 안 된다.** v1의 유산(`PlainAccountState`/`HashedAccounts` 이중 구조)이 왜 있는지 이해하는 데는 쓸모 있다.

---

## 1. 왜 계층을 나눴나 — 공식 답

`docs/design/database.md:5-6` (원문 그대로):

> We created a **Database trait abstraction** using Rust Stable GATs which **frees us from being bound to a single database implementation**. We currently use MDBX, but **are exploring** redb as an alternative.
>
> We then iterated on `Transaction` as a **non-leaky abstraction** with helpers for strictly-typed and unit-tested higher-level database abstractions.

두 표현을 정확히 읽을 것:

- **"frees us from being bound to a single database implementation"** — 트레이트/구현 분리가 존재하는 이유
- **"non-leaky abstraction"** — 하위 층 세부가 상위로 새지 않는다. 실제로 `DatabaseProvider`는 "MDBX"라는 단어를 몰라도 된다

### ⚠️ redb는 "탐색 중"이지 "전환 준비 중"이 아니다

원문이 `are exploring ... as an alternative`이고, 리포에 redb 구현체는 **없다.** 문서가 조심스럽게 쓴 걸 확정으로 옮기면 안 된다.

### 추상화가 빈 껍데기가 아니라는 진짜 증거

redb는 아직 없으니 증거로 약하다. 더 강한 건 **이미 컴파일되고 있는 두 번째 구현체**다.

```rust
// db-api/src/mock.rs:80, 150
impl DbTx for TxMock { ... }
impl DbTxMut for TxMock { ... }
```

②층(`db-api`↔`db`)에서는 `TxMock`이, ①층(`storage-api`↔구현체)에서는 `rpc-provider`가 같은 역할을 한다.

---

## 2. ★ GAT — 언어 기능 하나가 아키텍처를 가능하게 한 사례

문서가 굳이 *"using Rust **Stable** GATs"* 라고 언어 기능 이름을 박아넣은 이유.

```rust
// db-api/src/transaction.rs:23
pub trait DbTx: Debug + Send {
    type Cursor<T: Table>: DbCursorRO<T> + Send;    // ← GAT
    fn cursor_read<T: Table>(&self) -> Result<Self::Cursor<T>, DatabaseError>;
}

// db/src/implementation/mdbx/tx.rs:286
impl<K: TransactionKind> DbTx for Tx<K> {
    type Cursor<T: Table> = Cursor<K, T>;            // ← 한 줄로 29개 테이블 전부
}
```

**GAT = 자기 제네릭 파라미터를 가진 연관 타입.** 지금까지 본 `type TX;`는 빈칸 하나였지만, GAT는 빈칸이 함수처럼 인자를 받는다.

```
type TX;                → "타입 구멍 하나"
type Cursor<T: Table>;  → "테이블 T를 주면 그에 맞는 커서 타입을 내놓는 구멍"
```

Rust 1.65(2022-11)에 안정화됐다. **그 전에는 이 아키텍처가 표현 불가능했다.**

### GAT가 없다면 — 대안 3개, 전부 못 쓴다

재료: **테이블 29개**, **커서 호출 지점 280곳**.

#### 옵션 A — 트레이트 오브젝트

```rust
fn cursor_read<T: Table>(&self) -> Result<Box<dyn DbCursorRO<T> + '_>, DatabaseError>;
```

컴파일은 되지만 대가가 셋:

1. **★ `walk`/`walk_range`/`walk_back`을 잃는다.** `cursor.rs:39-60`을 보면 전부 `where Self: Sized`가 붙어 있다 → 트레이트 오브젝트에서 호출 불가. (`walk_range`는 `impl RangeBounds`를 받아 애초에 오브젝트 안전하지 않다.) 범위 스캔 API가 통째로 사라진다.
2. 커서 열 때마다 힙 할당
3. **매 `next()`가 vtable 간접 호출** — state root 계산은 `HashedAccounts`를 커서로 수억 번 `next()` 하는 일이다. 인라이닝이 막힌다.

#### 옵션 B — 트레이트에 테이블 타입을 올리기 (가장 그럴듯)

```rust
pub trait DbTxCursor<T: Table> {
    type Cursor: DbCursorRO<T> + Send;      // 파라미터 없는 평범한 연관 타입
    fn cursor_read(&self) -> Result<Self::Cursor, DatabaseError>;
}

impl<K: TransactionKind, T: Table> DbTxCursor<T> for Tx<K> {   // blanket impl 하나
    type Cursor = Cursor<K, T>;
}
```

**구현 쪽은 오히려 깔끔하다.** 성능 손실 0, 타입 안전성 유지. 문제는 세 군데의 난이도가 다르다는 것:

| | 옵션 B에서 |
|---|---|
| impl 쪽 | ✅ blanket impl 하나 |
| 타입 지칭 | ⚠️ `<TX as DbTxCursor<T>>::Cursor` — 장황하지만 성립 |
| **바운드 쪽** | ❌ **여기서 죽는다** |

```rust
impl<TX, N> DatabaseProvider<TX, N>
where
    TX: DbTx + DbTxMut
      + DbTxCursor<tables::TransactionSenders>
      + DbTxCursor<tables::BlockBodyIndices>
      + DbTxCursor<tables::HashedAccounts>
      + ...                                    // 만지는 테이블 전부 나열
```

그리고 **전염된다** — `BlockchainProvider`, `ConsistentProvider`, RPC 레이어까지 목록을 다시 적어야 한다.

줄이고 싶어서 이렇게 쓰면:

```rust
impl<TX: for<T: Table> DbTxCursor<T>, N> ...   // ❌ 문법 자체가 없음
```

**러스트의 HRTB(`for<...>`)는 라이프타임 전용이다.** `for<'a>`는 되지만 `for<T>`는 안 된다.

> **GAT가 메운 구멍이 정확히 이것이다** — "모든 T에 대해 커서 타입이 존재한다"를 트레이트 하나에 담는 것.

#### 옵션 C — 타입 소거

```rust
pub trait DbTx { type Cursor: RawCursor; }        // 테이블 무관
pub trait RawCursor {
    fn seek(&mut self, key: &[u8]) -> Result<Option<(Vec<u8>, Vec<u8>)>, DatabaseError>;
}
```

화요일에 배운 걸 통째로 잃는다:

```rust
tx.cursor_read::<PlainAccountState>()?.seek(some_b256)?;   // 현재: ❌ 컴파일 에러
cursor.seek(some_b256.as_slice())?;                        // 옵션 C: ✅ 통과, 런타임에 터짐
```

#### 정리

| | 타입 안전성 | 성능 | 실사용 |
|---|---|---|---|
| A 트레이트 오브젝트 | 유지 | ❌ `walk` 상실 + vtable + 힙 | 불가 |
| B `DbTxCursor<T>` | 유지 | ✅ 동일 | 바운드 지옥 — 사실상 불가 |
| C 타입 소거 | ❌ 상실 | 보통 | 가능하나 설계 후퇴 |
| **GAT (현재)** | 유지 | 유지 | ✅ |

---

## 3. `storage-api` / `provider`가 또 나뉜 이유 (문서에 없음 — 직접 추론)

질문을 바꾸면 답이 나온다: **"합쳐놨으면 뭐가 나빴을까?"**

크레이트를 나누는 이유는 거의 항상 **의존성**이다.

```toml
# storage-api/Cargo.toml — 트레이트만
reth-db-api = { workspace = true, optional = true }    # ← optional! DB 없이 컴파일됨

# provider/Cargo.toml — 구현
reth-db = { workspace = true, features = ["mdbx"] }    # MDBX (C)
rocksdb.workspace = true                               # RocksDB (C++)
reth-nippy-jar.workspace = true                        # static file
```

합쳐놨다면 **`AccountReader` 하나만 쓰고 싶은 크레이트도 MDBX·RocksDB 네이티브 라이브러리를 전부 빌드**해야 한다. 그런 크레이트가 **28개**다.

### 결정적 증거 — `rpc-provider`

`crates/storage/rpc-provider/src/lib.rs:1-8`:

> This crate provides an RPC-based implementation of reth's `StateProviderFactory` and related traits that fetches blockchain data via RPC **instead of from a local database**.
> - Useful for testing **without requiring a full database**

**데이터베이스가 아예 없는 provider다.** 네트워크로 원격 노드에 묻는다. `AccountReader`가 `provider` 크레이트 안에 있었다면 이걸 만들려고 MDBX를 링크해야 한다.

### `AccountReader` 구현체 12개 중 DB를 읽는 건 1개

| 구현체 | 실제 출처 |
|---|---|
| `DatabaseProvider` | **MDBX** ← 화요일에 본 것 |
| `HistoricalStateProviderRef` | MDBX + RocksDB (과거 시점) |
| `rpc-provider` | 네트워크 (원격 RPC) |
| `MemoryOverlayStateProviderRef` | 메모리 (아직 커밋 안 된 블록) |
| `CachedStateProvider` | 메모리 캐시 (래퍼) |
| `MockEthProvider`, `StateProviderTest` | 테스트용 `HashMap` |
| `DatabaseStateProvider` (revm) | 실행 엔진 어댑터 |

그리고 트레이트 자체가 극도로 가볍다 (`storage-api/src/account.rs:14`):

```rust
#[auto_impl(&, Arc, Box)]
pub trait AccountReader {                    // 슈퍼트레이트 0개. Send조차 없음
    fn basic_account(&self, address: &Address) -> ProviderResult<Option<Account>>;
}
```

> **"갈아끼우기"보다 "DB가 아닌 구현체가 애초에 가능해진다"가 핵심.**
> `rpc-provider`는 MDBX의 대체품이 아니라 **아예 다른 종류**다.

---

## 4. ★ 같은 원칙이 세 층에서 반복된다

```
╔══════════════════════════════════════════════════════════════════════════╗
║  "트레이트는 위, 구현은 아래"                                              ║
╚══════════════════════════════════════════════════════════════════════════╝

┌──────────────────────────────────────────────────────────────────────────┐
│ ① crates/storage/storage-api  ←→  구현체들                                │
│    질문: "계정 정보를 어디서 가져오나?"                                     │
├──────────────────────────────────────────────────────────────────────────┤
│   storage-api/src/account.rs:14        ┌─ reth-db-api = optional          │
│   ┌──────────────────────────────┐     │  → DB 없이도 컴파일됨            │
│   │ pub trait AccountReader {    │ ────┤                                  │
│   │   fn basic_account(&self,    │     └─ 의존 크레이트 28개              │
│   │     &Address)                │                                        │
│   │   -> Option<Account>         │        슈퍼트레이트 0개                 │
│   │ }                            │                                        │
│   └──────────────┬───────────────┘                                        │
│                  │  impl 12곳                                             │
│   ┌────────┬─────┴────┬──────────┬───────────┬──────────┐                │
│   ▼        ▼          ▼          ▼           ▼          ▼                │
│ Database  rpc-    Historical  Memory-    Cached-    MockEth              │
│ Provider  provider  State     Overlay     State     Provider            │
│   MDBX    원격RPC   과거시점   미커밋블록   캐시래퍼   테스트               │
│    │                                                                      │
│    └── ★ 실제로 DB를 읽는 건 이것뿐                                        │
│                                                                           │
│   ▸ 나눈 이유: 데이터 출처가 DB가 아닌 구현체가 존재한다.                    │
│     합쳤다면 28개 크레이트가 전부 MDBX·RocksDB를 링크해야 했다.             │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │  DatabaseProvider<TX, N> { tx: TX, ... }
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ ② crates/storage/db-api  ←→  crates/storage/db                            │
│    질문: "어떤 DB 엔진을 쓰나?"                                            │
├──────────────────────────────────────────────────────────────────────────┤
│   db-api (트레이트 + 테이블 정의)          db (MDBX 구현)                  │
│   ┌────────────────────────────┐          ┌──────────────────────────┐    │
│   │ trait Database             │  ──────▶ │ DatabaseEnv              │    │
│   │   type TX   : DbTx+Send+Sync│         │   type TX    = Tx<RO>    │    │
│   │   type TXMut: DbTxMut+DbTx  │         │   type TXMut = Tx<RW>    │    │
│   │                            │          │                          │    │
│   │ trait DbTx    (읽기 능력)   │  ──────▶ │ impl<K> DbTx for Tx<K>   │    │
│   │ trait DbTxMut (쓰기 능력)   │  ──────▶ │ impl DbTxMut for Tx<RW>  │    │
│   │ trait Table                │          │ Cursor<K, T>             │    │
│   │   NAME / Key / Value       │          └──────────────────────────┘    │
│   │ Encode/Decode  (Key)       │          ┌──────────────────────────┐    │
│   │ Compress/Decompress (Value)│  ┈┈┈┈┈▶ │ TxMock  (db-api/mock.rs) │    │
│   └────────────────────────────┘          │ redb    (문서상 탐색 중)  │    │
│                                           └──────────────────────────┘    │
│   ▸ 나눈 이유: database.md:5                                              │
│     "frees us from being bound to a single database implementation"       │
│   ▸ 가능하게 한 언어 기능: GAT — Rust 1.65 (2022-11)                       │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │  TX = Tx<RW>  (제네릭 → 구체 치환)
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ ③ DbTx 트레이트  ←→  Tx<K>  ←→  libmdbx C                                 │
│    질문: "실제 디스크는 누가 만지나?"                                       │
├──────────────────────────────────────────────────────────────────────────┤
│   Tx<K>                    db/implementation/mdbx/tx.rs:31   [러스트 어휘] │
│    │  inner ────────────────────────────────────────────┐        tx       │
│    │  dbis: Arc<HashMap<&str, MDBX_dbi>>  ← 이름→핸들 캐시│                │
│    ▼                                                     ▼                │
│   Transaction<K>           libmdbx-rs/transaction.rs:61                   │
│    └ Arc<TransactionInner<K>>                                             │
│         └ txn: TransactionPtr        :542                                 │
│              ├ *mut ffi::MDBX_txn   ← 진짜 핸들                            │
│              └ lock: Arc<Mutex<()>> ← ★ 모든 연산이 통과. 트랜잭션당 1개    │
│                     unsafe impl Send/Sync for TransactionPtr              │
│                     "SAFETY: synchronized by the lock"                    │
│  ─────────────────────────────────────────────────────────  [C 어휘] txn  │
│   ffi::mdbx_get / mdbx_put / mdbx_txn_begin_ex                            │
│                                                                           │
│   ▸ 이름이 tx → txn 으로 바뀌는 지점 = 러스트/C 경계                        │
│   ▸ 타입 안전성이 끝나는 지점:                                             │
│       PlainAccountState (타입) → T::NAME (문자열) → MDBX_dbi (u32)         │
└──────────────────────────────────────────────────────────────────────────┘
```

### 층마다 "왜 나눴나"가 다르다

| 층 | 경계 | 갈아끼우는 대상 | 근거 |
|---|---|---|---|
| ① | `storage-api` / 구현체 | **데이터 출처** (DB·네트워크·메모리·목) | `rpc-provider`가 실존 |
| ② | `db-api` / `db` | **DB 엔진** (MDBX ↔ redb) | `database.md:5` 명시 |
| ③ | `DbTx` / `Tx<K>` | **능력** (읽기 / 쓰기) | `Tx<RO>`는 `DbTxMut` 미구현 |

③은 크레이트 경계가 아니라 **같은 크레이트 안의 트레이트/구현 분리**다.

### `basic_account` 한 번의 여정

```
AccountReader::basic_account(&address)          storage-api/account.rs:18   [트레이트]
   └─ DatabaseProvider의 impl                   provider.rs:1462            [구현 선택]
        └─ self.tx.get_by_encoded_key::<T>(..)  TX = Tx<RW> or Tx<RO>
             └─ DbTx 트레이트 메서드             db-api/transaction.rs:32    [트레이트]
                  └─ Tx::get_by_encoded_key     db/mdbx/tx.rs:293           [구현]
                       └─ get_dbi::<T>()→T::NAME tx.rs:90                   [타입→문자열]
                       └─ Transaction::get      libmdbx-rs/transaction.rs:153
                            └─ txn_execute→lock() :598                      [뮤텍스]
                                 └─ ffi::mdbx_get                           [C]
```

**트레이트 → 구현 → 트레이트 → 구현**이 번갈아 나온다. 각 전환이 위 그림의 층 하나.

---

## 5. 왜 저장소가 셋인가 — hot/cold

`docs/vocs/docs/pages/run/storage.mdx:1-10`:

> description: Default **hot/cold** storage architecture in Reth.
>
> Reth uses the V2 hot/cold storage layout by default for new databases. This layout **routes history indices and transaction hash lookups to RocksDB, writes account and storage changesets to static files, and avoids the legacy plain state tables**.

목요일에 `either_writer.rs`를 뒤져서 만든 표가 **여기 한 문장으로 있다.**

### 설계 판단의 근거는 측정치다

`storage.mdx:27-33` — *"Measured on disk at block 24,396,823 on Ethereum mainnet"*:

| Node Type | Legacy V1 | V2 | Savings |
|---|---|---|---|
| Full | 1.46 TB | 1.02 TB | **-30%** |
| Minimal | 449 GB | 224 GB | **-50%** |
| Archive | 2.99 TB | 2.31 TB | **-23%** |

"이론상 좋을 것 같아서"가 아니라 **"재보니 30% 줄었다"**. 설계 의도를 추론할 때 **측정 가능한 목표가 있었는지**가 중요한 단서다.

### ⚠️ hot/cold의 구체적 대응은 공식 정의가 없다

문서는 이름만 쓰고 **어느 저장소가 hot이고 cold인지 정의하지 않는다.** 아래는 관찰에 근거한 대응이다.

| 목요일 분류 | hot/cold | 저장소 |
|---|---|---|
| 자주 바뀌는 현재 상태 + 트라이 | hot | MDBX |
| 안 바뀌는 과거 데이터 | cold | static file |
| 랜덤 키 대량 인덱스 | **분류 불가** | RocksDB |

**RocksDB가 이 축에 안 들어간다:**

| | 쓰기 | 읽기 |
|---|---|---|
| `AccountsHistory` / `StoragesHistory` | 매 블록 수천 건 (**hot**) | 과거 조회 시에만 (**cold**) |

> hot/cold는 v2 레이아웃의 **별칭**이고, 목요일에 만든 3분류가 더 세밀하다.

### ⚠️ 목요일 표 정정 — "전부 인덱스"는 구분 기준이 아니다

`data-flow.md:219`에 *"공통점: 전부 인덱스다"* 라고 적었는데, **MDBX에도 인덱스가 많다:**

```rust
self.tx.put::<tables::HeaderNumbers>(block.hash(), block_number)?;           // provider.rs:808
self.tx.cursor_write::<tables::TransactionBlocks>()?.append(...)?;           // provider.rs:836
```

진짜 경계선은 **키의 성격 × 쓰기 볼륨**이다.

| 테이블 | 저장소 | 키 | 키 순서 | 블록당 쓰기 | 방식 |
|---|---|---|---|---|---|
| `BlockBodyIndices` | MDBX | `BlockNumber` | **단조 증가** | **1건** | `append` |
| `TransactionBlocks` | MDBX | `TxNumber` | **단조 증가** | **1건** | `append` |
| `HeaderNumbers` | MDBX | `BlockHash` | 랜덤 | **1건** | `put` |
| `TransactionHashNumbers` | **RocksDB** | `TxHash` | 랜덤 | **수백 건** | put |
| `AccountsHistory` | **RocksDB** | `Address` | 랜덤 | **수천 건** | **read-modify-write** |
| `StoragesHistory` | **RocksDB** | `(Addr, Slot)` | 랜덤 | **수천 건** | **read-modify-write** |

근거 넷:

1. **MDBX 인덱스는 블록당 1건뿐.** `TransactionBlocks`는 트랜잭션이 300개여도 마지막 tx 번호 하나만 쓴다 (`provider.rs:832-839`).
2. **`append` vs `put`.** `DbTxMut::append` doc: *"typically from O(logN) down to O(1) thanks to no lookup"* — 키가 항상 증가할 때만 쓸 수 있다. `TxHash`/`Address`로는 불가능.
3. **v1 시절의 흔적** (`provider.rs:677-678`):
   ```rust
   // Sort by hash for optimal MDBX insertion performance
   all_tx_hashes.sort_unstable_by_key(|(hash, _)| *hash);
   ```
   랜덤 키를 B-tree에 대량 삽입하는 게 느려서 넣은 우회책. **v2에서는 이 블록 전체를 스킵한다** — LSM은 memtable에서 어차피 정렬하니까.
4. **history 두 개는 삽입이 아니라 수정이다.** `storage_history_shards_to_put` doc: *"reading the current last shard and appending indices"* — B-tree에서 최악의 패턴.

> **정정된 답**: 키가 랜덤(해시·주소)이고 **블록당 대량으로** 쓰이는 테이블이 RocksDB로 간다. "인덱스"는 관찰이지 설명이 아니다.

### static file로 보내는 데이터의 조건 (3가지)

1. **불변** — 과거 블록 데이터는 다시 안 바뀐다
2. **프루닝이 필요 없을 것** — static file은 append-only라 **중간을 못 지운다**
3. **블록 번호 순 접근** — 오프셋 계산으로 찾는 구조라 임의 키 조회 불가

②를 실증하는 코드 (`either_writer.rs:188-195`):

```rust
if !receipts_in_static_files && prune_modes.has_receipts_pruning() ||
    receipts_in_static_files && !prune_modes.receipts_log_filter.is_empty()
{
    EitherWriterDestination::Database      // ← 지워야 하니까 MDBX로
} else {
    EitherWriterDestination::StaticFile
}
```

**receipts는 성격상 static file에 딱 맞는데(불변, 블록 순), 프루닝을 켜면 MDBX로 보낸다.**

③ 때문에 `TransactionHashNumbers`(키가 tx 해시)는 static file 후보에 아예 오르지 못하고 MDBX냐 RocksDB냐만 따진다.

> ⚠️ `data-flow.md:173`의 *"전부 `storage_v2` 하나로 갈린다"* 는 과장이다. receipts와 senders는 **`storage_v2` × 프루닝 설정** 2차원이다.

---

## 6. `storage_v2`의 정체 — 코드 구조에 미친 영향

`storage.mdx:83-84`:

> Storage mode is **selected when a database is initialized**. After that, Reth reads the mode from database metadata.

**런타임 토글이 아니다.** DB 생성 시점에 정해져 메타데이터에 박힌다. (`--storage.v2` 기본값은 `true` — `cli/commands/src/common.rs:79`)

이게 코드 구조에 남긴 흔적이 넷:

### ① `EitherWriter`/`EitherReader`라는 전용 추상화가 생겼다 ★

`either_writer.rs`라는 파일이 **존재하는 이유** 자체가 이것이다. 런타임 토글이면 그때그때 `if`로 처리하면 되는데, **DB 수명 내내 고정된 두 세대를 동시에 지원**해야 하니 "목적지를 결정하는 계층"을 따로 만들었다.

```rust
pub enum EitherWriterDestination { Database, StaticFile, RocksDB }
pub enum EitherWriter<'a, CURSOR, N> { Database(..), StaticFile(..), RocksDB(..) }
pub enum EitherReader<...> { ... }
```

### ② 읽기와 쓰기가 쌍으로 갈라졌다

```rust
EitherWriter::new_accounts_history(provider, batch)     // 쓰기
EitherReader::new_accounts_history(provider, rocksdb)   // 읽기
```

**같은 플래그를 보고 같은 판단**을 해야 한다. v1 DB에 v2 리더를 붙이면 엉뚱한 저장소를 읽는다. 그래서 판단 로직이 대칭으로 중복돼 있다.

### ③ 플래그를 캐싱할 수 있다

```rust
// provider.rs:3932
fn cached_storage_settings(&self) -> StorageSettings {
    *self.storage_settings.read()      // RwLock에 담아둠
}
```

초기화 시점에 고정되니 캐싱이 안전하다. 런타임 토글이면 매번 DB 메타데이터를 읽거나 무효화 처리를 해야 했다. **화요일에 "왜 `cached_`가 붙었지?" 했던 게 이것.**

### ④ 마이그레이션 명령이 별도로 필요

`reth db migrate-v2`. 플래그를 바꿔도 데이터가 안 옮겨진다. `--storage.v2` 플래그가 *"does not rewrite an initialized data directory"* 라고 못박힌 이유(`storage.mdx:18-19`).

---

## 7. ★ v2에서 MDBX의 역할은 무엇으로 수렴하나 (문서에 답 없음)

이번 주 관찰 넷을 합친다.

| 출처 | 관찰 | MDBX의 역할 |
|---|---|---|
| 목 | v2에서 남는 건 현재 상태 + 트라이 + 체크포인트 | **"지금"의 정본** |
| 목 | static file은 MDBX가 든 위치 정보(`BlockBodyIndices` 등)로 접근됨 | **"어디"의 색인** |
| 수 | 커밋 순서에서 MDBX가 기준점 (전진 시 마지막, 후진 시 처음) | **확정 시점의 결정자** |
| 수 | MDBX만 트랜잭션을 provider **필드로** 갖는다 | 위를 가능하게 하는 수단 |

> **v2에서 MDBX는 데이터를 많이 담는 저장소가 아니라, 노드의 "지금"(현재 상태·트라이)과 "어디"(다른 저장소를 가리키는 색인)를 쥐고, 그 트랜잭션 커밋이 곧 상태 확정 시점이 되는 진실의 원천(source of truth)으로 수렴한다.**

**용량은 줄었는데 권위는 오히려 집중됐다.**

### ⚠️ "캐시"가 아니다

MDBX가 static file/RocksDB의 캐시라고 생각하기 쉬운데 틀렸다. **캐시는 원본이 딴 데 있고 사본을 두는 것**인데, MDBX의 현재 상태는 **원본(정본)** 이다.

```rust
// reth-primitives-traits/src/account.rs:32
pub struct Account {
    pub nonce: u64,
    pub balance: U256,
    pub bytecode_hash: Option<B256>,   // 코드는 Bytecodes 테이블에 따로 (역시 MDBX)
}
```

`PlainAccountState[Address] = Account` — **본체가 통째로** MDBX에 있다. 증거:

- `StaticFileSegment`에 현재 상태용 세그먼트가 **없다**: `Headers, Transactions, Receipts, TransactionSenders, AccountChangeSets, StorageChangeSets`
  (`AccountChangeSets`는 *과거 이력*이지 현재 상태가 아니다)
- `PlainAccountState`/`HashedAccounts`는 `providers/rocksdb/`·`providers/static_file/` 디렉터리에 **0회** 등장

**트랜잭션과 다른 이유**: 트랜잭션은 한 번 쓰면 안 바뀌니 append-only에 맞고 MDBX엔 색인만 둔다. **계정 잔액은 매 블록 덮어써야** 하니 append-only가 불가능하다.

### ⚠️ 커밋 순서는 하나가 아니라 둘이다

`data-flow.md:256`에 정방향만 적었다. 실제로는 (`provider.rs:3890-3913`):

| | 순서 | MDBX 위치 |
|---|---|---|
| 정방향 | SF → RocksDB → **MDBX** | 마지막 |
| 언와인드 | **MDBX** → RocksDB → SF | 첫 번째 |

원칙은 "기준점을 마지막에"가 아니라 **"MDBX가 기준점이고, 크래시 후 나머지를 MDBX에 맞출 수 있게 순서를 잡는다"**:

- 정방향: MDBX가 커밋 안 됐으면 앞서 쓴 SF/RocksDB는 그냥 안 보임 → 없던 일
- 언와인드: MDBX를 먼저 확정해두면, 재시작 때 **MDBX 체크포인트 기준으로 static file을 잘라내서** 복구 가능

`commit_unwind` doc이 그대로 말한다 (`provider.rs:265-266`):

> This keeps MDBX as the **first durable step** so an interrupted unwind can be recovered by truncating static files from checkpoints on the next startup.

---

## 8. 확인문제 답

**1. 계층을 나눈 공식 이유 + 실제로 쓰이는 증거**
`database.md:5` — *"frees us from being bound to a single database implementation"*. 증거는 redb(아직 **탐색 중**이라 약함)보다 **이미 컴파일되는 두 번째 구현체**가 강하다: ②층은 `db-api/mock.rs`의 `TxMock`, ①층은 `rpc-provider`.

**2. GAT가 없었다면**
`type Cursor<T: Table>`처럼 **테이블 타입에 따라 달라지는 커서 타입**을 트레이트가 약속할 수 없다. 우회로 셋이 전부 못 쓸 물건 → §2 참조. 핵심은 **`for<T>` HRTB가 없어서 "모든 테이블"을 한 번에 요구할 방법이 없다**는 것.

**3. hot/cold ↔ 목요일 분류**
MDBX↔hot, static file↔cold는 잘 대응된다. 다만 **RocksDB는 쓰기 hot / 읽기 cold라 이 축으로 분류되지 않으며, 공식 문서도 대응을 명시하지 않았다.** hot/cold는 별칭이고 3분류가 더 세밀하다.

**4. `storage_v2`가 초기화 시점 값인 게 코드 구조에 준 영향**
→ §6. 요약하면 **`EitherWriter`/`EitherReader`라는 전용 추상화의 존재 이유**이고, `cached_storage_settings()`의 `cached_` 접두사가 가능한 이유이며, 별도 마이그레이션 명령이 필요한 이유다.

**5. v2에서 MDBX의 본질적 역할 (한 문장)**
→ §7. **"지금"과 "어디"를 쥐고, 그 커밋이 곧 상태 확정 시점이 되는 진실의 원천.** 캐시가 아니다.

---

## 9. 이번 주 정정 목록 (다른 문서에 반영 필요)

| 문서 | 위치 | 정정 |
|---|---|---|
| `data-flow.md` | :58 | `in_place_scope`는 "호출 스레드도 일꾼으로 쓴다"가 아니라 **"클로저 본문을 호출 스레드에서 실행한다"**. 그리고 `scope`는 `OP: Send`를 요구하고 `in_place_scope`는 요구하지 않는다 — 이게 컴파일 가능 여부를 가른다 |
| `data-flow.md` | :173 | *"전부 `storage_v2` 하나로 갈린다"* — receipts/senders는 **프루닝 설정도** 본다 |
| `data-flow.md` | :181 | "헤더 / 바디 / 트랜잭션"을 한 행으로 묶으면 안 됨. **헤더·트랜잭션 본문은 SF, 바디 인덱스(`BlockBodyIndices`/`TransactionBlocks`/ommers/withdrawals)와 `HeaderNumbers`는 MDBX** |
| `data-flow.md` | :219 | *"전부 인덱스"* — MDBX에도 인덱스가 많다. 기준은 **랜덤 키 × 대량 쓰기** (→ §5) |
| `data-flow.md` | :256 | 커밋 순서가 **둘**이다 (정방향/언와인드가 정반대) |

---

## 10. 다음으로 넘길 질문

1. `DbTx`에서 `Sync`가 빠진 건 `chore(db): Remove Sync from DbTx (#20516)` (2025-12-22)다. 45개 파일을 건드린 **바운드 최소화 리팩터링**이었고 `#[auto_impl(&, Arc, Box)]`가 `(&, Box)`로 좁아졌다. **커밋 메시지에 동기가 없다** — PR 본문을 봐야 함
2. `Database::TX`/`TXMut`에는 `Sync`가 남아 있다. 즉 요구가 사라진 게 아니라 **필요한 곳으로 이동**했다. 이 패턴(트레이트는 최소, 사용처에서 명시)이 reth 전반의 방침인가?
3. RocksDB의 `AccountsHistory` read-modify-write는 매 블록 수천 건인데, 샤드 읽기는 어디서 일어나나? 배치에 쌓기 전에 읽는다면 그 읽기는 무슨 스냅샷을 보나
4. `--minimal` 모드가 449GB → 224GB로 **-50%** 인데 다른 모드보다 절감폭이 크다. 무엇이 빠지는 건가