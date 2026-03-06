---
name: tbc-dev
description: "TBC (TuringBitChain) blockchain development reference. Covers contract types, transaction structures, API endpoints, node ops. Use when user asks about TBC contracts, transactions, or API."
---

# TBC Development Skill

TBC = Bitcoin sidechain, SHA256 PoW, UTXO model, JS/TS smart contracts.

**Libraries**: `npm i tbc-lib-js tbc-contract`

**GitHub 源码**:
- 合约 SDK: https://github.com/TuringBitChain/tbc-contract
- 基础库: https://github.com/AidenChen6/tbc-lib-js
- 全节点: https://github.com/TuringBitChain/TBCNODE
- 学习资料: https://github.com/TuringBitChain/LearningMaterials

**API Base URLs**:
- Mainnet: `https://api.turingbitchain.io/api/tbc/`
- Testnet: `https://api.tbcdev.org/api/tbc/`
- API 文档(ShowDoc): https://www.showdoc.com.cn/2598842292642718 （密码: 123123）
- 区块浏览器: https://tbcscan.xyz

---

## Contract Types

| Contract | Purpose | Key Methods |
|----------|---------|-------------|
| **FT** | Fungible token | `MintFT`, `transfer`, `transferWithAdditionalInfo`, `batchTransfer`, `mergeFT` |
| **NFT** | Non-fungible token | `createCollection`, `createNFT`, `transferNFT`, `transferNFTWithTBC`, `batchCreateNFT` |
| **poolNFT** | AMM liquidity pool v1 | `createPoolNFT`, `initPoolNFT`, `increaseLP`, `consumeLP`, `swaptoToken_baseTBC`, `swaptoTBC_baseToken`, `mergeFTLP`, `mergeFTinPool` |
| **poolNFT2** | Pool v2 (lock, fee, plans) | v1 全部 + `burnFTLP`, `unlockFTLP`, `initPoolNFTWithLockTime`, `increaseLpWithLockTime`, `consumeLpWithLockTime`, `swaptoTBC_baseToken_local`, `fetchFtlpBalance/UTXOList/LockTime` |
| **OrderBook** | On-chain DEX | `makeSellOrder`, `makeBuyOrder`, `cancelSellOrder`, `cancelBuyOrder`, `matchOrder` (每种有 privateKey / Online / fillSigs 三种模式) |
| **MultiSig** | M-of-N multisig (1-6 sigs, 3-10 keys) | `createMultiSigWallet`, `p2pkhToMultiSig_sendTBC/transferFT`, `build/sign/finish MultiSigTransaction_sendTBC/transferFT` (含 batch 变体) |
| **HTLC** | Hash time-lock (atomic swap) | `deployHTLC`, `withdraw`, `refund`, `fillSigDepoly`(SDK拼写), `fillSigWithdraw`, `fillSigRefund`, `deployHTLCWithSign`, `withdrawWithSign`, `refundWithSign` |
| **stableCoin** | Admin-controlled FT (extends FT) + coinNFT cert | `createCoin` → `[coinNftTXRaw, coinMintRaw]`, `mintCoin`, `transfer`(override), `batchTransfer`(override), `mergeCoin`, `frozenCoinUTXO` |
| **piggyBank** | TBC time-lock freeze | `freezeTBC(address, tbcNumber, lockTime, utxos)`, `unfreezeTBC(address, utxos, network?)`, `fetchTBCLockTime(utxo)` |

### Contract Inheritance

```
FT (基础同质化代币)
├── stableCoin (extends FT, 管理员铸造/销毁)
└── poolNFT (uses FT)

NFT (基础非同质化代币)
└── coinNFT (稳定币证书 NFT, 记录供应量和发行历史)

MultiSig, HTLC, OrderBook, PiggyBank (standalone)
```

## 交易分析流程（核心能力）

当用户给你一个 txid 要求分析时，按以下步骤执行：

> **⚠️ 禁止用 browser 打开区块浏览器（如 tbcscan.xyz）来"看一眼"就结束。** 区块浏览器只显示表面信息，无法判断合约类型和事件。必须用 API 拿原始交易数据，然后结合合约结构自己分析判断。这才是你作为 TBC 开发助手的核心能力。
>
> **⚠️ 分析交易时必须一次性完成全部步骤给出最终结论，不要中途停下来问用户"要不要继续分析"。** 用户让你分析交易就是要完整结果：交易类型 + 具体事件 + 涉及地址 + 金额 + 代币信息，全部查完一起说。

### Step 1: 获取交易数据

用 `nodes run` 执行 curl（不要用 browser，不要用 web_fetch）：
```
["curl", "-s", "https://api.turingbitchain.io/api/tbc/decode/txid/{txid}"]
```
返回包含完整的 vin/vout/scriptPubKey 信息。如需原始 hex 可用 `/txraw/txid/{txid}`。

如果是 FT 相关交易，追加调用 `/ft/decode/txid/{txid}` 获取可读的 contract_id、地址、金额。

### Step 2: 判断交易类型

检查 vout 的 scriptPubKey，按以下优先级匹配：

**1) FT 交易** — vout 的 hex 末尾包含 `4654617065`（"FTape" 的 hex）

> **⚠️ 关键修正：判断 Mint vs Transfer 看输入类型，不是看输出脚本标记！**
> - `2Code`（32436f6465）出现在 Code 脚本中，Mint 和 Transfer 都有，不能用来区分
> - **FT Mint**：**输入是 TBC（无 FT 输入）**，输出是 FT（Code=500sat + Tape=0sat）
> - **FT Transfer**：**输入包含 FT UTXO**，输出是 FT，可能有找零 Code+Tape

- **FT Batch Transfer**：多个接收者的 Code+Tape 对
- **FT Merge**：多个 FT 输入 → 1 个 FT 输出（合并 UTXO）
- 进一步用 `/ft/decode/txid/{txid}` 获取 contract_id、地址、金额等可读信息
- 再用 `/ft/info/contract/{contract_id}` 获取代币名称、符号、精度、总量（返回含 `code_script`, `tape_script`, `creator_combine_script`）

**2) NFT 交易** — vout 的 hex 末尾包含 `4e54617065`（"NTape"）或 `4e486f6c64`（"NHold"）
- **NFT Collection Create**：Tape(0sat) + 多个 Mint script(100sat each, 含 `"V0 Mint NHold"`)
- **NFT Mint**：Code(200sat) + Hold(100sat, 含 `"V0 Curr NHold"`) + Tape(0sat, "NTape")
- **NFT Transfer**：需要两个输入 — Code input (index 0, 需合约解锁数据) + Hold input (index 1, 标准 P2PKH 签名)；输出 3 个 — Code(200sat) + Hold(100sat, 新 owner pubKeyHash) + Tape(0sat)
- **NFT Batch Mint**：多组 Code+Hold+Tape
- Tape 的 OP_RETURN 数据是 hex 编码的 JSON，解码后含 name/description/attributes
- NFT `file` 字段 = `collection_id` + `outputIndex`(4字节小端 hex)，唯一标识集合中的每个 NFT

NFT 流程: 创建集合(TBC输入) → 生成 Mint UTXO → 作为输入创建 NFT → 生成 NFT UTXO → 作为输入转移 NFT

**3) Pool 交易** — vout[0] 有超长 nonstandard 脚本（Pool NFT 合约脚本），且末尾含 `6269736f6e`（"bison"）+ `32436f6465`（"2Code"）
- **Pool Create**：vin 无 Pool NFT → vout 有 Pool NFT script
- **Pool Init**：首次注入流动性（FT + TBC）
- **Pool Add LP**：增加流动性，产生 LP token
- **Pool Remove LP**：消耗 LP，取回 FT + TBC
- **Pool Swap（TBC→FT）**：Pool 的 TBC 余额增加，FT 余额减少
- **Pool Swap（FT→TBC）**：Pool 的 FT 余额增加，TBC 余额减少
- 用 `/pool/poolinfo/poolid/{pool_contract_id}` 获取池子状态（tbc_balance, token_balance, lp_balance）
- swap 金额通过比较 vin/vout 的 Pool NFT 关联值变化计算

**4) OrderBook 交易** — vout 含 saleVolume/unitPrice/feeRate（3 个 8 字节 LE 值）的脚本
- **Sell Order**：创建卖单
- **Buy Order**：创建买单
- **Cancel Order**：撤销挂单
- **Match Order**：撮合成交

**5) MultiSig 交易** — vout 有 P2SH 脚本 `OP_HASH160 <20B hash> OP_EQUAL`（type: "scripthash"）
- 创建多签钱包输出结构：Output 1 = 主 P2SH 输出, Output 2~N+1 = 每个公钥一个 Hold(200sat), Output N+2 = Tape(0sat, 记录多签配置)
- 锁定脚本: `OP_<M> OP_SWAP <split/pick/cat验证ops> OP_HASH160 <hash> OP_EQUALVERIFY OP_<N> OP_CHECKMULTISIG`（pubkeys 在花费时提供并通过 hash 验证，不是裸嵌入），解锁需 `OP_0` + M 个签名 + redeemScript
- 多签交易流程: 构建(任一方) → 分发给各方签名 → 各方分别签名 → 聚合签名完成 → 广播
- 用 `/multisig/multisigaddress/address/{a}` 查询多签钱包信息

**6) HTLC 交易** — 脚本包含 `OP_IF OP_SHA256` 分支结构
- **Deploy**：创建 HTLC 合约
- **Withdraw**：提供 preimage 取款
- **Refund**：超时退款

**7) 纯 TBC 转账** — 所有 vout 都是标准 P2PKH（type: "pubkeyhash"），无合约标记

### Step 3: 组装结果（必须做完才回复用户）

Step 1-3 是一个完整流程，**必须全部执行完才回复用户**。不要执行到一半就发消息问"需要继续吗"。

最终输出应包含：
- **交易类型**：如 "FT Transfer" / "Pool Swap (TBC→FT)"
- **关键地址**：发送方、接收方
- **代币信息**（如适用）：contract_id、name、symbol、decimal
- **金额**：FT 数量（注意 decimal 转换）、TBC 数量
- **池子信息**（如适用）：pool_id、swap 方向、换入/换出金额

**示例输出**：
> 这是一笔 **FT Transfer** 交易。地址 `1ABC...` 向 `1XYZ...` 转了 **300,000 SATOSHI**（contract: `a2d7...86f3`，精度 6 位，即 300 枚）。

> 这是一笔 **Pool Swap (TBC→FT)** 交易。用户通过 Pool `d1ab...dfc6` 用 **5 TBC** 换得 **12,500 SpaceDoge**。

## Script Hex 标记速查

| 标记 | Hex | 含义 |
|------|-----|------|
| FTape | `4654617065` | FT Tape 输出标识 |
| NTape | `4e54617065` | NFT Tape 输出标识 |
| NHold | `4e486f6c64` | NFT Hold 输出标识 |
| bison | `6269736f6e` | Pool NFT 脚本标识 |
| 2Code | `32436f6465` | 合约 Code 输出标识 |

## Script Structure (Quick Ref)

**FT Code**: 数百操作码的智能合约脚本（含 OP_PARTIAL_HASH, OP_PUSH_META, 输入验证循环），末尾 `OP_RETURN 0x15 <pubKeyHash+00:21B> 0x05 0x32436f6465`("2Code"). **注意**: `OP_DUP OP_HASH160 ... OP_CHECKSIG OP_RETURN <flag>` 是 MintFT 的 source tx 输出（flag="for ft mint"），不是 FT Code 脚本本身
**FT Tape**: `OP_FALSE OP_RETURN <amount:48B = 6×uint64LE> <decimal:1B> <name> <symbol> "FTape"`
**NFT Hold**: `... OP_RETURN "V0 Curr NHold"`
**NFT Tape**: `OP_FALSE OP_RETURN <JSON hex> "NTape"`

FT amount = 6 slots of uint64LE (48 bytes, supports merging up to 6 UTXOs). Mint 时 slot 0 有金额, slots 1-5 为 0; Transfer 时 `buildTapeAmount` 将多个输入金额分配到各 slot. Each FT output: Code=500sat, Tape=0sat. Each NFT output: Code=200sat, Hold=100sat.
FT max supply constraint: `maxAmount = 10^(18 - decimal)`.
FT transfer 支持可选 `tbc_amount` 参数，可在一笔交易中同时发送 FT 和 TBC.

## Pool AMM Formula

**Pool v1** — 纯恒定乘积，无手续费:
```
poolMul = ft_a_amount * tbc_amount    // x * y = k
new_reserve = poolMul / updated_reserve
```
swap 方法: `swaptoToken`(deprecated), `swaptoToken_baseTBC`, `swaptoTBC`(deprecated), `swaptoTBC_baseToken`

**Pool v2** — 在 v1 基础上增加 service fee:
```
serviceFee = amount * (service_fee_rate + 10) / 10000    // 默认 service_fee_rate=25, 即万分之35
amount_swap = amount - serviceFee                         // 扣除手续费后参与 x*y=k
```
v2 额外参数: `service_fee_rate`(默认 25/万分), `lpPlan`(1 or 2), optional lock time. Service provider: "bison".

LP token amount initialized equal to TBC amount. Precision: `BigInt(1000000)` (6 decimals for TBC).

Pool v1 initPool 输出: `[index 0: Pool NFT (1000 sat), index 1: Pool NFT Tape (0 sat), index 2: FT_A Code (variable sat = TBC amount), index 3: FT_A Tape (0 sat), index 4: FTLP Code (500 sat), index 5: FTLP Tape (0 sat)]`

## API Endpoints (Complete)

All paths relative to base URL. Pagination max: 500. Full docs: `skills/tbc-dev/api-reference.md`

**TBC 原生**:
- `/decode/txid/{txid}` — 通用交易解码（任意类型交易）
- `/txraw/txid/{txid}` — 获取原始交易数据
- `/balance/address/{a}` | `/utxo/address/{a}?limit=N` | `/addressinfo/address/{a}`
- `/history/address/{a}/start/{s}/end/{e}` | `/history/scriptpubkeyhash/{h}/start/{s}/end/{e}`
- `/utxo/scriptpubkeyhash/{h}`
- `/broadcasttx` (POST) | `/broadcasttxs` (POST 批量)
- `/blockByHeight/height/{h}` | `/blockByHash/hash/{h}` | `/recentblocks/start/{s}/end/{e}`
- `/txbyblockhash/blockhash/{h}/start/{s}/end/{e}` | `/txbyblockheight/height/{h}/start/{s}/end/{e}`
- `/mempoolinfo` | `/mempooltxs` | `/nodeinfo` | `/chainstate` | `/health`

**FT 代币**:
- `/ft/decode/txid/{txid}` — FT 交易解码
- `/ft/info/contract/{id}` — 获取 FT 详情（含 code_script, tape_script）
- `/ft/tokenbalance/address/{a}/contract/{id}` | `/ft/tokenbalance/combinescript/{cs}/contract/{id}`
- `/ft/utxo/address/{a}/contract/{id}?limit=N` | `/ft/utxo/combinescript/{cs}/contract/{id}?limit=N`
- `/ft/history/address/{a}/contract/{id}/start/{s}/end/{e}` | `/ft/history/combinescript/{cs}/contract/{id}/start/{s}/end/{e}`
- `/ft/allhistory/address/{a}/start/{s}/end/{e}` | `/ft/allhistory/combinescript/{cs}/start/{s}/end/{e}`
- `/ft/tokenlist/address/{a}` | `/ft/tokenlist/combinescript/{cs}`
- `/ft/rank/contract/{id}/start/{s}/end/{e}` — 持币人排行

**NFT**:
- `/nft/nftinfo/nftid/{id}` | `/nft/collectioninfo/collectionid/{id}`
- `/nft/nftbyaddress/address/{a}/start/{s}/end/{e}` | `/nft/collectionbyaddress/address/{a}/start/{s}/end/{e}`
- `/nft/nftbycollection/collectionid/{id}/start/{s}/end/{e}`
- `/nft/history/address/{a}/nftid/{id}/start/{s}/end/{e}` | `/nft/allhistory/address/{a}/start/{s}/end/{e}`
- `/nft/collectionlist/start/{s}/end/{e}` | `/nft/nftlist/start/{s}/end/{e}`
- `/nft/utxo/scriptpubkeyhash/{h}`

**Pool**: `/pool/poolinfo/poolid/{id}` | `/pool/poollist/start/{s}/end/{e}` | `/pool/lputxo/scriptpubkeyhash/{h}`

**多签**: `/multisig/multisigaddress/address/{a}`

**注意**: 几乎所有 FT/地址端点都有对应的 `combinescript` 变体（如 `/ft/tokenbalance/combinescript/{cs}/contract/{id}`）。`combine_script` 是 21 字节 hex 字符串，用于标识持有者。

## SDK Complete Method Reference

### FT (Fungible Token)

```
constructor(txidOrParams?: string | { name, symbol, amount, decimal })
initialize(ftInfo: FtInfo): void
MintFT(privateKey, address_to, utxo) → string[] [txSourceRaw, txMintRaw]
transfer(privateKey, address_to, ft_amount, ftutxo_a[], utxo, preTX[], prepreTxData[], tbc_amount?) → string
transferWithAdditionalInfo(privateKey, address_to, amount, ftutxo_a[], utxo, preTX[], prepreTxData[], additionalInfo: Buffer) → string
batchTransfer(privateKey, receiveAddressAmount: Map<string, number|string>, ftutxo[], utxo, preTX[], prepreTxData[]) → {txraw}[]
mergeFT(privateKey, ftutxo[], utxo, preTX[], prepreTxData[], localTX[]) → {txraw}[] (递归合并, 每批最多5个)
```
Key constants: Code=500sat, Tape=0sat, max decimal=18, max amount=10^(18-decimal), merge batch=5, fee=80sat/KB

### NFT (Non-Fungible Token)

```
constructor(contract_id: string)
initialize(nftInfo: NFTInfo): void
transferNFT(address_from, address_to, privateKey, utxos[], pre_tx, pre_pre_tx, batch=false) → string
transferNFTWithTBC(address_from, address_to_nft, address_to_tbc, privateKey, utxos[], pre_tx, pre_pre_tx, tbc_amount) → string
static createCollection(address, privateKey, data: CollectionData, utxos[]) → string
static createNFT(collection_id, address, privateKey, data: NFTData, utxos[], nfttxo) → string
static batchCreateNFT(collection_id, address, privateKey, datas[], utxos[], nfttxos[]) → {txraw}[]
```
Key constants: Code=200sat, Hold=100sat, Tape=0sat, collection mint=100sat each

### poolNFT v1

```
constructor(config?: { txidOrParams?: string | { ftContractTxid, tbc_amount, ft_a }, network? })
async initCreate(ftContractTxid?) | async initfromContractId()
async createPoolNFT(privateKey, utxo) → string[]
async createPoolNftWithLock(privateKey, utxo) → string[]
async initPoolNFT(privateKey, address_to, utxo, tbc_amount?, ft_a?) → string
async increaseLP(privateKey, address_to, utxo, amount_tbc) → string
async consumeLP(privateKey, address_to, utxo, amount_lp) → string
async swaptoToken_baseTBC(privateKey, address_to, utxo, amount_tbc) → string
async swaptoTBC_baseToken(privateKey, address_to, utxo, amount_token) → string
async swaptoToken(deprecated) | async swaptoTBC(deprecated)
async mergeFTLP(privateKey, utxo) → boolean|string
async mergeFTinPool(privateKey, utxo) → boolean|string
```

### poolNFT2 (v2 独有方法)

```
async initPoolNFTWithLockTime(privateKey, address_to, utxo, tbc_amount, ft_a, lock_time) → string
async increaseLpWithLockTime(privateKey, address_to, utxo, amount_tbc, lock_time) → string
async consumeLpWithLockTime(privateKey, address_to, utxo, amount_lp) → string
async swaptoTBC_baseToken_local(privateKey, address_to, ftutxo, ftPreTX[], ftPrePreTxData[], amount_token, lpPlan?, utxo?) → string
async burnFTLP(privateKey, utxo) → string
async unlockFTLP(privateKey, utxo, lock_time?) → string|null
async fetchFtlpBalance(address) → bigint
async fetchFtlpUTXOList(address) → IUnspentOutput[]
async fetchFtlpLockTime(address) → {ftBalance, lockTime}[]
async getPoolNftExtraInfo() → {serviceFeeRate, lpPlan, withLock, withLockTime}
```
v2 createPoolNFT 新增参数: `tag`, `serviceFeeRate?`, `lpPlan?: 1|2`, `withLockTime?`
v2 createPoolNftWithLock 新增参数: `tag`, `lpCostAddress`, `lpCostTBC`, `pubKeyLock[]`
v2 移除: `swaptoToken`(deprecated), `swaptoTBC`(deprecated)

### OrderBook (On-chain DEX)

三种模式: ① privateKey 离线签名, ② privateKeyOnline 自动获取 UTXO, ③ build + fillSigs 分步签名

```
// 卖单
buildSellOrderTX(holdAddr, saleVolume, unitPrice, feeRate, ftID, ftPartialHash, utxos[]) → string
makeSellOrder_privateKey(privateKey, saleVolume, unitPrice, feeRate, ftID, ftPartialHash, utxos[])
async makeSellOrder_privateKeyOnline(privateKey, saleVolume, unitPrice, feeRate, ftID)
fillSigsSellOrder(txRaw, sigs[], publicKey, type: "make"|"cancel") → string
buildCancelSellOrderTX(sellutxo, utxos[]) → string
cancelSellOrder_privateKey(privateKey, sellutxo, utxos[])
async cancelSellOrder_privateKeyOnline(privateKey, sellutxo)

// 买单
buildBuyOrderTX(holdAddr, saleVolume, unitPrice, feeRate, ftID, utxos[], ftutxos[], preTXs[]) → string
makeBuyOrder_privateKey(privateKey, saleVolume, unitPrice, feeRate, ftID, utxos[], ftutxos[], preTXs[], prepreTxData[])
async makeBuyOrder_privateKeyOnline(privateKey, saleVolume, unitPrice, feeRate, ftID)
fillSigsMakeBuyOrder(txRaw, sigs[], publicKey, preTXs[], prepreTxData[]) → string
buildCancelBuyOrderTX(buyutxo, ftutxo, ftPreTX, utxos[]) → string
cancelBuyOrder_privateKey(privateKey, buyutxo, buyPreTX, ftutxo, ftPreTX, ftPrePreTxData, utxos[])
async cancelBuyOrder_privateKeyOnline(privateKey, buyutxo)
fillSigsCancelBuyOrder(txRaw, sigs[], publicKey, buyPreTX, ftPreTX, ftPrePreTxData) → string

// 撮合
matchOrder(privateKey, buyutxo, buyPreTX, ftutxo, ftPreTX, ftPrePreTxData, sellutxo, sellPreTX, utxos[], ftFeeAddr, tbcFeeAddr) → string
async matchOrderOnline(privateKey, buyutxo, sellutxo, ftFeeAddr, tbcFeeAddr)

// 工具
static getOrderData(codeScript) → { holdAddress, saleVolume, ftPartialHash, feeRate, unitPrice, ftID }
static updateSaleVolume(codeScript, newSaleVolume) → Script
```

### stableCoin (extends FT)

```
createCoin(privateKey_admin, address_to, utxo, utxoTX, mintMessage?) → string[] [coinNftTXRaw, coinMintRaw]
mintCoin(privateKey_admin, address_to, mintAmount, utxo, nftPreTX, nftPrePreTX, mintMessage?) → string
transfer(override, 继承 FT) | batchTransfer(override, 继承 FT)
mergeCoin(privateKey, utxo, ftutxo[], preTX[], prepreTxData[]) → {txraw}[]
frozenCoinUTXO(privateKey, ftutxo, utxo, preTX, prepreTxData, locktime) → string
static buildCoinNftOutput(data: coinNftData) | static buildCoinNftTX(privateKey, utxo, data)
static getCoinMintCode(adminAddr, toAddr, originCodeHash, tapeSize)
static setLockTimeInTape(tapeScript, locktime) | static getLockTimeFromTape(tapeScript)
static getAddressFromCode(codeScript)
```
使用 `feePerKb(80)`. coinNFT 证书为 "The sole issuance certificate for the stablecoin, dynamically recording cumulative supply and issuance history."

### HTLC

```
deployHTLC(sender, receiver, hashlock, timelock, amount, utxo) → string
withdraw(receiver, htlcUtxo) → string
refund(sender, htlcUtxo, timelock) → string
fillSigDepoly(deployHTLCTxRaw, sig, publicKey) → string   // 注意 SDK 拼写 Depoly
fillSigWithdraw(withdrawTxRaw, secret, sig, publicKey) → string
fillSigRefund(refundTxRaw, sig, publicKey) → string
deployHTLCWithSign(sender, receiver, hashlock, timelock, amount, utxo, privateKey) → string
withdrawWithSign(receiver, htlcUtxo, secret, privateKey) → string
refundWithSign(sender, htlcUtxo, timelock, privateKey) → string
```

### API Class (33 static methods)

**TBC 原生:**
```
getTBCbalance(address, network?) → number
fetchUTXOList(address, network?) → IUnspentOutput[]
fetchUTXO(privateKey, amount, network?) → IUnspentOutput (自动选择)
getUTXOs(address, amount_tbc, network?) → IUnspentOutput[]
mergeUTXO(privateKey, network?) → boolean
fetchTXraw(txid, network?) → Transaction
broadcastTXraw(txraw, network?) → string (txid)
broadcastTXsraw(txrawList: {txraw}[], network?) → any
fetchBlockHeaders(network?) → BlockHeader[]
```

**FT 相关:**
```
getFTbalance(contractTxid, addressOrHash, network?) → bigint
fetchFtUTXO(contractTxid, addressOrHash, amount, codeScript, network?) → IUnspentOutput
fetchFtUTXOs(contractTxid, addressOrHash, codeScript, network?, amount?) → IUnspentOutput[]
fetchFtUTXOList(contractTxid, addressOrHash, codeScript, network?) → IUnspentOutput[]
fetchFtUTXOsforPool(contractTxid, addressOrHash, amount, number, codeScript, network?) → IUnspentOutput[]
fetchFtInfo(contractTxid, network?) → FtInfo
fetchFtPrePreTxData(preTX, preTxVout, network?) → string
fetchFtUTXOS_multiSig(contractTxid, addressOrHash, codeScript, network?) → IUnspentOutput[]
getFtUTXOS_multiSig(contractTxid, addressOrHash, codeScript, amount, network?) → IUnspentOutput[]
```

**NFT 相关:**
```
fetchNFTInfo(contract_id, network?) → NFTInfo
fetchNFTTXO({script, tx_hash?, network?}) → IUnspentOutput
fetchNFTTXOs({script, tx_hash, network?}) → IUnspentOutput[]
fetchNFTs(collection_id, address, start, end, network?) → string[]
```

**Pool 相关:**
```
fetchPoolNftInfo(contractTxid, network?) → PoolNFTInfo
fetchPoolNftUTXO(contractTxid, network?) → IUnspentOutput
fetchFtlpBalance(ftlpCode, network?) → bigint
fetchFtlpUTXO(ftlpCode, amount, network?) → IUnspentOutput
```

**多签/冻结/其他:**
```
fetchUMTXO(script_asm, tbc_amount, network?) → IUnspentOutput (用于MultiSig等非标脚本)
fetchUMTXOs(script_asm, network?) → IUnspentOutput[]
getUMTXOs(script_asm, amount_tbc, network?) → IUnspentOutput[]
fetchFrozenTBCBalance(address, network?) → number
fetchFrozenUTXOList(address, network?) → IUnspentOutput[]
fetchUnfrozenUTXOList(address, network?) → IUnspentOutput[]
```

### Utility Functions (从 tbc-contract 导出)

```
buildUTXO(tx, vout, isFT?) → IUnspentOutput          // 从 Transaction 对象构造 UTXO
buildFtPrePreTxData(preTX, preTxVout, localTXs[]) → string  // 本地计算 prePreTxData
getFtBalanceFromTape(tape: string) → bigint            // 从 tape hex 读取 FT 余额
selectTXfromLocal(txs[], txid) → Transaction           // 从本地 TX 数组查找
parseDecimalToBigInt(amount, decimal) → bigint         // 金额转 BigInt
fetchInBatches(items[], batchSize, fetchFn, context) → R[]  // 批量请求
fetchWithRetry(fn, retries?, delay?, context?) → T     // 带重试的请求
getOpCode(number) → string                            // 数字转操作码
getLpCostAddress(poolCode) → string                    // 从 Pool 代码获取 LP 成本地址
getLpCostAmount(poolCode) → number                     // 从 Pool 代码获取 LP 成本金额
isLock(length) → 0|1                                   // 判断是否有锁
fetchTBCLockTime(utxo) → number                        // 获取 UTXO 锁定时间
```

## Node Deployment (Quick)

Build: `git clone https://github.com/turingbitchain/TBCNODE.git && cd TBCNODE && mkdir build && cd build && cmake .. && make -j$(nproc)`
RPC: `tbc-cli getinfo | getblockcount | getblockhash <h> | getrawtransaction <txid> | sendrawtransaction <hex>`
Key config: `excessiveblocksize=10000000000`, `txindex=1`, `rpcport=8332`

---

## Full Code Examples（写代码时必读）

**当用户要求写任何 TBC 相关代码时，先读 `skills/tbc-dev/code-reference.md`**，不要凭记忆写。里面有完整可运行的 TypeScript 代码。

| # | 内容 | 关键方法 |
|---|------|---------|
| 1 | TBC 转账 | `tbc.Transaction` |
| 2 | FT 铸造 | `FT.MintFT` |
| 3 | FT 转账 | `FT.transfer` |
| 4 | FT 批量转账 | `FT.batchTransfer` |
| 5 | FT 合并 UTXO | `FT.mergeFT` |
| 6 | NFT 创建合集 | `NFT.createCollection` |
| 7 | NFT 铸造 | `NFT.createNFT` |
| 8 | NFT 转账 | `NFT.transferNFT` |
| 9 | NFT 批量铸造 | `NFT.batchCreateNFT` |
| 10 | Pool v1 完整生命周期 | `createPoolNFT` → `initPoolNFT` → `increaseLP` / `consumeLP` → `swaptoToken_baseTBC` / `swaptoTBC_baseToken` |
| 11 | Pool v2 (Lock/Fee) | 同上 + `createPoolNftWithLock`, `burnFTLP`, lockTime |
| 12 | Pool Swap 高级（本地 UTXO） | 跳过 API 直接构造 swap |
| 13 | OrderBook DEX | `buildSellOrderTX` / `buildBuyOrderTX` / `matchOrder` / cancel |
| 14 | MultiSig 多签 | TBC 转账 + FT 转账（M-of-N） |
| 15 | Memo / TimeLock | 备注转账 + 时间锁转账 |
| 16 | buildUTXO 工具 | 从 TX 构造 UTXO 对象 |

> **`code-reference.md` 里没有覆盖的情况**（如 HTLC、StableCoin、复杂组合交易）：
> 去 GitHub 源码看具体实现 → https://github.com/TuringBitChain/tbc-contract/tree/main/lib/contract
> 每个合约类型一个文件（`ft.ts`, `poolNFT.ts`, `orderBook.ts`, `multiSig.ts`, `htlc.ts`, `stableCoin.ts` 等）。

## Skill 文件结构（发给别人时全部包含）

| 文件 | 内容 |
|------|------|
| `SKILL.md` | 主文件（你正在看的这个） |
| `code-reference.md` | 16 种交易类型完整 TypeScript 代码 |
| `api-reference.md` | API 全量文档（端点、参数、返回格式） |

## TBC 链特有常量

| 常量 | 值 | 说明 |
|------|-----|------|
| COINBASE_MATURITY | 1 | 矿工奖励 1 个确认后即可花费（Bitcoin = 100） |
| TX Version | 10 (0x0a) | TBC 交易版本号 |
| Genesis Height (mainnet) | 620538 | Genesis 升级激活高度 |
| TBC First Block Height | 824190 | TBC 链第一个区块 |
| OP_PUSH_META | 0xba | TBC 特有操作码，用于访问交易元数据（HTLC 时间验证、NFT Code 脚本） |
| OP_PARTIAL_HASH | 0xbb | TBC 特有操作码，部分 SHA256 哈希（Pool 合约脚本优化） |

Post-Genesis: P2SH 输出被禁止；OP_RETURN 格式从 `OP_RETURN <data>` 变为 `OP_FALSE OP_RETURN <data>`；脚本大小和操作码数量限制移除（最大 4GB）；启用 OP_MUL, OP_DIV, OP_MOD, OP_CAT, OP_SPLIT 等操作码.

## HTLC 脚本细节

HTLC 使用 OP_SHA256（非 OP_HASH256）做 hashlock 验证, OP_PUSH_META 做时间验证.
Refund 路径: `setInputSequence(0, 4294967294)` + `setLockTime(timelock)`.
三种使用方式: ① deploy/withdraw/refund (返回未签名 raw), ② fillSig* (分步填充签名), ③ *WithSign (直接用 privateKey 签名).
注意 SDK 中 `fillSigDepoly` 是 typo（不是 Deploy）。

## 中心化 Swap 模式（交易分析时注意）

Pool 合约有频率限制，因此存在中心化 Swap 服务模式：
1. 服务地址先从 Pool 批量 Swap 获取 FT
2. 通过连续的 FT_Transfer 分发给用户（每次找零给自己）
3. 可能达到 900+ 层转移链

**识别特征**：长转移链（>100 层）、同一地址反复出现、无 `bison` Pool 标记、连续 FT_Transfer。
**溯源策略**：快速跳过服务转移链，关注 Pool_Swap 边界作为关键节点。

## 交易广播可靠性（重要）

TBC 节点的 `CTxnPropagator` 每 250ms 批量处理新交易，存在以下已知问题：
- **250ms 窗口丢失**：节点断连期间的交易不会重试
- **filterInventoryKnown 污染**：节点收到 INV 但断连后，重连时认为已知该 tx，永远不再请求

**最佳实践**：
1. 广播后验证 tx 存在于多个节点（`/mempoolinfo` 或 `getmempoolentry`）
2. 缺失则用 `/broadcasttx` 重新广播
3. 关键交易实现重试循环：broadcast → wait 2s → check mempool → rebroadcast if missing
4. 用 `/broadcasttxs` 批量发送到多个 API 端点

## SDK Key Types / Interfaces

```typescript
FtInfo { contractTxid?, codeScript, tapeScript, totalSupply: bigint, decimal, name, symbol }
NFTInfo { collectionId, collectionIndex, collectionName, nftName, nftSymbol, nft_attributes, nftDescription, nftTransferTimeCount, nftIcon }
NFTData { nftName, symbol, description, attributes, file? }
CollectionData { collectionName, description, supply, file }
PoolNFTInfo { ft_lp_amount: bigint, ft_a_amount: bigint, tbc_amount: bigint, ft_lp_partialhash, ft_a_partialhash, ft_a_contractTxid, service_fee_rate, service_provider, poolnft_code, pool_version, currentContractTxid, currentContractVout, currentContractSatoshi }
coinNftData { nftName, nftSymbol, description, coinDecimal, coinTotalSupply: bigint }
```

## SDK Key Constants

| 常量 | 值 | 说明 |
|------|-----|------|
| FT Code satoshis | 500 | 所有 FT code 输出 |
| FT Mint source satoshis | 9900 | MintFT source TX |
| NFT Code satoshis | 200 | NFT code 输出 |
| NFT Hold satoshis | 100 | NFT hold 输出, collection mint slots |
| Pool NFT v1 satoshis | 1000 | Pool NFT code 输出 |
| Fee rate | 80 sat/KB | 或 flat 80 (tx < 1000 bytes) |
| FT merge batch size | 5 | 每轮最多合并 5 个 FT UTXO |
| TBC decimal places | 6 | parseDecimalToBigInt 中 TBC 用 6 位精度 |
| OrderBook buy_code_dust | 300 | 买单 code 输出 satoshis |

## 深入学习资源

需要更底层的合约实现细节时，去 GitHub 看源码：
- **合约代码**: https://github.com/TuringBitChain/tbc-contract → `lib/contract/` 目录下每个合约一个文件
- **FT 合约**: `lib/contract/ft.ts` — 铸造/转账/批量/合并的完整实现
- **Pool 合约**: `lib/contract/poolNFT.ts` / `poolNFT2.0.ts` — AMM 池子全生命周期
- **OrderBook**: `lib/contract/orderBook.ts` — 链上 DEX 挂单/撮合
- **MultiSig**: `lib/contract/multiSig.ts` — 多签钱包
- **HTLC**: `lib/contract/htlc.ts` — 哈希时间锁（含 fillSig 方法）
- **StableCoin**: `lib/contract/stableCoin.ts` — 稳定币（继承 FT）
- **API 封装**: `lib/api/api.ts` — 所有 API 调用方法的源码
