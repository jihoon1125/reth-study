# 데이터 흐름 — 블록 하나가 저장되기까지

> 8주차 목요일. 질문: **실행된 블록의 어떤 데이터가 어느 저장소로 갈라지는가?**

---

## 1. 진입점 — `save_blocks`

`crates/storage/provider/src/providers/database/provider.rs:579`

```rust
pub fn save_blocks(
    &self,
    blocks: Vec<ExecutedBlock<N::Primitives>>,
    save_mode: SaveBlocksMode,
) -> ProviderResult<()>
```

문서 주석: *"Writes executed blocks and state to storage."*

`SaveBlocksMode`(`provider.rs:166`)는 두 가지:

| 모드 | 쓰는 것 | 쓰는 쪽 |
|---|---|---|
| `Full` | 블록 구조 + receipts + state + trie | 엔진/프로덕션 |
| `BlocksOnly` | 블록 구조만 (헤더, tx, senders, indices) | `insert_block` |

---

## 2. 세 저장소에 병렬로 쓴다

`provider.rs:627`

```rust
// Write to all backends in parallel.
let runtime = &self.runtime;
runtime.storage_pool().in_place_scope(|s| {
    s.spawn(|_| {                                    // ① static file — 워커 스레드
        sf_result = Some(sf_provider.write_blocks_data(&blocks, &tx_nums, sf_ctx, runtime)...);
    });

    if rocksdb_enabled {
        s.spawn(|_| {                                // ② RocksDB — 워커 스레드
            rocksdb_result = Some(rocksdb_provider.write_blocks_data(...));
        });
    }

    // ③ MDBX — spawn 없이 호출 스레드에서 직접
    ...
    self.insert_block_mdbx_only(recovered_block, tx_nums[i])?;
    ...
    Ok::<_, ProviderError>(())
})?;
```

### `in_place_scope` — 성능이 아니라 컴파일 가능 여부의 문제

rayon-core 1.13 시그니처를 비교하면 차이가 명확하다.

```rust
pub fn scope<'scope, OP, R>(op: OP) -> R
where OP: FnOnce(&Scope<'scope>) -> R + Send,    // ← 클로저가 Send여야 함
      R: Send,

pub fn in_place_scope<'scope, OP, R>(op: OP) -> R
where OP: FnOnce(&Scope<'scope>) -> R,           // ← Send 요구 없음
```

`in_place_scope`는 **클로저 본문을 호출 스레드에서 실행하므로** `Send`를 요구하지 않는다.

여기서는 클로저가 `&self`를 캡처하는데, 아래 §"MDBX만 `spawn`이 없는 이유"에서 보듯
`&DatabaseProvider<TX,N>`은 `Send`가 아니다. 즉 **`scope`를 썼다면 애초에 컴파일이 안 된다.**
"호출 스레드를 놀리지 않는다"는 부수 효과이지 선택 이유가 아니다.

`spawn`한 작업이 전부 끝나야 반환되는 것(암묵적 배리어)은 양쪽 공통이다.

### MDBX만 `spawn`이 없는 이유 — 컴파일이 안 된다

`save_blocks`가 속한 impl 블록(`provider.rs:510`):

```rust
impl<TX: DbTx + DbTxMut + 'static, N: NodeTypesForProvider> DatabaseProvider<TX, N> {
//      ^^^^^^^^^^^^^^^^^^^^^^^^^ Sync가 없다
```

그리고 트레이트 선언(`db-api/transaction.rs:21, 52`):

```rust
pub trait DbTx: Debug + Send { ... }     // Send만
pub trait DbTxMut: Send { ... }          // Send만
```

추론 사슬:

```
1. 이 impl 안에서 TX는 Send일 뿐, Sync인지 컴파일러는 모른다
2. DatabaseProvider<TX,N>의 auto Sync는 "모든 필드가 Sync"여야 성립
   → 필드 tx: TX 가 Sync를 모르니 DatabaseProvider도 Sync 아님
3. &T: Send  ⟺  T: Sync   (러스트 규칙)
   → &DatabaseProvider<TX,N> 은 Send 아님
4. rayon의 s.spawn(f)는 f: Send 를 요구
5. ⟹ &self를 캡처한 클로저는 Send가 아니라 컴파일 에러
```

그래서 `provider.rs:617`의 주석이 정확히 이 얘기다:

```rust
// avoid capturing &self.tx in scope below.
let sf_provider = &self.static_file_provider;
```

`&self`를 통째로 캡처하면 `tx`까지 딸려오니, **`static_file_provider` 필드만 따로 빼서**
그것만 캡처하게 만든 것이다 (Rust 2021의 disjoint closure capture). `StaticFileProvider`는
내부가 `Arc<..>` + `DashMap`/`RwLock`이라 `Sync`다.

### ★ 구체 타입 vs 제네릭 컨텍스트

헷갈리기 쉬운 지점. **구체 타입은 실제로 `Sync`다.**

`crates/storage/libmdbx-rs/src/transaction.rs:714`

```rust
// SAFETY: Access to the transaction is synchronized by the lock.
unsafe impl Send for TransactionPtr {}
unsafe impl Sync for TransactionPtr {}

#[cfg(test)]
mod tests {
    const fn assert_send_sync<T: Send + Sync>() {}
    const fn test_txn_send_sync() {
        assert_send_sync::<Transaction<RO>>();
        assert_send_sync::<Transaction<RW>>();   // RW도 컴파일 타임에 단언
    }
}
```

| | 사실 |
|---|---|
| **구체 타입** `Transaction<RW>` | `Sync` 맞음 |
| **제네릭 컨텍스트의 `TX`** | `Sync` 아님 (바운드에 없으므로) |

> **제네릭 함수 안에서는 선언된 바운드만이 진실이다.** 실제로 들어올 타입이 `Sync`여도,
> 바운드에 안 적었으면 컴파일러에게는 없는 성질이다.

따라서 정확히는 **"물리적으로 불가능"이 아니라 "현재 선언된 바운드에서는 불가능"** 이다.
impl 블록에 `+ Sync`를 추가하면 컴파일은 통과할 것이다. 넓히지 않은 건 넓힐 이유가 없어서일
것이고, 그 위에 얹히는 부가 이유가 둘 더 있다:

1. **스코프 클로저가 `Result`를 반환한다** (`provider.rs:760`의 `Ok::<_, ProviderError>(())` +
   `})?`). 그래서 MDBX 구간은 `?`를 열 번 넘게 그냥 쓴다. `spawn`한 둘은 클로저가 `()`를
   반환해야 해서 결과를 `Option`에 담아뒀다 나중에 꺼낸다:
   ```rust
   timings.sf = sf_result.ok_or(StaticFileWriterError::ThreadPanic("static file"))??;
   ```
2. **`in_place_scope`니까 호출 스레드가 놀면 손해다.** 작업 3개를 전부 `spawn`하면 호출
   스레드는 대기만 한다.

> 참고: 락이 있으니 `Sync`가 **불필요**한 게 아니라, 락이 있으니 `Sync`가 **안전(sound)** 한 것이다.
> `unsafe impl`은 "내가 책임진다"는 선언이고 그 근거가 락이다.

---

## 3. 목적지는 누가 정하나

`crates/storage/provider/src/either_writer.rs:968`

```rust
/// Destination for writing data.
#[derive(Debug, EnumIs)]
pub enum EitherWriterDestination {
    Database,      // MDBX 테이블
    StaticFile,
    RocksDB,
}
```

3지 선다이고, 데이터 종류마다 결정 함수가 하나씩 있다.

```rust
pub fn senders<P>(provider: &P) -> Self {
    if provider.cached_storage_settings().storage_v2 {
        Self::StaticFile
    } else {
        Self::Database
    }
}
```

**대부분 `storage_v2` 하나로 갈린다.** 화요일에 본 `use_hashed_state()`도 같은 플래그였다
(`db-api/models/metadata.rs:85`). 즉 `storage_v2`는 **저장 레이아웃 세대를 통째로 결정하는
스위치**다.

단, **receipts는 예외로 2차원이다** — `storage_v2` × 프루닝 설정 (`either_writer.rs:187`):

```rust
if !receipts_in_static_files && prune_modes.has_receipts_pruning() ||
    receipts_in_static_files && !prune_modes.receipts_log_filter.is_empty()
{
    EitherWriterDestination::Database      // ← 지워야 하니까 MDBX로
} else {
    EitherWriterDestination::StaticFile
}
```

static file은 append-only라 **중간을 못 지운다.** 그래서 성격상 static file에 딱 맞는
receipts도 프루닝을 켜면 MDBX로 보낸다.

### 목적지 표

| 데이터 | v1 (`storage_v2 = false`) | v2 (`storage_v2 = true`) | 근거 |
|---|---|---|---|
| 헤더 본문 / 트랜잭션 본문 | **StaticFile** | **StaticFile** | 항상 |
| `HeaderNumbers` (해시→번호) | Database | Database | `provider.rs:808` |
| `BlockBodyIndices`, `TransactionBlocks`, ommers, withdrawals | Database | Database | `insert_block_mdbx_only` |
| senders | Database | **StaticFile** | `either_writer.rs:981` |
| receipts | Database | **StaticFile** (프루닝 켜면 Database) | `either_writer.rs:182` |
| account changesets | Database | **StaticFile** | `either_writer.rs:994` |
| storage changesets | Database | **StaticFile** | `either_writer.rs:1007` |
| `TransactionHashNumbers` | Database | **RocksDB** | `either_writer.rs:241` |
| `AccountsHistory` | Database | **RocksDB** | `either_writer.rs:259` |
| `StoragesHistory` | Database | **RocksDB** | `either_writer.rs:225` |
| 현재 계정 상태 | Database (`PlainAccountState`) | Database (`HashedAccounts`) | 화요일 §5 |

> ⚠️ **"블록"을 한 덩어리로 묶으면 안 된다.** 헤더·트랜잭션 **본문**은 static file로 가지만,
> 그것을 찾아가는 **인덱스**(`HeaderNumbers`, `BlockBodyIndices`, `TransactionBlocks`)는
> MDBX에 남는다. 목요일 §5의 "MDBX가 위치 정보를 든다"가 이 행들이다.

**v2로 가면 MDBX가 하는 일이 줄어든다.** 남는 건 현재 상태(state)와 트라이, 그리고 각종
체크포인트/인덱스뿐이다.

### static file로 보내는 데이터의 조건

표를 보면 규칙이 보인다.

1. **불변** — 과거 블록의 헤더/트랜잭션/receipts는 다시 안 바뀐다 (static file은 append-only)
2. **블록 번호 순으로 접근** — 순차 스캔이 자연스러운 데이터
3. **프루닝 단위가 블록 범위** — 잘라낼 때 파일 단위로 처리 가능

---

## 4. RocksDB의 정체 — 화요일 질문의 답

```bash
grep -rn "tables::" crates/storage/provider/src/providers/rocksdb/provider.rs \
  | grep -oE "tables::[A-Za-z]+" | sort -u
```

**딱 3개**:

| 테이블 | 무엇의 인덱스인가 |
|---|---|
| `TransactionHashNumbers` | tx 해시 → tx 번호 |
| `AccountsHistory` | 주소 → 이 계정이 바뀐 블록 번호들 |
| `StoragesHistory` | (주소, 슬롯) → 바뀐 블록 번호들 |

셋 다 인덱스이고 현재 상태는 하나도 없다. **다만 "인덱스라서 RocksDB"는 아니다** — MDBX에도
인덱스가 많다 (`HeaderNumbers`, `BlockBodyIndices`, `TransactionBlocks`).

진짜 기준은 **키의 성격 × 블록당 쓰기 볼륨**이다.

| 테이블 | 저장소 | 키 | 키 순서 | 블록당 쓰기 | 방식 |
|---|---|---|---|---|---|
| `BlockBodyIndices` | MDBX | `BlockNumber` | **단조 증가** | **1건** | `append` |
| `TransactionBlocks` | MDBX | `TxNumber` | **단조 증가** | **1건** | `append` |
| `HeaderNumbers` | MDBX | `BlockHash` | 랜덤 | **1건** | `put` |
| `TransactionHashNumbers` | **RocksDB** | `TxHash` | 랜덤 | **수백 건** | put |
| `AccountsHistory` | **RocksDB** | `Address` | 랜덤 | **수천 건** | **read-modify-write** |
| `StoragesHistory` | **RocksDB** | `(Addr, Slot)` | 랜덤 | **수천 건** | **read-modify-write** |

- MDBX에 남은 인덱스는 **블록당 1건**이다. `TransactionBlocks`는 트랜잭션이 300개여도 마지막
  tx 번호 하나만 쓴다 (`provider.rs:832-839`)
- 키가 단조 증가하면 `append`를 쓸 수 있다. `DbTxMut::append` doc: *"typically from O(logN)
  down to O(1) thanks to no lookup"*. `TxHash`/`Address`로는 불가능
- history 두 개는 삽입이 아니라 **수정**이다 — 기존 샤드를 읽어서 인덱스를 덧붙인다.
  B-tree에서 최악의 패턴

v1 시절의 흔적이 코드에 남아 있다 (`provider.rs:677-678`):

```rust
// Sort by hash for optimal MDBX insertion performance
all_tx_hashes.sort_unstable_by_key(|(hash, _)| *hash);
```

랜덤 키를 B-tree에 대량 삽입하는 게 느려서 넣은 우회책이고, **v2에서는 이 블록 전체를 스킵한다**
— LSM은 memtable에서 어차피 정렬하니까.

> **정정**: "전부 인덱스다"는 관찰이지 설명이 아니다. 기준은 **랜덤 키 × 대량 쓰기**.

RocksDB는 **LSM-tree** 기반이라 이런 쓰기에 강하다. 랜덤 키를 B+tree(MDBX)에 대량 삽입하면
페이지가 여기저기 쪼개지지만, LSM은 메모리에 모았다가 순차로 flush한다.

### 그래서 화요일 질문의 답

> *"`basic_account` 경로 어디에도 RocksDB가 안 나온다"*

**당연하다.** `basic_account`는 **현재 계정 상태**를 묻고, 그건 MDBX(`PlainAccountState` /
`HashedAccounts`)에 있다.

RocksDB가 나오는 건 다른 질문을 할 때다:

| 질문 | 쓰이는 저장소 |
|---|---|
| "이 주소의 지금 잔고는?" | MDBX (`latest.rs`) |
| "1000번 블록 시점의 이 주소 잔고는?" | **RocksDB**(`AccountsHistory`로 되감을 지점 찾기) → 그다음 changeset |
| "이 tx 해시가 몇 번 tx지?" | **RocksDB**(`TransactionHashNumbers`) |

`providers/state/historical.rs:361`의 `basic_account`가 두 번째 경로다. `account_history_lookup`
으로 **그 블록 시점에 이 주소가 어떤 상태였는지**(존재했는지, changeset에 포함되는지)를 먼저
판정한 뒤 값을 만든다. 화요일에 본 8줄짜리와 근본적으로 다른 일을 한다.

> **같은 함수 이름인데 저장소가 다른 이유는 질문의 종류가 다르기 때문이다.**
> 트레이트가 "무엇을 물어볼 수 있나"를 정의한다는 월요일 정리와 이어진다.

---

## 5. 쓰기는 병렬, 커밋은 순차

```
save_blocks:  SF ∥ RocksDB ∥ MDBX          ← 병렬. 순서 무관
                    ↓
commit (정방향):  SF → RocksDB → MDBX      ← 순차. MDBX가 마지막
commit (언와인드): MDBX → RocksDB → SF      ← 순차. MDBX가 처음
```

**커밋 순서는 하나가 아니라 둘이고, 정반대다** (`provider.rs:3890-3913`, `CommitOrder`).
어느 쪽이든 MDBX가 기준점이고, 크래시 후 나머지를 MDBX에 맞출 수 있게 순서를 잡는다.
자세한 건 `concurrency.md` §3.

병렬 쓰기와 순차 커밋은 모순이 아니다.

- **쓰기**는 각 저장소의 버퍼/임시 영역에 쌓는 단계 → 서로 무관하니 병렬 가능
- **커밋**은 "이제 진짜다"를 확정하는 단계 → 크래시 시점에 따라 결과가 갈리니 순서가 중요

수요일에 본 `pending_rocksdb_batches`가 정확히 이 구조다. RocksDB 쓰기는 병렬로 **배치만
만들어두고**, 실제 반영은 커밋 시점에 정해진 순서로 한다.

---

## 6. 오늘 나온 Rust 문법

### `scope` — 클로저가 바깥 변수를 수정할 수 있는 이유

```rust
let mut sf_result = None;
runtime.storage_pool().in_place_scope(|s| {
    s.spawn(|_| {
        sf_result = Some(...);      // 스코프 바깥 변수를 가변 대여
    });
});
// 여기서 sf_result를 읽음
```

`scope`는 **"스코프가 끝날 때까지 모든 스폰 작업이 끝난다"** 를 타입으로 보장한다. 그래서
컴파일러가 대여를 허용한다. `std::thread::spawn`은 언제 끝날지 모르니 이게 안 된다.

### `unsafe impl Send/Sync`

"컴파일러야, 안전성은 내가 책임질 테니 통과시켜"라는 선언. 반드시 SAFETY 주석으로 근거를
남긴다. 여기서는 "접근이 락으로 직렬화된다"가 근거.

### `#[derive(EnumIs)]`

`is_database()`, `is_static_file()`, `is_rocksdb()` 같은 판별 함수를 자동 생성하는 매크로.

```rust
write_senders: EitherWriterDestination::senders(self).is_static_file() && ...
```

---

## 7. 다음으로 넘길 질문

1. `storage_v2` 스위치는 언제 켜지나? v1 → v2 마이그레이션은 어떻게 이뤄지나
2. v2에서 MDBX에 남는 게 "현재 상태 + 트라이 + 체크포인트"뿐이라면, **MDBX의 역할이 결국
   무엇으로 수렴하는가?** (→ 금요일 설계 의도)
3. 저장소를 셋으로 나눈 기준을 한 문장으로 정리하면? — 불변/순차는 static file,
   랜덤 대량 쓰기는 RocksDB(LSM), 그 외 현재 상태는 MDBX(B+tree)? 이게 실제 판단 기준인지 검증
4. `write_blocks_data`가 static file과 RocksDB 양쪽에 같은 이름으로 있다. 트레이트인가 우연인가
