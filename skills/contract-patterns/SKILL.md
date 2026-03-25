---
name: contract-patterns
description: Understand TBC contract families, output structures, script markers, and how to distinguish similar-looking contracts. Use when comparing FT, NFT, Pool, OrderBook, HTLC, MultiSig, StableCoin, or PiggyBank behavior.
---

# Contract Patterns

Use this skill when the user asks how a TBC contract is structured or how to distinguish one contract family from another.

## Contract families

- FT
- NFT
- Pool v1 / v2
- FTLP
- OrderBook
- MultiSig
- HTLC
- StableCoin
- PiggyBank

## Shared model

Most TBC contracts follow a state-carrying UTXO model:

- immutable validation logic is embedded in the locking script
- mutable state is recreated in outputs
- contract operations consume old state and create new state

## Marker quick reference

- `32436f6465` -> Pool NFT v2 or normal FT code
- `31436f6465` -> Pool NFT v1
- `02436f6465` -> FTLP
- `FTape` -> FT / FTLP / StableCoin tape
- `NTape` -> NFT metadata or Pool state tape
- `NHold` -> NFT hold/right
- `MTape` -> MultiSig tape
- `multisig` -> MultiSig hold marker

## FT

Code + Tape pair. Every FT output is two consecutive vouts:

| UTXO | type | satoshis | marker |
|------|------|----------|--------|
| FT Code | nonstandard | 500 | ends with `32436f6465` ("2Code") |
| FT Tape | nulldata | 0 | ends with `4654617065` ("FTape") |

Sub-types:

| Operation | Example txid | Key feature |
|-----------|-------------|-------------|
| Mint | `566d9bcc...0848a0` | No FT input, two chained txs (source+mint) |
| Transfer | `58653f54...8bac68` | FT Code input -> new Code+Tape to receiver + optional change |
| Burn | `f187ecfe...4f95d` | Code goes to EaterAddress |
| Batch Transfer | — | Multiple Code+Tape pairs to different addresses |
| Merge | — | Multiple FT inputs -> single FT output |

## NFT

Code + Hold + Tape triple:

| UTXO | type | satoshis | marker |
|------|------|----------|--------|
| NFT Code | nonstandard | 200 | new: ends `ac6a`; old v0: ends `33436f6465` |
| NFT Hold | pubkeyhash | 100 | contains `NHold` (`4e486f6c64`) |
| NFT Tape | nulldata | 0 | ends with `NTape` (`4e54617065`), JSON metadata |

Identify NFTs by `NHold` and `NTape`, not by code ending alone.

Collection Mint outputs N slot UTXOs (100 sat each, P2PKH) that are consumed one-per-NFT during minting.

Sub-types:

| Operation | Example txid | Key feature |
|-----------|-------------|-------------|
| Collection Mint | `3e724f11...c9be1a3` | OP_RETURN metadata + N slot UTXOs |
| Mint | `86da8800...6933c4` | Consumes slot -> Code(200) + Hold(100) + Tape(0) |
| Transfer | `c524bdd9...56be79` | Code+Hold consumed, re-created with new Hold address |

## Pool

Complete pool ecosystem = 6 UTXOs:

| UTXO | marker | satoshis | role |
|------|--------|----------|------|
| poolNFT.Code | `2Code`/`1Code` | v1: 1000; v2: 1000+tbc_amount | Contract state anchor, holds TBC reserves (v2) |
| poolNFT.Tape | `NTape` | 0 | Pool state: ft_lp, ft_a, tbc_amount |
| FTAbyPool.Code | `01` prefix + `2Code` | 500 | FT held by pool (address = poolNFT.Code.hash160) |
| FTAbyPool.Tape | `FTape` | 0 | FT balance data |
| FTLP.Code | `02436f6465` ("\x02Code") | 500 | LP ownership token |
| FTLP.Tape | `FTape` | 0 | LP balance data |

MINT creates only poolNFT.Code + poolNFT.Tape (2 UTXOs). INIT creates all 6.

### FTAbyPool vs FTAbyAddress

Both end with `2Code` (`32436f6465`). Distinguished by a prefix byte before the 5-byte code marker:

- `00` + `05` + `32436f6465` = FTAbyAddress (user-controlled)
- `01` + `05` + `32436f6465` = FTAbyPool (pool-controlled)

### FTLP provenance verification

FTLP.Code embeds `poolNFT.Code.sha256` internally. When a parent Code's sha256 differs from the grandparent's, the contract requires the parent's sha256 to equal a fixed value bound to the poolNFT.

### Locked vs unlocked Pool

| Dimension | Unlocked | Locked |
|-----------|----------|--------|
| NTape flag byte 3 | `0x00` | `0x01` |
| Swap access | any address | designated pubkey only |
| LP INCREASE | no extra fee | extra fixed TBC output to hardcoded fee address |
| Code script | standard ending | appends `OP_EQUALVERIFY OP_CHECKSIGVERIFY` with embedded pubkey |

Sub-types (9 operations, same for locked/unlocked):

| Operation | Example txid (unlocked) | Example txid (locked) |
|-----------|------------------------|----------------------|
| MINT | `eb9b8cea...261f97` | `980f0014...9e1a77` |
| INIT | `67e65e79...a332e` | `8f9dfafd...d7aa4e` |
| LP INCREASE | `f5316b95...cbdc6` | `71082204...fd26ed` |
| LP CONSUME | `9ba1d3a0...d58429` | `bd84e0b0...58f99b` |
| SWAP TO TOKEN | `c8e74e57...2bd4ea` | `0ac1b812...ae9818` |
| SWAP TO TBC | `ccfbde95...d561d6` | `1568266e...bedb03` |
| mergeFTinPool | `92f56f1f...c96e19` | `12eb6f60...75f118` |
| FTLP MERGE | `11b578c9...c953c4` | `012579c6...4d9616` |
| LP BURN | `1020d17b...751f21` | `efb5a8de...e78fb3` |

Read Pool operations by NTape state transition, not output count.

## FTLP

FTLP is a specialized FT family for LP ownership.

- code ends with `02436f6465` ("\x02Code")
- tape ends with `FTape`
- can appear inside Pool operations or independently

Independent cases:

- merge FTLP: FTLP code goes to user-controlled address
- burn FTLP: FTLP code goes to EaterAddress (Pool NFT not involved, NTape unchanged)

## OrderBook

OrderBook uses fixed-length nonstandard scripts, not `2Code`/`1Code`/`02436f6465`.

| Feature | Sell | Buy |
|---------|------|-----|
| Script length | 946 bytes | 1010 bytes |
| Script value | saleVolume TBC | 300 sat (dust) |
| Locked asset | TBC (in script value) | FT (in associated Code+Tape) |
| Cancel path | scriptSig ends `52` (OP_2) | scriptSig ends `52` (OP_2) |

OrderData structure (114 bytes at script end):

- holdAddress: 20B (maker's hash160)
- saleVolume: 8B (uint64 LE)
- ftPartialHash: 32B
- feeRate: 8B (uint64 LE)
- unitPrice: 8B (uint64 LE)
- ftContractId: 32B

Sub-types:

| Operation | Example txid |
|-----------|-------------|
| Make Sell | `448817f4...591b89` |
| Make Buy | `2f340899...057a4c` |
| Cancel Sell | `ae1d70f8...a33e8f` |
| Cancel Buy | `79fe3561...33d979` |
| Match | `8084a278...333bad` |

Match requires three distinct addresses: buyer, seller, matchmaker.

## MultiSig

TBC-native M-of-N multisig (1-6 sigs, 3-10 keys). Not Bitcoin P2SH — uses custom Base58 address encoding.

- locking script uses `OP_CHECKMULTISIG`
- hold outputs contain `multisig` (`6d756c7469736967`)
- tape contains `MTape` (`4d54617065`)

Sub-types:

| Operation | Example txid |
|-----------|-------------|
| CreateWallet | `9b365639...6a7380` |
| P2pkhToMsigTBC | `aeb6d019...c6eb6b` |
| P2pkhToMsigFT | `8b78d329...d441a4` |
| MsigToP2pkhTBC | `594d1a52...f61a5d` |
| MsigToP2pkhFT | `e11d3139...3887a4` |
| MsigToMsigTBC | `d5010852...1fd165` |
| MsigToMsigFT | `266b72f2...d1b23f` |

## HTLC

Hashlock + timelock contract. Uses OP_SHA256 (not OP_HASH256).

- deployment: `OP_IF OP_SHA256 <hashlock> OP_EQUALVERIFY` (`63a820` in hex)
- withdraw path: scriptSig ends with `51` (OP_TRUE), reveals preimage
- refund path: scriptSig ends with `00` (OP_FALSE), requires nLockTime >= timelock and nSequence != 0xFFFFFFFF
- time verification uses `OP_PUSH_META` value 2 (reads nLockTime) and value 6 (reads nSequence)

Sub-types:

| Operation | Example txid |
|-----------|-------------|
| Deploy | `918beb1e...a7a5a1` |
| Withdraw | `1059d1fe...8c9a38` |
| Refund | `de431995...f3bdbc` |

## StableCoin

StableCoin extends FT with admin-controlled minting and a coinNFT certificate.

- on-chain FT representation identical to standard FT: Code(500sat) + Tape(0sat), `FTape` marker
- transfer and merge operations identical to FT
- differs in creation: `createCoin` produces two chained txs (coinNFT certificate + first FT mint)
- subsequent mints (`mintCoin`) consume coinNFT -> output updated coinNFT + new FT
- coinNFT uses NFT-like structure: Code(200sat, `3Code`) + Hold(100sat) + Tape(0sat, `NTape`)
- `frozenCoinUTXO`: writes lockTime into FT Tape, making UTXO unspendable until nLockTime >= frozen lockTime

Lifecycle: CreateCoinNFT -> CreateCoinMint -> MintCoin -> Transfer -> MergeCoin -> FrozenCoinUTXO

Sub-types:

| Operation | Example txid |
|-----------|-------------|
| CreateCoin | `86c59787...d106ba` + `13eccd66...36f238` |
| MintCoin | `7e3dad76...0087df` |
| Transfer | `5f6ba940...8135be` |
| MergeCoin | `2b783ab2...d493dd` |
| FrozenCoinUTXO | `3a0f0f4d...a128ed` |

## PiggyBank

TBC time-lock freeze contract for native TBC.

- identified by `OP_CHECKSIGVERIFY OP_6 OP_PUSH_META` (`ad56ba` in hex)
- Freeze: locks TBC to a block height, outputs nonstandard PiggyBank script
- Unfreeze: spends PiggyBank UTXO with nSequence < 0xFFFFFFFF and nLockTime >= script's lockTime
- script verifies: (1) P2PKH signature, (2) nSequence != 0xFFFFFFFF, (3) nLockTime >= embedded lockTime

Sub-types:

| Operation | Example txid |
|-----------|-------------|
| Freeze | `65a14814...86291a` |
| Unfreeze | `75a7b3b1...1546e5` |

## High-value distinctions

### Pool NFT vs FT

Do not classify from `2Code` alone.

- Pool NFT: `2Code/1Code` followed by `NTape`
- FT: `2Code` followed by `FTape`

### FTLP vs FT

- FTLP: `02436f6465`
- FT: `32436f6465`

### NFT vs Pool

Both may involve `NTape`, but:

- NFT has `NHold`
- Pool has `Pool NFT code + NTape state` in `vout[0]` and `vout[1]`

## Canonical reference

For full transaction semantics and exhaustive details, also consult:

- `tbc-dev/SKILL.md`
- `docs/code-reference.md`
