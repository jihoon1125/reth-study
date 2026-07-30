# Rust 개념 노트

> reth 읽으면서 만난 문법을 계속 쌓는 파일. 주차별 분석 문서와 분리해서 관리한다.

---

## 제네릭 / 트레이트 바운드

```rust
struct Foo<T>            // T는 아직 안 정해진 타입 자리
struct Foo<T: Display>   // T에 조건(트레이트 바운드)
```

조건이 있어야 `T`로 뭔가 할 수 있다. 조건이 없으면 그 타입에 대해 아무것도 못 한다.

여러 조건은 `+`로 연결: `T: Clone + Debug + Send + Sync + 'static`

`'static`은 라이프타임인데, 지금은 **"빌려온 참조가 없어서 얼마든지 오래 살 수 있는 타입"**
정도로만 알아두면 됨.

---

## 연관 타입 (associated type) ★

트레이트 안에 선언하는 **"타입 구멍"**. 구현체가 채운다.

```rust
pub trait NodeTypes {
    type Primitives: NodePrimitives;   // 구멍 + 그 구멍에 걸린 조건
}
```

읽는 법:

```
type Primitives            → "Primitives라는 타입 구멍이 있다"
          : NodePrimitives → "단, 그 구멍을 채우는 타입은 NodePrimitives를 구현해야 한다"
```

일반 제네릭 `<T: Display>`와 **문법 위치만 다르고 의미는 같다.**

비유 — 트레이트가 **양식지**, 연관 타입이 그 **빈칸**:

```
NodeTypesWithDB 양식지:        이더리움 노드가 제출한 양식:
  ChainSpec: [    ]      →       ChainSpec: [ ChainSpec     ]
  Primitives:[    ]      →       Primitives:[ EthPrimitives ]
  DB:        [    ]      →       DB:        [ DatabaseEnv   ]
```

### `N::DB`

"N이 자기 `DB` 구멍에 채워 넣은 타입". 덕분에 `ProviderFactory` 코드 한 벌로 이더리움용,
OP Stack용, 테스트용 노드를 전부 커버한다.

### `<DB as Database>::TX`

"DB를 `Database` 트레이트 **자격으로** 볼 때의 `TX` 타입".

왜 이렇게 장황하냐면, `DB`가 여러 트레이트를 구현하고 그중 둘 이상이 `TX`라는 연관 타입을
가지면 컴파일러가 어느 쪽인지 모르기 때문. `as`로 모호함을 없앤다.

### equality constraint — `Trait<Assoc = X>`

```rust
type ChainSpec: EthChainSpec<Header = <Self::Primitives as NodePrimitives>::BlockHeader>;
```

분해:

```
type ChainSpec:                          "ChainSpec 구멍이 있고"
  EthChainSpec<                          "EthChainSpec을 구현해야 하는데"
    Header = <Self::Primitives as NodePrimitives>::BlockHeader
  >                                      "단, 그것의 Header 타입이
                                          Primitives의 BlockHeader와 같아야 한다"
```

`Header = X`는 **연관 타입을 특정 타입으로 못박는 것**. 하는 말은:

> "체인 스펙이 말하는 헤더 타입과 프리미티브가 말하는 헤더 타입이 서로 달라선 안 된다."

이더리움 헤더용 프리미티브에 OP Stack 체인 스펙을 끼워 넣는 조합을 **컴파일 단계에서 막는다.**
reth 전체에 이 패턴이 깔려 있음.

---

## 연관 타입 바운드 ≠ 트레이트 상속 ★

혼동하기 쉬운 두 가지.

```rust
// (a) 트레이트 상속 — 구조가 고정됨
pub trait NodeTypesWithDB: NodeTypes { ... }
//                       ^ "NodeTypesWithDB를 구현하려면 NodeTypes도 반드시 구현해야 한다"

// (b) 연관 타입 바운드 — 계약 조항. 조정 가능
pub trait Database {
    type TXMut: DbTxMut + DbTx + ...;
//              ^^^^^^^^^^^^^^ "TXMut 자리에 넣을 타입은 이것들을 구현해야 한다"
}
```

`DbTxMut` 자체는 `DbTx`를 상속하지 **않는다**. 두 트레이트는 독립적이다.
그런데 `Database::TXMut`의 바운드가 둘 다 요구하므로, 실제 RW 트랜잭션 타입은 양쪽을 만족한다.

→ 그래서 `impl<TX: DbTx> ... for DatabaseProvider<TX, N>` (읽기 구현)이 RW provider에도 걸린다.

상속이면 구조가 못박히지만, 바운드는 계약이라 조정 가능하고 각 트레이트를 독립적으로 쓸 수 있다.

---

## ★ GAT (Generic Associated Type)

**자기 제네릭 파라미터를 가진 연관 타입.** 빈칸이 함수처럼 인자를 받는다.

```rust
type TX;                 // 빈칸 하나
type Cursor<T: Table>;   // "테이블 T를 주면 그에 맞는 커서 타입을 내놓는 빈칸"
```

```rust
// db-api/src/transaction.rs:23
pub trait DbTx: Debug + Send {
    type Cursor<T: Table>: DbCursorRO<T> + Send;
    fn cursor_read<T: Table>(&self) -> Result<Self::Cursor<T>, DatabaseError>;
}

// db/src/implementation/mdbx/tx.rs:286
impl<K: TransactionKind> DbTx for Tx<K> {
    type Cursor<T: Table> = Cursor<K, T>;    // 한 줄로 테이블 29개 전부
}
```

Rust 1.65(2022-11) 안정화. `docs/design/database.md:5`가 *"using Rust **Stable** GATs"* 라고
언어 기능 이름을 명시한 이유 — **그 전에는 이 아키텍처가 표현 불가능했다.**

### 없으면 어떻게 되나 — 대안 3개 전부 못 씀

| | 타입 안전성 | 성능 | 실사용 |
|---|---|---|---|
| `Box<dyn DbCursorRO<T>>` | 유지 | ❌ `walk`류 상실 + vtable + 힙 | 불가 |
| 트레이트에 파라미터 (`DbTxCursor<T>`) | 유지 | ✅ 동일 | **바운드 지옥** |
| 커서 타입 소거 (바이트) | ❌ 상실 | 보통 | 설계 후퇴 |

### ★ `for<...>` HRTB는 라이프타임 전용이다

트레이트에 파라미터를 올리는 우회로가 죽는 지점.

```rust
impl<TX: DbTxCursor<tables::A> + DbTxCursor<tables::B> + ...> ...   // 만지는 테이블 전부 나열
impl<TX: for<'a> Foo<'a>> ...                                       // ✅ 라이프타임은 됨
impl<TX: for<T: Table> DbTxCursor<T>> ...                           // ❌ 문법 자체가 없음
```

> **GAT가 메운 구멍이 정확히 이것** — "모든 T에 대해 그 타입이 존재한다"를 트레이트 하나에 담는 것.

---

## ★ 메서드는 struct가 아니라 (타입, 트레이트) 쌍에 속한다

흔한 오해: "impl을 여러 개 하면 struct 아래에 메서드가 쫘라락 평평하게 붙는다."

```
❌ 잘못된 그림                        ✅ 실제
struct Tx<RW>                        Tx<RW>
  ├─ cursor_read()  ← 트레이트 A        ├─ [트레이트 A] 서랍 ─ cursor_read()
  ├─ cursor_read()  ← 트레이트 B        ├─ [트레이트 B] 서랍 ─ cursor_read()
  └─ ...  이름 충돌!                    └─ [DbTx]     서랍 ─ get(), commit()
```

**서랍이 다르니 이름이 겹쳐도 된다.** 이미 쓰고 있는 증거:

```rust
u64::from(1u8)      // <u64 as From<u8>>::from
u64::from(1u16)     // <u64 as From<u16>>::from   ← u64에 from이 십수 개 있다
```

`tx.method()` / `Type::method()`는 **후보가 하나일 때만 작동하는 문법 설탕**이다.
애매하면 UFCS로 서랍을 지정한다: `<Type as Trait<X>>::method(&v)`

### 추론이 되는 조건 — 파라미터가 시그니처에 나타나는가

```rust
trait From<T>       { fn from(value: T) -> Self; }
//         ^                        ^ T가 인자에 있음 → 추론 가능

trait DbTxCursor<T> { fn cursor_read(&self) -> Result<Self::Cursor, E>; }
//               ^                   ^^^^^ T가 어디에도 없음 → 추론 불가
```

같은 문제를 겪는 표준 예가 `Into`:

```rust
let x: u64 = 1u8.into();   // ✅ 좌변이 단서
let x = 1u8.into();        // ❌ type annotations needed
```

**GAT는 파라미터를 트레이트가 아니라 연관 타입·메서드 쪽에 두어 서랍을 하나로 유지한다.**
그래서 `tx.cursor_read::<HashedAccounts>()` 한 줄로 끝난다.

---

## turbofish `::<T>`

함수에 **값이 아니라 타입**을 넘기는 문법.

```rust
tx.get_by_encoded_key::<tables::PlainAccountState>(address)
                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ turbofish
```

### 타입을 "설명서"로 쓰는 패턴 (reth의 테이블)

```rust
pub trait Table {
    const NAME: &'static str;
    type Key: Key;
    type Value: Value;
}

fn get<T: Table>(&self, key: T::Key) -> Result<Option<T::Value>, DatabaseError>;
//                           ^^^^^^                      ^^^^^^^^ T에 따라 자동 결정
```

테이블이 값이 아니라 **타입**이라, 함수 하나로 모든 테이블을 다루면서도 키/값 타입 불일치가
컴파일 에러가 된다.

```rust
tx.get::<PlainAccountState>(addr)       // Key=Address, Value=Account
tx.get::<Bytecodes>(hash)               // Key=B256,    Value=Bytecode
tx.get::<PlainAccountState>(some_b256)  // ❌ 컴파일 에러
```

---

## `?` vs `map_err` ★

```rust
tx.get(dbi, key)                               // Result<Option<Bytes>, mdbx::Error>
  .map_err(|e| DatabaseError::Read(e.into()))  // Result<Option<Bytes>, DatabaseError>
  ?                                            // Option<Bytes>
```

- **`map_err`** — 에러 타입만 갈아끼운다. **벗기지 않는다**
- **`?`** — 벗긴다. `Ok`면 내용물을 꺼내고, `Err`면 함수를 즉시 종료
- `?`는 함수 반환 타입의 에러로 **변환 가능할 때만** 통과시키므로,
  타입이 안 맞으면 `map_err`로 먼저 맞춰줘야 한다
  → **`map_err`는 `?`를 쓰기 위한 준비 작업**

## `.transpose()`

`Option<Result<T, E>>` ↔ `Result<Option<T>, E>` 뒤집기.

```rust
opt_bytes.map(decode)   // Option<Result<Account, E>>  "있으면 디코딩했는데 실패했을 수도"
         .transpose()   // Result<Option<Account>, E>  "실패면 에러, 성공이면 Option"
```

## 클로저 `|x| { ... }`

익명 함수.

```rust
self.execute_with_operation_metric(op, None, |tx| { tx.get(...) })
//                                            ^^^^^^^^^^^^^^^^^^^ 넘겨진 일감
```

"하고 싶은 일을 함수로 넘겨줘, 내가 감싸서(시간 재고/락 잡고) 실행해줄게" 구조.

---

## 공유와 가변

| 표현 | 의미 |
|---|---|
| `Arc<T>` | 여러 스레드가 공유 소유 (C++ `shared_ptr`). 값은 못 바꿈 |
| `RwLock<T>` | 읽기 여럿 or 쓰기 하나 |
| `Arc<RwLock<T>>` | 공유 + 가변 |

**읽는 요령: 필드에 락이 붙었나 안 붙었나 = 그 데이터가 런타임에 변하나 안 변하나.**

---

## 트레이트 객체 `dyn Trait`

```rust
reader_txn_tracker: Option<Arc<dyn ReaderTxnTracker>>
```

- **제네릭 `impl<TX: DbTx>`** — 컴파일 타임에 타입 확정, 타입마다 코드 생성(정적 디스패치)
- **`dyn Trait`** — 런타임에 함수 포인터를 따라감(동적 디스패치)

`dyn`을 쓰는 전형적 이유: 그 타입을 제네릭 파라미터로 들고 있지 않아서 타입으로 표현할 수 없을 때.
(`DatabaseProvider<TX, N>`에는 `DB` 파라미터가 없다)

### 오브젝트 안전성과 `where Self: Sized`

모든 트레이트를 `dyn`으로 만들 수 있는 건 아니다. 제네릭 메서드나 `impl Trait` 인자가 있으면
vtable을 만들 수 없다. 그런 메서드에 `where Self: Sized`를 붙이면 **트레이트는 오브젝트 안전해지지만
그 메서드는 `dyn`에서 호출할 수 없다.**

```rust
// db-api/src/cursor.rs:39-49
fn walk(&mut self, start_key: Option<T::Key>) -> Result<Walker<'_, T, Self>, DatabaseError>
where Self: Sized;                                    // ← dyn에서 호출 불가

fn walk_range(&mut self, range: impl RangeBounds<T::Key>) -> ...
where Self: Sized;                                    // impl Trait 인자 = 제네릭 메서드
```

→ `Box<dyn DbCursorRO<T>>`는 **컴파일은 되지만 `walk`/`walk_range`/`walk_back`을 잃는다.**
GAT 없이 커서를 트레이트 오브젝트로 만들 수 없는 이유 중 하나.

## blanket impl

```rust
impl<DB: Database> ReaderTxnTracker for DB { ... }
```

특정 타입 하나가 아니라 **"`Database`를 구현한 모든 타입"** 에 한꺼번에 구현. 담요를 덮듯이.

> **바운드는 능력을 요구하고, blanket impl은 능력을 공급한다.** 짝을 이루는 기법.

## let-else

```rust
let Some(sync_state) = &self.read_only_sync else { return Ok(()) };
```

`Some`이면 꺼내서 바인딩, 아니면 `else` 실행. `else` 블록은 **반드시 함수를 벗어나야** 한다
(return/break/panic). 중첩 없이 조기 반환하는 관용구.

## Mutex — 데이터 보호 vs 작업 직렬화

```rust
sync_lock: Mutex<()>    // 안이 비어 있다
```

`Mutex<T>`가 항상 데이터를 보호하는 건 아니다. `Mutex<()>`는 **"이 구간을 한 번에 한 스레드만"**
(상호배제) 만 강제하는 용도.

### 오염(poisoning)

```rust
lock().unwrap_or_else(|e| e.into_inner())
```

러스트 `Mutex`는 락을 쥔 스레드가 패닉하면 **오염** 상태가 되고, 이후 `lock()`이 `Err`를 반환한다.
`into_inner()` = "오염 무시하고 데이터 줘". 보호할 데이터가 없는 `Mutex<()>`라면 적절한 처리.

## 원자 변수와 `Ordering`

`Ordering`은 **"이 원자 연산 주변의 다른 메모리 연산까지 순서를 보장할 것인가"** 를 정한다.

| | 보장 |
|---|---|
| `Relaxed` | 이 변수 자체의 원자성만. torn read 없음. 주변 순서 보장 X |
| `Acquire`/`Release` | 발행/구독 시점에 **주변 메모리 변경도 함께** 보이게 |

`Relaxed`가 위험한 전형적 패턴:

```
A:  데이터를 씀 → flag = true (Relaxed)
B:  flag == true 확인 → 데이터 읽음 → ❌ 아직 옛 데이터일 수 있음
```

**`Relaxed`가 안전하려면 그 값이 "틀려도 되는 값"이어야 한다.** 정확성은 락이 책임지고,
원자 변수는 성능 필터 역할만 하는 구조라면 `Relaxed`로 충분하다.
(예: reth의 `sync_providers_if_needed` — double-checked locking)

### `Relaxed`가 보장하는 것을 과소평가하지 말 것

`Relaxed`라고 **없던 값이 튀어나오지는 않는다.** 원자 변수에는 **modification order**(그 변수에
일어난 저장들의 순서)가 있고, `Relaxed` 로드는 그 순서에 **실제로 존재했던 값 중 하나**만
반환한다. 미래 값도, 쓰레기 값도 안 나온다 (out-of-thin-air 금지).

그래서 "저장은 작업 완료 후에만 한다"는 규칙을 지키면,
**읽은 값 V ⟹ V까지의 작업이 실제로 완료됐다** 는 추론이 `Relaxed`로도 성립한다.

`Relaxed`에서 진짜 없는 것은 **가시성 보장**이다 — "V를 봤다 ⟹ V 이전의 메모리 쓰기도 보인다"가
성립하지 않는다. 따라서 원자 변수를 **데이터 발행 플래그로 쓰면 위험**하고,
발행은 다른 락/자료구조가 담당하고 원자 변수는 **힌트로만** 쓰면 안전하다.

## `unsafe impl Send` / `unsafe impl Sync`

"컴파일러야, 안전성은 내가 책임질 테니 통과시켜"라는 선언. 반드시 SAFETY 주석으로 근거를 남긴다.

```rust
// SAFETY: Access to the transaction is synchronized by the lock.
unsafe impl Send for TransactionPtr {}
unsafe impl Sync for TransactionPtr {}
```

**락이 있으니 `Sync`가 불필요한 게 아니라, 락이 있으니 `Sync`가 안전(sound)한 것이다.**
방향을 헷갈리기 쉬움.

### 표준 규칙 — `Mutex<T>`는 `T`가 `Sync`가 아니어도 `Sync`다

```rust
impl<T: ?Sized + Send> Sync for Mutex<T> {}     // T: Sync 요구 안 함
```

**"내부를 mutate하니까 `Sync`가 될 수 없다"는 정확히 거꾸로다.**
내부를 mutate하니까 → 락으로 감싸고 → **그 결과 `Sync`가 된다.** `Mutex`가 존재하는 이유가 이것.

MDBX 날 핸들(`*mut MDBX_txn`)은 스레드 안전하지 않다. 그래서 러스트 래퍼가 뮤텍스를 씌웠고,
씌운 결과 `Transaction<RO>`/`Transaction<RW>` 둘 다 `Send + Sync`다
(`libmdbx-rs/transaction.rs:724-730`의 `assert_send_sync` 컴파일 타임 단언).

### 바운드는 없앨 수 있지만 요구는 이동할 뿐이다

`DbTx`/`DbTxMut`에 `Sync`가 없는 건 위험해서가 아니라 **`chore(db): Remove Sync from DbTx`
(#20516, 2025-12-22)** 에서 뺐기 때문이다. 45개 파일을 건드린 바운드 최소화 리팩터링이었고
그 대가로 `#[auto_impl(&, Arc, Box)]`가 `(&, Box)`로 좁아졌다
(`Arc<T>: Sync`는 `T: Send + Sync`를 요구하므로).

그런데 `Database::TX`/`TXMut`에는 `Send + Sync`가 **그대로 남아 있다**(`db-api/database.rs:13-15`).

> **트레이트에서 요구를 빼도 사라지는 게 아니라, 진짜 필요한 곳으로 옮겨갈 뿐이다.**
> 트레이트는 자기 메서드가 요구하지 않는 바운드를 넣지 않는 게 원칙 (구현자 폭을 넓힌다).

## ★ 제네릭 컨텍스트에서는 선언된 바운드만이 진실이다

```rust
impl<TX: DbTx + DbTxMut + 'static, N: ...> DatabaseProvider<TX, N> {
//      ^^^^^^^^^^^^^^^^^^^^^^^^^ Sync 없음
```

실제로 들어올 구체 타입(`Transaction<RW>`)이 `Sync`여도, **바운드에 안 적었으면 컴파일러에게는
없는 성질**이다. 그래서 이 impl 안에서는 `&self`를 rayon `spawn`에 넘길 수 없다.

추론 사슬:

```
TX가 Sync를 모름
  → DatabaseProvider<TX,N>의 auto Sync 불성립 (필드 tx: TX 때문)
  → &DatabaseProvider<TX,N> 은 Send 아님   (&T: Send ⟺ T: Sync)
  → s.spawn(f)의 f: Send 불만족
  → 컴파일 에러
```

### `&T: Send ⟺ T: Sync`

외워둘 것. "참조를 다른 스레드로 보낼 수 있다" = "여러 스레드가 동시에 참조해도 된다".

### disjoint closure capture (Rust 2021)

클로저는 `self` 통째가 아니라 **실제로 쓰는 필드만** 캡처한다. 그래서 이런 우회가 가능하다:

```rust
// avoid capturing &self.tx in scope below.
let sf_provider = &self.static_file_provider;   // Sync인 필드만 미리 빼둔다
s.spawn(|_| { sf_provider.write(...) });        // tx는 캡처되지 않음
```

## `scope` — 클로저가 바깥 변수를 수정할 수 있는 이유

```rust
let mut sf_result = None;
pool.in_place_scope(|s| {
    s.spawn(|_| { sf_result = Some(...); });   // 바깥 변수를 가변 대여
});
// 여기서 sf_result를 읽음
```

`scope`는 **"스코프가 끝날 때까지 모든 스폰 작업이 끝난다"** 를 타입으로 보장하므로 컴파일러가
대여를 허용한다. `std::thread::spawn`은 언제 끝날지 모르니 이게 안 된다.

- 스코프 클로저가 `Result`를 반환하게 만들면 그 안에서 `?`를 그대로 쓸 수 있다
  (`Ok::<_, ProviderError>(())` + `})?`)

### ★ `scope` vs `in_place_scope` — 차이는 `Send` 바운드다

```rust
// rayon-core/src/thread_pool/mod.rs:283-316
pub fn scope<'scope, OP, R>(&self, op: OP) -> R
where OP: FnOnce(&Scope<'scope>) -> R + Send,   // ← Send 필요
      R: Send,
{ self.install(|| scope(op)) }                   // 클로저 본문을 풀 스레드로 옮겨 실행

pub fn in_place_scope<'scope, OP, R>(&self, op: OP) -> R
where OP: FnOnce(&Scope<'scope>) -> R,           // ← Send 불필요
{ do_in_place_scope(Some(&self.registry), op) }  // 클로저 본문을 현재 스레드에서 실행
```

| | 클로저 본문이 실행되는 곳 | `OP: Send` |
|---|---|---|
| `ThreadPool::scope` | 풀 안으로 **진입해서** | 필요 |
| `ThreadPool::in_place_scope` | **현재 스레드**에서 그대로 | 불필요 |

~~"호출 스레드도 일꾼으로 쓴다"~~ 는 부정확하다. 정확히는 **"클로저 본문을 현재 스레드에서
실행하고 풀에 합류하지 않는다"**. (스폰 작업을 전부 기다린 뒤 반환하는 건 **둘 다 같다** —
차이점이 아님)

**왜 이게 중요하냐**: `save_blocks`가 `scope`를 썼다면 바깥 클로저가 `Send`여야 하고
→ `&self`가 `Send`여야 하고 → `TX: Sync`가 필요한데 바운드에 없다 → **컴파일 에러**.
`in_place_scope`는 `Send`를 요구하지 않아 이 문제가 아예 생기지 않는다.

## `#[derive(EnumIs)]`

`is_database()`, `is_static_file()` 같은 variant 판별 함수를 자동 생성.

## `extern "C" fn`

C 라이브러리가 러스트 함수를 **거꾸로 호출**하는 콜백. 예: libmdbx가 "느린 리더 발견했는데
어떻게 할까?"를 러스트에 묻는 `handle_slow_readers`.

---

## 타입 별칭 vs newtype

```rust
type A = B;      // 그냥 줄임말(typedef). 새 타입이 아님
struct A(B);     // 새 타입(newtype). B와 다른 타입으로 취급됨
```

newtype에 `Deref`/`DerefMut`를 구현하면 `a.0.foo()` 대신 `a.foo()`로 쓸 수 있다.
→ 감싸긴 했지만 사용감은 안 감싼 것처럼 만드는 트릭.

### 실례 — `Address`(newtype 2겹) vs `B256`(별칭 1겹)

```rust
// alloy-primitives/src/bits/macros.rs:74 — wrap_fixed_bytes! 매크로가 생성
#[repr(transparent)]
pub struct Address(pub FixedBytes<20>);        // newtype

// alloy-primitives/src/bits/fixed.rs:36
pub struct FixedBytes<const N: usize>(pub [u8; N]);

// alloy-primitives/src/aliases.rs:65-86
pub type B256 = FixedBytes<32>;                // 별칭! 새 타입 아님
```

```
Address ──.0──▶ FixedBytes<20> ──.0──▶ [u8; 20]      → self.0 .0
B256 (== FixedBytes<32>)        ──.0──▶ [u8; 32]      → self.0
```

`Encode` 구현이 `self.0 .0` / `self.0`로 미묘하게 다른 이유가 **겹수 차이**다.

### `#[repr(transparent)]`

"메모리 레이아웃이 안쪽 필드와 **100% 동일**하다"는 선언. 그래서 `Address`와 `[u8; 20]`은
메모리에서 구분 불가능한 20바이트다.

```rust
impl Encode for Address {
    type Encoded = [u8; 20];
    fn encode(self) -> Self::Encoded { self.0 .0 }    // 껍데기만 벗김. 바이트 이동 0
}
```

`db-api/transaction.rs:29-31`의 주석 *"raw keys **that encode to themselves** like Address and B256"*
이 이 얘기다. 컴파일 후 명령어가 0개다.

대조군 (`db-api/models/accounts.rs:122-131`) — 얘는 진짜 인코딩:

```rust
fn encode(self) -> [u8; 52] {
    let mut buf = [0u8; 52];              // 새 버퍼 할당
    buf[..20].copy_from_slice(...);       // 진짜 memcpy
    buf[20..].copy_from_slice(...);
    buf
}
```

### deref coercion — `&Address`를 `&[u8; 20]` 자리에 넘기기

```rust
// provider.rs:1468 — 인자 타입은 &<T::Key as Encode>::Encoded = &[u8; 20]
self.tx.get_by_encoded_key::<tables::PlainAccountState>(address)   // address: &Address
```

캐스팅이 아니라 **컴파일러의 자동 역참조**다. `Address`와 `FixedBytes`가 둘 다 `Deref`를
파생해서 `&Address` → `&FixedBytes<20>` → `&[u8; 20]`로 **두 단계** 강제 변환된다.

> `get`(소유권 `T::Key`)과 `get_by_encoded_key`(참조 `&Encoded`) 두 개가 있는 이유:
> `encode(self)`가 소유권을 먹으므로 `&Address`만 있으면 복사가 필요한데,
> 후자는 참조를 그대로 넘겨 복사 0. 계정 조회는 블록마다 수만 번 일어난다.

---

## `#[auto_impl(&, Arc, Box)]`

매크로. "`T`가 이 트레이트를 구현하면 `&T`, `Arc<T>`, `Box<T>`도 자동으로 구현되게 해줘".

---

## Future / Pin (async — 나중에 다시)

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>
```

- **`self: Pin<&mut Self>`** — `&mut self`와 문법 구조는 같고 self의 **타입만** 명시한 것.
  "이동하지 않음이 타입으로 보장된 `&mut self`"
- **`cx: &mut Context<'_>`** — 준비되면 깨워달라고 등록하는 waker를 담은 상자.
  `'_`는 라이프타임 생략 표시로 특별한 의미 없음
- **`Poll<T>`** — `enum { Ready(T), Pending }`

전체 의미: "이 퓨처를, 절대 이동시키지 않는다는 전제 하에 한 번 확인해봐. 결과 나왔으면
`Ready(값)`, 아직이면 `cx`의 waker에 등록해두고 `Pending`."

### 자기참조가 뭔지

"퓨처 객체가 자기 자신을 참조"가 아니라,
**"퓨처 struct의 한 필드가 같은 struct의 다른 필드 주소를 들고 있는 것"**이 정확한 표현.

```rust
struct SelfReferential {
    value: String,                    // 필드 A
    pointer_to_value: *const String,  // 필드 B — 필드 A를 가리킴
}
```

`async fn` 안에서 `let x = ...; let r = &x; something().await; use(r);` 라고 쓰면 컴파일러가
이걸 struct 하나로 변환하는데, `r` 필드가 `x` 필드의 주소를 담게 된다.

### 왜 이동이 문제인가

러스트의 **이동(move)은 재계산이 아니라 바이트 복사(memcpy)**다. struct 전체가 옮겨지면
`x`는 새 주소로 잘 가지만, `r`에 저장된 "예전 x의 주소"라는 **숫자값은 그대로 복사만 될 뿐**
고쳐지지 않는다. → 깨진 참조.

### 함수를 다시 호출하는 건 문제 없음

| 상황 | 주소 재계산? |
|---|---|
| 함수를 처음부터 다시 호출 (`example()` 두 번) | **됨** — 매번 새 인스턴스라 자기참조도 새로 만들어짐 |
| 이미 만들어진(아직 실행 안 끝난) 퓨처를 다른 변수/`Vec`/`Box`로 옮김 | **안 됨** — 이동은 바이트 복사라 내부 포인터가 옛 주소를 그대로 들고 있음 |

**Pin이 하는 일**: 한 번 자리 잡은 퓨처 인스턴스를 그 실행이 끝날 때까지 못 옮기게 막는 것.
이동을 금지하면 "이동 후 주소 안 맞는" 상황 자체가 발생할 수 없다.

---

## MDBX 계층 구조

```
MDBX (엔진 = libmdbx. LMDB의 포크. mmap 기반 임베디드 KV 저장소)
 └─ Environment    ← DB 파일 전체를 대표하는 핸들. ProviderFactory의 `db` 필드
     └─ Transaction ← 한 번의 읽기/쓰기 세션. DatabaseProvider의 `tx` 필드
         └─ Cursor  ← 트랜잭션 안에서 테이블을 순회하는 도구
```

- 관련 크레이트: `crates/storage/libmdbx-rs`(안전한 러스트 래퍼),
  `mdbx-sys`(원본 C 라이브러리 FFI 바인딩)
- **MDBX 정책: 읽기 트랜잭션 여러 개 동시 가능 + 쓰기 트랜잭션은 한 번에 하나**
- 이 정책이 러스트 타입으로 반영된 게 `Database::TX` vs `Database::TXMut`

비유: **MDBX는 건물, Tx는 그 건물에 들어가 볼일 보는 한 번의 방문.**
`DatabaseProvider`가 건물 전체가 아니라 "지금 이 방문"을 들고 있는 이유는, 트랜잭션이 끝나면
락도 풀리고 자원도 정리돼야 하기 때문.

### 타입 치환 사슬 — `self.tx`가 결국 뭔가

```
self.tx : TX                                   provider.rs:189      [제네릭 이름]
   = Tx<RW>                                    mdbx/mod.rs:263      DatabaseEnv::TXMut
        └ inner: Transaction<RW>               libmdbx-rs:61
             └ Arc<TransactionInner<RW>>
                  └ txn: TransactionPtr        libmdbx-rs:542
                       ├ *mut ffi::MDBX_txn    ← 진짜 핸들
                       └ lock: Arc<Mutex<()>>  ← 트랜잭션당 1개
```

`TX`(제네릭 이름)와 `Tx<RW>`(구체 타입)는 **다른 게 아니라 같은 값**이다.

### 이름: `tx` vs `txn` vs `txnid`

| 표기 | 뜻 | 쓰이는 곳 |
|---|---|---|
| `txn` | DB 트랜잭션 — **C API 어휘** (`MDBX_txn`, `mdbx_txn_begin`) | `libmdbx-rs` 안쪽, FFI 근처 |
| `tx` (DB 문맥) | DB 트랜잭션 — reth 상위 어휘 | `DbTx`, `self.tx`, `tx.get()` |
| `tx` (이더리움 문맥) | **이더리움 트랜잭션** | `TxNumber`, `TxHash`, `tx_nums` |
| `txnid` | 트랜잭션 **ID(u64)** — MVCC 스냅샷 버전 | `oldest_reader_txnid`, `last_txnid` |

**판별 요령**: 뒤에 뭐가 붙으면(`tx_num`, `TxHash`) 이더리움 트랜잭션, 혼자 서서 메서드를
부르면(`self.tx`, `tx.get()`) DB 트랜잭션. `save_blocks` 첫 줄에 둘이 동시에 나온다:

```rust
let first_tx_num = self.tx.cursor_read::<tables::TransactionBlocks>()?  // provider.rs:597
//      ^^^^^^^^        ^^ DB 트랜잭션
//      └ 이더리움 트랜잭션 번호
```

**이름이 `tx` → `txn`으로 바뀌는 지점이 러스트/C 경계다.**

### ★ 스레드 종속성 — reth는 명시적으로 끈다

```rust
// libmdbx-rs/src/flags.rs:202 — 조건 없이 항상
flags |= ffi::MDBX_NOSTICKYTHREADS;
```

MDBX C 소스(`mdbx.c:2615-2623`)가 설명한다:

> The slot's address is saved in **thread-specific data** so that subsequent read transactions
> started by the same thread need no further locking. **If `MDBX_NOSTICKYTHREADS` is set, the
> slot address is not saved in thread-specific data.**

**`NOSTICKYTHREADS` = "트랜잭션을 스레드에 묶지 마라".** 그래서 `Tx<RO>`/`Tx<RW>` 둘 다
`Send + Sync`다. 대가로 매 begin마다 `lck_rdt_lock` 경합이 생기는데, 그걸 완화하려고
`ReadTxnPool`(리셋된 핸들 256개 재사용 큐, `txn_pool.rs`)이 있다.

### 뮤텍스는 트랜잭션당 하나 — RO 동시성의 조건

```rust
impl TransactionPtr {
    fn new(txn: *mut ffi::MDBX_txn) -> Self {
        Self { txn, lock: Arc::new(Mutex::new(())) }   // 트랜잭션 만들 때마다 새 뮤텍스
    }
}
```

| | 결과 |
|---|---|
| RO 트랜잭션을 **여러 개** 열기 | ✅ 완전 병렬. 각자 다른 뮤텍스. 제한은 reader slot(기본 126) |
| **하나의** 트랜잭션을 여러 스레드에서 | ❌ 직렬화. `get`/`put`/커서 전부 같은 락을 지나감 |

reth는 당연히 전자다 — RPC 요청마다 자기 `Tx<RO>`를 연다. `ReadTxnPool`의 존재
(*"Under high concurrency (e.g., prewarming), this becomes a contention point"*)가 그 증거.

`begin_ro_txn`(대기·재시도 없음)과 `begin_rw_txn`(*"will block while there are any other
read-write transactions open"*, `Error::Busy`면 250ms 자며 재시도)의 대비가 정책을 그대로 보여준다.

### 경합을 문제로 취급한다는 증거

```rust
// libmdbx-rs/transaction.rs:579-592
fn lock(&self) -> MutexGuard<'_, ()> {
    if let Some(lock) = self.lock.try_lock() { lock }
    else {
        tracing::trace!(target: "libmdbx",
            backtrace = %std::backtrace::Backtrace::capture(),   // ← 백트레이스를 뜬다
            "Transaction lock is already acquired, blocking...");
        self.lock.lock()
    }
}
```

같은 트랜잭션을 여러 스레드가 건드리는 건 **설계상 일어나선 안 되는 일**로 취급한다.

### `TxnManager` — 전용 스레드 하나

`begin_rw_txn`은 FFI를 직접 부르지 않고 `mdbx-rs-txn-mgr` 스레드에 메시지를 보내 대기한다
(`txn_manager.rs:66-114`). 반면 `begin_ro_txn`은 **호출 스레드에서 직접** `mdbx_txn_begin_ex`를
부른다(`transaction.rs:72-83`). writer 락 획득을 한 곳에 모으려는 구조로 보인다.