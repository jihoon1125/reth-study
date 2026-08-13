# 스펙이 약속한 것 vs 코드가 하는 것

> 12주차 금요일. Phase 4(백서 vs 소스 검증)의 축소판 리허설.
> 사실 수집은 화~목에 끝났고 오늘은 **판정**만 한다.
>
> **이 문서의 목적은 표가 아니라 방법이다.** Week 20~22에서 다시 읽을 사람은 결과가 아니라
> "어떻게 판정했는가"를 찾을 것이므로 §1·§2와 §6을 그쪽에 맞춰 썼다.

**대조 대상**: [`ethereum/execution-apis` `src/engine/paris.md`](https://github.com/ethereum/execution-apis/blob/main/src/engine/paris.md) (297줄)
**대조 코드**: reth `crates/engine/`, `crates/rpc/rpc-engine-api/` (2026-08 기준 `upstream/main`)

---

## 0. 조항 번호 읽는 법

이 문서의 `fcU §7` 같은 표기는 전부 `paris.md` 한 파일 안의 위치다.

```
paris.md
├─ ### Payload validation          (95줄~)   → "validation §N"
├─ ### Sync                        (120줄~)  → "Sync note"
├─ ### engine_newPayloadV1
│    └─ #### Specification         (162줄~)  → "newPayload §N"
└─ ### engine_forkchoiceUpdatedV1
     └─ #### Specification         (209줄~)  → "fcU §N"
```

각 `#### Specification` 아래 번호 매긴 목록의 그 번호다. **아래 표에는 원문을 같이 실었다** —
번호만 남기면 나중에 본인도 못 알아본다.

---

## 1. 판정 등급 여섯 개

대조를 하다 보면 "위반이다 / 아니다" 둘로만 나누게 되는데, 그러면 대부분의 항목을 잘못 분류한다.
실제로는 여섯 가지가 나온다.

| 등급 | 뜻 | 위반인가 |
|---|---|---|
| **준수** | MUST를 지킨다 | 아니오 |
| **재량 행사** | 스펙이 열어둔 지점(MAY / implementation dependent / "클라이언트가 정함")에서 선택했다 | **아니오** |
| **스펙 공백** | 그 경우를 스펙이 규정하지 않았다 | **판정 불가** |
| **적용 대상 아님** | 스펙이 상정한 개념과 코드의 개념이 다르다 | 아니오 |
| **범위 불일치** | 문구가 정한 범위와 구현 범위가 다르다 | **따져야 함** |
| **미구현** | MUST인데 코드가 없고, **그 조건이 실제로 발생한다** | **예** |

**마지막 하나만 위반이다.** 나머지 다섯은 "다르다"이지 "틀렸다"가 아니다.

---

## 2. 위반이라고 쓰려면 네 개가 필요하다

| # | 확인 | 놓치면 |
|---|---|---|
| 1 | **MUST 조항인가?** | MAY를 안 지켰다고 위반이라 하면 틀린다 |
| 2 | **조건절의 전제가 성립하는가?** | `if X, MUST Y`에서 X가 발생하지 않는 설계면 Y가 없어도 된다 |
| 3 | **코드가 그 요구를 안 지키는가?** | 다른 함수/계층에서 지키고 있을 수 있다 |
| 4 | **도달 가능한가? 재현했는가?** | 죽은 경로면 실질 위반이 아니다 |

**2번이 이번 주 최대 수확이다.** MUST 문장 앞의 `if`절을 안 읽으면 오판한다.
`grep`으로 "이 에러 코드가 없다"를 찾는 데는 5분이면 되는데, 그게 위반인지 판정하는 데는
조건절 한 줄을 읽어야 한다.

---

## 3. 판정 결과 9줄

### ① `-38006 Too deep reorg` → **재량 행사**

**스펙** (fcU §6, `paris.md:221`)

```
6. Client software **MUST** return `-38006: Too deep reorg` error if the depth of reorg to
   `forkchoiceState.headBlockHash` exceeds the limitation specific to the client software.
```

**코드**

```bash
grep -rn "38006\|TooDeepReorg\|too deep" --include=*.rs crates/
# → 0건
```

**판정 근거**: 조건절이 *"exceeds **the limitation specific to the client software**"*다.
**한계값을 정하는 주체가 클라이언트**이고, reth는 "제한 없음"을 선택했다. 그러면 "한계를 넘는"
상황이 발생하지 않으므로 MUST가 발동하지 않는다.

> ⚠️ `MIN_BLOCKS_FOR_PIPELINE_RUN = 32`(`engine/tree/src/tree/mod.rs:98`)를 한계값으로 착각하면
> 안 된다. 그건 **파이프라인 발동 문턱**이고 리오르그를 **거부**하는 값이 아니다. 깊으면
> 파이프라인으로 처리할 뿐 에러를 내지 않는다.

---

### ② fcU §5 + §7 (부분 갱신) → **스펙 공백**

**스펙** (fcU §5, `paris.md:219`)

```
5. Client software **MUST** return `-38002: Invalid forkchoice state` error if the payload
   referenced by `forkchoiceState.headBlockHash` is `VALID` and a payload referenced by
   either `forkchoiceState.finalizedBlockHash` or `forkchoiceState.safeBlockHash` does not
   belong to the chain defined by `forkchoiceState.headBlockHash`.
```

**스펙** (fcU §7, `paris.md:223`)

```
7. Client software **MUST** update its forkchoice state if payloads referenced by
   `forkchoiceState.headBlockHash` and `forkchoiceState.finalizedBlockHash` are `VALID`.
   ...
   * All updates to the forkchoice state resulting from this call **MUST** be made atomically.
```

**코드** (`engine/tree/src/tree/mod.rs:1353-1363`)

```rust
if let Some(chain_update) = self.on_new_head(state.head_block_hash)? {
    let tip = chain_update.tip().clone_sealed_header();
    self.on_canonical_chain_update(chain_update);          // ← head를 갱신한다

    // Update the safe and finalized blocks and ensure their values are valid
    if let Err(outcome) = self.ensure_consistent_forkchoice_state(state) {
        return Ok(Some(TreeOutcome::new(outcome)));        // ← -38002. head 갱신은 남는다
    }
```

**사실 정리**

- 롤백 경로가 없다. `on_canonical_chain_update`를 되돌리는 함수가 존재하지 않는다.
- 순서는 의도적이다 — `ensure_consistent_forkchoice_state`의 주석이
  *"…in the canonical chain **after the head block is canonicalized**"*라고 명시(`:3283`).
  §5의 판정 기준이 *"the chain defined by `headBlockHash`"* = 새 head의 체인이므로 먼저
  갱신해야 검사할 수 있다.
- `finalized`/`safe`를 갱신하는 코드는 두 줄뿐이고(`:3228`, `:3258`) 둘 다 fcU 처리 중에만
  불린다 → **부분 갱신은 다음 fcU가 성공해야 해소된다.**
- `-38002`를 받아 복구하는 코드는 없다.

**판정 근거 — 스펙의 침묵**

`MUST NOT update the forkchoice state`가 어느 조항에 붙어 있는지 보면:

| 조항 | 실패 원인 | 갱신 금지 문구 |
|---|---|---|
| fcU §3 | PoW terminal block 검증 실패 | ✅ 있음 |
| fcU §4 | payload 검증 실패 | ✅ 있음 |
| **fcU §5** | forkchoiceState 모순 | ❌ **없음** |
| **fcU §6** | 너무 깊은 리오르그 | ❌ **없음** |
| fcU §8.3 | payloadAttributes 실패 | ✅ 반대 방향("롤백 금지") |

**§3·§4에는 있고 §5·§6에는 없다.** 같은 문서, 같은 목록, 이웃한 조항인데 한쪽만 그 문장을
달고 있으므로 문체 실수로 보기 어렵다.

그리고 §7의 전제("head와 finalized가 VALID")가 §5 상황에서 **참이다.** 두 MUST가 서로를
해소하지 못하고 우선순위도 명시돼 있지 않다. → **판정 불가.**

**reth는 이 상태를 인지하고 있다**

같은 부분 갱신이 다른 경로에도 있고, 거기 로그가 그 상태를 명명한다
(`tree/mod.rs:1994-2017`).

```rust
if let Err(outcome) = self.ensure_consistent_forkchoice_state(sync_target_state) {
    debug!(
        target: "engine::tree", ...,
        "Canonicalized sync target head before safe/finalized could be applied"
    );
    return;                              // ← 로그만 남기고 조용히 넘어간다
}
```

| 경로 | 실패 시 |
|---|---|
| `apply_chain_update` (fcU) | `-38002` 반환 |
| `make_canonical` → `on_canonicalized_sync_target` | **`debug!` 로그만** |

**`debug!` 로그가 있다는 건 "이런 일이 일어날 수 있고 괜찮다"는 판단이 이미 내려져 있다는 뜻이다.**

**미확인**: 재현하지 않았다. finalized 아래 리오르그 자체가 극히 드물고, 정상 CL은 finalized보다
낮은 블록을 head로 지정하지 않는다(reth는 그 경우를 `reorgs.finalized` 메트릭으로 센다 —
`tree/mod.rs:2758-2772`).

---

### ③ fcU §8.3 "롤백 금지" → **준수**

**스펙** (`paris.md:233`)

```
    3. If `payloadAttributes` validation fails, the `forkchoiceState` update
       **MUST NOT** be rolled back.
```

**코드** (`rpc/rpc-engine-api/src/engine_api.rs:905-928`) — 주석에 스펙 원문이 그대로 붙어 있다.

```rust
if let Err(err) = attr_validation_res {
    let fcu_res = self.inner.beacon_consensus.fork_choice_updated(state, None).await?;
    //                                                                    ^^^^ attrs를 None으로
    if fcu_res.is_invalid() || fcu_res.payload_status.is_syncing() {
        return Ok(fcu_res)
    }
    return Err(err.into())
}
```

**"롤백하지 않는다"를 애초에 롤백할 상황을 만들지 않는 방식으로 지킨다.** attributes가 잘못되면
`None`으로 바꿔서 정본 갱신만 정상 수행하고 그 다음 `-38003`을 낸다.

엔진 계층에도 같은 요구가 문서화돼 있다 (`tree/mod.rs:3298-3311` `process_payload_attributes`) —
*"If it fails, returns `INVALID_PAYLOAD_ATTRIBUTES` without rolling back the forkchoice update."*

**두 계층에서 지킨다.**

---

### ④ validation §4 멱등성 → **준수**

**스펙** (`paris.md:114`)

```
4. Payload validation process **MUST** be idempotent with respect to payload's validity
   status (`VALID | INVALID`), i.e. a payload which validity status is
   `INVALID (INVALID_BLOCK_HASH)` **MUST NOT** become `VALID` and vice versa at any point
   in time when it subsequently runs through the validation process.
```

**코드** (`engine/tree/src/tree/invalid_headers.rs:11`, `payload_validator.rs:1448`)

```rust
/// Keeps track of invalid headers.
pub struct InvalidHeaderCache { ... }
```

```rust
if state.invalid_headers.get(&block.hash()).is_some() {
    // we already marked this block as invalid
    return
}
```

**한 번 무효로 표시한 블록은 재검증하지 않는다.** 재검증을 아예 안 하므로 결과가 뒤집힐 수 없다.
성능 최적화(비싼 실행 회피)와 규격 준수(멱등)가 같은 코드로 해결된다.

---

### ⑤ Sync "implementation dependent" → **재량 행사**

**스펙** (`paris.md:128`)

```
*Note:* Each of these stages is optional. Exact behavior of client software during the
sync process is implementation dependent.
```

**코드**: staged sync 파이프라인(`crates/stages/`)이 그 자리를 채운다. stage 15개를 순서대로
돌리고, 각 stage가 블록 범위 전체를 훑는다.

**스펙이 명시적으로 위임한 지점**이므로 위반 판정 대상이 아니다.

---

### ⑥ validation §6 "side branch" → **적용 대상 아님**

**스펙** (`paris.md:118`)

```
6. The process of validating a payload on the canonical chain **MUST NOT** be affected by
   an active sync process on a side branch of the block tree. For example, if side branch
   `B` is `SYNCING` but the requisite data for validating a payload from canonical branch
   `A` is available, client software **MUST** run full validation of the payload and
   respond accordingly.
```

**코드** (`tree/mod.rs:814-818`)

```rust
let mut outcome = if self.backfill_sync_state.is_idle() {
    self.try_insert_payload(payload)?.into_outcome()      // 정상: 검증하고 삽입
} else {
    TreeOutcome::new(self.try_buffer_payload(payload)?)   // 파이프라인 중: 버퍼에만
};
```

`try_buffer_payload`는 형식 변환만 하고 실행 검증 없이 `SYNCING`을 돌려준다(`:894-911`).

**판정 근거 — 개념이 다르다**

| | 대상 |
|---|---|
| 스펙 §6의 "sync process on a **side branch**" | **곁가지 하나**를 채우는 동기화 |
| reth의 backfill | **정본 체인**을 대량으로 따라잡는 동기화 |

결과적으로는 §6이 금지한 일(정본 검증 지연)이 벌어지지만 **이유가 스펙이 상정한 것과 다르다** —
*"it requires **exclusive access to the database**"*라는 물리적 제약이다(`tree/mod.rs:1232`).
스펙이 이 경우를 다루지 않는다.

---

### ⑦ fcU §2 skip 허용 범위 → **범위 불일치** (판정 보류)

**스펙** (`paris.md:213`)

```
2. Client software **MAY** skip an update of the forkchoice state and **MUST NOT** begin a
   payload build process if there is a known `finalizedBlockHash` and
   `forkchoiceState.headBlockHash` references a `VALID` ancestor of the latest known
   finalized block ...
```

**코드** (`tree/mod.rs:1340-1350`) — 주석이 스펙을 인용하는데 문구가 다르다.

```rust
// According to the Engine API specification, client software MAY skip an update of the
// forkchoice state and MUST NOT begin a payload build process if
// `forkchoiceState.headBlockHash` references a `VALID` ancestor of the head
// of canonical chain, ...
```

```
스펙 :  "a VALID ancestor of the latest known finalized block"
reth :  "a VALID ancestor of the head of canonical chain"
```

```
… ── finalized ── A ── B ── head(tip)
                  └──┴──── 스펙 §2 범위 밖. reth는 여기도 skip한다
       └───────────────── 스펙 §2가 허용한 범위
```

**판정 근거**

- §2는 **MAY**이므로 그것만으로는 위반이 아니다.
- 그런데 넓힌 구간(A·B)은 **fcU §7의 MUST update**에 걸린다. head와 finalized가 VALID이므로
  §7의 전제가 성립하고, 스펙대로면 head를 A로 되돌려야 한다.
- reth는 되돌리지 않는다. OpStack이거나 `unwind_canonical_header` 설정을 켠 경우만 한다
  (`tree/mod.rs:1319-1329`).

→ 체크리스트 1·2·3은 위반을 가리키는데 **4번(도달 가능성)이 약하다.** 정상 CL이 자기 head를
뒤로 돌리는 fcU를 보낼 일이 거의 없다.

**판정: 범위 불일치. 위반 소지 있으나 도달 가능성 낮고 재현하지 않음.**

> 억지로 등급을 붙이기보다 "판정 보류 + 이유"로 두는 것이 정확하다.

---

### ⑧ 초기 fcU에서 `safe`를 동기화 목표로 → **재량 행사**

**스펙**: 해당 조항 없음. **없다는 것 자체가 근거다.**

**코드** (`tree/mod.rs:1388-1398`)

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

`forkchoice_state_tracker.is_empty()` = fcU를 한 번도 못 받았다 = **노드를 막 켰다.** 그
상황에서 head까지 한 번에 가는 대신 safe까지 백필로 따라잡는다.

스펙이 금지하지 않았고, 동기화 방식은 ⑤에서 본 대로 위임 영역이다.

> 월요일에 `safe`를 "RPC 블록 태그용"으로 정리했는데 **동기화 목표로도 쓰인다.**

---

### ⑨ newPayload §6 `ACCEPTED` → **재량 행사**

**스펙** (`paris.md:172` 부근)

```
  * `{status: ACCEPTED, latestValidHash: null, validationError: null}` if the following
    conditions are met:
    - all `transactions` have non-zero length
    - the `blockHash` of the payload is valid
    - the payload doesn't extend the canonical chain
    - the payload hasn't been fully validated                    ← ★
    - ancestors of a payload are known and comprise a well-formed chain.
```

**코드**

```bash
grep -rn "Accepted" --include=*.rs crates/ | grep -v metrics.rs | grep -v test
```

나오는 곳이 셋뿐이고 **생성하는 코드가 없다.**

| 위치 | 무엇 |
|---|---|
| `ethereum/node/src/engine_ssz_containers.rs` | SSZ 직렬화/역직렬화 — 외부에서 받은 값 매핑 |
| `engine/primitives/src/forkchoice.rs:173` | `match` 팔에서 `Valid`와 같이 취급 |
| `tree/metrics.rs:477` | 메트릭 카운터 |

**판정 근거 — 조건절이 성립하지 않는다**

`BlockStatus`에는 "곁가지라서 검증 안 했다"는 상태가 없다 (`tree/mod.rs:3449-3459`).

```rust
pub enum BlockStatus {
    /// The block is valid and block extends canonical chain.
    Valid,
    /// The block may be valid and has an unknown missing ancestor.
    Disconnected { head: BlockNumHash, missing_ancestor: BlockNumHash },
}
```

그리고 삽입 함수가 곁가지도 똑같이 실행한다 (`tree/mod.rs:2962-2977`).

```rust
/// - Handles both canonical and fork chain insertions
///
/// Returns `InsertPayloadOk::Inserted(BlockStatus::Valid)` on successful execution,
```

| reth의 처리 | 반환 |
|---|---|
| 부모를 안다 → **실행해서 검증** (곁가지든 정본이든) | `VALID` |
| 부모를 모른다 → `Disconnected` | `SYNCING` |

**reth는 곁가지도 항상 완전 검증한다.** newPayload §5(*"Client software **MAY NOT** validate the
payload if the payload doesn't belong to the canonical chain"*)가 미룰 권리를 줬는데 안 쓴다.

그러면 `ACCEPTED`의 조건 *"hasn't been fully validated"*가 성립하지 않는다.
→ **스펙이 허용한 것보다 더 많이 검증하고, 그 결과 `ACCEPTED`를 낼 상황을 만들지 않는다.**

---

## 4. ★ ①과 ⑨는 같은 논리 구조다

```
①  한계값을 정하지 않았다   →  "한계를 넘는" 상황이 없다     →  -38006을 낼 일이 없다
⑨  항상 완전 검증한다       →  "검증 안 된 곁가지"가 없다    →  ACCEPTED를 낼 일이 없다
```

**둘 다 "구현을 빠뜨린 것"이 아니라 "그 조건을 만들지 않는 설계"다.**

`grep`으로 "이 에러 코드/상태가 코드에 없다"를 확인하는 데는 5분이면 된다. 그것이 위반인지
판정하려면 **스펙 문장의 조건절을 읽어야 한다.** 결론절(`MUST return X`)만 보고 "X가 없으니
위반"이라고 쓰면 두 항목 다 오판한다.

> **MUST 문장을 볼 때 결론절이 아니라 조건절을 먼저 읽어라.**

---

## 5. 종합

| 등급 | 개수 | 해당 |
|---|---|---|
| 준수 | 2 | ③ ④ |
| 재량 행사 | 4 | ① ⑤ ⑧ ⑨ |
| 적용 대상 아님 | 1 | ⑥ |
| 스펙 공백 | 1 | ② |
| 범위 불일치 (판정 보류) | 1 | ⑦ |
| **위반** | **0** | — |

### 위반 0개는 실패가 아니다

이번 주 목표는 "위반 찾기"가 아니라 **"스펙이 약속한 것과 코드가 하는 것을 대조하는 방법"**을
익히는 것이었다. 그리고 9줄 중 4줄이 금요일 목표에 정확히 답한다 —
**스펙이 클라이언트에 맡긴 지점에서 reth가 무엇을 골랐나.**

| # | reth의 선택 | 스펙이 맡긴 방식 |
|---|---|---|
| ① | 리오르그 깊이 **제한을 두지 않는다** | "the limitation specific to the client software" |
| ⑤ | 동기화를 **staged sync 파이프라인**으로 한다 | "implementation dependent" |
| ⑧ | 초기 동기화 때 head가 아니라 **safe를 먼저 목표로** 한다 | (해당 조항 없음) |
| ⑨ | 곁가지도 **항상 완전 검증한다** | "MAY NOT validate … if not canonical" (권리를 안 씀) |

**⑨가 특히 눈에 띈다** — 스펙이 "안 해도 된다"고 허락한 일을 reth는 한다. 대조에서 나오는
차이가 항상 "덜 지킴"인 것은 아니다.

---

## 6. Phase 4로 가져갈 방법

Week 20~22에서 백서와 소스를 대조할 때 그대로 쓸 것.

1. **MUST / MAY / MUST NOT을 구분해서 뽑는다.** MAY를 안 지켰다고 위반이라 하면 틀린다.
2. **★ 조건절을 먼저 읽는다.** `if X, MUST Y`에서 X가 발생하지 않는 설계면 Y가 없어도 된다.
   (①⑨가 그 실례)
3. **스펙의 침묵도 발견이다.** fcU §3·§4에는 "MUST NOT update"가 있고 §5·§6에는 없다.
   그 비대칭을 찾은 것이 ②의 판정을 "위반"에서 "공백"으로 바꿨다.
4. **주석이 인용한 스펙 문구를 원문과 나란히 놓는다.** ⑦은 그렇게 나왔다.
5. **로그 메시지가 설계 의도를 드러낸다.**
   *"Canonicalized sync target head before safe/finalized could be applied"* 한 줄이 ②의
   판정을 다시 바꿨다.
6. **재현 여부를 따로 적는다.** 못 했으면 못 했다고 쓴다. ②와 ⑦이 그 경우다.
7. **판정을 서둘러 내리지 않는다.** ②는 수요일 "문제일 수 있다" → 목요일 "reth가 인지한 상태"로
   두 번 바뀌었다.
8. **억지로 등급을 붙이지 않는다.** 애매하면 "판정 보류 + 이유"가 정확하다 (⑦).

### 백서에는 MUST/MAY가 없다

Engine API 스펙은 RFC 2119를 써서 강제성이 문장에 박혀 있지만, **백서는 그렇지 않다.**
Phase 4에서는 대신 이런 것을 기준으로 삼아야 한다.

- 백서가 **수치**로 못박은 것 (파라미터, 상한, 주기)
- 백서가 **불변식**으로 서술한 것 ("~하는 일은 없다", "항상 ~이다")
- 백서가 **의도적으로 비워둔 것** ("구현에 따라", "권장")

앞의 둘이 MUST에 대응하고 셋째가 MAY에 대응한다. **강제성 표시가 없으므로 그 분류를 직접
해야 하고, 그 분류가 곧 분석의 절반이다.**

---

## 7. 대조 대상이 아닌 별도 발견

스펙과 무관하지만 이번 주에 나온 것. 13주차 기여 후보로 남긴다.

### `BlockStatus::Valid`의 주석이 부정확하다

`engine/tree/src/tree/mod.rs:3449-3451`

```rust
pub enum BlockStatus {
    /// The block is valid and block extends canonical chain.
    Valid,
```

같은 파일 `:2971`의 `insert_block_or_payload` 문서가 반증한다.

```rust
/// - Handles both canonical and fork chain insertions
///
/// Returns `InsertPayloadOk::Inserted(BlockStatus::Valid)` on successful execution,
```

**fork chain 삽입에도 `Valid`를 돌려주므로 "extends canonical chain"은 좁다.**

10주차 후보 B(`BlockReader::block`의 *"from the database"*가 인메모리 반환을 누락)와 같은
성격이고, 근거가 **같은 파일 안의 다른 독 코멘트**라 더 직접적이다.
`docs(engine): ...` 2줄짜리 PR감.

---

## 8. 오늘의 수확

1. **★ 판정 등급이 둘이 아니라 여섯이다.** "위반/아님"으로만 나누면 대부분을 잘못 분류한다.
2. **★ MUST 문장은 조건절부터 읽는다.** ①⑨가 같은 구조로 "코드에 없는데 위반이 아닌" 경우였다.
3. **★ 위반 0개가 실패가 아니다.** 목표는 "스펙이 맡긴 지점에서 reth가 무엇을 골랐나"였고
   4개를 찾았다.
4. **스펙이 허락한 것을 안 쓰는 선택도 있다.** ⑨ — 곁가지 검증을 미룰 권리를 reth는 안 쓴다.
5. **애매하면 판정을 보류한다.** ⑦을 억지로 "위반"이나 "준수"로 밀어 넣으면 틀린 문서가 된다.
6. **백서에는 MUST/MAY가 없다.** Phase 4에서는 강제성 분류를 직접 해야 하고, 그 분류가 분석의
   절반이다.
