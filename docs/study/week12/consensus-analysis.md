# forkchoiceUpdated 한 번이 어디까지 가나

> 12주차 화요일. 스펙이 코드에서 어떻게 구현됐는지 추적. 그런데 추적하다가 **8~11주차에
> 몰랐던 개념이 계속 나왔다** — 다운로드가 P2P라는 것, stage가 언제 도는지, 리오르그가 되감기가
> 아니라는 것. 그것들을 먼저 정리한다.

---

## 1. 경로 한 장

```
engine_forkchoiceUpdatedV1  (JSON-RPC 요청, timeout 8s)
  │
  ├─ EngineApi::fork_choice_updated_v1        rpc-engine-api/src/engine_api.rs:315
  ├─ validate_and_execute_forkchoice          :899        ← §8.3 구현
  │
  ├─ BeaconConsensusEngineHandle::fork_choice_updated
  │                                           engine/primitives/src/message.rs:368
  ├─ [mpsc 채널] BeaconEngineMessage::ForkchoiceUpdated  ← ★ 스레드 경계
  │                                           message.rs:383
  │
  ├─ EngineApiTreeHandler::on_forkchoice_updated  engine/tree/src/tree/mod.rs:1175
  │     ① validate_forkchoice_state              :1216
  │     ② handle_canonical_head                  :1246
  │     ③ apply_chain_update                     :1308
  │     ④ handle_missing_block                   :1379  (fallback)
  │
  └─ canonical_in_memory_state.update_chain(...)  :1113
                                              ← 8~9주차에 판 그 인메모리
```

**마지막 줄이 이번 주의 목적이다.** 8~11주차는 `ConsistentProvider`가 인메모리를 먼저 읽는 쪽에서
봤고, 오늘은 그 인메모리를 **쓰는 쪽**에서 봤다.

```
[12주차] 엔진 트리                        [8~9주차] provider
   forkchoiceUpdated                          block()
        │  update_chain()                        │  읽기
        ▼                                        ▼
   ┌────────────────────────────────────────────────┐
   │          CanonicalInMemoryState                │
   └────────────────────────────────────────────────┘
```

그리고 provider 쪽 주석이 이미 엔진을 가리키고 있었다.

`storage/provider/src/providers/blockchain_provider.rs:997-1001`

```rust
/// NOTE: This cannot be called safely in a loop outside of the blockchain tree thread. This is
/// because the [`CanonicalInMemoryState`] could change during a reorg, causing results to be
/// inconsistent. Currently this can safely be called within the blockchain tree thread,
/// because the tree thread is responsible for modifying the [`CanonicalInMemoryState`] in the
/// first place.
```

*"the **tree thread** is responsible for modifying the `CanonicalInMemoryState`"* — 8~9주차에 읽던
파일이 12주차에 파는 스레드를 가리키고 있었다. 그때는 "blockchain tree thread"가 뭔지 몰라서
넘어갔다.

---

## 2. ★ 용어 — 여기서 제일 많이 헤맸다

### head ≠ header

| 말 | 무엇 | 타입 |
|---|---|---|
| **head** | **체인의 끝. 최신 블록을 가리키는 표식** | `BlockNumHash { number, hash }` |
| **header** | 블록의 머리 부분. `parentHash`, `stateRoot`, `timestamp` 등 | `SealedHeader` |

**둘은 아무 관계 없다.** 영어 단어가 비슷할 뿐이다.

```
체인:   블록 100 ── 블록 101 ── 블록 102
                                  ↑
                                 head (= tip)

블록 102의 내부:
   ┌─────────────────┐
   │ header          │ ← parentHash, stateRoot, timestamp …
   ├─────────────────┤
   │ body            │ ← transactions
   └─────────────────┘
```

한 문장에 둘 다 나오는 실례 — `stages/api/src/error.rs:45-48`

```
"stage encountered inconsistent chain: \
 downloaded header #{header_number} ({header_hash}) is detached from \
 local head #{head_number} ({head_hash})"
```

앞의 `header`는 방금 받은 블록의 헤더, 뒤의 `head`는 내 체인의 끝.

`head` 계열은 전부 "체인의 최신 블록"이다.

| 이름 | 어디 |
|---|---|
| `headBlockHash` | Engine API 스펙 — CL이 지정한 최신 블록 |
| `current_canonical_head` | `engine/tree/src/tree/state.rs:38` — 내가 삼고 있는 최신 블록 |
| `new_head` | `tree/mod.rs:920` — 새로 지정받은 최신 블록 |
| `local_head` / `local_tip` | 파이프라인이 아는 내 체인의 끝 |

> **`head`는 "어느 블록이냐"를 가리키는 포인터, `header`는 "그 블록 안에 뭐가 있냐"다.**
> 그래서 `current_canonical_head`의 타입이 데이터가 아니라 `BlockNumHash`(번호+해시)다.

### persisted = 디스크. 같은 말이다

- `persisted`는 reth의 코드 용어 (`last_persisted_block`, `PersistenceState`, `persistence_gap`)
- "디스크"는 설명할 때 쓰는 일상어

**같은 것을 가리킨다.** reth에서 "persistence"는 인메모리 블록을 MDBX + static file에 써서
영구 저장하는 작업의 이름이다.

### 함정 — `remove_persisted_blocks`는 디스크를 안 지운다

`chain-state/src/in_memory.rs:317-321`

```rust
/// Removes blocks from the in memory state that are persisted to the given height.
pub fn remove_persisted_blocks(&self, persisted_num_hash: BlockNumHash) {
```

*"Removes blocks **from the in memory state** that are persisted"* — **디스크에 잘 들어간 블록의
인메모리 사본을 치우는 것**이다. 정상 동작이고 데이터가 사라지는 게 아니다. 관리 주체가
메모리에서 디스크로 넘어가는 지점.

이름이 비슷한 함수 셋을 구분해둘 것.

| 이름 | 무엇을 지우나 | 언제 |
|---|---|---|
| `remove_persisted_blocks` | **메모리**에서 (디스크에 들어간 것) | 정상. persistence 완료 후 |
| `remove_blocks` | **디스크**에서 (리오르그로 무효가 된 것) | 예외. `find_disk_reorg` 감지 시 |
| `update_blocks`의 `reorged` 루프 | **메모리**에서 (버려진 가지) | 리오르그 시 |

### 다운로드 = P2P다. DB에서 불러오는 게 아니다

`stages/stages/src/stages/headers.rs:34-37`

```rust
/// The headers stage.
///
/// The headers stage downloads all block headers from the highest block in storage to
/// the perceived highest block on the network.
```

*"from the highest block in storage to the perceived highest block on the **network**"* — 내
저장소의 최신부터 네트워크가 알려주는 최신까지, **다른 노드들에게서 받아온다.**

`headers.rs:45-49`

```rust
pub struct HeaderStage<Provider, Downloader: HeaderDownloader> {
    /// Database handle.
    provider: Provider,
    /// Strategy for downloading the headers
    downloader: Downloader,
```

`provider`(내 DB)와 `downloader`(네트워크)를 둘 다 들고 있다. **네트워크에서 받아서 DB에 쓴다.**

reth에는 네트워킹 크레이트가 13개 있고 (`crates/net/`), 8~12주차에 한 번도 안 열었다.
계획서의 후보 B(devp2p)가 이것이다.

### 블록이 노드로 들어오는 경로가 셋

| 출처 | 어떻게 | 언제 |
|---|---|---|
| **합의 계층** | `engine_newPayload`로 **블록 전체를 준다** | 정상 운영 (1블록씩) |
| **P2P 피어** | 내가 요청해서 **받아온다** = 다운로드 | 대량 동기화, 빠진 부모 채우기 |
| 내 디스크 | 이미 갖고 있음 | 조회 |

**정상 운영 중에는 다운로드가 필요 없다.** CL이 블록을 통째로 넘겨준다 — 월요일에 본
`ExecutionPayloadV1`에 `transactions` 배열이 다 들어 있었다.

---

## 3. ★ 경로가 두 개다

```
【라이브 경로】 정상 운영 — 파이프라인 안 돔
  합의 계층
     │ engine_newPayload (블록 전체를 줌)
     ▼
  엔진 트리 ──► CanonicalInMemoryState (메모리)
     │
     │ 나중에, 버퍼를 넘으면
     ▼
  persistence ──► MDBX + static file


【대량 경로】 32블록 초과 뒤처짐 — 파이프라인 가동
  P2P 피어들
     │ 내가 요청해서 받아옴 (다운로드)
     ▼
  downloader (crates/net/downloaders)
     ▼
  Headers stage → Bodies stage → Execution stage → … (15개)
     │ 각 stage가 블록 범위 전체를 처리
     ▼
  MDBX + static file (직접 씀. 메모리 안 거침)
```

| | 라이브 경로 | 대량 경로 |
|---|---|---|
| 블록을 | **받는다** (CL이 줌) | **가져온다** (P2P 요청) |
| 처리 단위 | 블록 1개씩 | stage별로 블록 범위 전체 |
| 저장 | 메모리 먼저, 나중에 디스크 | 디스크에 바로 |
| 되감기 | 맵에서 remove | stage 역순 unwind |
| 도는 시점 | 항상 | **32블록 초과 뒤처졌을 때만** |

### stage는 파이프라인이 돌 때만 동작한다

`engine/tree/src/backfill.rs:20-30`

```rust
pub enum BackfillSyncState {
    /// The node is not performing any backfill synchronization.
    /// This is the initial or default state.
    #[default]
    Idle,
    Pending,
    Active,
}
```

**기본값이 `Idle`.** 파이프라인이 실제로 돌아가는 곳은 `backfill.rs:144` 한 줄이다.

```rust
let result = pipeline.run_as_fut(Some(target)).await;
```

`BackfillAction::Start`를 받았을 때만 불린다. **정상 운영 중에는 stage가 하나도 안 돈다.**

그리고 둘은 동시에 못 돈다 — `tree/mod.rs:1231-1236`

```rust
if !self.backfill_sync_state.is_idle() {
    // We can only process new forkchoice updates if the pipeline is idle, since it requires
    // exclusive access to the database
    trace!(target: "engine::tree", "Pipeline is syncing, skipping forkchoice update");
    return Ok(Some(OnForkChoiceUpdated::syncing()));
}
```

*"it requires **exclusive access to the database**"* — 파이프라인이 돌면 fcU를 안 받고 `SYNCING`을
돌려준다.

---

## 4. RPC → 엔진: 스레드가 갈린다

`engine/primitives/src/message.rs:383-392`

```rust
/// Sends a forkchoice update message to the beacon consensus engine and returns the receiver to
/// wait for a response.
fn send_fork_choice_updated(...) -> oneshot::Receiver<RethResult<OnForkChoiceUpdated>> {
    let (tx, rx) = oneshot::channel();
    let _ = self.to_engine.send(BeaconEngineMessage::ForkchoiceUpdated {
        state,
        ...
```

**함수를 직접 부르지 않고 메시지를 보낸다.**

```
RPC 스레드                          엔진 루프 (단일 스레드)
    │  to_engine.send(ForkchoiceUpdated)  │
    ├────────────────────────────────────▶│  처리
    │  rx.await                           │
    │◀────────────────────────────────────┤  tx.send(결과)
```

엔진은 상태(트리, 인메모리, 체크포인트)를 **단일 스레드에서만 고친다.** 8주차
[concurrency.md](../week8/concurrency.md)는 락으로 동시 접근을 막았는데, 여기는 **애초에 한
스레드만 만지게** 해서 락이 필요 없게 했다.

🎓 **두 종류의 채널을 조합하는 관용구**

| | 성질 | 용도 |
|---|---|---|
| `mpsc` | 여러 보내는 쪽 → 하나 받는 쪽. 계속 열림 | 요청 |
| `oneshot` | 딱 한 번만 보낼 수 있음 | 응답 |

**응답 받을 통로(`oneshot`의 `tx`)를 요청 메시지 안에 넣어 보낸다.** 답이 하나 오면 끝이므로
타입이 그것을 강제한다 — 두 번 보내려 하면 컴파일이 안 된다.

---

## 5. `on_forkchoice_updated`는 4단이다

`tree/mod.rs:1187-1204`

```rust
// Pre-validation of forkchoice state
if let Some(early_result) = self.validate_forkchoice_state(state)? { ... }

// Return early if we are on the correct fork
if let Some(result) = self.handle_canonical_head(state, &attrs)? { ... }

// Attempt to apply a chain update when the head differs from our canonical chain.
if let Some(result) = self.apply_chain_update(state, &attrs)? { ... }

// Fallback that ensures to catch up to the network's state.
self.handle_missing_block(state)
```

**마지막 줄에 `if let`이 없다** — 앞의 셋은 "내 담당이면 처리", 네 번째는 무조건 실행되는 fallback.

| # | 함수 | 상황 | 스펙 |
|---|---|---|---|
| 1 | `validate_forkchoice_state` | head 해시가 0 / 무효 조상 / 파이프라인 가동 중 | — / — / — |
| 2 | `handle_canonical_head` | 지정된 head가 이미 내 tip | §5 (`ensure_consistent_forkchoice_state`) |
| 3 | `apply_chain_update` | head가 트리에 있음 → 연장 또는 리오르그 | §2, §7 |
| 4 | `handle_missing_block` | **head를 아예 모름 → 다운로드** | **§1 + §9** |

### 4번이 스펙 두 조항을 한 번에 만족시킨다

`tree/mod.rs:1403-1407`

```rust
Ok(TreeOutcome::new(OnForkChoiceUpdated::valid(PayloadStatus::from_status(
    PayloadStatusEnum::Syncing,
)))
.with_event(TreeEvent::Download(DownloadRequest::single_block(target))))
```

| 스펙 | 이 코드 |
|---|---|
| fcU §1: *"**MAY** initiate a sync process if `headBlockHash` references an unknown payload"* | `TreeEvent::Download` |
| fcU §9: *"`{payloadStatus: {status: SYNCING, …}, payloadId: null}` if … unknown payload"* | `PayloadStatusEnum::Syncing` |

**MAY(동기화 시작)를 하면서 MUST(SYNCING 응답)를 지킨다.** 월요일에 정리한
"`SYNCING`은 타임아웃이 아니라 즉시 주는 답"의 구현이 이것 — 다운로드는 이벤트로 던져놓고
응답은 바로 돌려준다.

### 초기 fcU에서는 head가 아니라 safe로 먼저 간다

`tree/mod.rs:1388-1398`

```rust
// find the appropriate target to sync to, if we don't have the safe block hash then we
// start syncing to the safe block via backfill first
let target = if self.state.forkchoice_state_tracker.is_empty() &&
    !state.safe_block_hash.is_zero() &&
    self.find_canonical_header(state.safe_block_hash).ok().flatten().is_none()
{
    debug!(target: "engine::tree", "missing safe block on initial FCU, downloading safe block");
    state.safe_block_hash
} else {
    state.head_block_hash
};
```

`forkchoice_state_tracker.is_empty()` = fcU를 한 번도 못 받았다 = **노드를 막 켰다.**

월요일에 `safe`를 "RPC 블록 태그용"으로 정리했는데 **동기화 목표로도 쓰인다.** 스펙에 그런
말은 없다 → 대조표 후보.

---

## 6. §8.3은 RPC 계층에서 구현됐다

`rpc-engine-api/src/engine_api.rs:905-928`

```rust
if let Some(ref attrs) = payload_attrs {
    let attr_validation_res =
        self.inner.validator.ensure_well_formed_attributes(version, attrs);

    // From the engine API spec:
    //
    // Client software MUST ensure that payloadAttributes.timestamp is greater than
    // timestamp of a block referenced by forkchoiceState.headBlockHash. If this condition
    // isn't held client software MUST respond with -38003: Invalid payload attributes and
    // MUST NOT begin a payload build process. In such an event, the forkchoiceState
    // update MUST NOT be rolled back.
    //
    if let Err(err) = attr_validation_res {
        let fcu_res = self.inner.beacon_consensus.fork_choice_updated(state, None).await?;
        if fcu_res.is_invalid() || fcu_res.payload_status.is_syncing() {
            return Ok(fcu_res)
        }
        return Err(err.into())
    }
}

Ok(self.inner.beacon_consensus.fork_choice_updated(state, payload_attrs).await?)
```

**주석에 스펙 원문이 그대로 붙어 있다.**

구현 방식이 핵심이다.

```rust
let attr_validation_res = ...;                    // ① 검증하고 결과를 변수에 담아둠
if let Err(err) = attr_validation_res {
    ...fork_choice_updated(state, None).await?;   // ② attrs를 None으로 바꿔 그대로 실행
    return Err(err.into())                        // ③ 그 다음에 -38003
}
```

**"롤백하지 않는다"를 어떻게 지키나 — 애초에 롤백할 상황을 만들지 않는다.** attributes가
잘못됐으면 `None`으로 바꿔서 정본 갱신만 정상 수행하고, 그 다음 에러를 낸다.

### §4와 §8.3이 충돌 아닌 이유

시점이 다르다.

```
1단계  head payload 검증 → 실패면 갱신 안 함, 빌드 안 함     (§4)
2단계  forkchoice 갱신 — 원자적으로                          (§7)
3단계  payloadAttributes 검증 → 실패면 빌드만 안 함. 2단계 유지  (§8.3)
```

§4의 "실패"는 **블록 자체가 무효**. §8.3의 "실패"는 **빌드 요청이 잘못됨**(예: timestamp가
head보다 작음). 후자는 head의 유효성과 무관하므로 갱신을 유지한다.

에러 코드가 갈라진 것이 증거 — §5는 `-38002 Invalid forkchoice state`, §8.1은
`-38003 Invalid payload attributes`.

또 하나: `fcu_res.is_invalid() || is_syncing()`이면 `-38003`이 아니라 `fcu_res`를 돌려준다.
**head 자체가 문제일 때는 attributes 에러가 부차적**이기 때문이다. CL이 취할 행동이 다르다.

| 받은 응답 | CL이 할 일 |
|---|---|
| `-38003` | attributes를 고쳐서 다시 |
| `INVALID` | **head를 바꿔야 한다.** attributes를 고쳐도 소용없다 |
| `SYNCING` | **기다린다** |

---

## 7. ★ "갈아타야 한다"를 누가 판단하나 — 노드가 아니다

실행 계층에는 **"어느 가지가 옳은지"를 결정하는 코드가 없다.** 투표를 세는 로직도, 가지의
무게를 비교하는 로직도 없다.

```
[합의 계층]  투표 집계 → "0xBB가 이겼다"
     │  engine_forkchoiceUpdated({ headBlockHash: 0xBB, ... })
     ▼
[reth]      "알겠다. 0xBB로 갈아탄다"
```

| | 무엇을 판단 |
|---|---|
| `newPayload` | **유효한가** — reth가 판단 |
| `forkchoiceUpdated` | **무엇을 따를까** — CL이 판단, reth는 통보받음 |

reth가 `INVALID`를 돌려주면 CL이 다른 가지를 고를 뿐이고, **갈아탈지 말지는 여전히 CL이 정한다.**

### 코드에서 인지하는 지점

`tree/mod.rs:935-960`

```rust
// Walk back the new chain until we reach a block we know about
while current_number > current_canonical_number {
    if let Some(block) = self.state.tree_state.executed_block_by_hash(current_hash).cloned() {
        current_hash = block.recovered_block().parent_hash();   // 부모로 한 칸
        current_number -= 1;
        new_chain.push(block);
    } else { ... return Ok(None) }
}

// If we have reached the current canonical head by walking back from the target, then we
// know this represents an extension of the canonical chain.
if current_hash == self.state.tree_state.current_canonical_head.hash {
    new_chain.reverse();
    return Ok(Some(NewCanonicalChain::Commit { new: new_chain }))
}
```

**`if current_hash == current_canonical_head.hash` — 이 한 줄이 인지 지점이다.**

| 결과 | 뜻 |
|---|---|
| 같다 | 새 head가 내 head의 후손 → **연장** (`Commit`) |
| 다르다 | 같은 높이에 다른 블록 → **갈아타기** (`Reorg`) — `:962` 주석 `// We have a reorg.` |

**옳고 그름의 판단이 아니라 그래프 도달 가능성 계산이다.** 함수 문서에도 그렇게 나온다
(`:914-919`): *"Returns the new chain for the **given** head"* — head는 주어지는 것.

진입점이 둘인데 둘 다 CL에서 온다.

| 위치 | 문맥 | head 출처 |
|---|---|---|
| `:1354` | `apply_chain_update` — fcU 처리 중 | `state.head_block_hash` (방금 받음) |
| `:1984` | `make_canonical` — 백필 완료 후 | `forkchoice_state_tracker.sync_target_state()` (이전에 받아 저장) |

> **노드는 "내 체인이 잘못됐다"고 판단하지 않는다.**
> **"지시받은 head가 내 현재 head의 후손이 아니다"를 발견할 뿐이다.**
>
> "잘못됨"이라는 개념 자체가 실행 계층에 없다. 있는 것은 둘 — **유효/무효**(블록 하나에 대한
> 판정)와 **도달 가능/불가능**(두 해시 사이의 그래프 관계). "어느 가지가 진짜 체인이냐"는
> 세 번째 질문이고, 그 답은 합의 계층에서 정해져 넘어온다.

---

## 8. ★ 리오르그는 되감기가 아니라 갈아끼우기다

`chain-state/src/in_memory.rs:267-295`

```rust
fn update_blocks<I, R>(&self, new_blocks: I, reorged: R) {
    {
        let mut numbers = self.inner.in_memory_state.numbers.write();
        let mut blocks = self.inner.in_memory_state.blocks.write();

        // we first remove the blocks from the reorged chain
        for block in reorged {
            blocks.remove(&hash);          // ← 이게 "되감기"의 전부
            numbers.remove(&number);
        }

        // insert the new blocks
        for block in new_blocks {
            let parent = blocks.get(&block.recovered_block().parent_hash()).cloned();
            let block_state = BlockState::with_parent(block, parent);
            blocks.insert(hash, Arc::new(block_state));
            numbers.insert(number, hash);
        }
```

**해시맵에서 `remove` 두 줄.** 잔액을 되돌리거나 스토리지를 복원하는 코드가 없다.

### 왜 그것으로 충분한가 — 상태를 제자리에서 안 고쳤기 때문

`in_memory.rs:593-598`

```rust
pub struct BlockState<N: NodePrimitives = EthPrimitives> {
    /// The executed block that determines the state after this block has been executed.
    block: ExecutedBlock<N>,
    /// The block's parent block if it exists.
    parent: Option<Arc<Self>>,
}
```

**각 블록이 자기 실행 결과를 들고 있고 부모를 가리킨다.** 연결 리스트다.

읽을 때는 그 사슬을 모아서 DB 위에 겹친다 — `in_memory.rs:541-553`

```rust
pub fn state_provider(&self, hash: B256, historical: StateProviderBox)
    -> MemoryOverlayStateProvider<N>
{
    let in_memory = if let Some(state) = self.state_by_hash(hash) {
        state.chain().map(|block_state| block_state.block()).collect()
    } else { Vec::new() };

    MemoryOverlayStateProvider::new(historical, in_memory)
}
```

```
"블록 102 시점의 계정 X 잔액을 줘"
   → 102의 변경분에 X가 있나?  있으면 그것
   → 없으면 101의 변경분에?     있으면 그것
   → 끝까지 없으면 DB(historical)에서
```

**공용 상태 저장소를 고치는 게 아니라, 블록마다 자기 변경분을 들고 있고 읽을 때 위에서부터
찾는다.** 그래서 리오르그는 사슬에서 노드를 떼어내는 것으로 끝난다 — **되돌릴 게 없다.**

```
리오르그 전                        리오르그 후

 (102,0xAB)                        (102,0xAB)  ← 맵에서 제거. 아무도 못 찾음
     │  변경분 D                        (Arc 참조 0이면 drop)
 (101,0xAA)
     │  변경분 C                    (102,0xBC) ← 새로 삽입
 (100,0x0A) ───────────────┐            │  변경분 F
     │  변경분 B            └── (101,0xBB)
   DB (historical)                  │  변경분 E
                              (100,0x0A)   ← 양쪽이 Arc로 공유. 복사 없음
                                   │  변경분 B
                                 DB (historical)
```

공통 조상을 찾는 이유가 이것 — **그 아래는 손댈 필요가 없다.**

8주차에 본 static file의 append-only 성질, MDBX MVCC(옛 페이지를 안 덮어씀)와 **같은 발상이
인메모리 층에도 반복된다.** 덮어쓰지 않으면 되돌릴 일이 없다.

### 대가

겹쳐 읽으면 매번 사슬을 훑어야 한다. 그래서 `memory_block_buffer_target`에 상한이 있다.

---

## 9. ★ 깊이별로 무엇이 달라지나 — 기준은 숫자가 아니다

`engine/primitives/src/config.rs:6-14`

```rust
/// Triggers persistence when the number of canonical blocks in memory exceeds this threshold.
pub const DEFAULT_PERSISTENCE_THRESHOLD: u64 = 7;

/// Maximum number of blocks beyond the in-memory buffer target awaiting persistence before engine
/// API processing is stalled.
pub const DEFAULT_PERSISTENCE_BACKPRESSURE_THRESHOLD: u64 = 16;

/// How close to the canonical head we persist blocks.
pub const DEFAULT_MEMORY_BLOCK_BUFFER_TARGET: u64 = 5;
```

9주차에 남긴 질문의 답 — **7과 5는 다른 값이다.**

| 값 | 역할 |
|---|---|
| `PERSISTENCE_THRESHOLD = 7` | **언제** 저장을 시작할지 (메모리에 7개 넘게 쌓이면) |
| `MEMORY_BLOCK_BUFFER_TARGET = 5` | **어디까지** 저장할지 (head − 5까지) |
| `MIN_BLOCKS_FOR_PIPELINE_RUN = 32` | 파이프라인 발동 문턱 (`EPOCH_SLOTS`) |

```
        head
         │  ← 메모리에만 있음. 대략 5~7블록
  last_persisted_block  ══════ 경계선 ══════
         │  ← 디스크(MDBX + static file)에 있음
         ▼
```

### 판정은 해시 비교다. 깊이를 세지 않는다

`tree/mod.rs:2671-2713` `find_disk_reorg`

```rust
let mut canonical = self.state.tree_state.current_canonical_head;
let mut persisted = self.persistence_state.last_persisted_block;
...
// Happy path, canonical chain is ahead or equal to persisted chain.
// Walk canonical chain back to make sure that it connects to persisted chain.
while canonical.number > persisted.number {
    canonical = parent_num_hash(canonical)?;
}

// If we've reached persisted tip by walking the canonical chain back, everything is fine.
if canonical == persisted {
    return Ok(None);                       // ← 디스크 무사
}

// At this point, we know that `persisted` block can't be reached by walking the canonical
// chain back. In this case we need to truncate it to the first canonical block it connects to.
while persisted.number > canonical.number { persisted = parent_num_hash(persisted)?; }
while persisted.hash != canonical.hash {
    canonical = parent_num_hash(canonical)?;
    persisted = parent_num_hash(persisted)?;
}

debug!(target: "engine::tree", remove_above=persisted.number, "on-disk reorg detected");
Ok(Some(persisted.number))                 // ← 이 번호 위를 지워라
```

**"몇 블록이냐"가 아니라 "디스크에 있는 그 블록이 여전히 내 정본 사슬에 있느냐"를 본다.**

지우는 쪽 — `tree/mod.rs:1412-1421`

```rust
fn remove_blocks(&mut self, new_tip_num: u64) {
    if new_tip_num < self.persistence_state.last_persisted_block.number {
        let (tx, rx) = crossbeam_channel::bounded(1);
        let _ = self.persistence.remove_blocks_above(new_tip_num, tx);
        self.persistence_state.start_remove(new_tip_num, rx);
    }
}
```

가드가 `if new_tip_num < last_persisted_block.number` — **경계선 아래일 때만 동작한다.**
그리고 persistence 태스크(별도 스레드)에 보낸다 — 디스크 I/O로 엔진 루프를 막지 않으려고.

### 축 세 개가 독립적으로 겹친다

| 판정 | 기준 | 결과 |
|---|---|---|
| 디스크를 건드려야 하나 | **분기점이 `last_persisted_block`보다 아래인가** | `remove_blocks` |
| 새 가지 블록이 있나 | **트리(`blocks_by_hash`)에 있나** | 없으면 P2P 다운로드 |
| 다운로드로 될 거리인가 | **32블록 초과인가** | 초과면 파이프라인 |

| 깊이 | 디스크 | 다운로드 | 파이프라인 |
|---|---|---|---|
| 1~2블록 (메인넷 통상) | ❌ | 보통 ❌ | ❌ |
| ~5블록 (버퍼 안쪽) | ❌ | 상황에 따라 | ❌ |
| 5블록 초과 | ✅ `remove_blocks` | 상황에 따라 | ❌ |
| 32블록 초과 | ✅ | ✅ | ✅ backfill |

**"한두 블록을 넘는 순간" 바로 뭐가 바뀌지 않는다.** 5블록쯤에서 디스크가 개입하고 32블록쯤에서
파이프라인이 개입한다. 그리고 그 두 숫자는 **설정값**이다
(`--engine.persistence-threshold`, `--engine.memory-block-buffer-target`).

`crates/net/downloaders`가 하는 일이 여기서 갈린다 — 32블록 이하면 부모를 하나씩 더 받아온다
(`tree/mod.rs:2810` `// continue downloading the missing parent`).

---

## 10. pipeline unwind 트리거는 넷이다

9주차에 "정합성 깨짐"만 봤는데 그게 전부가 아니었다.

`stages/api/src/pipeline/mod.rs:549` `on_stage_error`

| # | 트리거 | unwind 목표 | 언제 |
|---|---|---|---|
| 1 | 기동 시 정합성 검사 | 저장소 3종 중 최소값 | 노드 시작 |
| 2 | **`StageError::DetachedHead`** | `local_head − (3 × 시도횟수)` | 헤더를 못 이어붙일 때 |
| 3 | `StageError::Block` + Validation/Execution | 직전 체크포인트 | stage가 검증/실행 실패 |
| 4 | CLI 명령 | 사용자 지정 | 수동 |

### ① 정합성 깨짐은 기동 시 한 번만 검출된다

```bash
grep -rn "check_consistency()" --include=*.rs crates/ bin/ | grep -v "fn check_consistency"
# → node/builder/src/launch/common.rs:546   ← 실질 호출자 하나
#    storage/provider/src/providers/database/mod.rs:496  ← assert_consistent (9주차: 호출자 0)
```

```
운영 중 ─── 💥 크래시 (커밋 도중) ───
              ↓
        저장소들이 서로 다른 높이에 남음  (아무도 모름. 프로세스가 죽었으니)
              ↓
        재시작 ──► check_consistency()  ← 여기서 처음 검출
              ↓
        pipeline unwind ──► 기동 계속
```

**운영 중에는 검출되지 않는다.** 8주차에 판 커밋 순서가 그래서 중요했다 — "MDBX가 기준점"이라는
규칙 덕에 재시작 시 무엇을 잘라낼지 계산할 수 있다.

### ② DetachedHead — 3블록씩 파면서 접점을 찾는다

깊은 분기의 실제 사슬:

```
① 32블록 넘게 뒤처짐 (또는 깊게 분기)
     ↓
② BackfillAction::Start → 파이프라인 가동 (Idle → Active)
     ↓
③ Headers stage: P2P에서 헤더를 거꾸로 받아옴
     ↓
④ 받은 헤더 끝이 내 local head에 안 붙는다 (다른 가지였다)
     ↓
⑤ StageError::DetachedHead
     ↓
⑥ 파이프라인이 3블록 되감고 재시도 (3 → 6 → 9 …)
```

다운로더 쪽이 다음 일을 예고한다 — `net/downloaders/src/headers/reverse_headers.rs:313-317`

```rust
// Reset trackers so that we can start over the next time the sync target is
// updated.
// The expected event flow when that happens is that the node will unwind the local
// chain and restart the downloader.
self.reset();
```

`pipeline/mod.rs:570-577`

```rust
// We unwind because of a detached head.
let unwind_to = local_head.block.number
    .saturating_sub(
        BEACON_CONSENSUS_REORG_UNWIND_DEPTH.saturating_mul(self.detached_head_attempts),
    )
    .max(1);
```

`BEACON_CONSENSUS_REORG_UNWIND_DEPTH = 3` (`reth-primitives-traits/src/constants/mod.rs:31`)

| 시도 | 되감는 깊이 |
|---|---|
| 1회 | local_head − 3 |
| 2회 | local_head − 6 |
| 3회 | local_head − 9 |

**분기 지점을 미리 모르므로 조금씩 파면서 찾는다.** 32블록을 한 번에 되감는 코드는 없다.
되감기가 비싸기 때문이다 — 6블록이면 충분한데 32를 되감으면 26블록을 헛수고로 버리고 다시 받는다.

`.max(1)`도 눈여겨볼 것 — 0까지는 안 내려간다. 9주차 §5의 `assert_ne!(unwind_block, 0)`과 같은 방어.

### 되감기 세 층 (최종판)

| 층 | 어떻게 | 트리거 | 규모 |
|---|---|---|---|
| **인메모리** | 맵에서 `remove` (뺄셈 없음) | 리오르그 | 1~2블록 |
| **디스크: disk reorg** | 저장 중 들어간 블록 제거 | `find_disk_reorg` | 몇 블록 |
| **디스크: pipeline unwind** | stage 15개 역순 (changeset 역적용, static file 절단) | 위 표의 4가지 | 3블록씩 반복 (①은 계산값) |

---

## 11. 스펙 대조 중간 결과

| # | 스펙 | 코드 | 판정 |
|---|---|---|---|
| 1 | fcU §6 `-38006 Too deep reorg` — "the limitation **specific to the client software**" | `grep -rn "38006\|TooDeepReorg\|too deep"` → **0건** | ⚠️ **미구현.** reth는 제한을 두지 않는다 |
| 2 | fcU §7 "MUST be made **atomically**" | 미확인 | 수요일 |
| 3 | fcU §8.3 "MUST NOT be rolled back" | `engine_api.rs:918` + `process_payload_attributes` 문서 | ✅ **두 계층에서** |
| 4 | validation §4 "MUST be **idempotent**" | `InvalidHeaderCache` (`invalid_headers.rs:11`) — 재검증 자체를 안 한다 | ✅ 확인 |
| 5 | Sync note "**implementation dependent**" | staged sync 파이프라인이 그 자리 | ✅ 스펙이 비운 자리 |
| 6 | validation §6 "MUST NOT be affected by an active sync process on a **side branch**" | 미확인 | 수요일 |
| 7 | fcU §2 skip 허용 범위 | 주석은 "ancestor of the **head of canonical chain**", 스펙은 "ancestor of the latest known **finalized** block" | ⚠️ **문구 불일치.** reth가 더 넓게 skip |
| 8 | (스펙에 없음) | 초기 fcU에서 `safe`를 동기화 목표로 사용 (`tree/mod.rs:1388`) | ⚠️ **reth만의 선택** |

**1번이 금요일 목표("스펙이 클라이언트에 맡긴 지점")의 답이다** — 맡겼는데 reth는 제한을 두지
않는 쪽을 골랐다.

> `MIN_BLOCKS_FOR_PIPELINE_RUN = 32`를 한계값으로 착각하면 안 된다. 그건 **파이프라인 발동
> 문턱**이고 리오르그를 **거부**하는 값이 아니다. 깊으면 파이프라인으로 처리할 뿐 에러를 내지
> 않는다.

---

## 12. 오늘의 수확

1. **★ head ≠ header.** `head`는 체인의 끝을 가리키는 포인터(`BlockNumHash`), `header`는 블록
   메타데이터(`SealedHeader`). 영어가 비슷할 뿐 무관하다.
2. **★ 다운로드는 P2P다.** DB에서 메모리로 옮기는 게 아니라 다른 노드에게서 받아온다.
   `crates/net/` 13개 크레이트가 그 일을 한다.
3. **★ 경로가 두 개다.** 라이브(CL이 블록을 줌 → 메모리 → 나중에 디스크)와 대량(P2P로 받아옴 →
   stage → 디스크 직접). 정상 운영 중에는 stage가 하나도 안 돈다.
4. **★ 갈아타기의 판단은 CL이 한다.** 실행 계층에는 "어느 가지가 옳은지" 결정하는 코드가 없다.
   있는 것은 유효/무효 판정과 그래프 도달 가능성 계산뿐.
5. **★ 리오르그는 되감기가 아니라 갈아끼우기다.** 상태를 겹쳐 읽는 구조라 맵에서 노드를 떼면
   끝난다. 8주차 SF append-only, MDBX MVCC와 같은 발상의 반복.
6. **★ 깊이 숫자가 기준이 아니다.** `find_disk_reorg`는 해시를 비교한다 — "분기점이
   `last_persisted_block`보다 아래인가". 5/7/32는 그 경계를 만드는 설정값일 뿐이다.
7. **엔진은 단일 스레드로 상태를 지킨다.** 락 대신 `mpsc`(요청) + `oneshot`(응답) 조합.
   파이프라인은 DB 배타 접근이 필요해서 fcU와 동시에 못 돈다.
8. **정합성 깨짐은 기동 시 한 번만 검출된다.** 운영 중에는 검출 주체가 없다.
9. **`on_forkchoice_updated`는 4단이다.** 마지막 `handle_missing_block`이 fallback이고,
   스펙 §1(MAY sync)과 §9(MUST SYNCING)를 한 번에 만족시킨다.
10. **주석이 인용한 스펙 문구를 원문과 나란히 놓아야 한다.** §2 불일치가 그렇게 나왔다.

## 13. 다음으로 넘길 질문

1. `apply_chain_update` 안에서 리오르그가 실제로 적용되는 전체 경로 (수요일)
2. fcU §7의 "atomically"를 어디서 보장하나 — 8주차 "저장소 셋을 순서로 해결"과 같은 방식인가
3. `DEFAULT_PERSISTENCE_BACKPRESSURE_THRESHOLD = 16`은 왜 필요한가? 주석이 *"before engine API
   processing is **stalled**"*라는데, 저장이 밀리면 엔진이 일부러 멈추는 이유는?
4. `detached_head_attempts`에 상한이 있나? 계속 늘어나면 어디까지 가나
5. `ACCEPTED`로 미뤄둔 곁가지가 정본이 되면 그 검증은 어느 함수에서 일어나나
6. 라이브 경로에서 CL이 준 블록의 부모를 모르면? (`on_disconnected_downloaded_block`의
   "downloaded"가 붙은 이유)
