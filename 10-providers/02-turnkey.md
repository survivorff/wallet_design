# Turnkey 剖析：可验证密钥管理，钱包基础设施里的"AWS"

## 一句话总结

Turnkey 把赌注押在"可验证"三个字上——开源 QuorumOS 让客户能远程证明飞地里跑的是什么代码，再用飞地内的策略引擎把权限控制做成产品，卖的是底层能力而不是钱包 App。

## 基本档案

| 项目 | 信息 |
|---|---|
| 定位 | 开发者钱包基础设施：密钥管理、签名、可编程访问控制 |
| 规模 | 平台托管资产 $100B+，生产环境运行 2+ 年 |
| 融资 | 累计 > $65M：B 轮 $30M（2025.06）+ 战略轮 $12.5M（2026.05） |
| 投资方 | Bain Capital Crypto、Sequoia、Lightspeed Faction、Galaxy Ventures、Variant、Coinbase Ventures、Circle Ventures、Archetype、Wintermute Ventures、Alchemy 等 |
| 核心技术 | 硬件安全飞地（TEE）+ QuorumOS（开源）+ 飞地内策略引擎 |
| 链支持 | 50+ |
| 计费 | 按签名计费，不按 MAU |

## 技术架构

### 核心思路：把"可验证"做成产品

Privy 和 Turnkey 都用 TEE，但对 TEE 的态度不一样。

```
Privy 的逻辑：
  TEE 是实现手段，用来保证私钥只在飞地里重组。
  部署治理（多方评审、硬件密钥）是流程性保证。

Turnkey 的逻辑：
  TEE 的信任根本身要可验证，所以自己造了一个 OS。
  QuorumOS 提供两件事：
    1. 部署应用需要多方参与（不是流程约定，是系统强制）
    2. 密码学证明 TEE 里跑的是预期的软件
  → 客户可以自己验，不需要相信 Turnkey 的内部流程
```

QuorumOS 于 2025 年 2 月开源。这一步的意义是把"我们的流程很严格"变成"你可以自己检查"。Turnkey 自称是**首个此类可验证密钥管理系统**。

### 组织模型：父组织 + 子组织隔离

这是 Turnkey 抽象里最实用的一层设计。

```
        Organization（你的应用）
        ├── Users（带 tag，供策略引用）
        ├── Policies
        ├── Wallets
        │
        ├── Sub-organization #1  ← 一个终端用户 / 一个企业客户
        │   ├── Users
        │   ├── Policies
        │   └── Wallets
        ├── Sub-organization #2
        └── Sub-organization #N

关键点：父组织对子组织只有读权限，不能修改其内容。
```

**为什么这个设计重要**：它在系统层面解决了"平台方能不能动用户资产"这个问题。你作为应用方，创建了子组织给用户，但你改不了子组织里的策略和钱包。这让"非托管"从一个法律声明变成了一个架构事实。

这也是 AllScale 这类新银行敢宣称自托管的技术基础（见 [05-allscale](./05-allscale.md)）。

### 核心概念

| 概念 | 含义 | 实务意义 |
|---|---|---|
| **Organization** | 顶层实体，代表你的应用 | 一个客户一个 |
| **Sub-organization** | 完全隔离的嵌套组织 | 一个终端用户或企业客户一个 |
| **User** | 用有效凭证提交操作的主体 | 可打 tag，策略按 tag 做 RBAC |
| **Root user / Root quorum** | 可绕过策略引擎的特权用户；quorum 设定行使特权所需的批准阈值 | **最关键的配置项**，配错了等于没有策略 |
| **Authenticator** | 用来给 API 请求盖章的凭证：Passkey / API key / 邮箱 OTP / OAuth | 认证方式与钱包解耦 |
| **Activity** | 提交给 Turnkey 的任何动作（签名、建钱包、改策略） | 全部经过策略引擎评估 |
| **Policy** | 求值为 ALLOW / DENY / REQUIRES_CONSENSUS 的规则 | 控制谁能签什么、在什么条件下 |
| **Wallet** | 飞地内的 HD 钱包（种子短语），派生多链账户 | 只返回地址和签名，种子不出飞地 |

### 策略引擎：Turnkey 的真正差异化

关键在于**策略引擎跑在飞地内部，签名产生之前就完成评估**。

```
请求进来
   ↓
凭证验证（Authenticator 盖章）
   ↓
┌──────────── 飞地内部 ────────────┐
│  策略引擎评估 Activity           │
│     ↓                           │
│  ALLOW / DENY / REQUIRES_CONSENSUS│
│     ↓ （仅 ALLOW 继续）           │
│  签名                            │
└─────────────────────────────────┘
   ↓
返回已签名 payload（也可直接广播）
```

对比链上多签，Turnkey 官方强调的优势是**策略语言与链无关**：

```
链上多签：
  要在每条链上部署合约，规则受合约实现限制，
  换链就要重做

Turnkey 策略引擎：
  "需要 2 个以上审批者"这种规则一次定义，
  对所有 50+ 条链生效
  → 一套治理规则，全链适用
```

这对多链的支付/交易平台来说价值很大。

### 工程化功能：解决真实痛点

Turnkey 有几个功能是明显从客户实践里长出来的：

```
Sessions（会话签名）
  问题：用户每笔小额支付都要弹窗签名 → 签名疲劳
  方案：在时间窗和限额内建立会话，多笔操作不重复确认
  实效：AllScale 用它消掉了高频收付款的逐笔签名

Delegated Access（委托访问）
  给后端服务限定范围的签名权限，不交出全部控制权

Claim Links（领取链接）
  向还没有钱包的人转账：生成链接，
  收款方填地址或用邮箱现场开钱包
  实效：AllScale 用它实现了链上退款

Passkeys / OAuth / 邮箱 OTP
  认证层和密钥层解耦，认证方式可以按地区和设备灵活组合
```

最后一点在新兴市场特别关键——Passkey 的支持度在不同手机品牌和系统版本上差异很大，能退回邮箱/Google 登录是刚需。

### 产品线

```
Embedded Wallets        —— 终端用户钱包
Company Wallets         —— 企业自有钱包
Key Management          —— 纯密钥管理 API
Turnkey Verifiable Cloud—— 可验证计算（超出钱包范畴）
```

最后一个值得注意。2026 年 5 月那笔 $12.5M 战略投资的主题就是"crypto verifiable compute"，Circle Ventures 参投。**Turnkey 在把"可验证飞地"这个能力从钱包扩展到通用可验证计算**——这是从"钱包公司"往"可信执行基础设施公司"转型的信号。

### 责任共担模型

Turnkey 明确划分了边界，这一点比很多同行诚实：

| Turnkey 负责 | 客户负责 |
|---|---|
| 飞地基础设施安全 | 配置 root quorum |
| 策略引擎正确性 | 界定用户权限范围 |
| 密钥机密性 | 编写策略 |
| 服务可用性 | 管理凭证 |

**实务提醒**：客户侧的第一条就是 root quorum。root user 能绕过策略引擎，如果你把 root quorum 设成 1 且那个凭证泄露，前面所有策略都是装饰。这是 Turnkey 架构下最容易踩的坑。

## 业务架构

### 定价：按签名，不按用户

| 档位 | 价格 |
|---|---|
| Pay as You Go | $0.10 / 签名（含 25 次免费额度） |
| Pro | $99 / 月起，单价更低 |
| Enterprise | 议价，可低至 ~$0.0015 / 签名，钱包数不限 |

这个模型和 Privy 是镜像关系：

```
Privy：按 MAU 分档（+ 签名和交易额限额）
  → 用户多但不活跃 → 贵
  → 用户少但高频   → 便宜

Turnkey：纯按签名
  → 用户多但不活跃 → 便宜（钱包数不限）
  → 用户少但高频   → 贵

所以选哪家的成本判断，取决于你的
「月签名数 ÷ 月活钱包数」这个比值。
```

粗算一个分界线（数量级估算，实际要按报价谈）：

```
假设 Turnkey 谈到 $0.01/签名，Privy 企业档 $0.05/交易钱包：

  人均月签名 < 5 次  → Turnkey 更便宜
  人均月签名 > 5 次  → Privy 的钱包计费更便宜

典型场景：
  空投/社交类（人均 1-2 次）  → Turnkey
  交易/游戏类（人均 50+ 次）  → Privy 或议价
```

详细的成本模型见 [09-business-model-comparison](./09-business-model-comparison.md)。

### 客群定位

官方列的解决方案方向：

```
DeFi and Trading      —— 高频签名，策略引擎防误操作
AI Agents             —— Agent 需要受限的自主签名权
Payments              —— 跨境支付、汇款
Stablecoins           —— 稳定币钱包基础设施
Consumer Applications —— 消费级应用
Developer Tooling     —— 给其它开发者工具做底层
```

注意 AI Agents 排得很靠前。Agent 钱包的核心需求正好是 Turnkey 的强项：**要让程序自主签名，但必须有硬约束**。策略引擎 + 委托访问天然适配。

### 开发者体验

```
REST API
SDK：React、React Native、Swift、Kotlin、Flutter 等 6 种语言/框架
web3 库适配器（ethers、viem 等）
交易管理工具
Embedded Wallet Kit（现成的 UI 套件）
AI-ready 文档（可直接喂给 Cursor / LLM）
开源：github.com/tkhq，QuorumOS 开源
支持：公开 Slack 社区
```

AllScale 的案例里提到实施是靠共享 Slack 频道、两边工程团队直接对接完成的。**这种高触达支持在早期是加分项，但也说明规模化自助能力还有提升空间**——和 Privy"几行代码接完"的定位有差别。

### 认可度信号

```
2025、2026 连续入选 CNBC 全球顶级 Fintech 公司
投资方含 Circle Ventures、Coinbase Ventures、Galaxy —— 战略而非纯财务
客户覆盖支付到 agentic finance
```

## 批判性分析

### 优势一：可验证性目前没有对手

```
Privy：TEE + 严格的部署治理流程（要相信 Privy 的流程）
Turnkey：TEE + QuorumOS 开源 + 远程证明（可以自己验）

对受监管的机构客户，这个差别是实质性的：
  "我们的供应商流程很严格" vs
  "我们可以密码学地证明供应商在跑什么代码"
第二句能过审计，第一句要靠尽调。
```

### 优势二：子组织隔离让非托管成为架构事实

父组织只读、改不了子组织，这让平台方在技术上就无法挪用用户资产。相比"我们承诺不动用户资金"，这是强得多的保证。

### 局限一：不是全栈，前端要自己拼

```
Turnkey 给你：密钥管理、签名、策略、认证
Turnkey 不给你（或较弱）：
  - 完整的钱包 UI 体验（有 Embedded Wallet Kit 但定位是套件）
  - 出入金 / 发卡 / 收益（对比 Privy 的资金流层）
  - 资产索引和组合展示

所以真实成本是 Turnkey 报价 + 你自己补齐的工程量。
```

### 局限二：EOA/种子模型的迁移问题

Turnkey 的 Wallet 是飞地内的 HD 钱包（种子短语）。和 Privy 一样，地址绑定签名器：

```
换供应商 → 要导出种子再导入别处 → 地址体系要迁移
后量子迁移 → 同样受地址派生方式约束
```

这是 Crossmint 对 TEE 阵营的共同攻击点。Turnkey 可以作为智能合约钱包的签名器层（很多客户就这么用），但那需要客户自己搭合约钱包。

### 局限三：按签名计费的反噬

```
高频场景下成本会失控：
  一个日活交易机器人，每天 1,000 次签名
  = 30,000 次/月
  × $0.01 = $300/月/用户

对比 Privy 的钱包计费，量级完全不同。

Turnkey 的 sessions 功能能减少弹窗，
但减少不了签名次数（会话内每笔仍是一次签名）。
```

超高频客户必须谈到 $0.0015 那一档才划算，这需要足够的议价体量。

### 局限四：$100B+ 托管资产的口径

```
这个数字来自平台整体，含机构客户的大额资产。
不能读成"嵌入式钱包业务规模"。

Turnkey 在嵌入式钱包这一细分的绝对用户量，
公开数据不如 Privy（120M+ 账户）和 Dynamic（50M+ 用户）。
它的强项是单客户价值，不是钱包数量。
```

## 什么情况下该选 Turnkey

```
适合：
  ✅ 需要向审计/监管证明密钥管理的可验证性
  ✅ 需要复杂权限控制（多人审批、限额、时间窗、角色）
  ✅ 多链，且希望治理规则一套通用
  ✅ AI Agent 场景：要自主签名但要硬约束
  ✅ 钱包数量大但签名频次低（钱包不限量，按签名付费）
  ✅ 团队有工程能力，愿意自己拼前端

不适合：
  ❌ 想要"几行代码接完"的全栈方案（选 Privy）
  ❌ 超高频签名且议价能力弱（成本会失控）
  ❌ 需要出入金/发卡/收益这些附加能力开箱即用
  ❌ 需要"换供应商不换地址"（选 Crossmint 路线）
  ❌ 团队没人能正确配置 root quorum 和策略
     （配错的后果比不用策略更危险，因为你会以为自己有保护）
```

## 小结

Turnkey 的定位可以概括成"钱包基础设施里的 AWS"：不做应用，只做底层，把可验证性和可编程性做深，让别人在上面盖楼。AllScale 这样的新银行建在它上面，就是这个定位的证明。

它的战略赌注很清晰——**随着监管和机构进场，"可证明的安全"会比"够好的安全"更值钱**。QuorumOS 开源、远程证明、责任共担模型都是为这个赌注服务的。2026 年那笔以"可验证计算"为主题的战略投资，说明它准备把这个能力卖到钱包之外。

风险在另一侧：如果市场最终证明大多数客户只要"够好"，那么 Privy 的全栈易用性和 Stripe 的分销会碾过可验证性的溢价。Turnkey 的护城河深，但护的是一个还没被验证一定会变大的市场。

## 数据来源

- [Turnkey 关于页 / 核心概念](https://docs.turnkey.com/get-started/about-turnkey)（官方）
- [Turnkey 白皮书](https://whitepaper.turnkey.com/) 与[应用章节](https://whitepaper.turnkey.com/applications)（官方）
- [Turnkey 安全方法](https://docs.turnkey.com/security/our-approach)、[安全飞地](https://docs.turnkey.com/security/secure-enclaves)（官方）
- [QuorumOS 开源公告](https://www.turnkey.com/blog/quorumos-is-now-open-source)（官方）
- [Turnkey 策略引擎：Web3 交易护栏](https://www.turnkey.com/blog/turnkey-policy-engine-guardrails-web3-transactions)（官方）
- [Turnkey 定价页](https://www.turnkey.com/pricing)（官方）
- [Turnkey $30M B 轮](https://www.turnkey.com/blog/30m-series-b-to-secure-the-next-era-of-crypto)（官方）
- [Turnkey $12.5M 战略投资](https://www.turnkey.com/blog/turnkey-strategic-investment-crypto-verifiable-compute)（官方）
- [Turnkey × AllScale 案例](https://www.turnkey.com/customers/allscale-cross-border-stablecoin-transactions)（官方案例）
- [Turnkey 开发者工具](https://www.turnkey.com/blog/developer-tooling-crypto-wallets-key-management)（官方）
- [Turnkey 入选 CNBC 全球顶级 Fintech](https://www.turnkey.com/blog/turnkey-2026-cnbc-worlds-top-fintech)（官方）
- [CoinDesk：Turnkey 融资报道](https://www.coindesk.com/business/2026/05/06/turnkey-raises-usd12-5-million-in-round-backed-by-circle-ventures-and-sequoia-capital)（第三方）
- [FintechFutures：Turnkey B 轮报道](https://www.fintechfutures.com/venture-capital-funding/turnkey-raises-30m-series-b)（第三方）
- [Sacra：Turnkey 融资与分析](https://sacra.com/c/turnkey/)（第三方）
- [Turnkey vs Privy 定价对比](https://www.turnkey.com/vs/privy-pricing-and-licensing)（**有立场**，Turnkey 自制）
- [apis.io：Turnkey 定价档位](https://apis.io/plans/turnkey/turnkey-plans-pricing/)（第三方汇总）

注：B 轮领投方在不同来源中记载不一致（FintechFutures 记为 Bain Capital Crypto 领投，Sacra 记为 Sequoia 领投），此处并列两家。内容经改写以符合来源许可要求。
