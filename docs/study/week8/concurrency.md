# 동시성 — 읽기/쓰기가 동시에 일어날 때 무엇이 보장되는가

> 8주차 수요일. 논지: **MVCC는 MDBX 한 통 안에서만 공짜다. 저장소가 셋이면 그 경계를 넘는
> 일관성은 코드로 직접 지켜야 한다.**

---

## 1. MVCC가 공짜로 주는 것

MDBX는 MVCC(Multi-Version Concurrency Control) 기반이다.

```
읽기 트랜잭션 여러 개  ─┐
                        ├─ 서로 안 막음. 락 대기 없음
쓰기 트랜잭션 하나    ─┘
```

**원리**: 읽기 트랜잭션을 열면 그 순간의 **스냅샷**에 고정된다. 이후 writer는 기존 페이지를
덮어쓰지 않고 **새 페이지에 쓴다.** 그래서 reader는 자기가 보던 옛 페이지를 계속 안전하게 읽는다.

| | 락 기반 | MVCC |
|---|---|---|
| reader vs writer | 서로 대기 | **대기 없음** |
| reader가 보는 것 | 락 잡은 시점의 현재 값 | **트랜잭션 연 시점의 스냅샷** |
| 대가 | 대기 시간 | **옛 버전을 보관할 공간** |

`db.tx()`를 호출하는 그 순간이 **스냅샷이 찍히는 시점**이다. provider가 살아있는 동안 세상이
아무리 바뀌어도 그 provider는 항상 같은 시점을 본다.

### 월요일 §7-1 질문의 답

> "왜 MDBX 트랜잭션을 먼저 열고 나서 나머지를 동기화하나?"
> (`providers/database/mod.rs:386-390` 주석)

**트랜잭션을 여는 게 곧 시간을 못박는 행위이기 때문.** 못박은 다음에 다른 저장소를 그 시점에
맞춘다. 순서가 반대면 맞춰놓는 사이에 시점이 흘러가 버린다.

---

## 2. MVCC의 청구서

`crates/storage/db/src/implementation/mdbx/mod.rs:434`

```rust
const MAX_SAFE_READER_SPACE: usize = 10 * GIGABYTE;

extern "C" fn handle_slow_readers(..., space: usize, ...) -> HandleSlowReadersReturnCode {
    if space > MAX_SAFE_READER_SPACE {
        let message = if is_current_process(process_id as u32) {
            "Current process has a long-lived database transaction that grows the database file."
        } else {
            "External process has a long-lived database transaction ... \
             Use shorter-lived read transactions or shut down the node."
        };
        warn!(...);
    }
    HandleSlowReadersReturnCode::ProceedWithoutKillingReader
}
```

reader가 옛 스냅샷을 붙들고 있으면 MDBX는 그 스냅샷에 필요한 페이지를 **회수할 수 없다.**
reader가 오래 살수록 DB 파일이 계속 커진다. 10GB를 넘으면 경고.

관찰할 것:

1. **`extern "C" fn`** — libmdbx(C)가 러스트 함수를 거꾸로 호출하는 콜백. "느린 리더 발견,
   어떻게 할까?"를 러스트에 묻는다
2. **`ProceedWithoutKillingReader`** — reth의 정책 판단. 리더를 죽이지 않고 경고만 한다.
   다르게 정할 수도 있었다(죽이기, 에러)
3. **외부 프로세스도 범인일 수 있다** — MDBX는 파일 기반이라 다른 프로세스가 같은 DB를 열 수
   있다. 동시성 범위가 프로세스를 넘어간다

> `TXMut`(쓰기)만 희소 자원인 게 아니다. `TX`(읽기)도 공짜가 아니다.
> 대가의 종류가 다를 뿐 — **쓰기는 배타성, 읽기는 공간.**

---

## 3. ★ 저장소가 셋이면 MVCC가 안 통한다 — 커밋 순서

`providers/database/provider.rs:89`

```rust
pub enum CommitOrder {
    /// Normal commit order: static files first, then RocksDB, then MDBX.
    #[default]
    Normal,
    /// Unwind commit order: MDBX first, then RocksDB, then static files.
    /// Used for unwind operations to allow recovery by truncating static files on restart.
    Unwind,
}
```

**순서가 정반대인 모드가 두 개.** 각 저장소는 자기만의 커밋을 하고, 셋을 한 번에 원자적으로
커밋할 방법은 **없다.** 그래서 순서가 곧 크래시 복구 전략이 된다.

`providers/database/provider.rs:3890` — Normal 경로

```rust
self.static_file_provider.finalize()?;   // 1. static file
for batch in batches {
    self.rocksdb_provider.commit_batch(batch)?;   // 2. RocksDB
}
self.tx.commit()?;                       // 3. MDBX ← 마지막
```

### 왜 MDBX가 마지막인가 (전진)

MDBX는 **static file의 어느 위치에 블록 데이터가 있는지**를 들고 있다. 즉 MDBX가
**"어디까지가 진짜인지"의 기준점**이다.

```
static file 커밋 완료 ─── 💥 크래시 ─── MDBX 미커밋
결과: static file엔 새 데이터가 있지만 MDBX는 그걸 모름 → 고아 데이터로 무시/절단
```

앞의 것들이 다 성공해야 마지막이 찍히므로 **"MDBX 커밋 = 전체 커밋"** 이 성립한다.

### 왜 MDBX가 처음인가 (후진)

`provider.rs:263-266` 주석:

> "This keeps MDBX as the first durable step so an interrupted unwind can be recovered by
> **truncating static files from checkpoints** on the next startup."

되감을 때는 MDBX를 먼저 낮춘다. 크래시가 나도 재시작 시
"MDBX는 100번까지인데 static file엔 105번까지 있네 → 잘라내자"가 가능하다.
**자르는 건 언제든 다시 할 수 있는 안전한 복구 동작**이다.

### 하나의 원칙

> **전진할 때는 기준점을 마지막에, 후진할 때는 기준점을 처음에.**
> 양쪽 모두 크래시 후 남는 상태가 **과잉된 데이터**가 되게 만드는 것.
> 과잉은 잘라내면 되지만 **결손은 복구할 수 없다.**

---

## 4. 읽기 쪽 정합성 — double-checked locking

`providers/database/mod.rs:294`

```rust
pub fn sync_providers_if_needed(&self) -> ProviderResult<()> {
    let Some(sync_state) = &self.read_only_sync else { return Ok(()) };
    let current_txnid = self.db.last_txnid().unwrap_or(0);      // ← 락 밖에서 읽음

    // Fast path
    if current_txnid == sync_state.last_synced_txnid.load(Ordering::Relaxed) {
        return Ok(());
    }

    // Slow path: serialize the actual catch-up I/O.
    let _guard = sync_state.sync_lock.lock().unwrap_or_else(|e| e.into_inner());

    // Double-check
    if current_txnid == sync_state.last_synced_txnid.load(Ordering::Relaxed) {
        return Ok(());
    }

    self.rocksdb_provider.try_catch_up_with_primary()?;
    self.static_file_provider.initialize_index()?;
    sync_state.last_synced_txnid.store(current_txnid, Ordering::Relaxed);
    Ok(())
}
```

```
1. 락 없이 확인 (원자 변수)      ← 대부분의 호출이 여기서 끝남. 락 경합 0
2. 다를 때만 락 획득
3. 락 잡고 다시 확인              ← 기다리는 동안 다른 스레드가 이미 했을 수 있음
4. 진짜 필요할 때만 무거운 I/O
```

### 역할 분담 — 여기가 이 코드의 핵심

`sync_lock`은 `Mutex<()>`로 **안이 비어 있다.** 데이터를 보호하는 게 아니라
**"동기화 I/O 두 줄을 한 번에 한 스레드만"** 을 강제하는 용도(= 상호배제).

| | 역할 | 틀리면? |
|---|---|---|
| `last_synced_txnid` (원자 변수) | 빠른 필터 | 헛수고가 늘 뿐 |
| `sync_lock` (뮤텍스) | **진짜 상호배제** | 진짜 문제 |

### "동기화를 건너뛸 수도 있지 않나" — 아니다

저장하는 값이 **락 잡기 전에 읽은 `current_txnid`** 라는 데 주목.
그런데 `try_catch_up_with_primary()`는 **"지금 시점의 최신"** 까지 따라잡는다. 락을 기다리는
동안 DB가 더 진행됐다면 실제 따라잡은 지점은 `current_txnid`보다 앞서 있다.

```
실제 동기화된 지점  ≥  last_synced_txnid에 기록된 값
```

**기록값은 뒤처질 수는 있어도 앞설 수는 없다.**

| 상황 | 결과 |
|---|---|
| 기록값이 뒤처짐 | 다음 호출이 한 번 더 동기화 → 헛수고 |
| 기록값이 앞섬 | **구조상 불가능** |

즉 **"안 해도 되는 걸 한 번 더"는 생겨도 "해야 하는 걸 건너뛰기"는 안 생긴다.**

### `Ordering::Relaxed`로 충분한 이유

`Ordering`은 "이 원자 연산 주변의 **다른** 메모리 연산까지 순서를 보장할 것인가"를 정한다.

- `Relaxed` — 이 변수 자체의 원자성만. torn read는 없다. 주변 순서는 보장 안 함
- `Acquire`/`Release` — 발행/구독 시점에 주변 메모리 변경도 함께 보이게 보장

### "Relaxed 때문에 틀린 값을 읽어서 조건을 통과해버리는" 케이스는 없나

두 가지가 각각 독립적으로 막는다.

**(a) `Relaxed`는 "아무 값이나" 주지 않는다**

원자 변수에는 **modification order**(그 변수에 일어난 저장들의 순서)가 있고, `Relaxed` 로드는
**그 순서에 실제로 존재했던 값 중 하나**만 반환한다. 미래 값도 쓰레기 값도 안 나온다
(out-of-thin-air 금지).

그리고 이 변수에 저장이 일어나는 지점은 **딱 한 곳, sync 완료 직후**다. 따라서:

> 값 `V`를 읽었다 ⟹ 과거에 누군가 `V`를 저장했다 ⟹ **`V`까지의 sync가 실제로 완료된 적이 있다**

`V == current_txnid`로 통과했다면 진짜로 거기까지 동기화가 끝난 것이다. 오래된 값을 읽어도 이
성질은 깨지지 않는다. 오래된 값은 `current_txnid`보다 **작아서** 조건을 통과 못 하고 한 번 더
sync 할 뿐이다.

**(b) 효과의 가시성 — 이게 진짜 `Relaxed` 이슈**

`Relaxed`가 위험한 전형적 패턴:

```
스레드 A:  데이터를 메모리에 씀 → flag = true (Relaxed)
스레드 B:  flag == true 확인 → 데이터를 읽음 → ❌ 아직 옛 데이터일 수 있음
```

**여기선 이 패턴이 아니다.** sync가 만드는 결과물이 평범한 공유 메모리가 아니기 때문이다.

`providers/static_file/manager.rs:273`

```rust
pub struct StaticFileProviderInner<N> {
    map: DashMap<(BlockNumber, StaticFileSegment), LoadedJar>,   // 자체 동기화
    indexes: RwLock<StaticFileMap<StaticFileSegmentIndex>>,      // 자체 락
    ...
}
```

- `initialize_index()`의 결과 → `DashMap` / `RwLock` 안으로 들어간다
- `try_catch_up_with_primary()`의 결과 → RocksDB 내부 상태 (RocksDB 자체 동기화)

나중에 이 데이터를 실제로 읽을 때 **저 락들을 반드시 지나가고, happens-before가 거기서 성립한다.**

> **발행(publish) 메커니즘이 `last_synced_txnid`가 아니라 각 자료구조의 내부 락이다.**
> 원자 변수는 데이터를 넘겨주는 통로가 아니라 "굳이 시도할 필요 있나?"만 답하는 표지판.
> 표지판이 오래됐으면 헛걸음할 뿐, 표지판을 믿고 데이터를 읽는 구조가 아니다.

> **`Relaxed`가 안전한 건 "틀려도 되는 값"이기 때문.** 정확성은 전부 `sync_lock`과 락 안의
> 재확인이 책임진다. 락과 원자 변수의 역할을 섞지 않은 게 이 코드의 미덕.

---

## 5. Unwind의 배리어 — 왜 리더를 기다리나

`providers/database/provider.rs:273`

```rust
fn commit_unwind(self) -> ProviderResult<()> {
    let reader_txn_tracker = self.reader_txn_tracker.clone();
    self.tx.commit()?;                                    // 1. MDBX 먼저

    if let Some(reader_txn_tracker) = reader_txn_tracker.as_ref() {
        reader_txn_tracker.wait_for_pre_commit_readers(); // 2. ★ 기다린다
    }

    if storage_v2 { /* RocksDB 배치 커밋 */ }              // 3.
    self.static_file_provider.commit()?;                  // 4. static file 마지막
}
```

`crates/storage/db-api/src/database.rs:120`

```rust
pub trait ReaderTxnTracker: Send + Sync {
    /// Waits until all readers older than the latest committed txnid have drained.
    fn wait_for_pre_commit_readers(&self);
}

impl<DB: Database> ReaderTxnTracker for DB {
    fn wait_for_pre_commit_readers(&self) {
        if let Some(committed_txnid) = Database::last_txnid(self) {
            while Database::oldest_reader_txnid(self).is_some_and(|oldest| oldest < committed_txnid) {
                std::thread::sleep(std::time::Duration::from_millis(10));
            }
        }
    }
}
```

옛 스냅샷을 보던 리더가 전부 빠질 때까지 10ms 폴링으로 기다리는 **배리어**다.

### 문제는 "못 읽는 것"이 아니라 "읽히는 것"

리더가 잘려나간 데이터를 못 읽어서 **에러가 나면 오히려 안전하다.** RPC가 실패하고 끝.
진짜 위험은 **에러가 안 나고 엉뚱한 값이 읽히는 경우**다.

static file의 truncate는 **행(row) 단위**다 (`static_file/writer.rs:976`).
헤더의 길이를 줄이는 것이고, unwind 이후에는 **새 체인의 블록이 잘려나간 바로 그 행 위치부터
append 된다.**

```
1. 리더가 MDBX 스냅샷을 잡음
   → "105번 블록의 트랜잭션은 static file 5000~5010행"

2. unwind: static file을 100번 블록(=4900행)까지 자름

3. 새 체인 블록 101'~106'이 append → 5000~5010행에 '다른 체인의 트랜잭션'이 들어감

4. 리더가 이제서야 조회 → "5000~5010행 줘"
   → ✅ 성공 (행이 존재하므로)
   → 반환값: 리오그로 사라진 블록을 요청했는데 새 체인의 트랜잭션이 돌아옴
```

**에러가 안 난다.** 행 번호는 유효하고 내용물만 다르다.

### use-after-free와 같은 구조

| C | reth |
|---|---|
| 포인터 | MDBX 스냅샷이 든 "행 번호" |
| `free()` | static file truncate |
| 할당자가 그 블록 재사용 | 새 체인 블록이 같은 행에 append |
| 읽으면 → 크래시가 아니라 **남의 데이터** | 조회 성공 → **남의 체인 데이터** |

use-after-free가 무서운 이유가 "죽어서"가 아니라 "**안 죽고 조용히 틀린 값을 주니까**"인 것과
같다.

### MVCC가 왜 못 막나

```
MDBX 스냅샷 ──(행 번호)──▶ static file
   ↑                            ↑
버전 관리 O                  버전 관리 X
```

MVCC는 **MDBX 페이지**를 버전 관리한다. 그런데 그 페이지에 들어있는 건 static file을 가리키는
**인덱스**다. 인덱스의 옛 버전은 잘 보존되는데 **가리키는 대상은 버전이 없다.**

즉 MDBX 스냅샷은 **옛 시점의 유효한 포인터**인데 대상이 갈아엎어진 것. 그래서
`wait_for_pre_commit_readers`가 **옛 포인터를 든 스레드가 다 나갈 때까지 free를 미룬다.**
RCU의 grace period와 같은 구조다.

### 실제 피해

| 오답이 흘러가는 곳 | 결과 |
|---|---|
| RPC 응답 | 리오그로 사라진 트랜잭션을 "정상 조회됨"으로 반환. 지갑/거래소가 없는 트랜잭션을 확정으로 취급 |
| state root 계산 | 잘못된 데이터로 루트 계산 → 합의 불일치, 노드가 체인에서 이탈 |

에러였다면 둘 다 안 일어난다. 실패하고 재시도하면 되니까.

> **결손(에러)은 감당 가능하고, 오염(조용한 오답)은 감당 불가능하다.**
> §3의 "과잉은 잘라내면 되지만 결손은 복구할 수 없다"와 짝을 이룬다 —
> **크래시 복구는 결손을 피하고, 동시성 제어는 오염을 피한다.**

---

## 6. 오늘 나온 Rust 문법

- **let-else** — `let Some(x) = expr else { return ... };`
  `else` 블록은 반드시 함수를 벗어나야 한다(return/break/panic). 중첩 없는 조기 반환 관용구
- **`Mutex<()>`** — 안이 빈 뮤텍스. 데이터 보호가 아니라 **작업 직렬화**가 목적
- **`lock().unwrap_or_else(|e| e.into_inner())`** — 러스트 `Mutex`는 락을 쥔 스레드가 패닉하면
  **오염(poisoned)** 된다. `into_inner()`는 "오염 무시하고 데이터 줘".
  여기선 보호할 데이터가 없으니 적절
- **`Arc<dyn Trait>` (트레이트 객체)** — 구체 타입을 컴파일 타임에 모르고 런타임에 정함
  (동적 디스패치). `DatabaseProvider<TX, N>`에 `DB` 제네릭 파라미터가 없어서 타입으로 표현할
  수 없으므로 `dyn`으로 담았다
- **blanket impl** — `impl<DB: Database> ReaderTxnTracker for DB`
  "`Database`를 구현한 **모든** 타입에 한꺼번에 구현". 바운드가 능력을 **요구**한다면
  blanket impl은 능력을 **공급**한다

---

## 7. 다음으로 넘길 질문

1. `disable_long_read_transaction_safety`(`db-api/transaction.rs:48`)는 어떤 상황에서
   일부러 켜는가? MVCC 대가를 감수하겠다는 선언일 텐데 어디서 쓰이나
2. static file은 왜 MVCC를 안 하나 — 못 하나, 안 하는 게 나은가?
   (append-only + 불변 과거 데이터라는 성격과 관련 있을 것)
3. `try_catch_up_with_primary`의 "primary/secondary"는 RocksDB의 어떤 모드인가
4. 커밋 3단계 중간에 실패하면(크래시가 아니라 `?`로 에러 반환) 앞서 커밋된 것은 어떻게 되나
