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

- Code + Tape pair
- code satoshis = `500`
- tape satoshis = `0`
- normal FT code ends with `2Code`
- merge, transfer, mint, batch transfer all reuse this pattern

## NFT

- Code + Hold + Tape triple
- code satoshis = `200`
- hold satoshis = `100`
- tape satoshis = `0`
- identify by `NHold` and `NTape`, not by code ending alone

## Pool

- `vout[0]` = Pool NFT code
- `vout[1]` = Pool NTape state
- later outputs represent pool-held FT, user-facing FT, or FTLP

Read Pool operations by state transition, not just count of outputs.

## FTLP

FTLP is a specialized FT family for LP ownership.

- code ends with `02436f6465`
- tape ends with `FTape`
- can appear inside Pool operations or independently

Independent cases:

- merge FTLP
- burn FTLP

## OrderBook

OrderBook is special because it is recognized more by fixed script geometry than by human-readable markers.

- sell script: `946B`
- buy script: `1010B`
- ends with order data structure
- buy order dust = `300 sat`

## MultiSig

- main locking script uses `OP_CHECKMULTISIG`
- hold outputs contain `multisig`
- tape contains `MTape`

## HTLC

- hashlock + timelock contract
- deployment output contains `OP_IF OP_SHA256`
- withdraw path reveals preimage
- refund path uses timelock satisfaction

## StableCoin

StableCoin extends FT.

- chain-level FT representation is still FT-like
- special behavior comes from coinNFT certificate + admin mint flow

## PiggyBank

- TBC time-lock freeze contract
- look for `OP_CHECKSIGVERIFY OP_6 OP_PUSH_META`

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
