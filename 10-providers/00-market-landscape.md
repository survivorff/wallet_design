# 市场格局与份额排序：钱包基础设施赛道谁在前面

## 一句话总结

这个赛道在 2025 年被并购潮重写了一遍——前三名现在是 Privy（Stripe）、Fireblocks Embedded Wallet（含 Dynamic）、Turnkey，但三家的"第一"含义完全不同：Privy 赢在钱包数量，Fireblocks 赢在资产规模，Turnkey 赢在可验证性和开发者口碑。

## 先分清两层市场

用户给的七家公司其实分属两层，混在一起比会得出错误结论。

```
        应用层（面向最终用户/企业客户）
   ┌──────────────────────────────────────────┐
   │  AllScale    Infini    Karsa             │
   │  稳定币新银行 / 企业卡 / 跨境美元账户       │
   └──────────────────────────────────────────┘
                      ↓ 采购钱包能力
        基础设施层（面向开发者，B2B）
   ┌──────────────────────────────────────────┐
   │  Privy   Turnkey   Fireblocks EW         │
   │  Crossmint                               │
   │  密钥管理 / 签名 / 策略引擎 / 合约钱包      │
   └──────────────────────────────────────────┘
```

一个具体的证据：AllScale 的钱包基础设施用的是 Turnkey（见 [05-allscale](./05-allscale.md)）。它们不是竞争对手，是上下游。

所以"市场份额前三"这个问题，得先问是哪一层。**本章按基础设施层来排**，因为那才是可比的同一个市场。

## 2025 年的并购潮：格局被重写

理解今天的格局，必须先看这一年发生了什么。

| 时间 | 事件 | 战略含义 |
|---|---|---|
| 2025 年 6 月 | **Stripe 收购 Privy** | 支付巨头补齐"最后一公里"：前端钱包（Privy）+ 后端稳定币清算（Bridge） |
| 2025 年 | **ConsenSys 收购 Web3Auth** | MetaMask 把嵌入式钱包能力收进自己体系 |
| 2025 年 10 月 | **Fireblocks 收购 Dynamic** | 机构托管龙头下沉消费级，Dynamic 成为其 Embedded Wallet 的前端 |

三起并购指向同一个结论：

```
钱包 SDK 不再是一个独立的生意，
而是更大平台的一个层。

买家的逻辑：
  Stripe    → 我有支付网络，缺钱包入口
  ConsenSys → 我有钱包品牌，缺 B2B 嵌入能力
  Fireblocks→ 我有机构客户，缺消费级前端

卖家的逻辑：
  单卖 SDK 的 ARPU 太低（$0.05/MAU 量级），
  想赚钱要么往上做支付/发卡，要么被有支付网络的人买走。
```

这也意味着：**现在挑供应商，本质是在挑一个母公司的生态站队**。用 Privy 就是在往 Stripe/Tempo 生态靠，用 Fireblocks EW 就是在往机构托管体系靠。

## 各家公开披露的规模指标

先摆事实，再排序。注意每一行的**口径都不一样**。

| 公司 | 披露指标 | 数值 | 口径问题 |
|---|---|---|---|
| **Privy** | 账户数 | 120M+ | 累计创建，非月活；一人多 App 会重复计 |
| | 开发者团队 | 2,000+ | 含免费层 |
| | 月签名量 | 115M+ | 这个指标最实在 |
| | 签名延迟 | < 20ms | 多区域部署后优化过 |
| **Fireblocks EW** | Dynamic 累计 onboard 用户 | 50M+ | Dynamic 口径，非 Fireblocks 整体 |
| | Fireblocks 平台企业客户 | 2,000+ | 含机构托管主业，不只 EW |
| | Fireblocks 累计交易额 | $10T+ | 主业为机构托管，不能算作 EW 业绩 |
| | 估值 | $8B（2025.10 E 轮 $550M） | 全公司 |
| **Turnkey** | 平台托管资产 | $100B+ | 含机构客户，非嵌入式钱包专属 |
| | 累计融资 | > $65M | B 轮 $30M（2025.06）+ $12.5M（2026.05） |
| | 支持链 | 50+ | |
| | 运行时长 | 2+ 年生产环境 | 白皮书口径 |
| **Crossmint** | 客户数 | 40,000+ | 含大量小客户；旗舰客户 MoneyGram、Western Union |
| | 累计融资 | $23.6M | Ribbit Capital 领投（2025.03），Franklin Templeton 参投 |

## 排序判断

### 第一：Privy（Stripe）

理由是三个维度同时领先：

```
✅ 钱包/账户数量：120M+，公开披露中最高
✅ 签名吞吐：115M+/月，是"活跃度"而非"累计数"的证据
✅ 分销渠道：进入 Stripe 体系后，触达的是 Stripe 的百万级商户
✅ 开发者心智：消费级 DApp 的默认选项，集成体验被普遍认为最好

隐忧：
⚠️ 独立性丧失，产品路线要服从 Stripe 的稳定币战略
⚠️ 2/2 分片 + TEE 架构下，"密钥重组风险"被公开质疑
   （详见 01-privy 的批判性分析一节）
```

### 第二：Fireblocks Embedded Wallet（含 Dynamic）

```
✅ 资本和资产规模最大：$8B 估值，$10T+ 累计交易额
✅ Dynamic 带来 50M+ 用户和成熟的前端/登录层
✅ 机构信任度最高：合规、审计、SLA 是它的主场
✅ MPC-CMP 是行业内被引用最多的 TSS 实现之一

隐忧：
⚠️ 收购整合期，Fireblocks NCW SDK 与 Dynamic 两套栈并存，
   官方甚至提供"从 Fireblocks 迁移到 Dynamic"的文档
⚠️ 定价和销售流程偏企业级，小团队自助上手成本高
⚠️ 主业是机构托管，嵌入式钱包在内部的优先级存疑
```

排在 Privy 之后而不是之前，核心原因是：**嵌入式钱包不是它的主业**，50M+ 是 Dynamic 的历史积累，整合后的真实活跃度没有公开数据。

### 第三：Turnkey

```
✅ 可验证性行业最强：QuorumOS 开源，可远程证明飞地运行的代码
✅ $100B+ 托管资产，说明拿下了高价值客户（交易、支付、机构）
✅ 定价模型对高频场景友好：按签名计费，不绑 MAU
✅ 战略投资方阵容说明问题：Sequoia、Bain Capital Crypto、
   Circle Ventures、Coinbase Ventures、Galaxy
✅ 是其它公司的底层：AllScale 等新银行建在它上面

隐忧：
⚠️ 绝对用户数不如前两家
⚠️ 定位偏"底层密钥管理"，前端 UI/登录层需要自己拼
⚠️ 按签名计费在超高频场景可能比 MAU 模型更贵
```

### 第四但值得单独看：Crossmint

Crossmint 没进前三，不是因为做得差，而是因为它在打一个**结构不同的仗**。

```
其它三家：私钥托管在服务商侧（TEE 或 MPC 分片）
Crossmint：钱包是链上智能合约，签名器可以随时换

这带来一个别人给不了的承诺：
  "换供应商不用换地址"

对企业客户（MoneyGram、Western Union 这个量级）来说，
这个承诺的价值可能高于任何性能指标——
因为它解决的是采购部门最怕的 vendor lock-in。
```

如果按"客户数"排，Crossmint 的 40,000+ 会很好看；如果按"钱包数"排就不占优。它的真实位置是**差异化第一，而非规模第三**。

## 一张图看清各家的位置

```
                 高（机构/合规导向）
                        ▲
                        │
      Fireblocks EW ●   │
                        │   ● Turnkey
                        │
   企业信任度            │
                        │            ● Crossmint
                        │              （可迁移性）
        Privy ●         │
                        │
                        ▼
                 高（消费级/增长导向）

        ◄─────────────────────────────►
        托管在服务商侧      链上可验证/可迁移
        （TEE / MPC）      （合约钱包 + 换签名器）
```

## 应用层三家的位置

AllScale / Infini / Karsa 不在同一个市场，但对照看很有价值——**它们代表了稳定币应用层的三种托管选择**。

| | AllScale | Infini | Karsa |
|---|---|---|---|
| 定位 | 自托管稳定币新银行 | 企业卡 + 资金管理 | 新兴市场美元账户 |
| 托管模型 | 非托管（建在 Turnkey 上） | 托管（资金在 Cobo 托管钱包） | 托管型账户 + 链上结算 |
| 主要客户 | 中小企业、自由职业者、创作者、AI Agent | 跨境企业 | 新兴市场个人与小企业 |
| 核心场景 | 发票、工资单、Checkout | 虚拟卡、支出管理、AI 记账 | 虚拟美国银行账户、跨境收付 |
| 融资 | $6.5M 种子（YZi Labs 领投，Animoca 参投） | 未完整披露 | YC W25 |
| 关键事件 | 2025.05 上线，1.5M+ 钱包 | 2025.02 被盗 $49.5M | 2025 年从 P2P 交易网络起步 |

一个很有意思的对照：

```
Infini 的教训（管理员权限被内部人利用，$49.5M USDC 被盗）
                    ↓
直接强化了 AllScale 这种"自托管 + 策略引擎"路线的说服力

而 AllScale 之所以敢做自托管，
是因为 Turnkey 这类基础设施把自托管的工程成本降到了可承受

结论：基础设施层的进步，改变了应用层的可行策略空间。
```

## 如果只记三件事

1. **前三是 Privy、Fireblocks EW、Turnkey，但"第一"的定义各不相同**——挑供应商别看排名，看你的瓶颈是数量、合规还是可验证性。
2. **2025 年并购潮意味着钱包 SDK 已经不是独立生意**，选供应商等于选生态。
3. **应用层和基础设施层是上下游而不是对手**，AllScale 建在 Turnkey 上就是最直接的证明。

## 小结

这个赛道的真实状态是：**技术趋同，分销分化**。

TEE、MPC、合约钱包三条路线的安全性差距在收窄，能不能赢越来越取决于分销——Privy 靠 Stripe 的商户网络，Fireblocks 靠机构客户名单，Turnkey 靠开发者口碑和策略引擎的深度，Crossmint 靠"不锁定"这个采购层面的说服力。

所以往后看，最值得盯的不是谁的密钥方案更漂亮，而是**谁能把钱包能力塞进已有的支付/银行/企业采购流程里**。这也解释了为什么这一年买家都是支付和托管公司。

## 数据来源

排序和数字来自以下公开来源。带立场的来源已标注。

- [Privy 定价页](https://www.privy.io/pricing)（官方，120M+ 账户 / 2,000+ 团队）
- [Privy 安全架构文档](https://docs.privy.io/security/wallet-infrastructure/architecture)（官方）
- [Turnkey 关于页](https://docs.turnkey.com/get-started/about-turnkey)、[Turnkey 白皮书](https://whitepaper.turnkey.com/)（官方）
- [Turnkey $30M B 轮](https://www.turnkey.com/blog/30m-series-b-to-secure-the-next-era-of-crypto)、[$12.5M 战略投资](https://www.turnkey.com/blog/turnkey-strategic-investment-crypto-verifiable-compute)（官方）
- [CoinDesk：Turnkey 融资报道](https://www.coindesk.com/business/2026/05/06/turnkey-raises-usd12-5-million-in-round-backed-by-circle-ventures-and-sequoia-capital)（第三方）
- [Fireblocks：嵌入式钱包 90 天上线计划](https://www.fireblocks.com/blog/embedded-wallets-90-day-rollout-plan)（官方，Dynamic 50M+ 用户、2025 年 10 月收购）
- [Fireblocks 嵌入式钱包产品页](https://www.fireblocks.com/products/embedded-wallets)（官方）
- [Crossmint 钱包架构文档](https://docs.crossmint.com/wallets/v0/architecture)（官方）
- [Crossmint $23.6M 融资](https://www.crossmint.com/announcement/crossmint-raises-23-6m-led-by-ribbit-capital)（官方）
- [Turnkey × AllScale 案例](https://www.turnkey.com/customers/allscale-cross-border-stablecoin-transactions)（供应商案例，数字为客户提供）
- [Spark：钱包 SDK 格局与并购](https://www.spark.money/tools/crypto-wallet-sdk-landscape)（第三方）
- [Fireblocks 嵌入式钱包对比报告](https://www.fireblocks.com/report/compare-embedded-wallet-infrastructure)（**有立场**：Fireblocks 自己做的 vs Privy/Turnkey 对比）
- [Turnkey vs Privy 定价页](https://www.turnkey.com/vs/privy-pricing-and-licensing)（**有立场**）
- [Crossmint：Privy 替代方案](https://www.crossmint.com/learn/privy-alternatives-for-programmable-wallets)（**有立场**）

内容经改写以符合来源许可要求。
