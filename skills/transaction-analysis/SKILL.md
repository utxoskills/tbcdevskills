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

## Pool transaction reading

Pool is a single-UTXO state machine.

Read:

- `vout[0]` = Pool NFT code/state carrier
- `vout[1]` = Pool NTape with reserves
- later FT outputs = FTAbyC / FTAbyA / FTLP depending on operation

Key interpretations:

- Pool Create: Pool NFT + NTape only
- Init: Pool NFT + NTape + FTAbyC + FTLP, reserves go from zero to nonzero
- Add LP: same structure as Init, but reserves were already nonzero
- Remove LP: Pool NFT + NTape + FT to user + TBC to user + FTLP burn
- Swap TBC->FT: Pool satoshis increase, user receives FT, no FTLP
- Swap FT->TBC: Pool satoshis decrease, user receives TBC, no FTLP
- Merge FT in Pool: Pool NFT + NTape + FTAbyC, no reserve semantic change

## FTLP independent operations

Two important cases:

- `Merge FTLP`: FTLP code goes to user-controlled address
- `Burn FTLP`: FTLP code goes to `1BitcoinEaterAddressDontSendf59kuE`

Burn check:

- read the ASM chunk right before `02436f6465`
- if hash160 is `759d6677091e973b9e9d99f19c68fbf43e3f05f9`, classify as burn

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
