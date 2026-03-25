# Expected Classification Notes

Use these fixtures as a regression checklist whenever transaction-analysis guidance changes.

## Required invariants

### Original invariants

- `ec87c044...` must classify as `Burn FTLP`, not FT Merge and not FT Transfer.
- `c12c67eb...` must classify as `Add LP`, not OrderBook.
- `f79e71a5...` must classify as an NFT atomic sale, not NFT mint and not plain NFT transfer.
- `d669a42f...` must remain an atomic NFT market-style settlement example.

### FT invariants

- `566d9bcc...` must classify as `FT Mint`, not FT Transfer (no FT input).
- `58653f54...` must classify as `FT Transfer`, not Pool operation (vout[1] is FTape, not NTape).
- `f187ecfe...` must classify as `FT Burn`, not FT Transfer (Code goes to EaterAddress).

### NFT invariants

- `3e724f11...` must classify as `NFT Collection Mint`, not NFT Mint (no NFT Code output, has slot UTXOs).
- `86da8800...` must classify as `NFT Mint`, not NFT Transfer (consumes slot, creates new Code+Hold+Tape).
- `c524bdd9...` must classify as `NFT Transfer`, not NFT Mint (Code+Hold already exist as inputs).

### Pool invariants

- `eb9b8cea...` must classify as `Pool MINT` (create empty pool), not Pool INIT (NTape amounts all zero).
- `67e65e79...` must classify as `Pool INIT`, not Add LP (NTape goes from zero to nonzero).
- `c8e74e57...` must classify as `Pool Swap TBC->FT`, not Add LP (ft_lp unchanged, no FTLP output).
- `ccfbde95...` must classify as `Pool Swap FT->TBC`, not Swap TBC->FT (pool sat decreases).
- `9ba1d3a0...` must classify as `Pool Remove LP`, not Swap (FTLP burned to EaterAddress).
- `92f56f1f...` must classify as `Pool mergeFTinPool`, not Swap (NTape amounts unchanged).
- `11b578c9...` must classify as `FTLP Merge`, not FT Merge (code ends 02436f6465, not 32436f6465).
- `1020d17b...` must classify as `LP Burn`, not Pool Remove LP (no Pool NFT in inputs).

### OrderBook invariants

- `448817f4...` must classify as `OrderBook Make Sell`, not P2PKH transfer (vout[0] is 946B nonstandard).
- `2f340899...` must classify as `OrderBook Make Buy`, not FT Transfer (vout[0] is 1010B nonstandard with 300sat).
- `8084a278...` must classify as `OrderBook Match`, not FT Transfer (buy+sell UTXOs consumed together).

### HTLC invariants

- `918beb1e...` must classify as `HTLC Deploy`, not PiggyBank Freeze (script contains OP_IF OP_SHA256).
- `1059d1fe...` must classify as `HTLC Withdraw`, not HTLC Refund (scriptSig ends OP_TRUE, preimage present).
- `de431995...` must classify as `HTLC Refund`, not HTLC Withdraw (scriptSig ends OP_FALSE, uses locktime).

### Other contract invariants

- `9b365639...` must classify as `MultiSig CreateWallet`, not P2PKH transfer (has OP_CHECKMULTISIG + MTape).
- `65a14814...` must classify as `PiggyBank Freeze`, not HTLC Deploy (script has ad56ba, no OP_IF/OP_SHA256).
- `75a7b3b1...` must classify as `PiggyBank Unfreeze`, not standard P2PKH (input is PiggyBank nonstandard script).

## What this protects against

- confusing FTLP with ordinary FT
- confusing Pool outputs with OrderBook or FT outputs
- ignoring atomic settlement and treating payment as change
- overfitting to a rigid output template instead of asset-flow reasoning
- confusing Pool MINT (empty pool) with Pool INIT (first liquidity injection)
- confusing Pool Swap with Add/Remove LP (check ft_lp change and FTLP presence)
- confusing NFT Collection Mint with NFT Mint (slots vs Code+Hold+Tape)
- confusing HTLC with PiggyBank (both use OP_PUSH_META, but HTLC has OP_IF/OP_SHA256)
- confusing OrderBook with FT operations (fixed script length 946B/1010B vs Code marker)
