# 전체 지도 — 무엇이 언제 어떻게 바뀌나

> 12주차 목요일. 8~11주차는 storage를 확대해서 봤고, 화·수에 그게 전체 어디에 붙는지 알았다.
> 오늘은 그것을 한 장으로 정리한다.
>
> **그림 자체는 별도로 관리한다.** 이 문서는 그림이 답해야 할 질문과, 그림으로는 담기 어려운
> **변수 단위 상태 전이**를 기록한다. "구조를 말하는 것"과 "어떤 변수가 어떤 시나리오에서 어떻게
> 바뀌는지 답하는 것"은 다른 깊이라서 후자를 표로 남긴다.

---

## 1. 그림이 답해야 하는 질문 7개

그림을 먼저 그리면 무조건 복잡해진다. **답해야 할 질문을 먼저 정하고, 그 목록이 검증 기준이 된다.**

| # | 질문 | 답 |
|---|---|---|
| 1 | 블록이 노드로 들어오는 경로가 몇 개인가? | **셋** — ① CL이 `newPayload`로 블록 전체를 줌 ② P2P 피어에게서 다운로드 ③ 내 디스크 |
| 2 | 합의 계층과 reth는 무엇을 주고받나? | CL→reth: `newPayload`(블록 전체) / `forkchoiceUpdated`(해시 3개). reth→CL: `PayloadStatus`(5종) 또는 `-38002`/`-38003`. timeout 8s |
| 3 | 유효성 판정과 정본 선택은 각각 누가 하나? | **유효성은 reth, 선택은 CL.** 실행 계층에는 "어느 가지가 옳은지" 결정하는 코드가 없다 |
| 4 | 파이프라인은 언제 도나? | 목표와 local tip의 차이가 **32블록 초과**일 때 (`MIN_BLOCKS_FOR_PIPELINE_RUN = EPOCH_SLOTS`). 그 외에는 `Idle` |
| 5 | 메모리와 디스크의 경계는? | **`last_persisted_block`.** 저장 시작 문턱 `PERSISTENCE_THRESHOLD = 7`, 저장 목표 `MEMORY_BLOCK_BUFFER_TARGET = 5` (head − 5까지) |
| 6 | 8~11주차에 판 provider는 어디에 붙나? | **읽는 대상이 둘** — `canonical_in_memory_state`(먼저) → 디스크(없으면). 쓰는 쪽은 엔진 트리와 파이프라인 |
| 7 | 리오르그는 어느 층에서 처리되나? | **엔진 트리 메모리.** 디스크는 분기점이 `last_persisted_block`보다 아래일 때만 (`find_disk_reorg` → `remove_blocks`) |

**7개에 다 답하는 그림 하나**가 목표다. 답 못 하는 질문이 있으면 그림이 부족한 것이고, 그림에
있는데 질문에 없는 요소는 지워야 한다.

---

## 2. ★ 지시와 실행이 분리돼 있다

표를 보기 전에 이걸 먼저 잡아야 한다. **`newPayload`가 정본을 바꾸는 경우가 있어서 헷갈리는데,
결정은 언제나 fcU가 한다.**

### 모든 fcU 직후에 tracker가 갱신된다

`engine/tree/src/tree/mod.rs:1645-1650`

```rust
let mut output = self.on_forkchoice_updated(state, payload_attrs);

if let Ok(res) = &mut output {
    // track last received forkchoice state
    self.state.forkchoice_state_tracker
        .set_latest(state, res.outcome.forkchoice_status());
```

`engine/primitives/src/forkchoice.rs:22-32`

```rust
pub const fn set_latest(&mut self, state: ForkchoiceState, status: ForkchoiceStatus) {
    if status.is_valid() {
        self.last_syncing = None;              // ← 미뤄진 지시를 지운다
        self.last_valid = Some(state);
    } else if status.is_syncing() {
        self.last_syncing = Some(state);       // ← 미뤄진 지시로 보관
    }
    ...
```

`forkchoice.rs:114-116`

```rust
pub const fn sync_target_state(&self) -> Option<ForkchoiceState> {
    self.last_syncing
}
```

**`sync_target_state()` = `last_syncing` = "SYNCING으로 답했던 fcU" = 아직 실행하지 못한 지시.**

### Case 1 — 블록이 이미 있다: fcU가 즉시 정본을 바꾼다

```
CL: fcU(head=0xAB)
  → on_forkchoice_updated
      ③ apply_chain_update
          on_new_head(0xAB) → Some(chain_update)
          on_canonical_chain_update  →  ★ 정본 바뀜
  → 반환 VALID
  → set_latest(state, VALID)
       last_valid   = state
       last_syncing = None       ← 미뤄진 지시 없음
```

통상 운영에서 거의 항상 이 경우다 — CL이 head로 지정하는 블록은 보통 직전에 `newPayload`로
이미 받아뒀다.

### Case 2 — 블록을 모른다: 지시를 보관하고 나중에 실행

```
T0  CL: fcU(head=0xAB)              ← 0xAB를 아직 못 받음
      ④ handle_missing_block
          SYNCING 반환 + Download 이벤트
      set_latest(state, SYNCING)
          last_syncing = state      ← ★ 지시 보관. 정본은 안 바뀜

T1  0xAB 도착 (newPayload 또는 다운로드)
      try_insert_payload  →  tree_state에 삽입
      is_sync_target_head(0xAB)?
          last_syncing.head_block_hash == 0xAB  →  true
      MakeCanonical 이벤트
      make_canonical(0xAB)
          on_new_head → on_canonical_chain_update  →  ★ 이제 정본 바뀜
          on_canonicalized_sync_target
              → ensure_consistent_forkchoice_state  (safe/finalized 적용)
              → promote_sync_target_to_valid
```

`tree/mod.rs:814-824`

```rust
// if the block is valid and it is the current sync target head, make it canonical
if outcome.outcome.is_valid() && self.is_sync_target_head(block_hash) {
    if self.state.tree_state.canonical_block_hash() != block_hash {
        outcome = outcome.with_event(TreeEvent::TreeAction(TreeAction::MakeCanonical {
            sync_target_head: block_hash,
        }));
    }
}
```

`tree/mod.rs:2366-2371`

```rust
fn is_sync_target_head(&self, block_hash: B256) -> bool {
    if let Some(target) = self.state.forkchoice_state_tracker.sync_target_state() {
        return target.head_block_hash == block_hash
    }
    false
}
```

**`newPayload`가 `is_sync_target_head`를 확인한다는 게 증거다** — *"내가 이 블록을 정본으로 삼아도
되나?"를 CL의 지시에 물어보는 것*이다. 조건이 거짓이면 `tree_state`에만 넣고 끝낸다.

### 정리

```
fcU 도착  →  블록이 있나?
              ├─ 있음  →  즉시 정본화
              └─ 없음  →  last_syncing에 지시 보관 + SYNCING
                              ↓
                          블록 도착 (newPayload)
                              ↓
                          is_sync_target_head? → 예 → 정본화
```

> **정본을 바꾸라고 결정하는 건 언제나 fcU다.**
> **`newPayload`는 그 지시를 실행할 재료가 도착했음을 알리는 것뿐이다.**
>
> 화요일 문장에 한 줄을 더하면 완성된다 — 노드는 "내 체인이 잘못됐다"고 판단하지 않고 지시받은
> head가 내 head의 후손이 아니라는 것을 발견할 뿐이며, **그 지시를 지금 실행할 수 없으면
> 보관해뒀다가 재료가 도착할 때 실행한다.**

---

## 3. 상태 변수 12개

| 그룹 | 변수 | 무엇 |
|---|---|---|
| **엔진 트리**<br/>`EngineApiTreeState`<br/>(`tree/mod.rs:147`) | `tree_state.blocks_by_hash` | **모든 후보** — 곁가지 포함 (`state.rs:25` *"This includes blocks of all forks"*) |
| | `tree_state.blocks_by_number` | 높이별. 같은 높이에 여러 블록 가능 |
| | `tree_state.parent_to_child` | 부모→자식 링크 |
| | `tree_state.current_canonical_head` | `BlockNumHash` — 엔진 내부 head |
| | `buffer` | 부모를 모르는 블록 (`BlockBuffer`) |
| | `invalid_headers` | 무효 캐시 (`InvalidHeaderCache`) |
| | `forkchoice_state_tracker` | CL이 준 마지막 상태 + **미뤄진 지시**(§2) |
| **provider가 읽는 쪽**<br/>`canonical_in_memory_state`<br/>(이하 `cims`) | `cims.blocks` / `numbers` | **정본 체인만** |
| | `cims.canonical_head` | `SealedHeader` — head **두 번째 사본** |
| | `cims.safe_block` | `watch::Sender<Option<SealedHeader>>` |
| | `cims.finalized_block` | `watch::Sender<Option<SealedHeader>>` |
| | `cims.persisted_block` | `watch::Sender<Option<BlockNumHash>>` |
| **진행 상태** | `persistence_state.last_persisted_block` | 디스크 경계 |
| | `backfill_sync_state` | `Idle` / `Pending` / `Active` |

### head가 두 곳에 있다

| | 타입 | 누구를 위한 것 |
|---|---|---|
| `tree_state.current_canonical_head` | `BlockNumHash` (번호+해시) | 엔진 내부 계산 — `on_new_head`의 조상 추적 |
| `cims.canonical_head` | `SealedHeader` (헤더 전체) | **provider가 읽는 값** — RPC `latest` 태그 등 |

전자는 **표식**이라 포인터만, 후자는 **데이터**라 헤더 전체를 들고 있다.
§2의 head ≠ header 구분이 여기서 실물로 나타난다.

### `tree_state`와 `cims`의 역할 차이

| | 담는 것 | 리오르그 시 |
|---|---|---|
| `tree_state` | 곁가지 포함 **모든 후보** | 버려진 가지를 **유지** (`reinsert_reorged_blocks`) |
| `cims` | **정본 체인만** | 버려진 가지를 **제거** (`update_blocks`의 `reorged` 루프) |

**"정본에서는 빼지만 후보 목록에는 남긴다."** 리오르그는 다시 뒤집힐 수 있고, 트리에서 지웠으면
`on_new_head`의 첫 `let Some(...) else { return Ok(None) }`에 걸려 갈아탈 수 없게 된다.

---

## 4. ★ 시나리오 × 변수 전이표

`cims` = `canonical_in_memory_state`

| # | 시나리오 | `tree_state` | `cims.blocks` / `canonical_head` | `safe`/`finalized` | 디스크 | 반환 |
|---|---|---|---|---|---|---|
| A | `newPayload` 유효, sync target 아님 | **+블록** | — | — | — | `VALID` |
| B | `newPayload` 유효, **sync target head** | +블록, **head 갱신** | **갱신** | **갱신 시도**(§5) | — | `VALID` |
| C | `newPayload` 부모 모름 | `buffer` **+블록** | — | — | — | `SYNCING` + Download |
| D | `newPayload` 무효 | `invalid_headers` **+헤더** | — | — | — | `INVALID` |
| E | `newPayload` backfill 중 | `buffer` **+블록** (검증 없이) | — | — | — | `SYNCING` |
| F | `fcU` head가 이미 tip | — | — | **갱신** | — | `VALID` |
| G | `fcU` 연장 (`Commit`) | **head 갱신** | **+new, head 갱신** | **갱신** | — | `VALID` |
| H | `fcU` 리오르그, 분기점 > `last_persisted` | head 갱신, **`old` 유지** | +new, **−old**, head 갱신 | **갱신** | — | `VALID` |
| I | `fcU` 리오르그, 분기점 < `last_persisted` | H와 같음 | H와 같음 | **갱신** | **`remove_blocks`** | `VALID` |
| J | `fcU` head 모름 | — | — | — | — | `SYNCING` + Download |
| **K** | **`fcU` safe/finalized 모순** | **head 갱신** | **갱신** | ❌ **불변** | — | **`-38002`** |
| L | persistence 완료 | `remove_before(...)` | **−persisted 이하** | — | (이미 써짐) | — |
| M | backfill 완료 | 백필 높이 이하 제거, unwind면 `invalid_headers` +bad_block | **리셋** | — | 대량 갱신됨 | — |

**모든 fcU 시나리오는 `forkchoice_state_tracker`를 갱신한다** (표에서 생략).

### 표에서 읽어야 할 것 넷

**① `newPayload`는 `safe`/`finalized`를 절대 안 만진다**

A·C·D·E 전부 불변. 그 값을 쓰는 코드는 `update_finalized_block`/`update_safe_block` 둘뿐이고
(`tree/mod.rs:3228`, `:3258`), 둘 다 `ensure_consistent_forkchoice_state`가 부른다.
**그 함수는 fcU 처리 경로에서만 불린다.**

**② `tree_state`와 `cims`가 반대로 움직인다 (H)**

**③ 디스크는 조건부다 (H vs I)**

리오르그 깊이 숫자가 기준이 아니라 **분기점이 `last_persisted_block`보다 아래인가**가 기준이다.
`find_disk_reorg`는 깊이를 세지 않고 해시를 비교한다.

**④ K만 부분 갱신을 남긴다**

head는 갱신됐고 `safe`/`finalized`는 옛 값. 되돌리는 코드가 없다. → §5

---

## 5. §13 보강 — 같은 부분 갱신이 두 곳에 있고, reth가 이름을 붙여놨다

수요일 [consensus-analysis.md](consensus-analysis.md) §13에서 K 시나리오를 팠는데, B 시나리오를
따라가니 **같은 상황이 한 곳 더** 있었다.

`tree/mod.rs:1994-2017`

```rust
fn on_canonicalized_sync_target(&mut self, target: B256) {
    let Some(sync_target_state) = self.state.forkchoice_state_tracker
        .sync_target_state()
        .filter(|state| state.head_block_hash == target)
    else { return; };

    if let Err(outcome) = self.ensure_consistent_forkchoice_state(sync_target_state) {
        debug!(
            target: "engine::tree",
            head = %sync_target_state.head_block_hash,
            safe = %sync_target_state.safe_block_hash,
            finalized = %sync_target_state.finalized_block_hash,
            ?outcome,
            "Canonicalized sync target head before safe/finalized could be applied"
        );
        return;                              // ← 로그만 남기고 조용히 넘어간다
    }

    self.state.forkchoice_state_tracker.promote_sync_target_to_valid(sync_target_state);
}
```

**로그 메시지가 그 상태를 정확히 명명한다.**

> *"Canonicalized sync target head **before** safe/finalized could be applied"*

= **"head는 정본화됐는데 safe/finalized는 아직 적용 못 한 상태"**

| 경로 | 실패 시 |
|---|---|
| `apply_chain_update` (fcU) — 시나리오 K | `-38002` 반환 |
| `make_canonical` → `on_canonicalized_sync_target` — 시나리오 B | **`debug!` 로그만** |

### 판정이 바뀐다

| 수요일 판정 | 수정 |
|---|---|
| "스펙 공백에서의 선택. 문제 소지 있음" | **"reth가 인지하고 이름까지 붙인 의도된 상태"** |

`debug!` 로그가 있다는 건 **"이런 일이 일어날 수 있고 괜찮다"는 판단이 이미 내려져 있다**는 뜻이다.
기여 후보로서는 약해진다 — 이슈를 열어도 *"알고 있고 의도된 것"*이라는 답을 받을 가능성이 크다.

**대신 분석으로서는 정확해진다.** 금요일에 이렇게 쓴다.

```
reth는 "head 정본화 후 safe/finalized 적용 실패" 상태를 인지하고 있다
(`on_canonicalized_sync_target`의 로그 메시지가 그 상태를 명명한다).
두 경로에서 같은 상황이 발생하며 처리가 다르다 — fcU 경로는 `-38002`를 반환하고,
sync target 정본화 경로는 debug 로그만 남긴다.
스펙 §5·§6에 갱신 금지 문구가 없으므로 명시적 위반은 아니고, reth의 의도된 동작으로 보인다.
```

> **이번 주 방법론의 실례다.** 수요일에는 "문제일 수 있다"까지 갔는데 코드를 한 겹 더 파니
> "reth가 이미 알고 있다"가 나왔다. **"위반이다"로 서둘러 결론 내지 않은 게 맞았다.**

---

## 6. 그림에서 의도적으로 뺀 것

한 장에 다 넣으면 아무 질문에도 답하지 못한다. **이번 주 주제(EL/CL 경계)의 축에 안 걸리는 것을
뺀다.** 나중에 velog에서 재사용할 때 "여기서는 넣어야 하나"를 판단하려면 이유를 남겨둬야 한다.

| 뺀 것 | 이유 |
|---|---|
| RocksDB | 8~9주차 주제(저장소 3종). EL/CL 경계와 무관 |
| static file 세그먼트 종류 | 같음 |
| ETL / `Collector` | Headers stage 내부 구현 |
| stage 15개 이름 | 파이프라인을 박스 하나로 두면 질문 4·7에 답이 된다 |
| `crates/net/` 13개 크레이트 | "P2P 피어" 박스 하나로 충분 |
| 트라이 / 상태 루트 | 실행 내부. 별도 주제 |
| `pending` 블록 / txpool | fcU·newPayload 축이 아님 |
| 파이프라인 unwind 4가지 트리거 | 별도 그림감 (아래) |

**두 번째 그림으로 뺀 것**: 되감기 세 층(인메모리 `remove` / disk reorg / pipeline unwind) +
경계 숫자(5/7/32). 수요일 §8~10을 옮기면 그것만으로 글 하나 분량이다.

## 그림이 잘못 말하면 안 되는 것 셋

이번 주에 바로잡은 오해들이다. 그림이 이렇게 보이면 틀렸다.

| 오해 | 그림에서 |
|---|---|
| 다운로드 = DB에서 불러오기 | 다운로드 화살표가 **디스크에서** 나오면 틀림. **P2P 피어에서** 나와야 함 |
| stage가 항상 돈다 | 파이프라인이 라이브 경로 **안**에 그려지면 틀림. 조건부(32블록)로 표시 |
| 리오르그 = 파이프라인 unwind | 리오르그 표시가 파이프라인 쪽에 붙으면 틀림. **엔진 트리 메모리** |

---

## 7. 오늘의 수확

1. **★ 지시(fcU)와 실행이 분리돼 있다.** 블록이 없으면 `last_syncing`에 지시를 보관하고, 재료가
   도착하면 `is_sync_target_head`로 확인해서 실행한다. `newPayload`가 정본을 바꾸는 경우는
   **미뤄진 fcU 지시의 실행**이지 독자적 판단이 아니다.
2. **★ head가 두 곳에 있고 타입이 다르다.** `BlockNumHash`(엔진 계산용)와
   `SealedHeader`(provider가 읽는 값). head ≠ header 구분이 여기서 실물로 나타난다.
3. **★ `newPayload`는 `safe`/`finalized`를 절대 안 만진다.** 그 값을 쓰는 코드가 둘뿐이고 둘 다
   fcU 경로에서만 불린다.
4. **★ 구조를 말하는 것과 변수 전이를 답하는 것은 다른 깊이다.** 그림으로는 3·4번 같은 사실이
   표현되지 않는다. 표가 따로 필요하다.
5. **그림은 질문 목록에서 시작한다.** 답해야 할 질문을 먼저 정하고 그것으로 검증한다.
   뺀 것과 그 이유를 남겨두면 재사용할 때 판단이 된다.
6. **로그 메시지가 설계 의도를 드러낸다.** *"Canonicalized sync target head before safe/finalized
   could be applied"* 한 줄이 §13의 판정을 바꿨다.

## 8. 다음으로 넘길 질문

1. 시나리오 M(backfill 완료)에서 `cims`를 "리셋"한다고 주석에 있는데, 정확히 어디까지 버리나
2. `buffer`(부모 모르는 블록)의 크기 제한은? 넘치면 무엇을 버리나
3. 시나리오 K를 재현할 방법 — §13의 유일한 미확인 항목
4. `cims.pending`은 언제 채워지나 (표에서 다루지 않았다)
