# 로그를 찍어 확인한 것 — 읽어서 세운 경로가 틀렸다

> 9주차 목요일. 처음으로 코드를 수정해서 실행 중인 노드를 관찰했다.
> 가장 큰 수확은 **월요일에 추적한 함수가 실제 RPC 경로가 아니었다**는 것.

---

## 1. 실험 설계

로그를 4곳에 넣었다. 두 쌍으로 놓은 이유는 **인메모리에서 걸렸는지 DB까지 내려갔는지**를
구분하려면 한 군데만으로는 안 되기 때문이다.

| 위치 | 파일:줄 |
|---|---|
| `ConsistentProvider::block` | `consistent.rs:703` |
| `DatabaseProvider::block` | `database/provider.rs:1846` |
| `ConsistentProvider::sealed_block_with_senders` | `consistent.rs:745` |
| `DatabaseProvider::sealed_block_with_senders` | `database/provider.rs:1916` |

```rust
tracing::debug!(target: "study", ?id, "DatabaseProvider::block");
```

`target: "study"`를 붙이면 `--log.stdout.filter=study=debug` 한 줄로 내 로그만 뽑을 수 있다.
`?id`는 `Debug`로 찍으라는 뜻이고 `Number(50)` / `Hash(0x…)`가 그대로 나와 어느 쪽으로
들어왔는지도 보인다.

### 실행

```bash
./target/release/reth node --dev --dev.block-time 1s --http \
  --datadir /tmp/reth-dev-study -q --log.stdout.filter=study=debug
```

`--dev`는 1초마다 블록을 직접 찍어낸다. sepolia를 처음부터 동기화하면 오늘 안에 안 끝나므로
관찰 대상을 몇십 초 만에 만들 수 있는 dev 모드를 썼다.

> stdout에서 로그가 안 보이면 dev 모드의 INFO 로그(1초마다 블록 생성)에 묻힌 것이다.
> `-q`로 기본 레벨을 끄거나, 파일을 따라보면 된다:
> `tail -f ~/.cache/reth/logs/dev/reth.log | grep --line-buffered "DEBUG study:"`
> (`grep study`만 하면 datadir 경로 `/tmp/reth-dev-study`까지 걸린다)

---

## 2. 예상 vs 실제

| # | 예상 | 실제 | |
|---|---|---|---|
| 1 | `block()` 로그는 안 찍힌다 | 안 찍힘 | ⭕ |
| 2 | 오래된 블록은 2줄(DB), 최신 블록은 1줄(인메모리) | 오래된 블록 2줄, **최신 블록 0줄** | ❌ |
| 3 | 같은 블록 두 번째 조회는 캐시 히트로 0줄 | 0줄 | ⭕ |
| 4 | 인메모리/DB 경계는 `--engine.persistence-threshold`(기본 7) 근처 | **경계는 캐시였고, 7은 관찰되지 않음** | ❌ |

**2번과 4번이 틀렸다.** RPC 경로에는 provider 앞에 캐시가 한 겹 더 있어서, 최신 블록은
provider까지 아예 내려가지 않는다.

---

## 3. ★ `eth_getBlockByNumber`는 `block()`을 부르지 않는다

로그 4개 중 `block()` 두 개는 **한 번도 찍히지 않았다.** 코드를 다시 따라가니 실제 경로는
이렇다.

```
eth_getBlockByNumber                         rpc-eth-api/src/core.rs:520
  → rpc_block()                              rpc-eth-api/src/helpers/block.rs:63
  → recovered_block()                        rpc-eth-api/src/helpers/block.rs:256
      → block_hash_for_id()                  (번호 → 해시)
      → cache().get_recovered_block(hash)    rpc-eth-types/src/cache/mod.rs:151
          ↓ 캐시 미스일 때만
        provider.sealed_block_with_senders(Hash(h), WithHash)
                                             rpc-eth-types/src/cache/mod.rs:536
```

`DatabaseProvider::block()`은 이 사슬 어디에도 없다.

### `block()`을 실제로 부르는 곳

| 호출자 | 위치 | 상황 |
|---|---|---|
| **Engine API** | `rpc-engine-api/src/engine_api.rs:657, 785` | `engine_getPayloadBodiesByRange/ByHash` |
| **P2P** | `net/network/src/eth_requests.rs:218` | 다른 노드의 `GetBlockBodies` 요청 응답 |

둘 다 하는 일이 같다 — **원본 블록을 그대로 넘겨주는 것**이다.

```rust
// eth_requests.rs:218 — P2P
if let Some(block) = self.client.block_by_hash(hash).unwrap_or_default() {
    let body = block.into_body();      // 바디를 꺼내 상대에게 전달
```

반면 `eth_getBlockByNumber` 응답에는 각 트랜잭션의 `from` 필드가 필요한데, 그것은 블록에
저장돼 있지 않고 발신자를 따로 채워야 나온다. 그래서 RPC는 `sealed_block_with_senders`
(= `RecoveredBlock`)를 쓴다.

> **`block()` = 헤더 + 바디, `sealed_block_with_senders()` = 거기에 발신자까지.**
> 필요한 것이 달라서 함수가 갈렸다.

---

## 4. 캐시가 1차 필터다

블록 오프셋별로 curl하면서 로그 증가분을 셌다. tip = 399.

| 블록 | 로그 | 경로 |
|---|---|---|
| 1 | 0줄 | 앞서 조회해서 캐시에 들어감 |
| 50, 150, 240, 250, 252 | **2줄** | Consistent → Database = **DB 경로** |
| 253, 254, 260, 280, 299~399 | 0줄 | 캐시 히트 |

경계가 **252/253**에서 딱 갈린다.

### 경계의 정체 — LRU 크기가 아니라 프로세스 재시작

`--rpc-cache.max-blocks` 기본값은 **5000**이라 블록 399개는 축출될 일이 없다.
로그에 원인이 있었다.

```
08:13:49.928301Z DEBUG payload_builder: building new payload
    parent_number=252
```

노드가 17:13에 재시작하면서 **253번부터 만들기 시작했다.** 캐시는 노드가 블록을 만들 때
채워지므로(`cache/mod.rs:379` `on_new_block`), **이 프로세스가 만든 253번 이상만 캐시에 있다.**
0~252번은 이전 프로세스가 만든 것이라 캐시에 없어서 provider까지 내려간다.

### 캐시 히트 재확인

블록 50을 다시 조회 → **0줄.** 첫 조회 때 캐시에 들어갔기 때문.
같은 블록도 **첫 조회만** provider를 부른다.

---

## 5. 인메모리 경로는 RPC로 관찰되지 않는다

월요일 §4에서 본 `get_in_memory_or_storage_by_block`의 인메모리 분기
(= `ConsistentProvider` 로그만 찍히고 `DatabaseProvider`는 안 찍히는 경우)는
**이번 실험에서 한 번도 나오지 않았다.**

이유는 구조적이다.

```
인메모리에 있는 블록   =  tip에서 persistence-threshold(기본 7) 안쪽
캐시에 있는 블록       =  이 프로세스가 만든 모든 블록 (최대 5000개)
```

**인메모리 구간이 캐시 구간에 완전히 포함된다.** 그래서 RPC 요청은 인메모리 블록에 도달하기
전에 항상 캐시에서 끝난다. 인메모리 분기를 보려면 캐시를 우회하는 호출자
(Engine API, P2P)를 관찰해야 한다.

---

## 6. 오늘의 수확

1. **★ 코드를 읽어서 세운 경로가 틀릴 수 있다.** `block()`이 블록 조회의 주 경로인 줄 알았는데
   RPC는 그 함수를 안 쓴다. 로그 한 줄이 그것을 즉시 드러냈다.
2. **RPC 앞에는 캐시가 한 겹 더 있다.** provider 호출 여부를 결정하는 1차 필터.
   provider 코드만 읽어서는 안 보인다.
3. **함수가 갈린 기준은 "발신자가 필요하냐"다.** P2P/Engine은 원본 바디만 넘기면 되니 `block()`,
   RPC는 `from` 필드가 필요하니 `sealed_block_with_senders`.
4. **관찰 결과가 예상과 다르면 그 자체가 결과다.** 예상 4개 중 2개가 틀렸고, 틀린 쪽이
   더 많은 것을 알려줬다.
5. **로그에 `target:`을 붙여라.** dev 모드는 초당 수십 줄을 쏟아내므로 필터가 없으면 못 읽는다.

## 7. 실무 메모

- **IDE format-on-save 주의.** stable rustfmt로 저장하면 파일 전체(113줄)가 재포맷된다.
  reth는 nightly + `rustfmt.toml`(`trailing_semicolon = false` 등)을 쓴다.
  이 저장소에서는 format-on-save를 끄거나 nightly로 지정할 것.
- **전체 빌드 전에 `cargo check -p reth-provider`** 로 오타를 먼저 거를 것.
- **백트레이스는 `--release`로 못 쓴다.** `[profile.release]`가 `debug = "none"`,
  `strip = "symbols"`다. 호출자를 런타임에 확인하려면 `--profile profiling`
  (release + 심볼 유지)을 써야 한다.

## 8. 내일(금) 실험으로 넘김

`--engine.persistence-threshold`는 RPC에서 관찰되지 않으므로 파라미터 실험 대상으로 부적합하다.
대신 **`--rpc-cache.max-blocks`** 를 바꾸면 오늘 관찰한 경계가 직접 움직인다.

```bash
--rpc-cache.max-blocks 10    # 최근 10블록만 캐시 → 그 밖은 전부 provider 호출
```

예상: 캐시 히트 구간이 tip-10 이내로 줄어들고, 그 바깥은 전부 2줄(DB 경로)이 찍힌다.
