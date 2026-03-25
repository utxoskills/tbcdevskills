# TBC Dev Skills

An AI skill pack for TBC (TuringBitChain) blockchain developers. Enables any skill-aware AI Agent to quickly master full-stack TBC development knowledge.

## What is this

A structured set of TBC development knowledge that can be loaded by AI Agent platforms such as [OpenClaw](https://openclaw.ai), the Cursor plugin system, and others. Once installed, an Agent can:

- Analyze any TBC transaction and identify all 9 contract types (FT, NFT, Pool, OrderBook, MultiSig, HTLC, StableCoin, PiggyBank, etc.)
- Write TBC smart contract interaction code (`tbc-lib-js` + `tbc-contract` SDK)
- Understand TBC's UTXO model, consensus rules, P2P protocol, and mempool policies
- Use the TuringWallet browser wallet API to build DApps
- Apply UTXO application-layer patterns (double-spend protection, chained transactions, concurrency control, fragmentation management)
- Query and call TBC full-node APIs

## Structure

### Skills

| Path | Description |
|------|-------------|
| `skills/transaction-analysis/` | Transaction identification, atomic swaps, payment vs change, asset flow analysis, full-contract vin/vout specifications |
| `skills/contract-patterns/` | FT / NFT / Pool / FTLP / OrderBook / HTLC / MultiSig / StableCoin / PiggyBank structure differentiation, sub-type index |
| `skills/sdk-coding/` | `tbc-lib-js` + `tbc-contract` SDK coding and construction patterns |
| `skills/utxo-design/` | UTXO atomicity, concurrency control, provenance tracing, fragmentation management |
| `skills/node-internals/` | Consensus, mempool, P2P, DAA, KYC miner verification |
| `skills/turingwallet-api/` | TuringWallet browser wallet API |

### Companion entry points

| Path | Description |
|------|-------------|
| `commands/analyze-tx.md` | Advanced transaction analysis entry |
| `commands/trace-asset.md` | Asset provenance tracing entry |
| `commands/write-tbc-code.md` | TBC code generation entry |
| `agents/tbc-analyst.md` | Dedicated TBC analyst agent |
| `docs/api-reference.md` | API documentation entry (compatibility wrapper) |
| `docs/code-reference.md` | Code examples entry (compatibility wrapper) |
| `tests/fixtures/` | Transaction classification regression fixtures (covering all 9 contract types) |

### Compatibility layer

| Path | Description |
|------|-------------|
| `tbc-dev/SKILL.md` | Original complete skill, retained to avoid breaking legacy workflows |
| `tbc-dev/api-reference.md` | Original full API documentation |
| `tbc-dev/code-reference.md` | Original code examples documentation |

## Installation

### Option 1: Cursor Plugin (Marketplace style)

This repo includes `.cursor-plugin/plugin.json`, conforming to the [Cursor plugin spec](https://cursor.com/docs/reference/plugins).

- **Install from Cursor**: Open the Plugins panel in Cursor, search for `tbc-developer` or add via Team Marketplace (see below)
- **Team Marketplace**: Go to Cursor team admin -> Settings -> Plugins -> Team Marketplaces -> Import, enter `https://github.com/utxoskills/tbcdevskills`
- **Local trial**: Clone the repo locally, then add it as a Team Marketplace directory in Cursor

### Option 2: OpenClaw

```bash
npx skills add utxoskills/tbcdevskills
```

### Option 3: Manual symlink

```bash
git clone https://github.com/utxoskills/tbcdevskills.git
ln -sf $(pwd)/tbcdevskills/tbc-dev ~/.openclaw/workspace/skills/tbc-dev
```

## Updates

- **Cursor**: Plugin updates require submitting a new version at [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish). Users will see the update in the Plugins panel after review.
- **Team Marketplace**: After `git push`, the team admin can refresh/re-import in the admin panel to pull the latest content.
- **OpenClaw / symlink**: Run `cd tbcdevskills && git pull` and the changes take effect in the next session.

## Design principles

- **No content loss**: The original `tbc-dev/` directory is retained
- **Product-shaped structure**: Added `skills/`, `commands/`, `agents/`, `docs/`, `tests/`
- **No regression**: The multi-skill structure improves discovery and routing; the legacy complete knowledge base remains as a fallback reference

## Use cases

- TBC DApp developers wanting AI-assisted contract interaction code
- Quick classification of on-chain transactions by contract type
- AI understanding of TBC vs BTC/BSV consensus differences
- Building browser-based applications with TuringWallet

## License

MIT
