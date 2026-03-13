# Expected Classification Notes

Use these fixtures as a regression checklist whenever transaction-analysis guidance changes.

## Required invariants

- `ec87c044...` must classify as `Burn FTLP`, not FT Merge and not FT Transfer.
- `c12c67eb...` must classify as `Add LP`, not OrderBook.
- `f79e71a5...` must classify as an NFT atomic sale, not NFT mint and not plain NFT transfer.
- `d669a42f...` must remain an atomic NFT market-style settlement example.

## What this protects against

- confusing FTLP with ordinary FT
- confusing Pool outputs with OrderBook or FT outputs
- ignoring atomic settlement and treating payment as change
- overfitting to a rigid output template instead of asset-flow reasoning
