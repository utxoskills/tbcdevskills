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

**4) OrderBook 交易** — vout 的脚本末尾有固定结构的 OrderData（6 段数据：holdAddress(20B) + saleVolume(8B LE) + ftPartialHash(32B) + feeRate(8B LE) + unitPrice(8B LE) + ftContractId(32B)）
- **Sell Order（创建卖单）**：vout[0] = 卖单脚本 (satoshis = saleVolume)，无 FT 输入
- **Buy Order（创建买单）**：vout[0] = 买单脚本 (satoshis = 300 = buy_code_dust)，vin 含 FT 输入（锁定 FT 到合约）
- **Cancel Sell**：vin 消费卖单 UTXO → vout 回到持有者地址
- **Cancel Buy**：vin 消费买单 UTXO → FT 退回持有者
- **Match Order**：vin 同时消费买单 + 卖单 UTXO → 完成交换，FT 给买方，TBC 给卖方，手续费给 ftFeeAddress/tbcFeeAddress
- 区分 sell vs buy: **卖单 satoshis = saleVolume（大额）**，**买单 satoshis = 300（固定小额）+ FT 输出**
- 用 `OrderBook.getOrderData(codeScript)` 可解析出 holdAddress, saleVolume, feeRate, unitPrice, ftID

**5) MultiSig 交易** — vout 有 P2SH 脚本 `OP_HASH160 <20B hash> OP_EQUAL`（type: "scripthash"）
- 创建多签钱包输出结构：Output 1 = 主 P2SH 输出, Output 2~N+1 = 每个公钥一个 Hold(200sat), Output N+2 = Tape(0sat, 记录多签配置)
- 锁定脚本: `OP_<M> OP_SWAP <split/pick/cat验证ops> OP_HASH160 <hash> OP_EQUALVERIFY OP_<N> OP_CHECKMULTISIG`（pubkeys 在花费时提供并通过 hash 验证，不是裸嵌入），解锁需 `OP_0` + M 个签名 + redeemScript
- 多签交易流程: 构建(任一方) → 分发给各方签名 → 各方分别签名 → 聚合签名完成 → 广播
- 用 `/multisig/multisigaddress/address/{a}` 查询多签钱包信息

**6) HTLC 交易** — 脚本包含 `OP_IF OP_SHA256` 分支结构
- **Deploy**：创建 HTLC 合约
- **Withdraw**：提供 preimage 取款
- **Refund**：超时退款

**7) StableCoin 交易** — stableCoin 继承 FT，链上表现与 FT 相同（有 FTape 标记），但有特殊操作：
- **CreateCoin（首次发行）**：产生两笔交易 — 第一笔创建 coinNFT 证书（含 NTape），第二笔铸造 FT（含 FTape）。识别：同一批广播的两笔 tx，一个有 NFT 输出 + 一个有 FT 输出
- **MintCoin（增发）**：vin 消费 coinNFT + 输出新 coinNFT（更新供应量）+ 新 FT 输出
- **transfer/batchTransfer**：与普通 FT 完全相同
- **frozenCoinUTXO**：FT transfer 但 tape 中含 locktime 信息
- 区分 stableCoin vs 普通 FT: 看该 contract_id 是否关联了 coinNFT 证书

**8) PiggyBank 交易** — vout 含特殊时间锁脚本：`OP_DUP OP_HASH160 <pkh> OP_EQUALVERIFY OP_CHECKSIGVERIFY OP_6 OP_PUSH_META ... <lockTimeHex> OP_BIN2NUM OP_2 OP_PUSH_META OP_BIN2NUM OP_LESSTHANOREQUAL OP_1 OP_EQUAL`
- **FreezeTBC**：vin 是标准 TBC → vout 有 PiggyBank 时间锁脚本（检查脚本含 `OP_CHECKSIGVERIFY OP_6 OP_PUSH_META`）
- **UnfreezeTBC**：vin 消费 PiggyBank UTXO（时间锁已过期）→ vout 回到标准 P2PKH
- 不是普通 P2PKH（含 OP_PUSH_META），也不是 HTLC（没有 OP_IF 分支）

**9) 纯 TBC 转账** — 所有 vout 都是标准 P2PKH（type: "pubkeyhash"），无合约标记

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
| V0 Mint NHold | `5630204d696e74204e486f6c64` | NFT 集合 Mint slot 标识 |
| V0 Curr NHold | `56302043757272204e486f6c64` | NFT 当前持有者标识 |
| for ft mint | `666f722066742(...)` | FT Mint source TX flag |
| OP_CHECKSIGVERIFY OP_6 OP_PUSH_META | `ad56ba` | PiggyBank 时间锁脚本特征（紧跟在 P2PKH 验证后） |

## 交易线格式 (Wire Format)

```
nVersion (int32 LE) → vin (CompactSize + vector) → vout (CompactSize + vector) → nLockTime (uint32 LE)
```

**CTxIn**: `prevout.txid(32B) || prevout.n(4B) || scriptSig(varint+data) || nSequence(4B)`
**CTxOut**: `nValue(8B int64 LE) || scriptPubKey(varint+data)`

区块头: 80 字节 = `nVersion(4) || hashPrevBlock(32) || hashMerkleRoot(32) || nTime(4) || nBits(4) || nNonce(4)`
区块: `header(80B) || tx_count(CompactSize) || transactions...`

**无 SegWit** — 线格式与 BSV 完全相同（但 TxId 计算不同，见"TBC v10 TxId"章节）。

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

## Node Deployment & Operations

Build: `git clone https://github.com/turingbitchain/TBCNODE.git && cd TBCNODE && mkdir build && cd build && cmake .. && make -j$(nproc)`
RPC: `tbc-cli getinfo | getblockcount | getblockhash <h> | getrawtransaction <txid> | sendrawtransaction <hex>`
Key config: `excessiveblocksize=10000000000`, `txindex=1`, `rpcport=8332`
DNS Seeds: `seed.tbcnode.org`, `seed.tbcnode.com` (测试网: `test.tbctestnet.top`)

**存储引擎** (LevelDB):
- DB cache 默认 450MB, batch 大小 16MB
- Key 前缀: `C`=UTXO, `c`=legacy UTXO, `f`=block file info, `t`=tx index, `b`=block index, `B`=best block, `H`=head blocks, `F`=flags, `R`=reindex, `l`=last block file
- Block 文件格式: `[4B magic][4B size][block]`，超大区块(>=4GB): `[4B magic][0xFFFFFFFF][8B size][block]`
- CDiskTxPos 支持 64 位偏移（`uint64_t nTxOffset`），兼容超大区块
- UTXO Coin 压缩: `VARINT((height << 1) | isCoinBase)` + `CTxOutCompressor`
- Undo 数据: `VARINT(height * 2 + isCoinbase)` + 被消费的 TxOut
- UTXO hash 使用 SipHash-2-4; 签名/脚本缓存使用 CuckooCache

**链工作量**: `work = 2^256 / (target + 1)` (chain.cpp:135)

**区块组装**:
- 默认使用 **JournalingBlockAssembler**（后台线程每 100ms 处理，最多 20,000 tx/时间片）
- Legacy assembler（祖先费率排序, 5% 高优先级保留）仍可用但非默认
- Coinbase 预留: 1000 bytes + 100 sigops
- Coinbase scriptSig: `<height> <extraNonce> <COINBASE_FLAGS>`，最大 100 字节

**挖矿 RPC**:
- `getblocktemplate`: 支持 BIP22/23, 长轮询, 缓存 5 秒
- `getminingcandidate` / `submitminingsolution`: 精简替代方案，只返回头部+Merkle proof（UUID 标识候选块）

**费率估算**: 仅有简单的 `CTxMemPool::estimateFee()` 方法，**无** `estimateSmartFee` RPC

**检查点**: 824190 (TBC 分叉块) 和 918000 已硬编码。分叉前不允许分叉。

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
| 17 | HTLC 哈希时间锁 | `deployHTLC` / `withdraw` / `refund`（两阶段 + 直接签名两种模式） |
| 18 | StableCoin 稳定币 | `createCoin` / `mintCoin` / `transfer` / `mergeCoin` / `frozenCoinUTXO` |
| 19 | PiggyBank 时间锁冻结 | `freezeTBC` / `unfreezeTBC` / `fetchTBCLockTime` / `fetchFrozenTBCBalance` |
| 20 | NFT 转账 + TBC 支付 | `NFT.transferNFTWithTBC`（一笔 tx 同时转 NFT + 发 TBC） |
| 21 | FT 转账 + 附加信息 | `FT.transferWithAdditionalInfo`（附加 OP_RETURN 数据） |

所有合约类型都有完整代码示例，覆盖所有 9 种合约。

## Skill 文件结构（发给别人时全部包含）

| 文件 | 内容 |
|------|------|
| `SKILL.md` | 主文件（你正在看的这个） |
| `code-reference.md` | 16 种交易类型完整 TypeScript 代码 |
| `api-reference.md` | API 全量文档（端点、参数、返回格式） |

## TBC 链特有常量（源码验证）

| 常量 | 值 | 源码位置 | 说明 |
|------|-----|----------|------|
| COINBASE_MATURITY | 1 | consensus/consensus.h | 矿工奖励 1 个确认后即可花费（Bitcoin = 100） |
| TX nVersion | 10 (0x0a) | validation.cpp:3474 | 所有交易必须 nVersion==10（区块 824190 后强制） |
| Genesis Height | 620538 | chainparams.cpp:21 | Genesis 协议升级激活高度 |
| TBC First Block Height | 824190 | validation.cpp:79 | TBC 链硬分叉高度 |
| TBC First Block Hash | `0000000058968601...` | chainparams.cpp:108 | 824190 区块 hash |
| KYC V1 Height | 824189 | validation.cpp:970 | Coinbase 矿工身份验证 V1 激活 |
| KYC V2 Height | 927000 | validation.cpp:971 | FilledMinerBillV2 验证激活 |
| OP_PUSH_META | 0xba | script/opcodes.h:144 | TBC 特有操作码，访问交易元数据 |
| OP_PARTIAL_HASH | 0xbb | script/opcodes.h:145 | TBC 特有操作码，部分 SHA256 哈希 |
| OP_CHECKDATASIG | 0xbc | script/opcodes.h:146 | 数据签名验证（TBC 4 参数版本） |
| SIGHASH_FORKID | 0x40 | script/sighashtype.h:18 | 签名强制包含 FORKID |
| LOCKTIME_THRESHOLD | 500000000 | script/script.h:33 | < 此值 = 区块高度, >= 此值 = Unix 时间戳 |
| SEQUENCE_LOCKTIME_GRANULARITY | 9 | primitives/transaction.h | BIP68 时间锁粒度 = 2^9 = 512 秒 |
| Net Magic (mainnet) | `e3e1f3e8` | chainparams.cpp:151 | P2P 网络 magic bytes |
| Disk Magic | `f9beb4d9` | chainparams.cpp:147 | 磁盘存储 magic bytes |

### OP_PUSH_META 元数据映射（interpreter.cpp:786-882）

取栈顶 1 字节，有效值 1-7:

| 值 | 返回数据 | 大小 | 说明 |
|-----|----------|------|------|
| 1 | `tx.nVersion` | 4B | 交易版本号 |
| 2 | `tx.nLockTime` | 4B | 交易时间锁 |
| 3 | `tx.vin.size()` | 4B | 输入数量 |
| 4 | `tx.vout.size()` | 4B | 输出数量 |
| 5 | `SHA256(all inputs)` | 32B | 所有输入的 SHA256 摘要 |
| 6 | `prevout.txid + prevout.n + nSequence` | 40B | 当前输入的 outpoint + 序列号 |
| 7 | `SHA256(all outputs)` | 32B | 所有输出的 SHA256 摘要 |

合约用法：值 2 用于 PiggyBank 时间锁验证；值 5/7 用于 FT/Pool 合约的 partial hash 验证。

### Post-Genesis 脚本规则（区块 >= 620538）

| 规则 | Before Genesis | After Genesis |
|------|---------------|--------------|
| 脚本最大大小 | 10,000 字节 | UINT32_MAX (~4GB) |
| 操作码数量限制 | 500 | UINT32_MAX (无限制) |
| 栈元素大小 | 520 字节 | 无限制（受内存限制: 100MB policy, INT64_MAX consensus） |
| P2SH 输出 | 允许 | **禁止** (拒绝码: `bad-txns-vout-p2sh`) |
| OP_RETURN 格式 | `OP_RETURN <data>` | `OP_FALSE OP_RETURN <data>` |

Post-Genesis 启用的操作码: `OP_MUL(0x95)`, `OP_DIV(0x96)`, `OP_MOD(0x97)`, `OP_CAT(0x7e)`, `OP_SPLIT(0x7f)`, `OP_NUM2BIN`, `OP_BIN2NUM`, `OP_AND`, `OP_OR`, `OP_XOR`. 仅禁用: `OP_2MUL`, `OP_2DIV`.

签名方案: ECDSA (变长) + Schnorr (固定 64 字节)。Schnorr 多签由 `SCRIPT_ENABLE_SCHNORR_MULTISIG (1<<20)` 控制。

### 脚本验证标志位（script_flags.h）

| 标志 | 值 | 说明 |
|------|-----|------|
| SCRIPT_ENABLE_SIGHASH_FORKID | 1<<16 | 强制签名含 FORKID |
| SCRIPT_GENESIS | 1<<18 | Genesis 规则激活 |
| SCRIPT_UTXO_AFTER_GENESIS | 1<<19 | UTXO 在 Genesis 后创建 |
| SCRIPT_ENABLE_SCHNORR_MULTISIG | 1<<20 | 启用 Schnorr 多签 |

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

## TBC vs BSV 关键差异（共识级）

TBC 从 BSV 在区块 824,190 硬分叉。以下是经源码验证的所有共识级差异：

| 特性 | BSV | TBC |
|------|-----|-----|
| 默认 tx 版本 | 1 或 2 | **10** (CURRENT_VERSION=10, MAX_STANDARD_VERSION=12) |
| TxId 计算 (v10+) | 全序列化 double-SHA256 | **结构化哈希树**（见下方） |
| OP_PUSH_META (0xBA) | 无 | **交易内省操作码** |
| OP_PARTIAL_HASH (0xBB) | 无 | **增量 SHA256 (midstate 支持)** |
| KYC 矿工验证 | 无 | **共识级强制** (V1=ECDSA, V2=Schnorr) |
| Dust 阈值 | 动态计算 (基于费率) | **固定 10 satoshis** |
| 显示精度 | 8 位小数 (COIN=10^8) | **6 位小数** (TBCCOIN=10^6) |
| 货币单位 | BSV | **TBC** |
| Pre-fork UTXO | 可花费 | **824190 后冻结** (inputScriptBlockHeight <= 824190 不可花费) |
| COINBASE_MATURITY | 100 | **1** |

### TBC v10 TxId 计算方式（关键！）

nVersion >= 10 的交易使用完全不同于标准 Bitcoin 的哈希算法：

```
root = double-SHA256(
  nVersion (4B LE)
  || nLockTime (4B LE)
  || vin_count (4B uint32, 非 CompactSize!)
  || vout_count (4B uint32, 非 CompactSize!)
  || SHA256(foreach vin: prevout.txid || prevout.n || nSequence)
  || SHA256(foreach vin: SHA256(scriptSig, 无长度前缀))
  || SHA256(foreach vout: nValue || SHA256(scriptPubKey, 无长度前缀))
)
```

注意：
- 输入/输出数量用 **固定 4 字节 uint32**（不是 CompactSize）
- 子哈希用 **单次 SHA256**（`GetSingleHash()`）
- 最终 root 用 **双次 SHA256**（`GetHash()`）
- **不能用标准 BSV/BTC 库计算 TBC v10 TxId**

源码：`hash.h:260-313 TxSerializeHash()`

### Pre-fork UTXO 冻结规则

```
if (spendHeight >= 824190 && inputScriptBlockHeight <= 824190) → REJECT
```
区块 824190 之后，所有在 824190 或之前创建的 UTXO **不可花费**，有效冻结了所有分叉前的币。

---

## 共识规则详解（源码验证）

### 区块验证流程

1. **CheckBlockHeader**: PoW 验证 — hash 满足 nBits 目标
2. **ContextualCheckBlockHeader**: 难度匹配 + 时间戳 > MTP(11) + 时间戳 < 当前+20分钟
3. **CheckBlock**: Merkle root 匹配 + 无 CVE-2012-2459 重复子树 + CheckTransactionCommon
4. **ContextualCheckBlock**: BIP34 coinbase 高度编码 + coinbase 奖励 ≤ 补贴+手续费

### 交易验证规则

| 检查项 | 规则 | 违规码 |
|--------|------|--------|
| 非空输入 | `vin.size() > 0` | `bad-txns-vin-empty` (DoS 10) |
| 非空输出 | `vout.size() > 0` | `bad-txns-vout-empty` (DoS 10) |
| 交易大小 | Before Genesis ≤ 1MB, After Genesis ≤ 1GB | `bad-txns-oversize` (DoS 100) |
| 输出值非负 | `nValue >= 0` | `bad-txns-vout-negative` |
| 输出值不超限 | `nValue ≤ MAX_MONEY (21M×10^8)` | `bad-txns-vout-toolarge` |
| 无重复输入 | prevout 唯一 | `bad-txns-inputs-duplicate` |
| nVersion == 10 | spendHeight >= 824190 | `bad-rgtx-nVersion` |
| 无 P2SH 输出 | After Genesis | `bad-txns-vout-p2sh` (DoS 100) |
| Coinbase 成熟 | `spendHeight - coinHeight >= 1` | `bad-txns-premature-spend-of-coinbase` |
| 不凭空造币 | `inputValue >= outputValue` | `bad-txns-in-belowout` |

### 交易大小限制

| 上下文 | 限制 | 源码 |
|--------|------|------|
| Before Genesis (共识) | 1 MB | consensus/consensus.h:26 |
| After Genesis (共识) | **1 GB** | consensus/consensus.h:28 |
| Before Genesis (策略) | 99,999 bytes | policy/policy.h:69 |
| After Genesis (策略默认) | 10 MB | policy/policy.h:71 |

### 区块大小与奖励

| 参数 | 值 | 源码 |
|------|-----|------|
| 最大区块大小 (共识) | **INT64_MAX** (无限制) | policy/policy.h:37 |
| 最大生成区块 (Before) | 32 MB | policy/policy.h:46 |
| 最大生成区块 (After) | 128 MB | policy/policy.h:47 |
| 初始区块奖励 | 50 × COIN (5,000,000,000 sat) | validation.cpp:2933 |
| 减半间隔 | 210,000 块 | chainparams.cpp:95 |
| 目标出块时间 | 600 秒 (10 分钟) | chainparams.cpp:113 |
| MAX_MONEY | 21,000,000 × COIN | amount.h:161 |
| COIN | 100,000,000 sat | amount.h:144 |
| TBCCOIN | 1,000,000 sat (6 位精度) | amount.h:145 |

### 费率与 Dust

| 参数 | 值 | 源码 |
|------|-----|------|
| 默认最小转发费率 | 60 sat/KB | validation.h:83 |
| Dust 阈值 | **固定 10 satoshis** | primitives/transaction.h:208 |
| 默认区块最小交易费 | 500 sat/KB | policy/policy.h:67 |
| 默认最大交易费 | COIN/10 = 10,000,000 sat | validation.h:85 |
| OP_RETURN 数据限制 | UINT32_MAX (~4GB, 无限) | script/standard.h:30 |

### 签名哈希类型

| 类型 | 值 | 组合 |
|------|-----|------|
| SIGHASH_ALL | 0x01 | +FORKID: 0x41, +ANYONECANPAY: 0x81, +Both: 0xC1 |
| SIGHASH_NONE | 0x02 | +FORKID: 0x42, +ANYONECANPAY: 0x82, +Both: 0xC2 |
| SIGHASH_SINGLE | 0x03 | +FORKID: 0x43, +ANYONECANPAY: 0x83, +Both: 0xC3 |
| SIGHASH_FORKID | 0x40 | TBC 交易必须包含此标志 |
| SIGHASH_ANYONECANPAY | 0x80 | 仅签名自己的输入 |

签名方案: ECDSA (DER 编码, 变长) + **Schnorr** (固定 64 字节, x-only 32 字节公钥)。签名方法自动检测。

### 公钥格式

| 类型 | 格式 | 用途 |
|------|------|------|
| 压缩公钥 | `02`/`03` + 32B x坐标 | ECDSA 标准 |
| 未压缩公钥 | `04` + 64B (x+y) | ECDSA 兼容 |
| X-only 公钥 | 32B x坐标 | Schnorr (BIP-340) |

### 地址格式 (Base58Check)

| 网络 | 前缀 (P2PKH) | 前缀 (P2SH) | 前缀 (WIF) | 起始字符 |
|------|-------------|-------------|------------|---------|
| 主网 | 0x00 | 0x05 | 0x80 | `1...` / `3...` / `5/K/L...` |
| 测试网 | 0x6F | 0xC4 | 0xEF | `m/n...` / `2...` / `c...` |

HD 钱包: 主网 `xpub`/`xprv`, 测试网 `tpub`/`tprv` — 与 BTC/BSV 相同。

---

## 难度调整算法 (DAA)

### 分叉点重置

区块 824188 → 难度强制重置为 `0x1d00ffff`（最低难度/difficulty 1）。

### TBC DAA（区块 >= 824189，与 BSV DAA 不同）

```
NewBlockSpacing = GetNewBlockSpacing(pindexPrev, 8064, params)  // 8064 块回溯窗口
nextTarget = ComputeTarget(pindexFirst, pindexLast, NewBlockSpacing)  // 144 块采样

// 每块调整幅度限制 ±6.25%
prevTargetUpLimit = prevTarget + (prevTarget >> 4)
prevTargetDnLimit = prevTarget - (prevTarget >> 4)
nextTarget = clamp(nextTarget, prevTargetDnLimit, prevTargetUpLimit)

// 紧急条件：最近 12 块 MTP 超过 6 小时 → 强制降低难度
if (mtp12blocks > 6 * 3600) → nextTarget = prevTargetUpLimit
```

| 参数 | BCH/BSV DAA | TBC DAA |
|------|-------------|---------|
| 回溯窗口 | 144 块固定 | 144 采样 + **8064 块**计算动态出块间距 |
| 出块间距 | 固定 600s | **动态**（根据 8064 块实际产出调整） |
| 每次调整幅度 | [0.5x, 2x] 每周期 | **±6.25%** 每块 |
| 紧急机制 | 无 | 12 块 > 6 小时 → 立即降难度 |
| 中位数选择 | 3 块中位数 | 可变奇数块排序网络 |

---

## KYC 矿工验证（共识级）

### V1 (ECDSA, 区块 >= 824189)

Manager 公钥: `0318a274337b8f52726ee6080e6432a643dbaffd8f183c5b52338acfa90cd92f43`

Coinbase `vout[0].scriptPubKey` 结构:
```
P2PKH(25B) || OP_RETURN(1B) || KYC标记(3B) || 矿工公钥B(33B) || Manager签名A(变长)
|| 许可高度(3B BE) || 费率(1B) || [矿工签名B(变长)]
```

验证:
1. 许可高度 >= 当前区块高度
2. `vout[0].nValue >= 总输出 × 费率 / 100`
3. ECDSA 验证 Manager 签名
4. ECDSA 验证矿工签名 (`Hash(blockHeight_BE)`)

### V2 (Schnorr, 区块 >= 927000)

两个 Manager x-only 公钥:
- `84ddaab460c3e2d460a5c706746d3894928cfcac87a41840e1e5992ebf047b49`
- `f54e6c6619cfd2c2b62ce17fa0366b8a69ee0d97a4d9752e7286b71530dbf02e`

Coinbase 结构变化: 使用 Schnorr 64 字节签名，增加 countryCode(4B)，矿工签名放在 `vin[0].scriptSig` 中。

---

## P2P 网络协议详解

### 握手流程

```
Outbound: 我→VERSION → 对方→VERSION → 我→VERACK → 我→PROTOCONF → 对方→VERACK → 对方→PROTOCONF
          → SENDHEADERS (v>=70012) → SENDCMPCT (v>=70014)
Inbound:  对方→VERSION → 我→VERSION → 我→VERACK → 我→PROTOCONF → ...
```

### 协议版本

| 常量 | 值 | 说明 |
|------|-----|------|
| PROTOCOL_VERSION | **90015** | TBC 协议版本（非 BTC 70015） |
| MIN_PEER_PROTO_VERSION | 90014 | 最低接受版本（低于此拒绝连接） |

### 全部 27 种 P2P 消息类型

| 消息 | 说明 | TBC 特有 |
|------|------|---------|
| VERSION/VERACK | 握手 | |
| ADDR/GETADDR | 地址交换 | |
| INV/GETDATA/NOTFOUND | 库存公告/请求 | |
| TX/BLOCK | 交易/区块传输 | |
| HEADERS/GETHEADERS | 区块头同步 (max 2000/次) | |
| GETBLOCKS | 请求区块 INV | |
| MEMPOOL | 请求内存池 TXID | |
| PING/PONG | 心跳 (120s 间隔) | |
| REJECT | BIP61 拒绝 | |
| FEEFILTER | BIP133 费率过滤 | |
| SENDHEADERS | BIP130 头优先偏好 | |
| SENDCMPCT/CMPCTBLOCK/GETBLOCKTXN/BLOCKTXN | BIP152 紧凑区块 | |
| MERKLEBLOCK/FILTERLOAD/FILTERADD/FILTERCLEAR | BIP37 布隆过滤 | |
| **PROTOCONF** | **协议配置协商**（交换最大接收载荷大小） | **是** |

### 连接管理

| 参数 | 值 | 说明 |
|------|-----|------|
| 最大出站连接 | 8 | net.h:83 |
| 最大 addnode 连接 | 8 | net.h:85 |
| 最大总连接 | 125 | 即最大入站 = 109 |
| Ping 间隔 | 120 秒 | |
| 不活跃超时 | 20 分钟 | |
| Ban 时长 | 24 小时 | |
| Ban 分数阈值 | 100 | 累计 DoS 分数达到 100 则 ban |
| Feeler 间隔 | 120 秒 | 探测新地址 |

### PROTOCONF 协议配置

- VERACK 之后立即发送
- 每连接只允许一次（重复则断开）
- 协商最大接收载荷大小 (默认 2 MiB, 最大 4 MiB)
- 对方 `maxRecvPayloadLength < 1 MiB` 则断开
- 动态计算每对端的 `maxInvElements = (maxPayloadLength - 8) / 36`

### 入站连接驱逐策略（5 层保护）

满连接时驱逐策略：
1. 保护 4 个网络组最多样化的节点
2. 保护 8 个 ping 最低的节点
3. 保护 4 个最近转发过交易的节点
4. 保护 4 个最近转发过区块的节点
5. 保护剩余一半连接时间最长的节点
→ 从最大连接组中驱逐最新加入的节点

### 区块下载参数

| 参数 | 值 |
|------|-----|
| 每对端并行下载区块数 | 16 |
| 下载窗口 | 1024 块 |
| 区块停滞超时 | 10 秒 |
| 最低下载速率 | 100 KB/s |
| 紧凑区块最大深度 | 5 |
| 孤儿交易池 | 100 MB, 20 分钟过期 |
| INV 广播间隔 | 5 秒 |
| INV 广播延迟 | 150 ms (默认) |

### 拒绝码

| 码 | 值 | 说明 |
|-----|-----|------|
| REJECT_MALFORMED | 0x01 | 格式错误 |
| REJECT_INVALID | 0x10 | 无效 |
| REJECT_OBSOLETE | 0x11 | 过时 |
| REJECT_DUPLICATE | 0x12 | 重复 |
| REJECT_NONSTANDARD | 0x40 | 非标准 |
| REJECT_DUST | 0x41 | 灰尘输出 |
| REJECT_INSUFFICIENTFEE | 0x42 | 费率不足 |
| REJECT_CHECKPOINT | 0x43 | 检查点不匹配 |
| **REJECT_TOOBUSY** | **0x44** | **TBC 特有**: 节点过忙，5 秒后重试 |

---

## 内存池策略详解

| 参数 | 值 | 源码 |
|------|-----|------|
| 默认大小 | **1000 MB (1GB)** | policy/policy.h:91 |
| 非最终交易池 | 50 MB | policy/policy.h:93 |
| 过期时间 | 336 小时 (14 天) | validation.h:103 |
| 非最终交易过期 | 672 小时 (28 天) | validation.h:105 |
| mapRelay 过期 | 15 分钟 | net_processing.cpp:4057 |
| 祖先数量限制 | 10,000 | validation.h:92 |
| 后代数量限制 | 10,000 | validation.h:93 |
| RBF (替换交易) | **不支持** | 无相关代码 |
| CPFP (子付父费) | **支持** | 祖先费率排序 |

### 时间锁交易 (CTimeLockedMempool)

TBC 有独立的非最终交易池（`time_locked_mempool.h`）：
- 持有 `nLockTime` 未来或 `nSequence` 非最终的交易
- 每 10 分钟检查：当 `IsFinalTx()` 变为 true 时，提交到主内存池
- 更新规则：相同输入 + 至少一个 `nSequence` 增加 + 不减少任何 `nSequence`
- 持久化到 `non-final-mempool.dat`
- **注意：Genesis 后 BIP68 相对时间锁在策略层禁用**

### 交易广播可靠性

**CTxnPropagator**: 每 250ms 批量处理新交易（txn_propagator.h:70），存在已知问题：
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

## UTXO 应用层开发模式（重要）

### 核心思维模型

**UTXO ≠ 余额。你拥有的是一组离散的"硬币"，每个操作都是选择、消费、创造硬币。**

与账户模型（Ethereum）的根本区别：

| 特性 | UTXO 模型 (TBC) | 账户模型 (ETH) |
|------|-----------------|----------------|
| 并行性 | **天然并行** — 不同 UTXO 独立处理 | 同账户交易必须顺序执行 (nonce) |
| 双花防护 | **结构性** — 每个 UTXO 只能花一次 | 需要 nonce 管理 |
| 状态验证 | SPV Merkle proof（轻量） | 需要完整状态树 |
| 智能合约状态 | 需要显式状态携带模式 | 原生可变状态 |

### 1. 双花防护（节点共识级保证）

**节点如何防止双花**（源码验证）：
- **mapNextTx**: 内存池维护 `outpoint → transaction` 映射表（txmempool.cpp:603），每个 outpoint 只映射一笔交易
- **首见规则 (First-Seen)**: 先到的交易占据 outpoint，后到的被拒绝（`txn-mempool-conflict`）
- **无 RBF**: TBC 完全没有 Replace-By-Fee 代码，注释写明 "Disable replacement feature for good"
- **并行检测**: `CTxnDoubleSpendDetector`（txn_double_spend_detector.cpp）在并行验证时用互斥锁保护 outpoint 分配
- **区块重组清理**: `removeConflictsNL` 递归删除与已确认交易冲突的内存池交易

**开发者意义**: 你构建的交易一旦被节点接受进内存池，该 UTXO 就被"锁定"了，其他试图花费同一 UTXO 的交易会被拒绝。这是 UTXO 原子性的第一道防线。

### 2. 交易原子性保证

**UTXO 的全有或全无保证**（源码验证）：

1. **交易级**: `UpdateCoins()` 先花费所有输入，再添加所有输出 — 不存在部分消费（validation.cpp:3346）
2. **区块级**: `ConnectBlock` 在临时 `CCoinsViewCache` 上操作，只有整个区块通过验证才 `Flush()` 到主 UTXO 集（validation.cpp:4838-4882）
3. **磁盘级**: LevelDB 使用 `DB_HEAD_BLOCKS` 二阶段提交，崩溃恢复时可重放（txdb.cpp:85-150）

**开发者意义**: 一笔交易中的所有输入和输出是原子的 — 要么全部生效，要么全部无效。这意味着：
- **原子交换**: HTLC 中，preimage 公开 + 资金转移在一笔交易中原子完成
- **多方交易**: MultiSig 交易中，所有签名者的资金转移是原子的
- **合约状态转换**: Pool swap 中，FT 储备变化 + TBC 储备变化 + LP 变化在一笔交易中原子完成

### 3. 0-conf 链（未确认交易链）

**可以花费未确认的输出**。节点通过 `CCoinsViewMemPool` 层叠视图支持：
- `CCoinsViewDB`（磁盘 UTXO）→ `pcoinsTip`（内存缓存）→ `CCoinsViewMemPool`（加上内存池 UTXO）
- 内存池中的交易输出可以作为新交易的输入（txmempool.cpp:1419）

**TBC 的宽松限制**:
- 祖先链最长 **10,000** 笔交易（默认，可配置）
- 后代链最长 **10,000** 笔交易
- 远超 BTC 的 25 笔限制

**限制**: 非最终交易（nLockTime 未到）的输出**不可被花费**（`too-long-non-final-chain`）

### 4. TTOR（拓扑交易排序规则）

TBC 强制执行 TTOR（validation.cpp:5742）：同一区块内，如果交易 B 花费交易 A 的输出，A 必须排在 B 之前。这意味着 **同一区块内可以包含互相依赖的交易链**。

### 5. OP_FALSE OP_RETURN 与 UTXO 集

`OP_FALSE OP_RETURN` 输出（如 FT Tape、NFT Tape）被识别为不可花费，**永远不进入 UTXO 集**（coins.cpp:118）。这意味着：
- 0 值的 Tape 输出不会膨胀 UTXO 集
- 数据存储不增加节点的 UTXO 集大小
- 只有可花费的输出（Code、Hold、标准 P2PKH）才占用 UTXO 集空间

### 6. 时间锁机制（OP_PUSH_META 替代 CLTV/CSV）

**关键**: Genesis 后 `OP_CHECKLOCKTIMEVERIFY` 和 `OP_CHECKSEQUENCEVERIFY` 被**禁用**（当作 NOP）。

TBC 智能合约使用 **`OP_PUSH_META` 值 2**（读取 `nLockTime`）替代 CLTV 做时间锁验证：
```
<lockTimeHex> OP_BIN2NUM OP_2 OP_PUSH_META OP_BIN2NUM OP_LESSTHANOREQUAL OP_1 OP_EQUAL
```
含义：将锁定时间与交易的 nLockTime 比较，要求 nLockTime >= 锁定时间。

`nLockTime` 规则（仍然有效）：
- `< 500,000,000` → 解释为区块高度
- `>= 500,000,000` → 解释为 Unix 时间戳
- `nSequence == 0xFFFFFFFF` 的输入忽略 nLockTime

---

## SDK UTXO 管理模式

### UTXO 选择策略（SDK 源码验证）

| 方法 | 算法 | 最大数量 | 自动合并 |
|------|------|---------|---------|
| `API.fetchUTXO()` | **首次适配** (first-fit: 第一个 > amount) | 1 | 是（递归合并+3s等待） |
| `API.getUTXOs()` | 返回全部 UTXO | 全部 | 否 |
| `API.fetchFtUTXO()` | 首次适配 (ftBalance >= amount) | 1 | 否（需手动 mergeFT） |
| `API.fetchFtUTXOs()` | **降序排序** + 贪心累加 | 5 | 否 |
| `API.fetchUMTXO()` | 首次适配 + 上限 3200 TBC | 1 | 否 |
| `findMinFiveSum()` | **最优 5-sum** (双指针) | 5 | 否 |

**FT 5 个输入限制**: FT 合约脚本限制每笔交易最多 5 个 FT 输入（tape 有 6 个 uint64LE 槽位），因此 SDK 所有 FT 选择方法都限制最多 5 个。

### 链式交易构建（关键模式）

SDK 大量使用 `addInputFromPrevTx()` + `buildUTXO()` 模式构建未确认交易链：

```typescript
// 模式: 交易 B 花费交易 A 的输出（A 尚未广播）
const txA = new tbc.Transaction().from(utxo).addOutput(...).sign(key);
const txB = new tbc.Transaction()
  .addInputFromPrevTx(txA, 0)  // 花费 txA 的 output[0]
  .addOutput(...)
  .sign(key);

// 必须按顺序广播
await broadcastTXsraw([{ txraw: txA.serialize() }, { txraw: txB.serialize() }]);
```

**使用此模式的合约操作**:

| 操作 | 链结构 | 说明 |
|------|--------|------|
| `FT.MintFT()` | 2 笔: source → mint | mint 花费 source 的 output[0] |
| `stableCoin.createCoin()` | 2 笔: NFT创建 → FT铸造 | 第二笔花费第一笔的 3 个输出 |
| `poolNFT.createPoolNFT()` | 2 笔: source → pool | pool 花费 source 的 output[0] |
| `FT.batchTransfer()` | N 笔链 | 每笔花费上一笔的找零（output[2]=FT, output[4]=TBC） |
| `FT.mergeFT()` | 树状归并 | 5 个一组合并 → 递归合并结果 |

### FT 祖先验证（preTX / prepreTxData）

FT 合约的链上脚本执行**两级祖先验证**：
1. **preTX** (父交易): 创建当前 FT UTXO 的交易，验证其输出结构合法
2. **prepreTxData** (祖父交易数据): 创建父交易输入的交易数据，验证 FT 余额链的完整性

这意味着每笔 FT 交易都携带了可追溯到铸造的**监管链证明**。

**本地构建 vs 网络获取**:
- 已确认的 preTX: 通过 `API.fetchFtPrePreTxData()` 从网络获取
- 未确认链中的 preTX: 通过 `buildFtPrePreTxData()` 从本地交易数组构建（`selectTXfromLocal()`）

### 并发问题与解决方案

**SDK 没有内置并发控制**（无锁、无 UTXO 预留、无冲突检测）。两个进程同时调用 `fetchUTXO()` 可能获取同一个 UTXO，导致第二笔广播失败。

**Pool 合约瓶颈**: Pool NFT 是单 UTXO 状态机，同一时刻只能处理一笔操作。

**应用层解决方案**:

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **UTXO 预留锁** | 数据库 `SELECT FOR UPDATE` 或 Redis 分布式锁，TTL 过期释放 | 多进程共享钱包 |
| **UTXO 分片** | 不同进程分配不同 UTXO 池，无需协调 | 高并发支付 |
| **单写入者队列** | 每个合约实例一个 FIFO 队列 | Pool/OrderBook 操作 |
| **预分裂** | 提前将一个大 UTXO 拆成 N 个，每个独立可用 | 并行支付 |
| **乐观重试** | 失败后重新获取 UTXO 重建交易 | 低并发场景 |

### UTXO 碎片化管理

**问题**: 每次转账产生找零，逐渐积累大量小 UTXO。当需要大额操作时，没有单个 UTXO 足够大。

**SDK 的合并策略**:
- **TBC UTXO**: `API.mergeUTXO()` — 将所有 UTXO 合并为一个，递归执行直到只剩 1 个
- **FT UTXO**: `FT.mergeFT()` — 每轮合并 5 个，树状递归
- **自动触发**: `fetchUTXO()` 在找不到足够大的 UTXO 时自动调用 `mergeUTXO()`

**最佳实践**:
1. **低频合并**: 在低负载时段定期合并（TBC 费率极低，合并成本可忽略）
2. **维持 UTXO 池深度**: 保持 10-50 个 UTXO 用于并行操作
3. **预分裂**: 大额收款后立即分裂成多个适中的 UTXO
4. **避免灰尘**: 不要创建 < 10 satoshi 的输出（共识级 dust 阈值）

### UTXO 状态机模式

TBC 合约使用"状态携带 UTXO"模式：
- **不可变代码** + **可变状态** 编码在 UTXO 的锁定脚本中
- 状态转换 = 花费旧 UTXO + 创建含新状态的 UTXO

```
Pool NFT: [state: reserves, LP] → swap → [state: updated reserves, LP]
OrderBook: [state: order data] → match → [state: fulfilled/partial]
FT: [state: balance] → transfer → [state: new balances]
```

**优势**: 状态转换的原子性由 UTXO 模型天然保证
**限制**: 单 UTXO = 单线程状态机，高频合约需要分片设计

### 并行处理模式（UTXO 独有优势）

UTXO 天然支持并行：不同 UTXO 之间零依赖。

```
Fan-out: 1 个 UTXO → N 个输出 → N 个独立交易可并行执行
Fan-in:  N 个 UTXO → 1 笔交易合并（需要所有 UTXO 都可用）
```

**实际应用**:
- 批量支付: 一笔交易多个输出（`batchTransfer`）比 N 笔单独交易高效
- 并行 swap: 预分裂 TBC 到 N 个 UTXO，同时发起 N 笔不同 Pool 的 swap
- 并行收款: 每个客户付到不同地址，收款后可并行处理

**反模式**: 所有操作串联在一个 UTXO 上（像账户模型一样用单地址收发），完全丧失并行优势。

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
