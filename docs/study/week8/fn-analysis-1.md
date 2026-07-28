# `basic_account()` 라인 단위 추적

> 8주차 화요일 — "이 주소의 계정 정보를 어떻게 가져오는가"를 진입점부터 MDBX 조회까지.

---

## 0. 전체 경로

```
DatabaseProvider::basic_account(&address)      provider.rs:1462          진입점
        ↓  self.tx.get_by_encoded_key::<PlainAccountState>(address)
DbTx::get_by_encoded_key<T: Table>             db-api/transaction.rs:32  트레이트 = 약속
        ↓  MDBX가 이 약속을 구현
Tx::get_by_encoded_key                         db/mdbx/tx.rs:293         실제 구현
        ↓  tx.get(dbi, key.as_ref())
[ libmdbx (C) ]                                                          디스크
```

`basic_account` 구현체는 6곳에 있다(`grep -rn "fn basic_account"`). 여기서는 가장 저수준인
`DatabaseProvider`의 것을 추적한다. 나머지(`latest.rs`, `historical.rs`, `blockchain_provider.rs`
등)는 결국 이 경로로 수렴하거나, 인메모리 상태를 먼저 보고 없으면 여기로 내려온다.

---

## 1. 진입점 — `DatabaseProvider::basic_account`

`crates/storage/provider/src/providers/database/provider.rs:1462`

```rust
impl<TX: DbTx, N: NodeTypes> AccountReader for DatabaseProvider<TX, N> {
    fn basic_account(&self, address: &Address) -> ProviderResult<Option<Account>> {
        if self.cached_storage_settings().use_hashed_state() {
            let hashed_address = keccak256(address);
            Ok(self.tx.get_by_encoded_key::<tables::HashedAccounts>(&hashed_address)?)
        } else {
            Ok(self.tx.get_by_encoded_key::<tables::PlainAccountState>(address)?)
        }
    }
}
```

계정 조회의 전부가 8줄이다.

### impl 한 줄 읽기

```
impl<TX: DbTx, N: NodeTypes>       "TX가 DbTx이고 N이 NodeTypes인 모든 경우에 대해"
    AccountReader                   "AccountReader 트레이트를"
    for DatabaseProvider<TX, N>     "DatabaseProvider<TX,N>에 구현한다"
```

바운드가 `DbTx`(읽기)인데 **RW provider에도 이 impl이 적용된다.** 이유는 §4 참조.

### 반환 타입에 담긴 두 겹의 불확실성

`ProviderResult<Option<Account>>`

| 상황 | 표현 |
|---|---|
| 계정이 존재하지 않음 | `Ok(None)` — **정상 결과의 일종** |
| 디스크 읽기/디코딩 실패 | `Err(DatabaseError)` |

"계정이 없다"는 에러가 아니라는 판단이 타입에 드러나 있다.

### 분기: `use_hashed_state()`

`crates/storage/db-api/src/models/metadata.rs:85`

```rust
pub const fn use_hashed_state(&self) -> bool {
    self.storage_v2
}
```

**storage v2인지 아닌지**로 갈린다. v2면 `HashedAccounts`, 아니면 `PlainAccountState`.
두 테이블의 차이는 §5에서 다룬다.

---

## 2. ★ 테이블은 값이 아니라 타입이다

```rust
self.tx.get_by_encoded_key::<tables::PlainAccountState>(address)
                          └──────── turbofish ────────┘
```

**turbofish `::<T>`** = 함수에 값이 아니라 **타입**을 넘기는 문법.

보통이라면 `tx.get("PlainAccountState", address)`처럼 테이블 이름을 문자열로 넘겨서 오타가
런타임 에러가 된다. 러스트는 테이블 자체를 타입으로 넘기므로 **오타면 컴파일이 안 된다.**

### `tables::PlainAccountState`의 정체

`crates/storage/db-api/src/tables/mod.rs:390`

```rust
table PlainAccountState {
    type Key = Address;
    type Value = Account;
}
```

매크로 문법이고, 실제로는 **struct + `Table` 트레이트 구현**으로 펼쳐진다.

`crates/storage/db-api/src/table.rs:79`

```rust
pub trait Table: Send + Sync + Debug + 'static {
    const NAME: &'static str;   // 테이블 이름
    const DUPSORT: bool;        // 중복 키 허용 여부
    type Key: Key;              // ← 연관 타입
    type Value: Value;          // ← 연관 타입
}
```

```
Table 양식지:          PlainAccountState가 제출한 양식:
  NAME  [    ]   →       NAME  "PlainAccountState"
  Key   [    ]   →       Key   Address
  Value [    ]   →       Value Account
```

### 이게 왜 강력한가

`crates/storage/db-api/src/transaction.rs:28`

```rust
fn get<T: Table>(&self, key: T::Key) -> Result<Option<T::Value>, DatabaseError>;
                             ^^^^^^                      ^^^^^^^^
```

인자 타입도 반환 타입도 `T`에 따라 **자동으로 결정된다.** 함수는 하나인데 테이블마다 타입이 다르다.

```rust
tx.get::<PlainAccountState>(addr)   // Key=Address, Value=Account   → Option<Account>
tx.get::<Bytecodes>(hash)           // Key=B256,    Value=Bytecode   → Option<Bytecode>

tx.get::<PlainAccountState>(some_b256)                          // ❌ 컴파일 에러 (Key 불일치)
let x: Option<Bytecode> = tx.get::<PlainAccountState>(addr)?;   // ❌ 컴파일 에러 (Value 불일치)
```

> **핵심**: 테이블은 값이 아니라 타입이고, 각 테이블 타입이 자기 Key/Value를 연관 타입으로
> 들고 있어서, 잘못된 키·값 조합은 컴파일 단계에서 걸린다.

### `get` vs `get_by_encoded_key`

```rust
fn get<T: Table>(&self, key: T::Key) -> ...;
fn get_by_encoded_key<T: Table>(&self, key: &<T::Key as Encode>::Encoded) -> ...;
```

`<T::Key as Encode>::Encoded` = "T의 Key 타입을 `Encode` 자격으로 봤을 때의 `Encoded` 타입"
= **키를 바이트로 인코딩한 결과 타입**.

`Address`, `B256`처럼 **자기 자신이 곧 바이트인 타입**은 인코딩이 항등이라, 복제 없이 참조로
바로 넘기려는 최적화가 `get_by_encoded_key`다.

---

## 3. MDBX 구현부

`crates/storage/db/src/implementation/mdbx/tx.rs:293`

```rust
fn get_by_encoded_key<T: Table>(
    &self,
    key: &<T::Key as Encode>::Encoded,
) -> Result<Option<T::Value>, DatabaseError> {
    self.execute_with_operation_metric::<T, _>(Operation::Get, None, |tx| {
        tx.get(self.get_dbi::<T>()?, key.as_ref())
            .map_err(|e| DatabaseError::Read(e.into()))?
            .map(decode_one::<T>)
            .transpose()
    })
}
```

**트레이트(약속)와 구현(실제)이 만나는 지점.** §2가 "이런 함수가 있다"는 선언이었다면 여기는
MDBX가 그걸 실제로 어떻게 하는지다.

### 타입 변환 사슬

```rust
tx.get(dbi, key)                               // Result<Option<Bytes>, mdbx::Error>
  .map_err(|e| DatabaseError::Read(e.into()))  // Result<Option<Bytes>, DatabaseError>  에러 타입만 교체
  ?                                            // Option<Bytes>                          여기서 벗김
  .map(decode_one::<T>)                        // Option<Result<Account, DatabaseError>>
  .transpose()                                 // Result<Option<Account>, DatabaseError>
```

- **`map_err`는 벗기지 않는다.** 에러 타입만 갈아끼운다. 벗기는 건 `?`
- `map_err`가 필요한 이유: `?`는 함수 반환 타입의 에러로 **변환 가능할 때만** 통과시킨다.
  `mdbx::Error` → `DatabaseError`로 먼저 바꿔놔야 `?`가 작동한다.
  즉 **`map_err`는 `?`를 쓰기 위한 준비 작업**
- `.transpose()`: `Option<Result<T,E>>` ↔ `Result<Option<T>,E>` 뒤집기. 함수 반환 타입에 맞추려고

### `|tx| { ... }` — 클로저

익명 함수. `execute_with_operation_metric`은 "**하고 싶은 일을 함수로 넘겨줘, 내가 시간 재고
실행해줄게**"라는 구조. reth 전반에 반복되는 패턴이다.

### `get_dbi::<T>()`

`T::NAME`(테이블 이름 상수)을 꺼내서, 해당 이름의 테이블 핸들을 Tx 캐시에서 찾고, 등록돼
있지 않으면 DB를 열어 dbi를 받아온다. → **타입 정보(`T::NAME`)가 런타임 핸들로 바뀌는 지점.**

---

## 4. `impl<TX: DbTx>`가 RW에도 적용되는 이유

`DbTxMut`은 `DbTx`의 하위 트레이트가 **아니다.** 둘은 완전히 독립적이다.

`crates/storage/db-api/src/transaction.rs:21, 52`

```rust
pub trait DbTx: Debug + Send { ... }
pub trait DbTxMut: Send { ... }        // DbTx를 상속하지 않음
```

그런데도 읽기 impl이 RW provider에 걸리는 이유는 **연관 타입의 바운드** 때문이다.

`crates/storage/db-api/src/database.rs:15`

```rust
pub trait Database {
    type TX: DbTx + ...;
    type TXMut: DbTxMut + DbTx + TableImporter + ...;
                          ^^^^
}
```

`Database`가 "`TXMut` 자리에 넣을 타입은 `DbTxMut`**과** `DbTx`를 둘 다 구현해야 한다"고
요구한다. 그래서 실제 RW 트랜잭션 타입은 두 트레이트를 모두 만족하고, `impl<TX: DbTx>` 블록이
걸린다.

**트레이트 상속이 아니라 계약 조항**이라는 게 중요하다. 상속이면 구조가 고정되지만, 바운드는
필요하면 조정할 수 있고 `DbTxMut` 자체는 다른 맥락에서 독립적으로 쓸 수 있다.
(`EthChainSpec<Header = ...>`와 같은 계열의 장치)

---

## 5. `PlainAccountState` vs `HashedAccounts`

| 테이블 | Key | Value | 정렬 기준 |
|---|---|---|---|
| `PlainAccountState` | `Address` | `Account` | 주소 순 |
| `HashedAccounts` | `B256` (= `keccak256(address)`) | `Account` | **해시 순** |

### 주소를 해싱하는 건 reth의 선택이 아니다

이더리움 MPT는 규격상 **secure trie** — 트리 키가 `keccak256(address)`여야 한다.
이유는 **depth 쏠림 방지**다. 해시는 분포가 균일하므로 트리 깊이가 고르게 유지된다.
해싱이 없으면 공격자가 공통 접두사를 가진 주소를 대량 생성해 특정 경로를 비정상적으로 깊게
만드는 DoS가 가능하다.

### reth가 정한 것

프로토콜이 "트리 키 = 해시"라고 정했으니, **아예 해시를 키로 삼은 테이블을 따로 두자**는 게
reth의 판단이다.

MDBX는 B+tree라 키 순서대로 정렬 저장된다. `HashedAccounts`는 해시 순으로 평평하게 정렬돼
있고, 이 순서가 **MPT를 조립할 때 필요한 순회 순서와 정확히 일치**한다.

- `HashedAccounts` 사용 → state root 계산 시 **순차 스캔 한 번**으로 트리를 아래에서 위로 조립
- `PlainAccountState` 사용 → 조회할 때마다 해싱 + **재정렬 부하**

### 조립된 트리도 테이블에 저장된다

`crates/storage/db-api/src/tables/mod.rs:484`

```rust
table AccountsTrie {
    type Key = StoredNibbles;        // 니블 경로가 곧 키
    type Value = BranchNodeCompact;  // 트리 노드 자체
}
```

MPT 노드가 테이블에 그대로 들어간다. 즉 **`HashedAccounts`(정렬된 리프 데이터) →
`AccountsTrie`(조립된 노드)** 라는 두 단계가 테이블 구조에 드러나 있다.

MPT 탐색은 니블 단위로 내려가며 각 단계에서 노드를 DB에서 읽는 반복이므로, 리프 접근이
O(log N)이 된다.

---

## 6. 오늘의 수확

1. **테이블 = 타입.** 연관 타입 덕에 키/값 타입 불일치가 컴파일 에러가 된다.
   어제 배운 연관 타입이 실제 효용으로 나타나는 첫 지점.
2. **`map_err`는 에러 타입 변환, `?`가 벗기기.** 둘은 역할이 다르다.
3. **`DbTx`/`DbTxMut`은 상속 관계가 아니다.** RO/RW 양쪽에 읽기 impl이 걸리는 건
   `Database::TXMut`의 바운드 때문.
4. **저장 형식이 두 벌인 이유**는 state root 계산 비용. 프로토콜 제약(secure trie)을
   저장 레이아웃 설계로 흡수한 사례.

## 7. 다음으로 넘길 질문

1. storage v1 → v2 전환은 언제/어떻게 일어나는가? 두 형식이 공존하는 기간이 있나?
2. `latest.rs` / `historical.rs`의 `basic_account`는 이 경로와 어떻게 다른가?
   (historical = 과거 블록 시점의 계정 조회 → changeset을 되감는 방식일 것으로 추정)
3. `execute_with_operation_metric`이 재는 메트릭은 어디서 소비되는가?
4. 이 경로 어디에도 RocksDB가 안 나온다. 계정 조회는 RocksDB를 안 쓰는가,
   아니면 다른 진입점에서 갈리는가? (→ 목요일 데이터 흐름)
