# 고칠 곳 찾기 — 이슈 트래커는 입구가 아니었다

> 10주차 월요일. 처음으로 "읽기"가 아니라 "고칠 것 찾기"를 했다.
> 가장 큰 수확은 후보 자체가 아니라 **이 리포에서 기여가 시작되는 경로가 내 예상과 달랐다**는 것.

---

## 1. 이슈 트래커부터 — 그리고 첫 번째 오류

`good first issue`로 검색해서 0개가 나왔다. 그런데 라벨 이름이 틀렸다.

```bash
curl -s "https://api.github.com/repos/paradigmxyz/reth/labels?per_page=100" \
  | python3 -c "import json,sys;[print(l['name']) for l in json.load(sys.stdin)]"
```

라벨이 **83개**고, 실제 이름은 `D-good-first-issue`다. 이름을 찍어서 맞히는 게 아니라 접두사로 축이 나뉘어 있었다.

| 접두사 | 축 | 개수 |
|---|---|---|
| `A-` | 영역 (db, rpc, trie, staged-sync…) | 29 |
| `C-` | 종류 (bug, docs, perf, enhancement…) | 16 |
| `D-` | 난이도 (`good-first-issue`, `complex`) | 2 |
| `S-` | 상태 (needs-triage, blocked, stale…) | 12 |
| 기타 | `P-` 우선순위 / `E-` 하드포크 / `M-` 메타 / `O-` OS | 24 |

**직교 축이라 교집합으로 뽑아야 한다.** `A-db` 하나만 본 게 잘못이었다.

## 2. 열린 이슈 66개 — 미배정 32개

라벨로 거를 게 아니라 전수가 가능한 규모였다.

```bash
for p in 1 2 3 4; do
  curl -s "https://api.github.com/repos/paradigmxyz/reth/issues?state=open&per_page=100&page=$p"
done
```

| | |
|---|---|
| 열린 이슈 (PR 제외) | 66 |
| 배정됨 | 34 (51%) |
| 미배정 | 32 (49%) |

"다 배정돼 있어서 할 게 없다"는 절반만 맞다. 미배정 32개는 sparse trie 상태 루트 버그, 엔진 리오르그, "의존성 1093개를 줄여라" 같은 것들이다.

> **미배정 = 남아 있는 게 아니라 아무도 손 안 대는 것.**

8~9주차 범위와 겹치는 셋은 전부 배정 상태였다.

| 이슈 | 라벨 | 상태 |
|---|---|---|
| #19946 `--engine.memory-block-buffer-target` 문서화 | `C-docs, D-good-first-issue, A-cli` | 2025-11부터 배정, 8개월 무진전. 인계 요청 2명 대기 |
| #20900 `ProviderFactory` 정합성 문서 | `C-docs, A-db, A-static-files` | 메인테이너 배정 |
| #20301 replay/trace EVM 에러 메시지 | `A-rpc, D-good-first-issue` | 배정 + EVM 실행 영역(미학습) |

#19946은 9주차 목·금에 판 주제(캐시 vs 인메모리, `persistence-threshold`)와 정확히 겹쳐서 아쉬웠지만, 이미 줄이 서 있어서 배제했다.

## 3. ★ reth는 이슈 기반 리포가 아니다

여기서 전제를 의심했다. 이슈에서 자리를 못 잡으면 정말 기여를 못 하나?

최근 머지된 PR 150개(#25285~#26546)를 조사했다.

| | |
|---|---|
| 외부 저자 PR | **114 / 150 (76%)** |
| `Closes/Fixes #N`으로 이슈를 닫는 PR | **1 / 150** |

**150개 중 1개.** 나머지는 전부 이슈 없이 올라왔다.

실제로 머지된 외부 PR들:

| PR | diff | 리뷰 | 소요 |
|---|---|---|---|
| #26484 `perf(provider): avoid range scan…` | +6/-7, 1파일 | 1 | 당일 |
| #26508 `docs(engine): document sparse trie…` | +28/-1, 2파일 | 0 | 당일 |
| #26513 `fix(net): reply to GetNodeData…` | +34/-2, 1파일 | 0 | 1일 |
| #26479 `fix(rpc): align debug trace error codes` | +51/-2, 2파일 | 0 | 당일 |

> "배정 안 된 이슈 찾기"는 이 리포에서 잘못된 전략이다.
> **코드를 읽다 발견한 것을 바로 PR로 올리는 게 정상 경로다.**

그래서 8~9주차 노트로 돌아갔다.

---

## 4. 후보 3개

### 후보 A — `BlockExpired`가 담은 정보를 RPC가 버린다 ✅ 선정

9주차 화요일 [error-paths.md](../week9/error-paths.md) §1에 이렇게 썼다.

> ③만 "가능한 범위"를 필드로 같이 준다 — 호출자(RPC)가 사용자에게 대안을 알려줄 수 있게.

**그 호출자가 실제로는 버리고 있다.**

provider는 담는다 (`storage/provider/src/providers/database/provider.rs:1849`):

```rust
return Err(ProviderError::BlockExpired { requested: number, earliest_available })
```

RPC 경계에서 사라진다 (`rpc/rpc-eth-types/src/error/mod.rs:543`):

```rust
ProviderError::BlockExpired { .. } => Self::PrunedHistoryUnavailable,
```

사용자가 받는 건 `{"code": 4444, "message": "Pruned history unavailable"}` 한 줄. "그래서 몇 번부터 되냐"를 못 알려준다.

### 후보 B — `BlockReader::block` 독 코멘트가 틀렸다

`storage/storage-api/src/block.rs:93`:

```rust
/// Returns the block with given id from the database.
```

"from the database"가 아니다. reth 자기 테스트가 반증한다 (`blockchain_provider.rs:1400`, `// First in memory block should be found`). 8주차에 판 `get_in_memory_or_storage_by_block` 분기 그대로다.

2~4줄짜리. **보류** — A를 먼저 끝내고 시간이 남으면 별도 브랜치로.

### 후보 C — 죽은 pub API ❌ 탈락

`new_checked` / `assert_consistent` / `MustUnwind`. 9주차에 확인한 대로 워크스페이스 내 호출자 0이 그대로다.

**버린 이유는 "고칠 게 없어서"가 아니다.** pub API 삭제는 "외부 사용자가 쓸 수도 있다"는 정책 판단이 필요하고, 그 판단은 메인테이너 몫이다. 코드로 답할 수 있는 문제가 아니라 물어봐야 하는 문제다. 나중에 이슈로 여는 게 맞다.

---

## 5. A를 고른 근거 넷

### ① 같은 enum 안의 비대칭 — 가장 강한 근거

`rpc-eth-types/src/error/mod.rs`에서 문제의 변형 **8줄 위**:

```rust
/// Requested block number is beyond the head block
#[error("request beyond head block: requested {requested}, head {head}")]
RequestBeyondHead { requested: u64, head: u64 },

/// Thrown when historical data is not available because it has been pruned
#[error("Pruned history unavailable")]
PrunedHistoryUnavailable,
```

**위쪽 경계를 넘으면 두 숫자를 알려주고, 아래쪽 경계를 넘으면 아무것도 안 알려준다.** 같은 enum, 같은 성격, 8줄 간격.

### ② 같은 커밋에서 만들고 같은 커밋에서 버렸다

```bash
git log --oneline -S "PrunedHistoryUnavailable" -- crates/rpc
```

| 커밋 | 날짜 | 한 일 |
|---|---|---|
| `6ddc756489` #16780 | 2025-06 | 유닛 변형 생성. 9줄, 호출자 0 — 미리 만들어둠 |
| `2305c3ebeb` #21270 | 2026-01 | `BlockExpired` 생성 **+ `{ .. }`로 버리는 매핑** |
| `cc242f83fd` #21304 | 2026-01 | `eth_getLogs` 2곳 |
| `e9507f5907` #23600 | 2026-05 | `trace_filter` 1곳 (**외부 기여자, +7줄**) |

`#21270` 한 커밋 안에서 이 둘이 같이 들어갔다.

```rust
// storage/errors/src/provider.rs — 메시지를 공들여 씀
#[error("block #{requested} is not available, history has expired (earliest available: #{earliest_available})")]
BlockExpired { requested: BlockNumber, earliest_available: BlockNumber },
```
```rust
// rpc-eth-types/src/error/mod.rs — 같은 커밋, 한 줄
+ ProviderError::BlockExpired { .. } => Self::PrunedHistoryUnavailable,
```

공들여 쓴 메시지를 옆에서 버린다. **설계 판단이 아니라 배선 누락이다.**

### ③ 생성 지점 4곳이 전부 이미 값을 갖고 있다

```bash
grep -rn "PrunedHistoryUnavailable" --include=*.rs .
```

전 저장소 6곳뿐이고, 만드는 곳 4개는 모두 바로 윗줄에 두 값이 있다.

| 위치 | requested | earliest |
|---|---|---|
| `rpc/src/trace.rs:373` | `start` | `earliest_block` |
| `rpc/src/eth/filter.rs:495` | `block_number` | `earliest_block` |
| `rpc/src/eth/filter.rs:586` | `from_block_number` | `earliest_block` |
| `rpc-eth-types/src/error/mod.rs:543` | `BlockExpired`에서 그대로 | 〃 |

```rust
// trace.rs:371-374 — 구해놓고 안 쓴다
let earliest_block = self.provider().earliest_block_number()...?;
if start < earliest_block {
    return Err(EthApiError::PrunedHistoryUnavailable.into());
}
```

`Option`으로 감쌀 이유가 없다. 구현 비용이 사실상 0이다.

### ④ 죽은 코드 경로가 아니다

처음엔 "EIP-4444 미래용이라 발생 안 하는 것 아닌가" 의심했는데, 지금 있는 플래그로 켜진다.

```
--prune.bodies.distance <BLOCKS>          node/core/src/args/pruning.rs:241
  → config.segments.bodies_history        pruning.rs:312
  → Bodies 프루너                          prune/segments/user/bodies.rs
  → delete_segment_below_block(Transactions, …)
                                          static_file/manager.rs:841
  → delete_jar → initialize_index()       manager.rs:1114
  → earliest_history_height = 최저 jar 시작 블록
                                          manager.rs:1224-1231
  → block_number < earliest_available  →  BlockExpired
```

`--full`이나 `--prune.bodies.distance`로 돌리는 노드에서 실제로 난다.

---

## 6. 반론 검토

| 반론 | 검토 |
|---|---|
| "그 정보는 이미 조회 가능하다" | 맞다. `storage-api/src/block_id.rs:64`가 `Earliest` 태그를 `earliest_block_number()`로 푼다. **하지만** 그 논리면 `RequestBeyondHead`도 `eth_blockNumber`로 알 수 있으니 없어야 한다. 동시에 이건 **새로 노출되는 정보가 0**이라는 뜻이기도 하다 |
| "브레이킹 체인지다" | **인정.** 유닛 → 구조체 변형이라 `EthApiError`를 쓰는 downstream이 깨진다. 반박하지 않고 `S-breaking` 라벨로 표시한다 |
| "기존 메시지에 의존하는 테스트가 있나" | 없다. `grep -rn "Pruned history unavailable"` → 정의 1곳뿐 |
| "규격을 어기지 않나" | 에러 코드 `4444`는 그대로 둔다. JSON-RPC `message`는 자유 텍스트다 |

리뷰어가 "메시지 말고 `rpc_err`의 `data` 필드에 넣어라"고 할 수는 있다. 그래도 **enum에 필드를 넣는 것은 어느 쪽이든 필요**하므로 방향은 안 바뀐다.

---

## 7. 오늘의 수확

1. **★ 라벨을 이름으로 찍지 마라.** 83개가 `A-/C-/D-/S-` 4축으로 나뉜 직교 분류인데 `good first issue`라고 찍어서 "0개"라는 틀린 결론을 냈다. 분류 체계를 먼저 보고 교집합을 뽑아야 했다.
2. **★ reth에서 이슈는 기여 입구가 아니다.** 머지 PR 150개 중 이슈를 닫는 건 1개. "배정 안 된 이슈 찾기"라는 전략 자체가 이 리포에 안 맞았다.
3. **미배정 ≠ 여유분.** 32개가 비어 있지만 전부 아무도 안 건드리는 난제다.
4. **내가 쓴 노트가 후보를 만들어줬다.** 후보 A는 9주차 화요일에 내가 쓴 문장("호출자가 사용자에게 대안을 알려줄 수 있게")이 코드에서 안 지켜지는 걸 발견한 것이다. 노트를 안 썼으면 못 찾았다.
5. **`git log -S`가 의도를 알려준다.** "왜 이렇게 돼 있나"는 추측할 게 아니라 그 줄이 들어온 커밋을 찾으면 된다. 같은 커밋에서 만들고 버렸다는 사실이 "설계냐 누락이냐"를 그 자리에서 끝냈다.
6. **"이건 죽은 코드 아닌가"를 먼저 의심해라.** 안 그랬으면 발생하지도 않는 경로를 고치는 PR을 쓸 뻔했다. 플래그에서 에러까지 사슬을 끝까지 이은 다음에야 후보로 확정했다.

## 8. 다음으로 넘길 질문

1. `EthApiError`의 브레이킹 체인지 기준은 뭔가? reth는 에러 enum을 자주 바꾸는데 `S-breaking`/`M-migration-notes`를 붙이는 선이 어디인가
2. `#16780`은 호출자 없이 변형만 먼저 만들었다. reth에서 "먼저 만들어두고 나중에 배선"이 얼마나 흔한 패턴인가 — `MustUnwind`(9주차 §5)도 같은 모양이었다
3. static file은 500K 블록 단위라 dev 노드에서는 `BlockExpired`를 재현할 수 없다. 노드 쪽에 blocks-per-file 오버라이드가 없는 이유는?
4. 후보 C(`MustUnwind` 죽은 API)는 이슈로 여는 게 맞나, 아니면 그냥 놔두는 게 맞나
