---
name: trace-asset
description: Trace the real source of an asset through TBC transactions using asset-focused reasoning.
---

# Trace TBC Asset

When the user asks to trace where an asset came from:

1. Use the `utxo-design` skill first.
2. Identify the core asset in the transaction:
   - FT -> FT Code inputs
   - NFT -> NFT Code + Hold inputs
   - Pool remove LP -> LP token input
   - Pool swap TBC->FT -> exchanged TBC input
3. Ignore fee-only TBC inputs unless TBC is the core asset.
4. Follow only the asset-carrying lineage.
5. Explain where naive tracing would go wrong if relevant.

Also consult `tbc-dev/SKILL.md` for full edge-case guidance.
