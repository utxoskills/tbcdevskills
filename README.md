# TBC Dev Skills

面向 TBC (TuringBitChain) 区块链开发者的 AI 技能包，让任何支持 skills 的 AI Agent 快速掌握 TBC 全栈开发知识。

## 这是什么

一套结构化的 TBC 开发知识，可以被 [OpenClaw](https://openclaw.ai) 等 AI Agent 加载使用。装上之后，Agent 就能：

- 分析任意 TBC 交易，识别所有 9 种合约类型（FT、NFT、Pool、OrderBook、MultiSig、HTLC、StableCoin、PiggyBank 等）
- 编写 TBC 智能合约交互代码（`tbc-lib-js` + `tbc-contract` SDK）
- 理解 TBC 的 UTXO 模型、共识规则、P2P 协议、内存池策略
- 调用 TuringWallet 插件钱包 API 构建 DApp
- 掌握 UTXO 应用层开发模式（双花防护、链式交易、并发控制、碎片化管理）
- 查询和调用 TBC 全节点 API

## 内容

| 文件 | 说明 |
|------|------|
| `tbc-dev/SKILL.md` | 主技能文件 — 合约结构、交易分析、共识差异、节点知识、UTXO 开发模式、TuringWallet API |
| `tbc-dev/api-reference.md` | 完整 API 文档 — 所有节点接口的参数与返回示例 |
| `tbc-dev/code-reference.md` | 21 个 TypeScript 代码示例 — 覆盖全部合约类型的构造与调用 |

## 安装

```bash
git clone https://github.com/utxoskills/tbcdevskills.git
ln -sf $(pwd)/tbcdevskills/tbc-dev ~/.openclaw/workspace/skills/tbc-dev
```

## 更新

```bash
cd tbcdevskills && git pull
```

软链接安装后，`git pull` 即可让 Agent 下次会话使用最新知识，无需额外操作。

## 适用场景

- TBC DApp 开发者想让 AI 辅助写合约交互代码
- 想快速分析链上交易属于哪种合约类型
- 需要 AI 理解 TBC 与 BTC/BSV 的共识差异
- 构建基于 TuringWallet 的浏览器端应用

## License

MIT
