# TBC Dev Skills

面向 TBC (TuringBitChain) 区块链开发者的 AI 技能包，让任何支持 skills 的 AI Agent 快速掌握 TBC 全栈开发知识。

## 这是什么

一套结构化的 TBC 开发知识，可以被 [OpenClaw](https://openclaw.ai)、Cursor 插件系统等 AI Agent 入口加载使用。装上之后，Agent 就能：

- 分析任意 TBC 交易，识别所有 9 种合约类型（FT、NFT、Pool、OrderBook、MultiSig、HTLC、StableCoin、PiggyBank 等）
- 编写 TBC 智能合约交互代码（`tbc-lib-js` + `tbc-contract` SDK）
- 理解 TBC 的 UTXO 模型、共识规则、P2P 协议、内存池策略
- 调用 TuringWallet 插件钱包 API 构建 DApp
- 掌握 UTXO 应用层开发模式（双花防护、链式交易、并发控制、碎片化管理）
- 查询和调用 TBC 全节点 API

## 结构

### 多 skill 入口

| 路径 | 说明 |
|------|------|
| `skills/transaction-analysis/` | 交易识别、原子交换、支付 vs 找零、资产流向分析 |
| `skills/contract-patterns/` | FT / NFT / Pool / FTLP / OrderBook / HTLC / MultiSig 等结构区分 |
| `skills/sdk-coding/` | `tbc-lib-js` + `tbc-contract` SDK 编码与构造模式 |
| `skills/utxo-design/` | UTXO 原子性、并发控制、溯源、碎片化管理 |
| `skills/node-internals/` | 共识、mempool、P2P、DAA、KYC 矿工验证 |
| `skills/turingwallet-api/` | TuringWallet 浏览器钱包 API |

### 配套入口

| 路径 | 说明 |
|------|------|
| `commands/analyze-tx.md` | 高级交易分析入口 |
| `commands/trace-asset.md` | 资产溯源入口 |
| `commands/write-tbc-code.md` | TBC 代码生成入口 |
| `agents/tbc-analyst.md` | 专用 TBC 分析 agent |
| `docs/api-reference.md` | API 文档入口（兼容包装） |
| `docs/code-reference.md` | 代码示例入口（兼容包装） |
| `tests/fixtures/` | 交易识别回归样例 |

### 兼容层

| 路径 | 说明 |
|------|------|
| `tbc-dev/SKILL.md` | 原始完整技能，继续保留，避免旧工作流失效 |
| `tbc-dev/api-reference.md` | 原始 API 全量文档 |
| `tbc-dev/code-reference.md` | 原始代码示例文档 |

## 安装

### 方式 1：Cursor 插件（Marketplace 风格）

本仓库已包含 `.cursor-plugin/plugin.json`，符合 [Cursor 插件规范](https://cursor.com/docs/reference/plugins)。

- **从 Cursor 安装**：在 Cursor 中打开 Plugins 面板，搜索 `tbc-developer` 或添加 Team Marketplace（见下方）
- **Team Marketplace**：在 Cursor 团队后台 → Settings → Plugins → Team Marketplaces → Import，填入 `https://github.com/utxoskills/tbcdevskills`
- **本地试用**：将仓库 clone 到本地，在 Cursor 中添加 Team Marketplace 并选择该目录

### 方式 2：OpenClaw

```bash
npx skills add utxoskills/tbcdevskills
```

### 方式 3：手动软链接

```bash
git clone https://github.com/utxoskills/tbcdevskills.git
ln -sf $(pwd)/tbcdevskills/tbc-dev ~/.openclaw/workspace/skills/tbc-dev
```

## 更新

- **Cursor**：插件更新需在 [cursor.com/marketplace/publish](https://cursor.com/marketplace/publish) 提交新版本，审核通过后用户可在 Plugins 面板看到更新
- **Team Marketplace**：仓库 `git push` 后，团队管理员在后台刷新/重新导入即可拉取最新内容
- **OpenClaw / 软链接**：`cd tbcdevskills && git pull`，下次会话即生效

## 设计原则

- **内容不丢**：原始 `tbc-dev/` 目录继续保留
- **结构更像插件产品**：新增 `skills/`、`commands/`、`agents/`、`docs/`、`tests/`
- **效果不退化**：多 skill 结构用于更好的发现与路由，旧的完整知识库继续作为兜底参考

## 适用场景

- TBC DApp 开发者想让 AI 辅助写合约交互代码
- 想快速分析链上交易属于哪种合约类型
- 需要 AI 理解 TBC 与 BTC/BSV 的共识差异
- 构建基于 TuringWallet 的浏览器端应用

## License

MIT
