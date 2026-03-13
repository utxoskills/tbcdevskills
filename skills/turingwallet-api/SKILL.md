---
name: turingwallet-api
description: Use the TuringWallet browser wallet API for connection, signing, encryption, transaction sending, associated transaction signing, and cross-chain support. Use when building browser dapps or integrating wallet UX.
---

# TuringWallet API

Use this skill when the user asks about browser wallet integration, wallet-side signing, or DApp transaction flows.

## Main surface

`window.Turing`

## Core capabilities

- connect / disconnect
- get address and pubkey
- get wallet info
- sign messages
- encrypt / decrypt
- send transaction
- sign transaction
- sign associated transaction
- batch request
- EVM support
- BTC support

## TBC transaction sending

Key `flag` families include:

- `P2PKH`
- `FT_MINT`
- `FT_TRANSFER`
- `FT_MERGE`
- `COLLECTION_CREATE`
- `NFT_CREATE`
- `NFT_TRANSFER`
- `POOLNFT_MINT`
- `POOLNFT_INIT`
- `POOLNFT_LP_INCREASE`
- `POOLNFT_LP_CONSUME`
- `POOLNFT_LP_BURN`
- `POOLNFT_SWAP_TO_TOKEN`
- `POOLNFT_SWAP_TO_TBC`
- `POOLNFT_MERGE`
- `FTLP_MERGE`

## When to prefer each method

- `sendTransaction`: wallet handles UTXO selection, signing, and broadcast
- `signTransaction`: your app builds raw txs and only delegates signatures
- `signAssociatedTransaction`: your app builds parent/child chains and lets the wallet sign the linked set

## Important integration reminders

- preserve the correct contract flag
- pass chain-specific domain only when needed
- understand whether the wallet returns `txid` or `txraw`
- keep contract-level parameters precise for Pool and FT flows

## Canonical reference

For full method signatures and parameter tables, also consult:

- `tbc-dev/SKILL.md`
- `docs/code-reference.md`
