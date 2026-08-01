# 모듈 경계 — 누가 provider를 쓰고, 무엇을 쓰나

> 9주차 수요일. 결론: 소비자는 두 부류로 갈리고, 갈리는 기준이 Cargo.toml에 그대로 찍혀 있다.

---

## 1. 의존 관계가 이미 답이다

```bash
awk '/^\[/{sec=$0} /^reth-provider/{print sec"  ->  "$0}' <Cargo.toml>
```

| 크레이트 | `reth-provider` 위치 |
|---|---|
| `rpc/rpc-eth-api` | **없음** (`reth-storage-api`만) |
| `rpc/rpc` | `[dev-dependencies]` |
| `net/network` | `[dev-dependencies]` |
| `stages/stages` | **`[dependencies]`** |

`rpc`와 `network`의 **프로덕션 코드는 `reth-provider`를 컴파일조차 하지 않는다.**
트레이트(`reth-storage-api`)만 알고, 구현체는 테스트에서 mock을 쓸 때만 끌어온다.

정식 의존은 `stages`뿐이다.

---

## 2. 경계선 — 저장소를 직접 만지느냐

`stages`가 import하는 것들을 정의 위치로 갈라보면 선이 뚜렷하다.

| 트레이트 | 정의 위치 |
|---|---|
| `BlockReader`, `BlockWriter`, `DBProvider`, `StatsReader`, `HeaderSyncGapProvider` | **storage-api** |
| `StaticFileWriter` | **provider** (`providers/static_file/manager.rs`) |
| `StaticFileProviderFactory` | **provider** (`traits/static_file_provider.rs`) |
| `RocksDBProviderFactory` | **provider** (`traits/rocksdb_provider.rs`) |
| `EitherWriter` | **provider** (`providers/either_writer.rs`) |

> **storage-api = 저장소 종류와 무관한 읽기/쓰기 트레이트**
> **provider = 저장소 3종(MDBX / static file / RocksDB)에 직접 묶인 것**

`BlockWriter`가 storage-api에 있다는 게 이 선을 잘 보여준다. **쓰기냐 읽기냐가 기준이 아니라,
특정 저장소를 지목하느냐가 기준이다.** `BlockWriter`는 "블록을 쓴다"만 말하고 어디에 쓰는지는
말하지 않으므로 storage-api에 있다. `StaticFileWriter`는 이름부터 저장소를 지목하므로 provider에 있다.

`stages`가 `reth-provider`를 정식 의존하는 이유가 이것이다 — **static file writer와 RocksDB
팩토리를 직접 만져야 한다.**

### stage가 요구하는 바운드

`crates/stages/stages/src/stages/bodies.rs:145-152`

```rust
impl<Provider, D> Stage<Provider> for BodyStage<D>
where
    Provider: DBProvider<Tx: DbTxMut>       // MDBX 쓰기 트랜잭션
        + StaticFileProviderFactory         // SF writer 접근
        + StatsReader
        + BlockReader
        + BlockWriter<Block = D::Block>,
```

`DBProvider<Tx: DbTxMut> + StaticFileProviderFactory`가 stage들의 공통 세트고, 거기에 stage별로
필요한 것이 붙는다.

| stage | 추가로 요구하는 것 | 이유 |
|---|---|---|
| `headers` | `BlockHashReader`, `HeaderSyncGapProvider` | 헤더 다운로드 구간 계산 |
| `bodies` | `BlockWriter`, `StatsReader` | 바디 저장 |
| `tx_lookup` | **`RocksDBProviderFactory`, `EitherWriter`**, `TransactionsProviderExt` | tx 해시 인덱스 |
| `prune` | `PruneCheckpointReader/Writer`, `ChainStateBlockReader`, `RocksDBProviderFactory` | 프루닝 체크포인트 |

### `tx_lookup`만 RocksDB를 요구하는 이유

v2에서 **tx 해시 조회가 MDBX에서 RocksDB로 옮겨갔다.** `tx_lookup`은 그 인덱스를 만드는
stage이므로 두 저장소 중 어디에 쓸지 골라야 한다.

`crates/stages/stages/src/stages/tx_lookup.rs:165`

```rust
EitherWriter::new_transaction_hash_numbers(provider, rocksdb_batch)?;
```

함수 이름에 대상 테이블이 그대로 박혀 있다. `EitherWriter`가 storage 버전에 따라 MDBX/RocksDB로
라우팅한다 (8주차 목요일에 본 그 라우팅).

월요일 §8-5의 "`block()` 경로에 RocksDB가 없다"는 관찰과 짝이 맞는다 — 블록 조회는 번호 기반이라
RocksDB를 안 타고, **해시 기반 조회만** 탄다.

---

## 3. rpc는 읽기만 쓴다

```bash
grep -rhno "reth_storage_api::{\?[A-Za-z_, :]*" crates/rpc/ \
  | sed 's/.*reth_storage_api:://; s/[{}]//g' | tr ',' '\n' | sort | uniq -c | sort -rn
```

```
   8 BlockReaderIdExt
   7 BlockReader
   6 ProviderTx
   4 StateProviderFactory
   3 StateProvider
   3 BalProvider
   2 HeaderProvider
   2 BlockNumReader
   2 BlockIdReader
```

**전부 읽기다.** `BlockWriter`도 `StaticFileWriter`도 없다.
그래서 rpc는 저장소가 MDBX인지 SF인지 RocksDB인지 몰라도 되고, `reth-provider`가 필요 없다.

### 8주차 §3의 검증

8주차 금요일에 "`storage-api`와 `provider`를 왜 또 나눴나"를 추론하면서 근거로 `rpc-provider`를
짚었다. 오늘 Cargo.toml이 그것을 직접 확인해준다 — `rpc-eth-api`는 `reth-provider`를 의존 목록에
아예 올리지 않는다. **분리가 실제로 작동하고 있다.**

---

## 4. 오늘의 수확

1. **소비자는 두 부류다.** 저장소를 직접 만지는 쪽(`stages`)과 트레이트만 아는 쪽(`rpc`, `network`).
   그 차이가 `[dependencies]` vs `[dev-dependencies]`로 드러난다.
2. **크레이트 경계 기준은 읽기/쓰기가 아니라 "특정 저장소를 지목하느냐"다.**
   `BlockWriter`는 storage-api, `StaticFileWriter`는 provider.
3. **`tx_lookup`만 RocksDB를 요구한다.** v2에서 tx 해시 인덱스가 그리로 갔기 때문.
4. **의존 관계는 문서보다 정확하다.** 설계 의도를 추측하기 전에 Cargo.toml부터 보면 된다.

## 5. 다음으로 넘길 질문

1. `net/network`가 provider를 어떻게 받는가? rpc처럼 제네릭 바운드인가, 다른 방식인가
2. `stages` 중 static file을 안 만지는 stage가 있나? 있다면 그 stage의 바운드는 얼마나 얇은가
3. `EitherWriter`는 storage 버전 분기를 감싼 것인데, v1 지원이 끝나면 사라질 타입인가
