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
