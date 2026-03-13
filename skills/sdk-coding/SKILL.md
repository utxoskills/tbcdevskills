---
name: sdk-coding
description: Write and explain TBC SDK code using tbc-lib-js and tbc-contract. Use when generating transaction-building code, SDK usage examples, or mapping contract methods to implementation patterns.
---

# SDK Coding

Use this skill when the user asks for TBC code, SDK usage, or contract interaction examples.

## Libraries

```bash
npm i tbc-lib-js tbc-contract
```

## Main SDK surfaces

- `FT`
- `NFT`
- `poolNFT`
- `poolNFT2`
- `orderBook`
- `MultiSig`
- `HTLC`
- `stableCoin`
- `piggyBank`
- `API`

## Coding rules

1. Prefer canonical SDK methods over hand-built raw scripts unless the user explicitly wants low-level construction.
2. When writing FT code, preserve `preTX` and `prepreTxData` handling.
3. When writing Pool code, remember the Pool NFT is the state anchor and FTLP is separate from ordinary FT.
4. When writing NFT transfer logic, remember the Code + Hold + Tape triple.
5. When writing broadcast flows, preserve chain order for chained transactions.

## Common patterns

### FT

- Mint: `MintFT`
- Transfer: `transfer`
- Attach TBC during transfer: `transfer(..., tbc_amount?)`
- Batch transfer: `batchTransfer`
- Merge UTXOs: `mergeFT`

### NFT

- Create collection: `createCollection`
- Mint NFT: `createNFT`
- Transfer NFT: `transferNFT`
- NFT + TBC in one tx: `transferNFTWithTBC`

### Pool

- v1 and v2 share lifecycle concepts
- v2 adds lock and FTLP burn/unlock semantics
- do not confuse `mergeFTLP` with `mergeFTinPool`

### OrderBook

Three usage styles exist:

- privateKey
- online auto-fetch
- build + fillSigs

### StableCoin

- `createCoin`
- `mintCoin`
- transfer behavior still follows FT patterns

## UTXO-aware coding guidance

- `API.fetchUTXO()` may auto-merge TBC UTXOs
- `API.fetchFtUTXOs()` is limited by FT multi-input structure
- `buildUTXO()` and `addInputFromPrevTx()` are core chaining tools

## Must-read references

Before writing production code, consult:

- `docs/code-reference.md`
- `docs/api-reference.md`
- `tbc-dev/SKILL.md`

## Output expectations

When answering coding requests:

- choose the right contract class
- explain required inputs and prefetches
- show correct broadcast order
- include decimal handling where relevant
