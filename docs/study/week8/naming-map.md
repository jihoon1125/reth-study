# 이름 지도 — Database / DbTx / Tx / Transaction / RO / RW

> 헷갈릴 때 돌아오는 참조 문서.
> 근본 원인: **"DB 전체"와 "트랜잭션"이라는 딱 두 개념이 5개 층에서 이름을 바꿔 반복된다.**

---

## 1. 계층 구조 (정적) — 누가 누구를 감싸는가

| 층 | 크레이트 | **DB 전체** | **트랜잭션** | 읽기/쓰기 구분 |
|---|---|---|---|---|
| ④ 고수준 | `reth-provider` | `ProviderFactory<N>` | `DatabaseProvider<TX,N>` | `DatabaseProviderRO` / `RW` |
| ③ **추상 트레이트** | `reth-db-api` | `Database` (trait) | `DbTx` / `DbTxMut` (trait) | `Database::TX` / `::TXMut` |
| ② MDBX 구현 | `reth-db` | `DatabaseEnv` | `Tx<K>` | `Tx<RO>` / `Tx<RW>` |
| ① 안전 래퍼 | `reth-libmdbx` | `Environment` | `Transaction<K>` | `Transaction<RO>` / `<RW>` |
| ⓪ C | `mdbx-sys` → libmdbx | `MDBX_env` | `MDBX_txn` | 플래그 |

**세로로 읽으면 전부 같은 것이다.** 위로 갈수록 한 겹씩 감싼 것뿐.

**③층만 트레이트(약속)이고 나머지는 구체 타입(실물)이다.** ③이 있어서 ④가 ②를 몰라도 된다.

```
                ④ reth-provider ──────────────────────────────
                   ProviderFactory<N>              "연결"
                          │ .provider() / .provider_rw()
                          ▼
                   DatabaseProvider<TX, N>         "한 번의 세션"
                   (RO = TX가 ::TX / RW = TX가 ::TXMut)
    ─────────────────────┼───────────────────────────────────
                ③ reth-db-api  (트레이트 = 약속)
                   trait Database { type TX; type TXMut; }
                   trait DbTx (읽기)  /  trait DbTxMut (쓰기)
    ─────────────────────┼───────────────────────────────────
                ② reth-db  (MDBX 구현)
                   DatabaseEnv ──impl Database──▶ TX=Tx<RO>, TXMut=Tx<RW>
                   Tx<K> ── dbi 캐시 + 메트릭 추가
    ─────────────────────┼───────────────────────────────────
                ① reth-libmdbx  (안전 래퍼)
                   Environment
                   Transaction<K> ── 내부에 TransactionPtr
                                        └─ Mutex로 C 호출 직렬화   ← 락 (a)
    ─────────────────────┼───────────────────────────────────
                ⓪ libmdbx (C)
                   MDBX_env / MDBX_txn
                   writer 뮤텍스 ← 락 (b)  │  MVCC ← (c)
```

---

## 2. 호출 추적 (동적) — 그 계층을 관통하는 동선

`factory.provider()` 한 번이 계층을 **아래로 내려갔다 되돌아온다.** 위 구조의 다른 관점일 뿐이다.

```
factory.provider()                                        [④]
  │
  └─ self.db.tx()               db: N::DB = DatabaseEnv    [③ 트레이트 호출]
       │
       └─ DatabaseEnv::tx()     impl Database for DatabaseEnv   [②]
            │
            └─ inner.begin_ro_txn()      Environment       [①]
                 │
                 └─ mdbx_txn_begin()      C 함수           [⓪]
                 ◀── MDBX_txn 포인터
            ◀── Transaction<RO>                            [①]
       ◀── Tx<RO>     (Transaction<RO>를 감싸고 dbi 캐시 + 메트릭 추가)   [②]
  ◀── DatabaseProvider<Tx<RO>, N>   = DatabaseProviderRO   [④]
```

### 연관 타입의 빈칸이 메워지는 지점

`crates/storage/db/src/implementation/mdbx/mod.rs:261`

```rust
impl Database for DatabaseEnv {
    type TX = tx::Tx<RO>;        // ← 여기서 채워진다
    type TXMut = tx::Tx<RW>;

    fn tx(&self) -> Result<Self::TX, DatabaseError> { ... }
    fn tx_mut(&self) -> Result<Self::TXMut, DatabaseError> { ... }
}
```

`N::DB` = `DatabaseEnv`, 그것의 `::TX` = `Tx<RO>`.
`<DB as Database>::TX` 같은 표기가 결국 무엇을 가리키는지가 여기서 확정된다.

---

## 3. 이름 읽는 규칙

| 규칙 | 예 |
|---|---|
| **`Db~`로 시작 = 트레이트** | `DbTx`, `DbTxMut`, `DbCursorRO` |
| **대문자 `TX` / `TXMut` = 연관 타입** (`Database`의 빈칸) | `Database::TX`, `N::DB` |
| **`RO` / `RW` = 마커 타입** (값이 없음) | `Tx<RO>`, `Transaction<RW>` |

주의: `TX`가 **제네릭 파라미터**로 쓰일 때(`DatabaseProvider<TX, N>`)는 "여기 트랜잭션 타입이
들어온다"는 빈칸 표시다. 연관 타입 `Database::TX`와 **이름만 같고 다른 것.**

---

## 4. `RO` / `RW`는 실체가 없는 타입이다

`crates/storage/libmdbx-rs/src/transaction.rs:43`

```rust
pub struct RO;          // ← 필드가 하나도 없다 (크기 0)
pub struct RW;

impl TransactionKind for RO {
    const OPEN_FLAGS: MDBX_txn_flags_t = MDBX_TXN_RDONLY;
    const IS_READ_ONLY: bool = true;
}
impl TransactionKind for RW { ... }
```

런타임 실체가 없고, **오직 컴파일러에게 구분을 알려주려고** 존재한다.

### 타입 수준 강제가 실제로 구현되는 자리

`crates/storage/db/src/implementation/mdbx/tx.rs:285, 399`

```rust
impl<K: TransactionKind> DbTx for Tx<K> { ... }   // 읽기: 모든 K에 대해
impl DbTxMut for Tx<RW> { ... }                   // 쓰기: RW일 때만!
```

**쓰기 메서드는 `Tx<RW>`에만 붙는다.** 그래서 `Tx<RO>`로 `put`을 부르면 "그런 메서드 없음"
컴파일 에러가 난다.

이번 주 내내 말한 "RO/RW가 타입 수준에서 강제된다"의 근원이 이 두 줄이다.

---

## 5. "락"이 가리키는 세 가지

같은 단어가 서로 다른 층의 세 가지를 가리켜서 헷갈린다.

| | 무엇을 막나 | 누가 | 범위 |
|---|---|---|---|
| **(a)** | 한 트랜잭션 객체를 여러 스레드가 동시에 건드리는 것 | **reth가 추가** (`TransactionPtr`의 `Mutex`) | 객체 하나 |
| **(b)** | 쓰기 트랜잭션이 둘 이상 열리는 것 | MDBX 자체 (writer 뮤텍스) | DB 파일 전체 |
| **(c)** | 읽기와 쓰기가 서로 막는 것 → **안 막는다** | MDBX 자체 (MVCC 스냅샷) | DB 파일 전체 |

`concurrency.md`에서 다룬 건 (b)(c), `unsafe impl Sync`의 근거는 (a)다.

### (a) 락이 도는 방식

`crates/storage/libmdbx-rs/src/transaction.rs:540`

```rust
pub(crate) struct TransactionPtr {
    txn: *mut ffi::MDBX_txn,      // 그냥 C 포인터. 원래 Send도 Sync도 아님
    lock: Arc<Mutex<()>>,         // 이 포인터 접근을 직렬화
}
```

`transaction.rs:598`

```rust
pub(crate) fn txn_execute_fail_on_timeout<F, T>(&self, f: F) -> Result<T> {
    let _lck = self.lock();      // ① 락 획득
    ...
    Ok((f)(self.txn))            // ② 락 잡은 채로 C 함수 호출 → drop 시 자동 해제
}
```

**MDBX C 함수를 부르는 모든 경로가 이 함수를 통과한다.** 즉 한 트랜잭션에 대한 C 호출은
항상 한 번에 하나다.

이게 `transaction.rs:714`의 정확한 의미:

```rust
// SAFETY: Access to the transaction is synchronized by the lock.
unsafe impl Sync for TransactionPtr {}
```

원래 `*mut`는 `Sync`가 아니다. 그런데 모든 접근을 뮤텍스로 감쌌으니 여러 스레드가 참조를 나눠
가져도 안전하다 — **락이 `Sync`를 정당화하는 구조.**

### 락과 무관한 것

`save_blocks`에서 `&self`를 rayon `spawn`에 못 넘기는 문제(`data-flow.md` §2)는 이 락과
**무관하다.** 락은 ②①층 얘기고, 그건 ④층 **제네릭 바운드** 얘기다.
구체 타입은 `Sync`인데 impl 블록 바운드에 안 적혀서 못 쓰는 것뿐.

---

## 기억할 한 문장

> **③층만 트레이트고 나머지는 전부 구체 타입이며, 위로 갈수록 감싸는 것뿐이다.**
