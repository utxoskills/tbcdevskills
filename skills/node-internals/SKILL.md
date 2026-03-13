---
name: node-internals
description: Understand TBC consensus, mempool, P2P, validation, DAA, and node operational behavior. Use when explaining how the chain works or validating behavior against node source code.
---

# Node Internals

Use this skill when the user asks about TBC node behavior, consensus differences, mempool policy, P2P, validation, or deployment/ops.

## High-value topics

- TBC vs BSV consensus differences
- transaction version 10
- TBC txid hashing
- pre-fork UTXO freeze
- mempool rules and no-RBF behavior
- TTOR
- P2P handshake and PROTOCONF
- DAA
- KYC miner validation

## Deployment and operations

Know how to explain:

- node build flow
- RPC basics
- LevelDB storage model
- journaling block assembly
- mining candidate flow

## Consensus details

Important constants and ideas:

- `nVersion == 10`
- `COINBASE_MATURITY = 1`
- fixed dust threshold behavior
- `OP_PUSH_META`, `OP_PARTIAL_HASH`, `OP_CHECKDATASIG`
- post-Genesis rule changes

## Mempool and broadcast

Be able to explain:

- first-seen conflict behavior
- no RBF
- ancestor/descendant limits
- `CTimeLockedMempool`
- broadcast reliability caveats

## P2P

Be able to explain:

- handshake flow
- `PROTOCOL_VERSION = 90015`
- TBC-specific `PROTOCONF`
- connection eviction policy
- inventory and download parameters

## Difficulty and mining

Be able to explain:

- fork reset
- TBC DAA vs BCH/BSV DAA
- per-block adjustment limits
- emergency slowdown rule
- KYC V1 / V2 miner validation

## Canonical reference

For the full verified constants and source-backed details, also consult:

- `tbc-dev/SKILL.md`
