---
name: transaction-analysis
description: Analyze TBC transactions by reading script markers, output structure, asset flow, and atomic relationships. Use when classifying txids, explaining what a transaction does, or distinguishing similar contract operations.
---

# Transaction Analysis

Use this skill when the user provides a `txid` or asks what a TBC transaction is doing.

## Core mindset

Do not pattern-match mechanically. Read the transaction in this order:

1. Identify what assets appear in outputs.
2. Identify which contract family is present from script markers and neighboring outputs.
3. Identify who receives the core asset.
4. Distinguish payment vs change.
5. Infer the business intent.

## Fetching data

Use:

```bash
curl -s https://api.turingbitchain.io/api/tbc/decode/txid/{txid}
```

FT-related transactions may also need:

```bash
curl -s https://api.turingbitchain.io/api/tbc/ft/decode/txid/{txid}
```

## Contract DNA

Read script endings and output neighbors together:

- `32436f6465` + next output is `NTape` -> Pool NFT v2
- `31436f6465` + next output is `NTape` -> Pool NFT v1
- `32436f6465` + next output is `FTape` -> normal FT
- `02436f6465` -> FTLP
- `NHold` / `NTape` without Pool NFT context -> NFT
- fixed nonstandard script length `946B` / `1010B` -> OrderBook
- all outputs standard `pubkeyhash` -> pure TBC transfer

## Must-know distinctions

### Pool vs FT

Both Pool NFT v2 and normal FT code can end with `2Code`.

The difference is structural:

- Pool NFT: `vout[0] = 2Code/1Code`, `vout[1] = NTape`
- FT: `vout[0] = 2Code`, `vout[1] = FTape`

### FTLP vs FT

FTLP is not normal FT.

- FTLP code ends with `02436f6465`
- FTLP tape ends with `4654617065`
- independent FTLP operations have no Pool NFT output

### NFT identification

Do not rely on NFT code endings.

- New NFT code ends with `ac6a`
- Old v0 NFT code ends with `33436f6465`
- In practice, identify NFTs by `NHold` and `NTape`

## UTXO type overview

All TBC outputs fall into four script families:

- `pubkeyhash` (P2PKH): standard address outputs — payments, change, fee
- M-of-N multisig (`nonstandard` with `OP_CHECKMULTISIG`): TBC-native multisig, not Bitcoin P2SH
- Contract scripts (`nonstandard`): FT Code, NFT Code, Pool NFT Code, FTLP Code, OrderBook scripts, HTLC, PiggyBank
- Eater address: `1BitcoinEaterAddressDontSendf59kuE` — burn destination for FTLP

## FT vin/vout structures

FT Mint:

```
vin:  [0] P2PKH (source)
vout: [0] FT Code (500 sat)
      [1] FT Tape (0 sat, nulldata)
      [?] P2PKH change
```

No FT input. Two chained transactions (source + mint).

FT Transfer:

```
vin:  [0..N-1] FT Code inputs
      [N]      P2PKH (fee)
vout: [0] FT Code (500 sat, to receiver)
      [1] FT Tape (0 sat)
      [2] FT Code (500 sat, change, optional)
      [3] FT Tape (0 sat, optional)
      [?] P2PKH change
```

FT Burn: same as Transfer but Code goes to EaterAddress.

FT Merge: multiple FT inputs -> single FT output.

## NFT vin/vout structures

Collection Mint:

```
vin:  [0] P2PKH
vout: [0] OP_RETURN (collection metadata, nulldata)
      [1..N] P2PKH 100 sat each (mint slot UTXOs, one per NFT)
      [N+1] P2PKH change
```

NFT Mint:

```
vin:  [0] mint slot UTXO (100 sat, from collection)
      [1] P2PKH (fee)
vout: [0] NFT Code (200 sat, nonstandard)
      [1] NFT Hold (100 sat, pubkeyhash, points to minter)
      [2] NFT Tape (0 sat, nulldata, NTape with JSON metadata)
      [3] P2PKH change
```

NFT Transfer:

```
vin:  [0] NFT Code
      [1] NFT Hold
      [2..] P2PKH (fee)
vout: [0] NFT Code (200 sat, to new owner)
      [1] NFT Hold (100 sat, P2PKH to new owner)
      [2] NFT Tape (0 sat, references original mint txid)
      [3] P2PKH change
```

## Pool transaction reading

Pool is a single-UTXO state machine. The complete pool ecosystem has 6 UTXOs: poolNFT.Code + poolNFT.Tape + FTAbyPool.Code + FTAbyPool.Tape + FTLP.Code + FTLP.Tape. MINT creates only the first 2; INIT creates the full 6.

### FTAbyPool vs FTAbyAddress

Both end with `2Code` (`32436f6465`) but differ by a prefix byte before the `2Code` marker:

- `00` prefix + `2Code` = FTAbyAddress (user-controlled)
- `01` prefix + `2Code` = FTAbyPool (pool-controlled, address = poolNFT.Code.hash160)

### NTape byte structure

```
OP_FALSE OP_RETURN
  [ft_lp_partialhash]   32B
  [ft_a_partialhash]    32B
  [ft_lp_amount]         8B  uint64 LE
  [ft_a_amount]          8B  uint64 LE
  [tbc_amount]           8B  uint64 LE (satoshis)
  [ft_a_contractTxid]   32B
  [serviceFeeRate]       var
  [flags]                6B  (byte 3 = 0x00 unlocked, 0x01 locked)
  "NTape"                5B
```

### Pool operations vin/vout

Pool Create (MINT):

```
vin:  [0] P2PKH (fee)
vout: [0] poolNFT.Code (1000 sat, nonstandard)
      [1] poolNFT.Tape (0 sat, NTape, all amounts zero)
      [2] P2PKH change
```

Init / Add LP:

```
vin:  [0] poolNFT.Code
      [1] FTAbyAddress.Code (user FT)
      [2] P2PKH (user TBC + fee)
vout: [0] poolNFT.Code (updated)
      [1] poolNFT.Tape (updated NTape)
      [2] FTAbyPool.Code
      [3] FTAbyPool.Tape
      [4] FTLP.Code (new LP tokens)
      [5] FTLP.Tape
      [?] FT change, P2PKH change
```

Init vs Add LP: Init has NTape amounts going from zero to nonzero; Add LP has them already nonzero.

Remove LP (Consume):

```
vin:  [0] poolNFT.Code
      [1] FTLP.Code (LP to consume)
      [2] FTAbyPool.Code
      [3] P2PKH (fee)
vout: [0] poolNFT.Code
      [1] poolNFT.Tape
      [2] FTAbyAddress.Code (FT returned to user)
      [3] FTAbyAddress.Tape
      [4] P2PKH (TBC returned to user)
      [5] FTLPBurn.Code (to EaterAddress)
      [6] FTLPBurn.Tape
      [?] FTLP change, FTAbyPool change, P2PKH change
```

Swap TBC->FT:

```
vin:  [0] poolNFT.Code
      [1] P2PKH (user TBC)
      [2] FTAbyPool.Code (pass-through)
vout: [0] poolNFT.Code (sat increases)
      [1] poolNFT.Tape
      [2] FTAbyAddress.Code (FT to user)
      [3] FTAbyAddress.Tape
      [4] P2PKH (service fee)
      [?] FTAbyPool change, P2PKH change
```

Swap FT->TBC:

```
vin:  [0] poolNFT.Code
      [1] FTAbyAddress.Code (user FT)
      [2] P2PKH (fee)
vout: [0] poolNFT.Code (sat decreases)
      [1] poolNFT.Tape
      [2] P2PKH (TBC to user)
      [3] FTAbyPool.Code (FT into pool)
      [4] FTAbyPool.Tape
      [5] P2PKH (service fee)
      [?] FT change, P2PKH change
```

Merge FT in Pool:

```
vin:  [0] poolNFT.Code
      [1..N] FTAbyPool.Code (multiple pool FT UTXOs)
      [N+1] P2PKH (fee)
vout: [0] poolNFT.Code
      [1] poolNFT.Tape (amounts unchanged)
      [2] FTAbyPool.Code (merged)
      [3] FTAbyPool.Tape
      [?] P2PKH change
```

### NTape state changes

| Operation | ft_lp | ft_a | tbc_amount |
|-----------|-------|------|------------|
| Create (MINT) | 0 | 0 | 0 |
| Init / Add LP | increases | increases | increases |
| Remove LP | decreases | decreases | decreases |
| Swap TBC->FT | unchanged | decreases | increases |
| Swap FT->TBC | unchanged | increases | decreases |
| Merge FT in Pool | unchanged | unchanged | unchanged |
| FTLP Merge | N/A | N/A | N/A |
| LP Burn | N/A | N/A | N/A |

### Locked Pool differences

Locked Pool has the same 9 operations but with access control:
- Swap operations restricted to a designated public key (embedded in Pool NFT Code script)
- LP INCREASE adds an extra fee output (fixed TBC amount to hardcoded fee address)
- NTape flag byte 3 = `0x01` instead of `0x00`

## FTLP independent operations

Two important cases:

- `Merge FTLP`: FTLP code goes to user-controlled address
- `Burn FTLP`: FTLP code goes to `1BitcoinEaterAddressDontSendf59kuE`

Burn check:

- read the ASM chunk right before `02436f6465`
- if hash160 is `759d6677091e973b9e9d99f19c68fbf43e3f05f9`, classify as burn

## MultiSig vin/vout structures

Create Wallet:

```
vin:  [0] P2PKH (creator TBC)
vout: [0] nonstandard (multisig lock, OP_CHECKMULTISIG)
      [1] OP_RETURN (MTape, member pubkeys and config)
      [2] P2PKH change
```

P2PKH -> MultiSig (TBC): standard P2PKH spend, output to multisig nonstandard script.

P2PKH -> MultiSig (FT): FT Code inputs with P2PKH unlock, FT Code output controlled by multisig address.

MultiSig -> P2PKH (TBC/FT): multisig scriptSig contains M signatures + redeem script, output to P2PKH.

MultiSig -> MultiSig (TBC/FT): multisig spend to another multisig address.

Seven sub-types exist: CreateWallet, P2pkhToMsigTBC, P2pkhToMsigFT, MsigToP2pkhTBC, MsigToP2pkhFT, MsigToMsigTBC, MsigToMsigFT.

## OrderBook vin/vout structures

OrderBook uses fixed-length nonstandard scripts (not `2Code`/`1Code`/`02436f6465`). OrderData (114 bytes) is embedded at the script end: holdAddress(20B) + saleVolume(8B) + ftPartialHash(32B) + feeRate(8B) + unitPrice(8B) + ftContractId(32B).

Make Sell:

```
vin:  [0] P2PKH (user TBC)
vout: [0] sell script (946B nonstandard, value = saleVolume TBC)
      [1] P2PKH change
```

Make Buy:

```
vin:  [0] FTAbyAddress.Code (user FT)
      [1] P2PKH (fee)
vout: [0] buy script (1010B nonstandard, 300 sat dust)
      [1] FT Code (locked FT, 500 sat)
      [2] FT Tape (0 sat)
      [3] FT Code (FT change, optional)
      [4] FT Tape (optional)
      [5] P2PKH change
```

Cancel Sell: `vin[0]` = sell UTXO (scriptSig ends with `52`/OP_2 for cancel path), outputs P2PKH returning TBC.

Cancel Buy: `vin[0]` = buy UTXO, `vin[1]` = associated FT Code, outputs FTAbyAddress returning FT.

Match:

```
vin:  [0] buy UTXO (300 sat)
      [1] FT Code (buy-associated FT)
      [2] sell UTXO
      [3] P2PKH (matchmaker fee)
vout: [0] FT Code (FT to seller)
      [1] FT Tape
      [2] FT Code (FT fee)
      [3] FT Tape
      [4] P2PKH (TBC to buyer, saleVolume - tbcTax)
      [5] P2PKH (TBC fee)
      [6] P2PKH (matchmaker change)
      [?] new order UTXO (partial fill)
```

Buyer, seller, and matchmaker must be three different addresses.

## HTLC vin/vout structures

Identified by `OP_IF OP_SHA256` (`63a820` in hex). Uses OP_SHA256 (not OP_HASH256). Two spend paths: Withdraw (OP_TRUE at scriptSig end) and Refund (OP_FALSE).

Deploy:

```
vin:  [0] P2PKH (sender TBC)
vout: [0] HTLC script (nonstandard, locked TBC)
      [1] P2PKH change
```

Withdraw:

```
vin:  [0] HTLC UTXO (scriptSig = sig + pubkey + preimage + OP_TRUE)
vout: [0] P2PKH (TBC to receiver)
```

Refund:

```
vin:  [0] HTLC UTXO (scriptSig = sig + pubkey + OP_FALSE, nSequence < 0xFFFFFFFF)
      tx-level: locktime >= script's timelock
vout: [0] P2PKH (TBC back to sender)
```

## StableCoin vin/vout structures

StableCoin extends FT. On-chain FT representation is identical (Code+Tape with FTape marker). Difference is the coinNFT certificate mechanism.

CreateCoin (two chained transactions):

TX1 — create coinNFT certificate:

```
vin:  [0] P2PKH (admin)
vout: [0] coinNFT.Code (200 sat, ends with 33436f6465)
      [1] coinNFT.Hold (100 sat, "For Coin {name} NFT NHold")
      [2] coinNFT.Tape (0 sat, NTape, totalSupply=0)
      [3] P2PKH change
```

TX2 — first FT mint:

```
vin:  [0] coinNFT.Code (from TX1 vout[0])
      [1] coinNFT.Hold (from TX1 vout[1])
      [2] P2PKH (from TX1 vout[3])
vout: [0] coinNFT.Code (updated, 200 sat)
      [1] coinNFT.Hold (100 sat)
      [2] coinNFT.Tape (0 sat, totalSupply updated)
      [3] FT Code (500 sat, minted stablecoin)
      [4] FT Tape (0 sat)
      [5] OP_RETURN (optional mint message)
      [6] P2PKH change
```

MintCoin: consumes current coinNFT -> outputs updated coinNFT + new FT Code+Tape.

Transfer / MergeCoin: identical to standard FT operations.

FrozenCoinUTXO: sets lockTime in FT Tape. Frozen UTXO requires tx nLockTime >= frozen lockTime to spend.

## PiggyBank vin/vout structures

Identified by `OP_CHECKSIGVERIFY OP_6 OP_PUSH_META` (`ad56ba` in hex). Time-lock freeze for native TBC.

Freeze:

```
vin:  [0..N] P2PKH (user TBC, supports batch)
vout: [0] PiggyBank script (nonstandard, locked TBC)
      [1] P2PKH change
```

Unfreeze:

```
vin:  [0] PiggyBank UTXO (nSequence < 0xFFFFFFFF)
      [?] additional expired PiggyBank UTXOs
      tx-level: locktime >= script's lockTime
vout: [0] P2PKH (TBC to user)
      [1] P2PKH change (if multiple inputs)
```

## Atomic and mixed transactions

If multiple asset families appear in one transaction, or asset flows diverge across addresses, switch to atomic analysis:

1. Group outputs into NFT / FT / TBC.
2. Identify who received the core asset.
3. Treat large TBC to a different address as payment.
4. Treat smaller TBC as change.
5. Infer the exchange relation.

Common outcomes:

- NFT to A + large TBC to B -> NFT sale
- FT to A + large TBC to B -> FT-TBC OTC swap
- NFT to A + FT to B -> NFT-FT swap

## Output requirements

Always return:

- transaction type
- key addresses and roles
- token/contract metadata when relevant
- amounts with decimal interpretation
- atomic relation when relevant

## Canonical reference

For the complete canonical knowledge base and edge cases, also consult:

- `tbc-dev/SKILL.md`
- `docs/api-reference.md`
- `docs/code-reference.md`
