# tbcdevskills

Development skills, tools, and experiments for building the TBC ecosystem.

TBC (TuringBitChain) blockchain development skills for [OpenClaw](https://openclaw.ai) agents.

## tbc-dev

TBC 区块链开发参考 — 合约类型、交易分析、API 接口、SDK 用法、节点部署。

**包含文件：**
- `SKILL.md` — 主技能文件（合约类型、交易分析流程、脚本标记、AMM 公式、SDK 方法）
- `api-reference.md` — 完整 API 文档（所有接口的参数与返回示例）
- `code-reference.md` — 16 个完整 TypeScript 代码示例（TBC 转账、FT、NFT、Pool、OrderBook、MultiSig 等）

## 安装

```bash
git clone https://github.com/utxoskills/tbcdevskills.git
ln -sf $(pwd)/tbcdevskills/tbc-dev ~/.openclaw/workspace/skills/tbc-dev
```

## 更新

```bash
cd tbcdevskills && git pull
```

通过软链接安装的情况下，`git pull` 后下次 OpenClaw 会话即可生效。

## License

MIT
