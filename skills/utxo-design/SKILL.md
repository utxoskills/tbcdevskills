---
name: utxo-design
description: Apply TBC UTXO design principles such as atomic settlement, asset-focused tracing, concurrency control, fragmentation management, and chain construction. Use when designing applications or reasoning about transaction safety and asset flow.
---

# UTXO Design

Use this skill when the user asks about TBC application architecture, double-spend safety, transaction atomicity, tracing strategy, or UTXO operational design.

## Core model

UTXO is not a balance model. You own discrete spendable outputs.

That means:

- concurrency is natural across different UTXOs
- every spend is explicit
- state must be carried forward intentionally

## Atomicity

A TBC transaction is all-or-nothing.

Use this property to design:

- NFT sales
- FT-TBC swaps
- multi-asset settlement
- HTLC-based conditional transfer
- pool operations with simultaneous reserve update

## Asset-focused tracing

Do not trace by “last input”.
Trace by the input that carries the core asset:

- FT transfer -> trace FT Code inputs
- NFT transfer -> trace NFT Code + Hold inputs
- Pool swap TBC->FT -> user TBC is the exchanged core asset
- Remove LP -> LP token input is the user-side core asset

## Payment vs change

When a transaction includes both asset transfer and TBC outputs:

- large TBC to a different role/address is likely payment
- smaller TBC is usually change
- infer roles from the asset destination first, not from the input address alone

## Double-spend safety

Important chain properties:

- first-seen mempool behavior
- no RBF
- detector-based conflict handling
- block/UTXO updates are atomic

## 0-conf chains

TBC allows deep unconfirmed chains relative to BTC-like systems.

Design implications:

- chained workflows are practical
- `addInputFromPrevTx()` is a first-class construction pattern
- ordering still matters at broadcast time

## Concurrency and operational design

SDKs do not give you built-in UTXO reservation.

Application-level strategies:

- reservation locks
- shard UTXO pools
- single-writer queues for stateful contracts
- optimistic retry for low-contention flows

## Fragmentation management

Use merges intentionally:

- TBC: `API.mergeUTXO()`
- FT: `FT.mergeFT()`

Avoid building systems that continually create unusable dust-like operational fragments.

## State machine mindset

Pool, OrderBook, FT, and other contracts are best read as state transitions:

- spend old state
- create new state
- interpret outputs as the new truth

## Canonical reference

For the full detailed theory and chain internals, also consult:

- `tbc-dev/SKILL.md`
- `docs/code-reference.md`
