# 파이프라인과 stage — 그리고 리오르그가 그걸 안 쓰는 이유

> 12주차 월요일 보충. 8~11주차 내내 `stages` 크레이트 이름과 stage별 트레이트 바운드는 봤지만
> **파이프라인이 실제로 뭘 하는 물건인지는 모르고 있었다.** Engine API를 보려면 그 반대편에
> 뭐가 있는지 알아야 해서 먼저 세웠다.

---

## 1. reth 자신의 설명

📖 `crates/stages/api/src/pipeline/mod.rs:48-66`

```rust
/// A staged sync pipeline.
///
/// The pipeline executes queued [stages][Stage] serially. An external component determines
/// the tip of the chain and the pipeline then executes each stage in order from the current
/// local chain tip and the external chain tip. When a stage is executed, it will run until
/// it reaches the chain tip.
///
/// After the entire pipeline has been run, it will run again unless asked to stop.
///
/// # Unwinding
///
/// In case of a validation error (as determined by the consensus engine) in one of the stages,
/// the pipeline will unwind the stages in reverse order of execution.
```

네 문장이 전부다.

| 문장 | 뜻 |
|---|---|
| *"executes queued stages **serially**"* | 동시에 안 돌린다. 하나 끝나면 다음 |
| *"An **external component** determines the tip"* | 목표는 외부(합의 계층)가 알려준다 |
| *"it will run **until it reaches the chain tip**"* | ★ 한 stage가 tip까지 다 하고 나서 다음 stage |
| *"unwind … in **reverse order**"* | 되감기는 역순 |

세 번째가 "staged sync"라는 이름의 뜻이다.

```
❌ 블록 단위가 아니다
   블록1: 헤더→바디→실행   블록2: 헤더→바디→실행   …

✅ stage 단위다
   Headers:   블록 1 ~ 1,000,000 헤더 전부
   Bodies:    블록 1 ~ 1,000,000 바디 전부
   Execution: 블록 1 ~ 1,000,000 실행 전부
```

---

## 2. stage는 트레이트 하나다

📖 `crates/stages/api/src/stage.rs:241-298` (군더더기 제거)

```rust
pub trait Stage<Provider>: Send {
    /// Get the ID of the stage.
    fn id(&self) -> StageId;

    /// Execute the stage.
    fn execute(&mut self, provider: &Provider, input: ExecInput) -> Result<ExecOutput, StageError>;

    /// Unwind the stage.
    fn unwind(&mut self, provider: &Provider, input: UnwindInput) -> Result<UnwindOutput, StageError>;
}
```

필수 함수가 셋. 나머지(`poll_execute_ready`, `post_execute_commit`)는 기본 구현이 있다.

### `unwind`가 필수라는 게 중요하다

세 함수 다 **본문 없이 세미콜론으로 끝난다** = 구현 필수. 만약 `unwind`에 기본 구현이
있었다면, 새 stage를 만든 사람이 잊어도 **컴파일이 통과한다.** 그리고 되감기가 필요한 순간까지
아무도 모른다.

10주차에 본 것과 같은 구조다 — `{ .. }`가 "의도적으로 안 쓴다"는 선언이라 경고가 없었다.
**기본 구현도 "안 해도 된다"는 선언이다.**

그리고 되감기를 못 하면 단순히 "복구 불가"가 아니라 **더 나쁜 상태**가 된다. unwind는 stage
15개를 역순으로 전부 돌아야 완결되는데, 중간 하나가 실패하면 앞의 몇 개만 되감긴 채 멈춘다.
**stage마다 체크포인트가 다른 높이에 남는 것** — 9주차 [error-paths.md](../week9/error-paths.md)
§4에서 판 "저장소들이 서로 다른 지점에 멈춘" 상태다.

### 실물 — 제일 단순한 stage 전체

📖 `crates/stages/stages/src/stages/finish.rs`

```rust
/// The finish stage.
///
/// This stage does not write anything; it's checkpoint is used to denote the highest fully
/// synced block.
#[derive(Default, Debug, Clone)]
#[non_exhaustive]
pub struct FinishStage;

impl<Provider> Stage<Provider> for FinishStage {
    fn id(&self) -> StageId { StageId::Finish }

    fn execute(&mut self, _provider: &Provider, input: ExecInput) -> Result<ExecOutput, StageError> {
        Ok(ExecOutput { checkpoint: StageCheckpoint::new(input.target()), done: true })
    }

    fn unwind(&mut self, _provider: &Provider, input: UnwindInput) -> Result<UnwindOutput, StageError> {
        Ok(UnwindOutput { checkpoint: StageCheckpoint::new(input.unwind_to) })
    }
}
```

**stage 하나가 실제로 이만큼이다.** 구조체는 필드가 없고(`struct FinishStage;`) 함수 셋이 전부.

`_provider`에 밑줄이 붙었다 — **이 stage는 provider를 안 쓴다.** 아무것도 읽지도 쓰지도 않고,
**오직 체크포인트를 찍는 게 목적**이다.

🎓 `#[non_exhaustive]` — 외부 크레이트에서 `FinishStage {}`처럼 직접 만들지 못하게 막는다.
나중에 필드를 추가해도 남의 코드가 안 깨진다. 10주차 "유닛 → 구조체 변형은 브레이킹"과 같은
문제를 미리 막는 장치.

---

## 3. `ExecInput` / `ExecOutput` — 체크포인트가 뼈대다

```rust
input.target()      // "어디까지 해라"    ← 파이프라인이 알려줌
input.checkpoint()  // "지난번에 어디까지 했나"

ExecOutput {
    checkpoint: StageCheckpoint::new(...),  // "여기까지 했다"
    done: true,                            // "목표까지 다 했다"
}
```

`done: false`를 돌려주면 파이프라인이 같은 stage를 다시 부른다. 그런데 **핵심은 반복마다
커밋한다는 것**이다.

📖 `crates/stages/api/src/pipeline/mod.rs:280-306` (`loop` 안)

```rust
loop {
    let prev_checkpoint = self.provider_factory.get_stage_checkpoint(stage_id)?;   // ① DB에서 읽기
    ...
    let provider_rw = self.provider_factory.database_provider_rw()?;               // ② 새 쓰기 트랜잭션

    match self.stage(stage_index).execute(&provider_rw, exec_input) {
        Ok(out @ ExecOutput { checkpoint, done }) => {
            // Update stage checkpoint.
            provider_rw.save_stage_checkpoint(stage_id, checkpoint)?;              // ③
            // Commit processed data to the database.
            provider_rw.commit()?;                                                // ④
```

**왜 배치로 자르나:**

- 100만 블록을 한 트랜잭션에 담으면 MDBX 트랜잭션이 거대해진다 (8주차
  [concurrency.md](../week8/concurrency.md)에서 본 MVCC 비용 — 오래 살수록 DB 파일이 커진다)
- 중간에 크래시가 나면 전부 날아간다

배치로 자르면 각 배치가 독립적으로 커밋되고, 크래시가 나도 마지막 커밋된 체크포인트부터
재개한다. ①에서 DB에서 체크포인트를 **다시 읽는** 것도 그래서다 — 메모리 변수가 아니라
디스크에 남은 값을 신뢰한다.

> `save_stage_checkpoint`와 데이터가 **같은 트랜잭션**에서 커밋되는 게 핵심이다. 따로 커밋되면
> "데이터는 들어갔는데 체크포인트는 안 올라간" 상태가 생긴다.

---

## 4. stage 15개

📖 `crates/stages/types/src/id.rs:45-62` — **이 배열 순서가 곧 실행 순서다.**

| # | stage | 하는 일 | 8~10주차 접점 |
|---|---|---|---|
| 1 | `Era` | ERA 파일(과거 블록 아카이브)에서 가져오기 | |
| 2 | `Headers` | 헤더를 tip까지 다운로드 | 9주차 `HeaderSyncGapProvider` 바운드가 이것 |
| 3 | `Bodies` | 바디(트랜잭션) 다운로드 | **10주차의 그 stage.** static file `Transactions`에 쓴다. `--prune.bodies.distance`가 이걸 지운다 |
| 4 | `SenderRecovery` | 서명에서 발신자 주소 복원 | 9주차 `sealed_block_with_senders`가 쓰는 데이터 |
| 5 | `Execution` | EVM 실행, 상태 변경 | |
| 6 | `PruneSenderRecovery` | 4번 결과 중 불필요한 것 정리 | |
| 7 | `MerkleUnwind` | 되감기 전용 머클 처리 | |
| 8 | `AccountHashing` | 계정 주소 해싱 | |
| 9 | `StorageHashing` | 스토리지 키 해싱 | |
| 10 | `MerkleExecute` | 상태 루트 계산 | |
| 11 | `TransactionLookup` | tx 해시 → 위치 인덱스 | 9주차 §2, **RocksDB를 요구하는 유일한 stage** |
| 12 | `IndexStorageHistory` | 스토리지 변경 이력 인덱스 | |
| 13 | `IndexAccountHistory` | 계정 변경 이력 인덱스 | |
| 14 | `Prune` | 설정에 따라 오래된 데이터 삭제 | 10주차 `delete_segment_below_block`이 여기서 불린다 |
| 15 | `Finish` | 체크포인트만 찍음 | |

**순서에 의존 관계가 박혀 있다.** `Bodies`(3) 없이 `SenderRecovery`(4)를 할 수 없고,
`Execution`(5) 없이 `AccountHashing`(8)을 할 수 없다.

9주차 [interfaces.md](../week9/interfaces.md)에서 stage별 트레이트 바운드를 표로 정리했는데,
**그 표의 각 행이 이 목록의 한 항목이었다.** 그때는 "stage"가 뭔지 모르고 바운드만 봤던 것.

---

## 5. 루프 두 겹

📖 `crates/stages/api/src/pipeline/mod.rs:223-256`

```rust
pub async fn run_loop(&mut self) -> Result<ControlFlow, PipelineError> {
    self.move_to_static_files()?;

    let mut previous_stage = None;
    for stage_index in 0..self.stages.len() {           // ← 배열 순회
        let next = self.execute_stage_to_completion(previous_stage, stage_index).await?;

        match next {
            ControlFlow::NoProgress { block_number } => { ... }
            ControlFlow::Continue { block_number } => self.progress.update(block_number),
            ControlFlow::Unwind { target, bad_block } => {
                self.unwind(target, Some(bad_block.block.number))?;
                return Ok(ControlFlow::Unwind { target, bad_block })
            }
        }

        previous_stage = Some(
            self.provider_factory.provider()?
                .get_stage_checkpoint(stage_id)?
                .unwrap_or_default()
                .block_number,
        );
    }
    Ok(self.progress.next_ctrl())
}
```

**`for stage_index in 0..self.stages.len()`** — 파이프라인의 정체가 이 한 줄이다.

`previous_stage`는 직전 stage가 도달한 지점이고, 다음 stage의 목표가 된다 (`:270`).

```rust
let target = self.max_block.or(previous_stage);
```

`max_block`이 우선한다 — CLI로 `--debug.max-block 1000`처럼 상한을 걸면 그게 이긴다.

그리고 바깥에 루프가 하나 더 (`:182`):

```rust
pub async fn run(&mut self) -> Result<(), PipelineError> {
    loop {
        let next_action = self.run_loop().await?;
        ...
    }
}
```

*"After the entire pipeline has been run, it will run again"* — 15개를 다 돌면 처음부터 다시.
그동안 체인이 더 자랐을 테니까.

### unwind는 `.rev()` 한 줄

📖 `crates/stages/api/src/pipeline/mod.rs:320`

```rust
let unwind_pipeline = self.stages.iter_mut().rev();
```

`Finish`(15) → `Prune`(14) → … → `Headers`(2) 순으로 각 stage의 `unwind`를 부른다.

역순인 이유는 **전진할 때의 의존 관계를 거꾸로 풀어야 하기 때문**이다. `Execution`(5)의 결과로
`AccountHashing`(8)이 만들어졌으니, `Execution`을 되감기 전에 `AccountHashing`을 먼저 되감아야
한다.

> **의존 관계를 따로 선언하는 자료구조가 없다.** 그래프도 위상 정렬도 없다.
> `StageId::ALL` 배열 순서 하나가 "정순 = 의존 순서, 역순 = 되감기 순서"를 동시에 표현한다.
> 그래서 새 stage를 추가할 때 **배열의 어디에 넣느냐가 곧 설계 결정**이다.

각 stage의 진행도가 다를 수 있으니 이미 목표 아래인 stage는 건너뛴다 (`:334-338`).

---

## 6. 리오르그란 무엇인가

### 체인은 하나가 아니다

밸리데이터들이 서로 다른 정보를 갖고 동시에 일하기 때문에 같은 부모 위에 블록이 둘 생길 수 있다.

```
시각 T   : 밸리데이터 A가 블록 101을 만들어 전파 시작
시각 T+ε : 네트워크가 느려서 B는 101을 아직 못 받음
시각 T+ε : B의 차례가 되어, B는 100 위에 또 101을 만듦
```

**둘 다 형식상 유효하다.** 부모도 맞고 서명도 맞고 실행도 된다.

```
                (101, 0xAA) ── (102, 0xAB)              ← 내가 따라가던 쪽
(100, 0x0A) ────┤
                (101, 0xBB) ── (102, 0xBC) ── (103, 0xBD)   ← 이긴 쪽
                ↑
          공통 조상 (fork point)
```

**번호가 겹친다** — 101이 둘, 102가 둘. 구별되는 건 해시뿐이다.

### 리오르그 = 믿었던 게 뒤집히는 것

내 노드는 0xAA를 먼저 받아 실행하고 상태를 바꿨고, 그 위 0xAB까지 처리했다. 그런데 합의 계층이
투표를 집계해보니 0xBB 쪽이 이겼다. 그러면:

1. **0xAA, 0xAB가 만든 상태 변경을 취소한다** — 잔액, 스토리지, 리시트 전부
2. **0xBB, 0xBC, 0xBD를 실행해서 적용한다**
3. 0xAA, 0xAB에 있던 트랜잭션은 처리 안 된 셈이므로 mempool로 돌려보낸다

**1번이 "되감기"다.**

### 코드 — 연장인가 갈아타기인가를 판정한다

📖 `crates/engine/tree/src/tree/mod.rs:920` `fn on_new_head(&self, new_head: B256)`

인자와 필드의 관계를 먼저 봐야 한다.

| | 무엇 |
|---|---|
| `new_head` (인자) | 합의 계층이 **방금 지시한** head. 아직 적용 안 됨 |
| `current_canonical_head` (필드) | 내 노드가 **이미 삼고 있는** head. 곧 교체될 것 |

`:953-960` — 먼저 단순 연장인지 확인한다.

```rust
// If we have reached the current canonical head by walking back from the target, then we
// know this represents an extension of the canonical chain.
if current_hash == self.state.tree_state.current_canonical_head.hash {
    new_chain.reverse();
    return Ok(Some(NewCanonicalChain::Commit { new: new_chain }))
}
```

새 head에서 부모를 따라 거슬러 올라가다 **현재 head를 만나면** 그냥 뒤에 붙은 것이다.
대부분이 이 경우다.

`:962-996` — 못 만나면 리오르그다.

```rust
// We have a reorg. Walk back both chains to find the fork point.
let mut old_chain = Vec::new();
let mut old_hash = self.state.tree_state.current_canonical_head.hash;

// If the canonical chain is ahead of the new chain, gather all blocks until new head number.
while current_canonical_number > current_number { ... }        // ① 높이 맞추기

// Walk both chains from specified hashes at same height until a common ancestor is reached.
while old_hash != current_hash { ... }                        // ② 나란히 거슬러 올라가기

Ok(Some(NewCanonicalChain::Reorg { new: new_chain, old: old_chain }))
```

**두 단계다** — ① 옛 가지가 더 길면 짧은 쪽 높이까지 내려오고, ② 같은 높이에서 나란히 거슬러
올라가며 만나는 지점을 찾는다.

### 타입이 두 경우를 구분한다

📖 `crates/chain-state/src/in_memory.rs:887-901`

```rust
pub enum NewCanonicalChain<N: NodePrimitives = EthPrimitives> {
    /// A simple append to the current canonical head
    Commit {
        new: Vec<ExecutedBlock<N>>,
    },
    /// A reorged chain consists of two chains that trace back to a shared ancestor block at
    /// which point they diverge.
    Reorg {
        /// All blocks of the _new_ chain
        new: Vec<ExecutedBlock<N>>,
        /// All blocks of the _old_ chain
        old: Vec<ExecutedBlock<N>>,
    },
}
```

**`Reorg`만 `old`를 들고 있다.** `Commit`은 붙이기만 하니 `new`만 있으면 되고, `Reorg`는
**버릴 것 목록이 필요하다.** 위의 1번(취소한다)의 재료가 그것이다.

### canonical head 위에는 디스크에 없다

`:936-939`

```rust
// Walk back the new chain until we reach a block we know about
//
// This is only done for in-memory blocks, because we should not have persisted any blocks
// that are _above_ the current canonical head.
while current_number > current_canonical_number {
```

`above` = **블록 번호가 더 큰 쪽**(최신 방향). 이 문장의 근거는 두 겹이다.

**(1) 디스크에 쓸 블록은 정본 체인에서만 고른다.**

`:2143` `get_canonical_blocks_to_persist`

```rust
let mut current_hash = self.state.tree_state.canonical_block_hash();
```

canonical head에서 부모를 따라 내려가며 모은다. 곁가지는 이 순회에 안 들어온다.

**(2) 목표는 canonical head보다 낮게 잡는다.**

`:2147-2152`

```rust
let target_number = match target {
    PersistTarget::Head => canonical_head_number,
    PersistTarget::Threshold => {
        canonical_head_number.saturating_sub(self.config.memory_block_buffer_target())
    }
};
```

`:534-541`

```rust
/// How many blocks the canonical tip is ahead of the last persisted block. A large gap means
/// persistence is falling behind execution.
const fn persistence_gap(&self) -> u64 {
    self.state.tree_state.canonical_block_number()
        .saturating_sub(self.persistence_state.last_persisted_block.number)
}
```

**"canonical tip이 last persisted block보다 얼마나 앞서 있나"** — 항상 0 이상이다.

따라서 `on_new_head`가 걷는 구간(번호가 현재 head보다 큰 구간)은 **아직 정본 체인에 들어온 적이
없으므로 디스크에 있을 수 없다.** 메모리(`blocks_by_hash`)만 찾으면 된다. 그 아래로 내려가면
디스크도 봐야 하니 `canonical_block_by_hash`로 바뀐다 (`:969`).

### 예외는 따로 처리한다 — disk reorg

`:2669-2671`

```rust
/// This method tries to detect whether on-disk and in-memory states have diverged. It might
/// happen if a reorg is happening while we are persisting a block.
fn find_disk_reorg(&self) -> ProviderResult<Option<u64>> {
```

**저장하는 중에 리오르그가 나면** 방금 정본이었던 블록이 디스크에 들어간 뒤 버려질 수 있다.

`:1451-1452`

```rust
if let Some(new_tip_num) = self.find_disk_reorg()? {
    self.remove_blocks(new_tip_num)
} else if self.should_persist() {
```

**지우는 게 먼저다.** 새로 저장하기 전에 잘못 들어간 것을 걷어낸다. 10주차
[measurement.md](../week10/measurement.md)의 "과잉은 잘라낼 수 있지만 결손은 복구할 수 없다"와
같은 처리 순서.

---

## 7. "뒤처진다"는 게 뭐가 뒤처지나

**내 노드가 처리 완료한 최신 블록 번호 vs 합의 계층이 지정한 head의 번호.**

```
합의 계층이 지정한 head : 23,500,000
내 노드 local tip       : 23,499,997
                          ─────────
                          3블록 뒤처짐
```

| 상황 | 뒤처짐 |
|---|---|
| 노드를 처음 켰다 | **2,300만 블록** |
| 3일 껐다 켰다 | 약 21,600 블록 (12초에 1블록) |
| 잠깐 네트워크 끊김 | 몇 블록 |
| 정상 운영 | 0~1블록 |

📖 `crates/engine/tree/src/tree/mod.rs:2575-2577`

```rust
const fn exceeds_backfill_run_threshold(&self, local_tip: u64, block: u64) -> bool {
    block > local_tip && block - local_tip > MIN_BLOCKS_FOR_PIPELINE_RUN
}
```

**인자 이름이 그대로 답이다** — `local_tip`(내 노드)과 `block`(목표).

`:98`

```rust
pub(crate) const MIN_BLOCKS_FOR_PIPELINE_RUN: u64 = EPOCH_SLOTS;   // = 32
```

**32블록.** 그보다 적게 뒤처지면 파이프라인을 안 돌린다.

`:2797-2809`

```rust
/// This handles downloaded blocks that are shown to be disconnected from the canonical chain.
fn on_disconnected_downloaded_block(...) -> Option<TreeEvent> {
    if let Some(target) = self.backfill_sync_target(head.number, missing_parent.number, ...) {
        trace!(target: "engine::tree", %target, "triggering backfill on downloaded block");
        return Some(TreeEvent::BackfillAction(BackfillAction::Start(target.into())));
    }
```

---

## 8. ★ 리오르그와 뒤처짐은 다른 축이다

|  | 리오르그 | 뒤처짐 |
|---|---|---|
| 상황 | **다른 가지**로 갈아타야 함 | **같은 가지**에서 앞으로 못 따라감 |
| 방향 | 후진 + 전진 | 전진만 |
| 규모 | 보통 1~2블록 | 몇 블록 ~ 수천만 블록 |
| 처리 | 엔진 트리가 메모리에서 (`NewCanonicalChain::Reorg`) | 32블록 초과면 파이프라인 (`BackfillAction::Start`) |

```
뒤처짐 —— 같은 사슬, 앞으로만
   내 위치 ●━━━━━━━━━━━━━━━━━━━━━━▶ ○ 목표

리오르그 —— 갈라진 지점까지 후진 후, 다른 가지로 전진
                    ┌─── ○ ── ○  새 가지 (전진)
   공통조상 ●───────┤
                    └─── ● ── ●  내가 있던 가지 (후진해서 버림)
```

### 그래서 리오르그에 파이프라인을 안 쓴다

파이프라인은 stage 15개를 순서대로 돌리고 각 stage가 블록 범위 전체를 훑는 기계다. 2300만
블록을 따라잡을 때 쓰는 구조다.

리오르그는 2블록이다. 여기다 그 기계를 돌리면 stage 15개를 역순 순회하며 체크포인트를 읽고
되감고, 다시 15개를 정순으로 돌려 2블록을 전진해야 한다. **12초 안에 끝내야 하는 일에 과잉이다.**

그리고 물리적으로도 필요 없다 — 리오르그가 건드리는 구간은 canonical head 근처이고,
**그 구간은 아직 메모리에 있다** (`memory_block_buffer_target` 안쪽). 디스크를 안 건드리니
파이프라인이 할 일이 없다.

### 세 용어 구분

| 말 | 무엇 | 위치 |
|---|---|---|
| **reorg** | 다른 가지로 갈아타기. 엔진 트리가 메모리에서 | `engine/tree/src/tree/mod.rs:920` |
| **pipeline unwind** | stage 15개를 역순으로 되감기. **기동 시 정합성 깨짐에만** | `stages/api/src/pipeline/mod.rs:303` |
| **disk reorg** | 저장 중 리오르그로 디스크에 잘못 들어간 것을 지우기 | `engine/tree/src/tree/mod.rs:2669` |

**9주차 정리가 절반만 맞았다** — 되감기는 맞지만 파이프라인 unwind가 아니다.

---

## 9. 오늘의 수확

1. **★ "staged"의 뜻은 stage 하나가 블록 범위 전체를 훑는다는 것이다.** 블록 단위로 돌지 않는다.
2. **★ `StageId::ALL` 배열 순서 하나가 의존 그래프다.** 정순은 실행 순서, 역순은 되감기 순서.
   따로 선언된 그래프도 위상 정렬도 없다.
3. **`unwind`가 트레이트 필수 함수다.** 기본 구현을 주면 "안 만든 것"이 조용히 통과한다.
   10주차 `{ .. }`와 같은 구조.
4. **`done: false`는 배치 처리를 위한 것이고, 배치마다 커밋된다.** 데이터와 체크포인트가 같은
   트랜잭션에서 커밋되는 게 핵심.
5. **★ 리오르그는 번호가 겹치는 두 가지 사이의 갈아타기다.** 그래서 forkchoice가 해시로 지정한다.
6. **★ 리오르그와 뒤처짐은 다른 축이다.** 전자는 엔진 트리 메모리, 후자는 32블록 초과 시 파이프라인.
7. **"canonical head보다 번호가 큰 블록은 디스크에 없다"** — 저장 대상을 정본 체인에서만 고르고,
   저장은 항상 canonical head보다 뒤처져 진행되기 때문. 예외는 `find_disk_reorg`가 따로 처리한다.

## 10. 다음으로 넘길 질문

1. `Bodies`(3)와 `TransactionLookup`(11) 사이에 stage가 7개 있다. 이 간격에 의존 관계가 실제로
   있나, 아니면 순서가 바뀌어도 되나?
2. `move_to_static_files()`가 `run_loop` 첫 줄에 있다 (`:224`). 왜 stage 순회 **전에** 부르나?
3. `ControlFlow::NoProgress`는 언제 나오나? `Continue`와 뭐가 다른가
4. `memory_block_buffer_target`과 9주차에 본 `--engine.persistence-threshold`는 같은 값인가
   다른 값인가
