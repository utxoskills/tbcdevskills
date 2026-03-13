---
name: write-tbc-code
description: Write TBC SDK code with the correct contract class, UTXO flow, and broadcast sequence.
---

# Write TBC Code

When the user asks for TBC code:

1. Use the `sdk-coding` skill.
2. Pick the correct contract family before writing code.
3. Use the canonical SDK methods instead of inventing raw scripts unless low-level control is requested.
4. Preserve:
   - decimal handling
   - `preTX` / `prepreTxData` where needed
   - correct order of chained broadcasts
   - correct distinction between FT, FTLP, Pool state, and NFT Hold/Tape structures

Consult:

- `docs/code-reference.md`
- `docs/api-reference.md`
- `tbc-dev/SKILL.md`
