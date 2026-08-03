# Engine API — 합의 계층이 하는 말은 두 종류뿐이다

> 12주차 월요일. 8~11주차는 storage 하나를 확대해서 봤고, 이번 주는 그게 전체 어디에 붙어
> 있는지 좌표를 잡는다. 대상은 Engine API — reth(실행 계층)와 Lighthouse 등(합의 계층)의 경계.

---

## 1. 대상 선택 — A(Engine API)

후보는 둘이었다.

| 후보 | 왜 매력적인가 | 왜 골랐나/버렸나 |
|---|---|---|
| **A. Engine API** | 8~10주차에 판 storage와 직접 연결 | ✅ |
| B. devp2p | 분산시스템 감각, 완전히 다른 표면 | ❌ storage와 접점이 적어 "좌표 잡기"에 안 맞음 |

### 그런데 A를 고른 근거 하나가 코드와 안 맞았다

계획서의 추천 이유는 이랬다.

> forkchoice가 바뀌면 어떤 블록을 정본으로 볼지가 바뀌고, 그게 unwind로 이어진다

확인해보니 아니었다.

```bash
grep -rn "PipelineTarget::Unwind" --include=*.rs crates/
```

프로덕션 코드에서 이걸 만드는 곳은 두 곳뿐이다.

| 위치 | 상황 |
|---|---|
| `node/builder/src/launch/common.rs:566` | 기동 시 정합성 검사 — **9주차에 분석한 그것** |
| `cli/commands/src/common.rs:229` | CLI 명령 |

**리오르그 경로가 없다.** 9주차 [error-paths.md](../week9/error-paths.md) §3에 "되감기가 필요한 상황은 ① 리오르그 ② 정합성 깨짐"이라고 썼는데, **①이 파이프라인 unwind가 아니었다.**

리오르그는 엔진 트리가 자기 메모리에서 처리한다. 자세한 것은 [pipeline-and-stages.md](pipeline-and-stages.md)에 정리했다.

> 결과적으로 A가 맞는 선택이었다. 다만 이유가 계획서와 다르다 — **"잃어버린 퍼즐 조각"이
> 있는 게 아니라, 내가 끼워 맞춰뒀던 조각이 잘못 끼워져 있었다.**

---

## 2. 두 함수가 약속한 것

📖 `execution-apis/src/engine/paris.md` (297줄). 이후 하드포크 파일(`shanghai`/`cancun`/
`prague`/`osaka`/`amsterdam.md`)은 필드 추가 델타라 의미론은 전부 여기 있다.

| | 합의 계층이 하는 말 | reth가 하는 일 |
|---|---|---|
| `engine_newPayloadV1` | "이 블록 받아라. 유효한가?" | 검증하고 상태를 돌려줌 |
| `engine_forkchoiceUpdatedV1` | "이 해시를 head로 삼아라" | 정본 체인을 갱신 (+ 선택적으로 블록 생성 시작) |

둘 다 `timeout: 8s`. 슬롯이 12초라 그 안에서 끝나야 한다.

### 왜 하나로 안 합쳤나

블록을 받는 것과 그 블록을 정본으로 삼는 것이 **다른 시점에 일어날 수 있기 때문**이다.

```
newPayload(A)      → "받았고 유효함"
newPayload(A')     → "받았고 유효함"    ← 경쟁 블록. 이것도 유효
forkchoiceUpdated(A) → "A를 head로"     ← 여기서 비로소 정본이 정해짐
```

**유효한 블록이 둘 이상 있을 수 있고, 그중 무엇을 따를지는 검증이 아니라 합의의 문제다.**
실행 계층은 유효성만 판정하고 선택은 합의 계층이 한다. 함수가 갈린 이유가 그것이다.

### 갈라진 블록은 번호가 같다

```
                (101, 0xAA) ── (102, 0xAB)              ← 내가 먼저 받은 쪽
(100, 0x0A) ────┤
                (101, 0xBB) ── (102, 0xBC) ── (103, 0xBD)   ← 투표에서 이긴 쪽
```

101이 둘, 102가 둘 있다. **번호만으로는 특정할 수 없다.**

그래서 `forkchoiceState`가 번호가 아니라 **해시**를 쓴다. 같은 이유가 provider 쪽에도
적혀 있다 (`storage-api/src/block.rs`).

```rust
/// Note: this only operates on the hash because the number might be ambiguous.
```

---

## 3. `ForkchoiceStateV1` — 세 개의 포인터

```
- `headBlockHash`: block hash of the head of the canonical chain
- `safeBlockHash`: the "safe" block hash ... This value **MUST** be either equal to
  or an ancestor of `headBlockHash`
- `finalizedBlockHash`: block hash of the most recent finalized block
```

```
제네시스 ─── … ─── finalized ─── … ─── safe ─── … ─── head
                   되돌릴 수 없음      되돌리기 어려움    최신
```

`safe`에 붙은 조건이 *"MUST be either equal to or an ancestor of headBlockHash"* — **세 개가
항상 한 사슬 위에 있어야 한다.** 어기면 `-38002`.

### `safe`와 `finalized`가 둘 다 있는 이유

리오르그 처리와는 **관계 없다.** 리오르그는 `current_canonical_head`만 보고 공통 조상을
찾는다. 실제 용도는 **RPC 블록 태그**다.

`storage-api/src/block.rs:344-352`

```rust
self.sealed_header_by_id(BlockNumberOrTag::Safe.into())
self.sealed_header_by_id(BlockNumberOrTag::Finalized.into())
```

`eth_getBlockByNumber("safe")` / `("finalized")`로 조회된다. 쓰는 쪽은 거래소·앱이다 —
"입금을 몇 블록 기다렸다가 확정 처리할까"의 기준.

| | 언제 정해지나 | 뒤집힐 가능성 |
|---|---|---|
| `safe` | 몇 슬롯 (수십 초) | 낮음 |
| `finalized` | 2 에폭 ≈ **12.8분** | 검증자 1/3 이상이 슬래싱을 감수해야 가능 |

하나만 있으면 **13분을 기다리든가, 보장 없이 head를 쓰든가** 둘뿐이다. 중간 선택지를 주려고
둘을 노출한다.

### finalized가 뒤집히면?

프로토콜상 불가능하지 않다. 그런데 **reth에는 그 경우에 대응하는 코드가 없다.** finalized
아래는 안 바뀐다고 가정하고 프루닝하고 static file로 내려보낸다 — 10주차에 본
`earliest_history_height`가 그렇게 만들어진 값이다. 되돌릴 데이터가 이미 디스크에서 사라져
있다.

> 이건 코드로 처리할 문제가 아니라 **처리하지 않기로 한 문제**다.
> 9주차 §5의 `assert_ne!(unwind_block, 0)`도 같은 종류였다 — 고치느니 죽는 지점이 있다.

---

## 4. ★ "판정 못 함"을 다섯 가지로 쪼갠 것

```
- `status`: "VALID" | "INVALID" | "SYNCING" | "ACCEPTED" | "INVALID_BLOCK_HASH"
- `latestValidHash`: DATA|null - the hash of the most recent *valid* block in the
  branch defined by payload and its ancestors
- `validationError`: String|null
```

| status | 뜻 |
|---|---|
| `VALID` | 검증했고 유효 |
| `INVALID` | 검증했고 무효 |
| `INVALID_BLOCK_HASH` | `blockHash`가 헤더와 안 맞음 (내용을 볼 것도 없음) |
| `SYNCING` | **검증에 필요한 데이터가 없어서** 판정 못 함 |
| `ACCEPTED` | 데이터는 있는데 **정본을 연장하지 않아서** 완전 검증을 안 함 |

### 검사가 두 부류로 갈린다

`newPayloadV1` §1~2는 *"MUST run this validation **in all cases** even if this branch or any
other branches of the block tree are in an active sync process"*로 끝난다.

**두 검사(트랜잭션 길이, `blockHash`)는 payload 안의 데이터만으로 판정된다.** 조상도 상태도
디스크도 필요 없다. 그래서 무조건 한다.

반면 "실행하면 `stateRoot`가 맞나"는 부모 상태가 필요하다. 그건 조건부(§4)다.

```
게이트 1: 다른 데이터가 필요한가?  ─ 아니오 → 무조건 검증 (§1, §2)
                                   ─ 예 ↓
게이트 2: 데이터가 있고 정본인가?  ─ 예   → 반드시 검증 (§4)
                                   ─ 아니오 → 미룸 (§5) → SYNCING 또는 ACCEPTED
```

### `SYNCING` vs `ACCEPTED` — 반대 방향의 "모름"

```
SYNCING  : 조상을 모른다   → 데이터가 없어서 못 했다
ACCEPTED : 조상을 안다     → 정본이 아니라서 안 했다 (일부러)
```

`ACCEPTED`의 조건 마지막 줄이 결정적이다 — *"ancestors of a payload are known and comprise
a well-formed chain"*.

**`SYNCING`은 타임아웃이 아니다.** 8초 안에 "지금은 모른다"고 **답하는** 것이다. 데이터를 다
받아온 뒤에 답하려 하면 타임아웃이 되고, 합의 계층은 노드가 죽었는지 일하는 중인지 구분할 수
없다. **답을 안 하는 것과 "모른다"고 답하는 것은 완전히 다르다.**

### `forkchoiceUpdated`는 status가 3개로 제한된다

```
- `payloadStatus`: PayloadStatusV1; values of the `status` field in the context of
  this method are restricted to the following subset:
  * "VALID"  * "INVALID"  * "SYNCING"
```

같은 구조체를 쓰면서 허용 값을 좁힌다. 왜 둘이 빠졌는지가 두 함수의 차이를 그대로 보여준다.

| 빠진 값 | 이유 |
|---|---|
| `ACCEPTED` | "정본이 아니라서 안 했다"인데, **이 함수는 정본으로 삼으라는 명령**이다. 성립하지 않음 |
| `INVALID_BLOCK_HASH` | payload 본문을 받았을 때만 가능한 검사. **이 함수는 해시만 받는다** |

### `latestValidHash` — "틀렸다"만 말하지 않는다

Payload validation §3이 세 값을 규정한다.

| 값 | 뜻 |
|---|---|
| 해시 | "여기까지는 유효하다" — 합의 계층이 여기서부터 다시 시도 가능 |
| `0x000…000` | 유효한 조상이 PoW 블록 (머지 전 구간) |
| `null` | **어디까지 유효한지 모른다** |

`null`은 클라이언트가 최대한 피해야 하는 응답이다 — 받는 쪽이 **어디서 다시 시작할지 알 수 없다.**

9주차 [error-paths.md](../week9/error-paths.md) §1에 이렇게 썼다.

> ③만 "가능한 범위"를 필드로 같이 준다 — 호출자(RPC)가 사용자에게 대안을 알려줄 수 있게

`latestValidHash`가 정확히 그 역할이고, **이쪽은 스펙 레벨에서 MUST로 강제돼 있다.**
10주차 PR에서 고친 것과 같은 발상이다.

---

## 5. §4와 §8.3은 충돌이 아니다

`forkchoiceUpdated`를 읽다가 걸린 지점. 두 조항이 반대로 보인다.

```
§4:   "If the validation process fails, client software MUST NOT update the
       forkchoice state and MUST NOT begin a payload build process."
§8.3: "If payloadAttributes validation fails, the forkchoiceState update
       MUST NOT be rolled back."
```

**대상이 다르고 시점이 다르다.**

```
1단계  headBlockHash가 가리키는 payload 검증
         └─ 실패 → 갱신 안 함, 빌드 안 함              (§4)
         └─ 성공 → 2단계

2단계  forkchoice 갱신 (head/safe/finalized) — 원자적으로   (§7)

3단계  payloadAttributes 검증 (timestamp 등)
         └─ 실패 → 빌드만 안 함. 2단계는 유지            (§8.3)
         └─ 성공 → 빌드 시작, payloadId 반환
```

§4의 "실패"는 **블록 자체가 무효**다. 무효한 블록을 정본으로 삼을 수 없으니 갱신하면 안 된다.

§8.3의 "실패"는 **블록 만들기 요청이 잘못됐다**는 뜻이다(예: `timestamp`가 head보다 작음).
그건 head 블록의 유효성과 무관하다. head는 멀쩡하니 갱신은 유지하고 빌드 요청만 거절한다.

에러 코드가 갈라져 있는 것이 증거다.

| 실패 지점 | 코드 |
|---|---|
| §5 forkchoiceState가 모순 | `-38002 Invalid forkchoice state` |
| §8.1 payloadAttributes가 잘못됨 | `-38003 Invalid payload attributes` |

> §8.3을 명시한 이유는 **"함수 중간에 실패하면 전부 롤백"이 기본 감각**이기 때문이다.
> 스펙이 반대를 못박아 뒀다.

---

## 6. 금요일 대조표용 주장 목록

`spec-vs-code.md`에서 "스펙이 약속한 것 vs 코드가 하는 것"을 대조할 재료. **원문 그대로** 옮겨
둔다 (요약하면 금요일에 스펙을 다시 읽어야 한다).

| # | 출처 | 스펙 문장 | 확인할 것 |
|---|---|---|---|
| 1 | fcU §6 | "MUST return `-38006: Too deep reorg` … if the depth of reorg … exceeds **the limitation specific to the client software**" | **스펙이 값을 안 정했다. reth의 한계값은?** |
| 2 | fcU §7 | "All updates to the forkchoice state resulting from this call MUST be made **atomically**" | 저장소가 셋인 reth가 어떻게 지키나 |
| 3 | fcU §8.3 | "the `forkchoiceState` update **MUST NOT** be rolled back" | 지키나 (직관과 반대라 틀리기 쉬움) |
| 4 | validation §4 | "Payload validation process MUST be **idempotent**" | `InvalidHeaderCache`로 지키는 것으로 보임 → 확인 |
| 5 | Sync note | "Exact behavior of client software during the sync process is **implementation dependent**" | **스펙이 비워둔 자리.** staged sync가 이 자리 |
| 6 | validation §6 | "The process of validating a payload on the canonical chain MUST NOT be affected by an active sync process on a side branch" | 지키나 |

**1번과 5번이 성격이 같다** — 스펙이 명시적으로 구현에 위임한 지점. 금요일 목표("스펙에 없는
reth만의 것")에 정확히 맞는다.

**4번은 이미 후보 코드를 찾았다.**

`engine/tree/src/tree/invalid_headers.rs:11`

```rust
/// Keeps track of invalid headers.
pub struct InvalidHeaderCache {
```

`engine/tree/src/tree/payload_validator.rs:1448`

```rust
if state.invalid_headers.get(&block.hash()).is_some() {
    // we already marked this block as invalid
    return
}
```

**한 번 무효로 표시한 블록은 재검증하지 않는다.** 결과가 뒤집힐 수 없다. 성능 최적화(비싼 실행
회피)와 규격 준수(멱등)가 같은 코드로 해결된다.

### 멱등성을 왜 MUST로 못박았나

합의 계층은 그 판정을 근거로 **이미 투표(attestation)를 했다.** `INVALID`이라고 답했으면 그
블록에 투표하지 않고 다른 가지를 지지했고, 그 서명은 네트워크에 뿌려져 회수할 수 없다.

그리고 판정은 노드 밖으로도 나간다 — validation §3 마지막 줄:

```
* Client software MUST NOT surface an INVALID payload over any API endpoint and
  p2p interface.
```

---

## 7. 오늘의 수확

1. **★ 함수가 둘로 갈린 기준은 "유효성이냐 선택이냐"다.** 유효한 블록이 둘 이상 있을 수 있고,
   그중 무엇을 따를지는 실행 계층이 정할 문제가 아니다.
2. **★ 갈라진 블록은 번호가 같다.** 그래서 forkchoice가 해시로 지정한다. provider의
   `find_block_by_hash` 주석("the number might be ambiguous")과 같은 이유.
3. **`SYNCING`은 타임아웃이 아니라 정상 응답이다.** 8초 제한이 있으니 "모른다"를 규격 안의
   답으로 만들어야 했다.
4. **`SYNCING`과 `ACCEPTED`는 반대 방향의 모름이다.** 조상을 모르는 것 vs 알지만 일부러 안 한 것.
5. **`latestValidHash`는 "어디까지는 맞다"를 같이 준다.** 10주차 PR과 같은 발상이 스펙 레벨에
   MUST로 박혀 있다.
6. **`safe`/`finalized`는 리오르그 기계가 아니라 RPC 블록 태그다.** 확실성과 대기 시간의
   선택지를 앱에게 주는 용도.
7. **계획서의 전제도 검증 대상이다.** "forkchoice → unwind"가 코드에 없었고, 그걸 확인한 게
   이번 주 방향을 정했다.
8. **대조는 MUST 조항으로만 해야 한다.** MAY를 안 지켰다고 위반이라 하면 틀린다.

## 8. 다음으로 넘길 질문

1. `-38006 Too deep reorg`의 reth 한계값은 얼마인가? (금요일)
2. forkchoice 세 포인터의 **원자적** 갱신을 어디서 보장하나? 8주차에 본 "저장소 셋을 순서로
   해결"과 같은 방식인가, 다른 방식인가
3. `ACCEPTED`로 미뤄둔 곁가지가 나중에 정본이 되면, 그 검증은 정확히 어느 함수에서 일어나나
4. `newPayload`가 받은 `stateRoot`/`receiptsRoot`는 실행 결과와 대조하는 용도인데, 그 대조가
   실패하면 8~10주차에 본 storage에는 무엇이 남나 (부분 실행 상태가 남는가)
