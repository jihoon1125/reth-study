# PR을 올리기까지 — 근거는 사후에 더 강해졌다

> 10주차 목요일. reth에 첫 PR을 올렸다 ([#26550](https://github.com/paradigmxyz/reth/pull/26550)).
> 코드는 46줄. 그런데 **왜 이 코드가 이렇게 생겼는지**를 파는 과정에서 후보를 고를 때는 몰랐던
> 근거가 계속 나왔다.

---

## 1. 올린 것

| | |
|---|---|
| PR | [#26550](https://github.com/paradigmxyz/reth/pull/26550) `fix(rpc): include block numbers in pruned history error` |
| 베이스 | `paradigmxyz:main` (8bfea20a91) |
| 변경 | 3파일, +46 / -7, 커밋 1개 |

`EthApiError::PrunedHistoryUnavailable`를 유닛 변형에서 구조체 변형으로 바꿔서
`requested` / `earliest_available`를 싣는다. 생성 지점 4곳은 그 값을 이미 갖고 있었다.

---

## 2. 이 버그의 출처 — 실사용자 리포트였다

월요일에 "이 에러 경로가 실제로 발생하긴 하나"를 의심했는데, **애초에 메인넷 버그 리포트에서
시작된 코드였다.**

[이슈 #20038](https://github.com/paradigmxyz/reth/issues/20038) (SamWilsn, 2025-11-28):

```bash
curl --json '{"method":"debug_getRawBlock","params":["0xb443"]}' localhost:8545
# 634바이트여야 하는데 528바이트만 옴
```

블록 46147(프론티어)을 요청했는데 **트랜잭션이 빠진 채 헤더만 조용히 반환됐다.**
리포터는 `--full` 노드였다. mattsse 진단:

> ah yeah that makes sense, hence the txs are missing, tho **we're missing a check for the
> retired height** here

### 왜 그런 일이 생겼나

`--full`은 오래된 트랜잭션을 static file에서 지우지만 **헤더는 남긴다.** 그래서 `block()` 입장에서
"바디가 없다"가 두 가지 뜻이 된다.

| 상황 | 의미 |
|---|---|
| 아직 안 들어옴 | 동기화 중. 나중에 다시 물어보면 있음 |
| 이미 지워짐 | 프루닝. 영원히 없음 |

**둘을 구분할 기준이 없었다.** 그래서 둘 다 똑같이 처리했고, 사용자는 잘린 데이터를 받고도
잘린 줄 몰랐다. #21270이 `earliest_history_height`를 기준선으로 세워서 이걸 갈랐다.

### 그리고 리포터의 마지막 질문

> Is the most efficient way to get the early transaction data to resync entirely?

**자기 노드가 어디부터 갖고 있는지 몰라서 물어본 것이다.** 그 답이 바로 우리가 되살리려는
`earliest_available`이다. 후보를 고를 때는 몰랐고, 사후에 찾은 이 한 줄이 근거로는 제일 셌다.

---

## 3. 고쳐온 이력 — 5개월간 작은 PR 네 개

```bash
git log --oneline -S "PrunedHistoryUnavailable" -- crates/rpc
```

| PR | 날짜 | 저자 | 크기 | 추가한 것 |
|---|---|---|---|---|
| #16780 | 2025-06 | mattsse | +9 | 변형만 미리 만듦 (호출자 0) |
| #21270 | 2026-01 | gakonst | +21 | `earliest_history_height` 검사 + `BlockExpired` |
| #21304 | 2026-01 | — | — | `eth_getLogs` 2곳 |
| #23600 | 2026-05 | **외부 기여자** | **+7** | `trace_filter` |
| #24760 | 2026-06 | mattsse | **+4** | `recovered_block()` |

한 번에 다 하지 않고 **빠진 곳을 발견할 때마다 몇 줄씩 채워 넣었다.** 외부 기여자 PR도 그 안에
있다. 이걸 보고 나서 "46줄짜리를 올려도 되나"라는 망설임이 사라졌다.

---

## 4. ★ PR #21270 본문이 사실과 다르다

#21270 본문:

> This enables endpoints like `debug_getRawBlock` to properly return the
> **`PrunedHistoryUnavailable`** error …

**그렇게 안 된다.** 경로를 끝까지 따라가면 갈라지는 지점이 있다.

```
debug_getRawBlock                  rpc/src/debug.rs:777
  → provider().block_by_id(id)     debug.rs:780
      → block_by_number_or_tag → self.block(num)   storage-api/src/block.rs:320
      → DatabaseProvider::block → BlockExpired ✓
  → .to_rpc_result()               debug.rs:781    ← 여기
```

`to_rpc_result()`는 `From<ProviderError> for EthApiError`를 **거치지 않는다.**

```rust
// rpc-server-types/src/result.rs:14
fn to_rpc_result(self) -> RpcResult<Ok> {
    self.map_internal_err(|err| err.to_string())   // → INTERNAL_ERROR_CODE (-32603)
}
```

`impl_to_rpc_result!(reth_errors::ProviderError)`(result.rs:108)가 걸려 있어서 **모든
`ProviderError`가 내부 오류 -32603으로 뭉개진다.** 4444로 매핑하는 코드는 실행되지 않는다.

### 두 경로 중 어느 쪽도 온전하지 않았다

| 경로 | 코드 | 메시지 |
|---|---|---|
| `debug_getRawBlock` (`to_rpc_result`) | **-32603** ❌ | provider의 완전한 메시지 ✅ |
| `eth_getLogs` / `trace_filter` (`EthApiError`) | **4444** ✅ | `"Pruned history unavailable"` ❌ |

우리 PR은 아래쪽을 고친다. 위쪽은 **원인도 범위도 다른 문제**라 섞지 않았다.

| | 이번 PR | -32603 문제 |
|---|---|---|
| 원인 | `EthApiError`가 숫자를 안 실음 | `to_rpc_result`가 모든 provider 에러를 뭉갬 |
| 범위 | 변형 하나 | `to_rpc_result` 호출부 전체 |

→ **후보 D로 남긴다.** 이번 PR이 머지되면 별도 이슈나 후속 PR로.

---

## 5. 절차에서 알게 된 것

- **PR 브랜치는 `upstream/main`에서 딴다.** 로컬 `main`이 아니라. `git remote -v`를 먼저 볼 것.
- **리뷰어를 지정할 수 없다.** 포크 기여자에게는 그 권한이 없고 `.github/CODEOWNERS`가 자동으로
  건다. 우리 3파일은 `crates/rpc/ @mattsse @Rjected`에 걸린다 (CODEOWNERS는 나중에 매칭된
  패턴이 이겨서 `* @gakonst`가 아니다). 같은 성격 PR 4건(#26479, #26502, #26540, #23600)을
  조사하니 전부 Rjected가 자동 요청되고 **실제로는 mattsse가 리뷰·머지**했다.
- **라벨도 못 단다.** `.github/scripts/label_pr.js`가 자동으로 붙이지만 **본문에 `Closes #N`이
  있을 때만** 원본 이슈 라벨을 복사한다. 우리는 이슈를 안 닫으니 메인테이너가 손으로 단다.
- **첫 기여자는 CI가 자동으로 안 돈다.** 워크플로 10개가 전부 `action_required`로 멈춰 있고
  `mergeable_state: unstable`로 뜬다. 실패가 아니라 메인테이너의 "Approve and run workflows"
  대기다.

---

## 6. 오늘의 수확

1. **★ PR의 근거는 사후에 더 강해질 수 있다.** 원본 이슈 #20038을 찾고 나서야 "실사용자가
   자기 노드 범위를 몰라 물어봤다"는 증거가 나왔다. 후보를 고를 때는 몰랐던 것이다.
2. **★ PR 본문도 틀릴 수 있다.** #21270이 "`debug_getRawBlock`이 4444를 반환한다"고 써놨는데
   호출 경로를 따라가니 -32603이었다. 설명보다 코드가 정확하다.
3. **같은 종류의 실수가 계층 경계에서 반복된다.** provider가 담은 정보를 RPC가 버리고,
   `to_rpc_result`가 에러 종류를 뭉갠다. 둘 다 "경계를 넘을 때 정보가 준다"는 같은 모양이다.
4. **작게 나눠 올리는 게 이 리포의 방식이다.** 같은 문제를 5개월간 +4, +7, +21짜리 PR 네 개로
   메웠다. 한 번에 완결하려 들 이유가 없다.

## 7. 다음으로 넘길 질문

1. **후보 D** — `to_rpc_result`가 모든 `ProviderError`를 -32603으로 뭉개는 문제.
   `debug_*` 계열 전체에 영향이 있는데, 고치려면 호출부를 몇 개나 봐야 하나?
2. #16780(변형만 미리 생성)과 `MustUnwind`(9주차 §5, 호출자 0)가 같은 모양이다.
   reth에서 "먼저 만들어두고 나중에 배선"이 얼마나 흔한 패턴인가?
3. static file은 500K 블록 단위라 dev 노드로는 `BlockExpired`를 재현할 수 없다.
   노드 쪽에 blocks-per-file 오버라이드가 없는 이유는?
4. 리뷰에서 `S-breaking` 라벨이 붙을까? reth가 에러 enum 변경을 브레이킹으로 취급하는
   기준선은 어디인가?
