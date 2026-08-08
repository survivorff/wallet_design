# Privy 剖析：TEE + 分片，规模最大的嵌入式钱包基础设施

## 一句话总结

Privy 用"TEE 里临时重组私钥"这一招同时拿到了体验和安全的中间态，做成了 120M+ 账户的规模，然后被 Stripe 收购，成了全球最大支付网络的链上入口。

## 基本档案

| 项目 | 信息 |
|---|---|
| 定位 | 嵌入式钱包 / 可编程钱包基础设施 |
| 归属 | Stripe（2025 年 6 月宣布收购）；法律实体 Horkos, Inc. |
| 规模 | 120M+ 账户，2,000+ 开发者团队 |
| 吞吐 | 115M+ 月签名量，签名延迟 < 20ms |
| 核心技术 | AWS Nitro Enclaves（TEE）+ Shamir 秘密共享 2-of-2 |
| 账户模型 | EOA 为主（BIP-39 / HD 钱包），可叠加 ERC-4337 |
| 计费 | MAU 分档 + 签名量 + 交易额三重限额；企业签名价可低至 $0.001 |

## 技术架构

### 核心思路：不做 MPC，做"TEE 内临时重组"

这是 Privy 最关键的架构选择，也是它和 Turnkey、Fireblocks 的分界线。

```
MPC / TSS 路线（Fireblocks、早期 Web3Auth）：
  私钥从来不完整存在，各方各持分片，
  通过多轮通信协作产出签名
  → 安全模型最强，但每次签名要跑协议，工程复杂

Privy 路线：
  私钥被拆成加密分片存两个地方，
  签名时在 TEE 内存里临时拼回完整私钥，用完即弃
  → 简单、快、跨链适配容易
  → 代价：存在一个"私钥完整存在过"的瞬间
```

### 密钥分片：2-of-2 结构

```
钱包创建时（全程在 TEE 内完成）：

  TEE 内的 CSPRNG 生成 128 位熵
        ↓
  BIP-39 转成助记词
        ↓
  派生公钥 / 私钥（HD 钱包）
        ↓
  用 Shamir 秘密共享拆成 2 片
        ↓
  ┌─────────────────┐   ┌─────────────────┐
  │  Enclave Share  │   │   Auth Share    │
  │  （TEE 分片）    │   │  （认证分片）    │
  ├─────────────────┤   ├─────────────────┤
  │ 用 TEE 的密钥加密 │   │ 加密后存在 Privy │
  │ 只能在 TEE 内解密 │   │ 需要有效凭证才能 │
  │                 │   │ 取出（bearer    │
  │                 │   │ token / secret）│
  └─────────────────┘   └─────────────────┘

  两片都拿到才能重组。任一片单独存在都不泄露任何信息。
```

Privy 用的 Shamir 实现是[开源的](https://github.com/privy-io/shamir-secret-sharing)，这一点值得肯定——密码学核心组件开源可审计，是行业里应该被标准化的做法。

### 签名流程

```
1. 你的服务端 POST 到 Privy API
   携带：API 凭证（bearer token 或 app secret）
        + authorization key 签名

2. Privy API 验证凭证
   通过 → 把请求和 Auth Share 一起转发进 TEE

3. TEE 内校验授权签名和钱包策略
   （authorization 公钥验签 + policy 检查）

4. TEE 解密 Enclave Share，与 Auth Share 合并
   → 内存中重组出完整私钥

5. 签名

6. 私钥随即销毁，只返回签名
   （Privy 也可以直接帮你广播上链）
```

### TEE 到底提供了什么保证

Privy 用的是 AWS Nitro Enclaves，关键特性有三条：

```
无持久化存储  → 飞地里的东西不会落盘
无交互式访问  → 没有 SSH，运维进不去
无网络连接    → 数据出不去

加上 Attestation（远程证明）：
  飞地里跑的代码哈希被签名，
  外部可以用对应公钥验证"跑的确实是这份代码"
```

### 代码部署的治理控制

这一层常被忽略，但对 TEE 架构来说是真正的信任根。Privy 披露的控制手段：

```
- 所有进 TEE 的代码需要多个指定 owner 评审
- 需要硬件安全密钥（hardware security key）
- 自动化安全测试必须通过
- 分阶段部署，每阶段额外审批
- CI/CD 只从受保护分支构建，有分支保护、安全扫描、签名要求
- 流程本身定期审计
```

**为什么这很重要**：TEE 能证明"跑的是代码 X"，但不能证明"代码 X 是善意的"。所以对 TEE 架构的钱包，真正的攻击面是**部署流程**，不是飞地本身。Privy 把这套流程写进文档是负责的做法。

### 从设备端迁移到 TEE

Privy 早期是"用户设备存一片 + 服务商存一片"的模型。现在提供了迁移路径：

```
开启 TEE execution 后：
  新钱包 → 直接在 TEE 内创建
  老钱包 → 用户下次登录时自动迁移到 TEE

注意：这是单向变更，不可回退
```

架构上这是一次重要转向：**从"用户设备是一个安全边界"变成"两个边界都在服务商这侧"**。体验上更好（换设备无痛），去中心化程度上更弱。这个取舍值得留意。

### 多区域部署

Privy 后来做了多区域钱包基础设施，全球用户签名延迟下降约 50–75%。这解释了 <20ms 的签名时间从哪来——对交易类和游戏类应用，这个指标直接影响可用性。

## 业务架构

### 产品线：早就不只是钱包 SDK

看 Privy 现在的产品目录，能清楚看到它在往哪走：

```
钱包层
  User wallets      —— 终端用户嵌入式钱包（起家业务）
  Agent wallets     —— AI Agent 钱包
  Treasury wallets  —— 企业资金钱包
  Custodial wallets —— 托管钱包
  Digital asset accounts

安全与控制层
  Key management    —— 底层密钥管理 API
  Policy engine     —— 可编程规则、审批、转账控制
  Roles & permissions

资金流层（← 这才是变现重心）
  Cards and spend   —— 卡计划直连钱包余额
  Funding           —— 法币/数字资产出入金
  Treasury management
  Yield integrations —— 链上收益
```

**判断**：Privy 的钱包本身是获客工具，真正的收入曲线在"资金流层"——出入金、发卡、收益分成。这和 Stripe 的收购逻辑完全吻合。

### 目标客群

官方划分的方向很说明问题：

| 类别 | 场景 |
|---|---|
| 机构 | 银行、资产管理 |
| Fintech | 支付、新银行、工资单、汇款 |
| 交易 | DeFi、交易所、预测市场 |
| 消费级 | 游戏、社交与创作者平台、市场 |
| AI | Agent 与自主系统 |

注意"银行"和"新银行"排在很前面。**Privy 已经不把自己当 Web3 工具，而是当金融基础设施。**

### 定价

| 档位 | MAU 区间 | 价格 |
|---|---|---|
| Free | 0 – 499 | $0 |
| Core | 500 – 2,499 | $299 / 月 |
| Scale | 2,500 – 9,999 | $499 / 月 |
| Enterprise | 定制 | 按交易或按交易钱包计价，签名可低至 $0.001 |

所有 Developer 档共享的免费额度：

```
50K 月签名量  +  $1M 月交易额

超过 10K MAU、50K 签名或 $1M 交易额 → 转为用量计费
```

这个定价结构有个值得注意的设计：**三个维度同时设限**（MAU、签名数、交易额）。意思是你可以用户少但交易大，也可以用户多但交易小，两种都会触发升级。对客户来说预测成本更难，对 Privy 来说收入捕获更充分。

### Stripe 收购的战略意义

```
Stripe 的链上栈拼图：

  前端入口 ──── Privy（钱包、账户、签名）
  清算结算 ──── Bridge（稳定币发行与转移）
  底层网络 ──── Tempo（与 Paradigm 共同孵化的稳定币 L1）
  商户网络 ──── Stripe 自身

结论：Stripe 可以端到端提供"美元在链上流动"的全栈服务，
而 Privy 是用户实际触碰到的那一层。
```

对开发者的含义：

```
好处：
  ✅ 供应商倒闭风险几乎为零
  ✅ 出入金、发卡、合规能力被 Stripe 补齐
  ✅ 可能吃到 Stripe 商户网络的分销

风险：
  ⚠️ 产品路线服从 Stripe 战略，不再中立
  ⚠️ 如果你是 Stripe 的竞争对手（其它支付公司），
     用 Privy 等于把用户数据交给对手
  ⚠️ 可能被推向 Tempo 等 Stripe 系资产/链
```

## 批判性分析

### 争议一：2-of-2 + 两片都在服务商侧 = 重组风险

这是当前对 Privy 最实质的质疑。

```
事实链条：
  1. Enclave Share 由 Privy 的 TEE 持有
  2. Auth Share 由 Privy 加密存储
  3. 两片合起来 = 完整私钥
  4. 两片都在 Privy 的控制域内

阻止 Privy 重组用户私钥的，不是密码学，
而是 TEE 的代码约束 + 部署治理流程。

也就是说：这是"流程性不可能"，不是"数学性不可能"。
```

对照 MPC 路线：真正的 MPC 里私钥从未完整存在过，重组在数学上就不成立。Privy 的模型下，如果攻击者同时拿到 TEE 代码部署权限和 Auth Share 存储，理论上是可以重组的。

Privy 的应对是把部署流程做到多方审批 + 硬件密钥 + 审计。这在工程上是合理的，但**性质上是信任假设的转移，不是消除**。

在 [02-architecture/11-self-custody-spectrum](../02-architecture/11-self-custody-spectrum.md) 的谱系里，Privy 现在的位置比它早期（设备端持片）更靠向托管一侧。

### 争议二：EOA 路线的天花板

Privy 的核心是 EOA，签名器和地址是绑定的。带来两个约束：

```
1. 换供应商就要换地址
   要迁移，得导出私钥再导入别处——
   资产、历史、地址全部要处理

2. 后量子迁移麻烦
   EOA 地址由公钥派生，换密码学算法就得换地址
```

这正是 Crossmint 攻击 Privy 的点（见 [04-crossmint](./04-crossmint.md)）。Privy 可以在 EOA 上叠 ERC-4337，但那是加一层，不是改地基。

### 争议三：120M 账户这个数字该怎么读

```
"120M+ 账户"是累计创建数，不是月活。

拆解一下：
  - 一个人用同一邮箱登录 10 个 Privy 客户的 App
    → 可能算 10 个账户（取决于客户是否共享 app id）
  - 空投猎人批量创建
  - 大量长期零余额账户

更可信的活跃度指标是 115M+ 月签名量。
按每个活跃钱包月均 10 次签名粗估，
量级大概是千万级月活钱包——依然是行业第一，
但和 120M 是两个数量级的概念。
```

引用这些数字时最好同时给口径，否则容易做出错误的市场规模推算。

## 什么情况下该选 Privy

```
适合：
  ✅ 消费级应用，Onboarding 转化率是核心指标
  ✅ 需要快速上线，团队没有密钥管理专家
  ✅ 需要出入金 / 发卡 / 收益这些"钱包之外"的能力
  ✅ 已经在 Stripe 生态里（或愿意进）
  ✅ 用户量大但单用户交易频次不高（MAU 计费更划算）

不适合：
  ❌ 你是支付公司，和 Stripe 直接竞争
  ❌ 需要"数学上不可重组"的托管保证（选真 MPC）
  ❌ 高频签名场景（MAU 分档 + 签名限额组合可能很贵，
     对比 Turnkey 的纯签名计费）
  ❌ 需要"换供应商不换地址"（选合约钱包路线）
  ❌ 需要开源自建或私有部署
```

## 小结

Privy 的成功来自一个务实的判断：**大多数应用需要的不是最强的密码学保证，而是最低的集成摩擦加上"够好"的安全**。TEE + 2-of-2 分片正好落在那个点上，再配上开源的分片库和严格的部署治理，把"够好"讲成了一个可审计的故事。

被 Stripe 收购是这个策略的自然终点。钱包 SDK 单卖的 ARPU 撑不起大公司，但作为全球最大支付网络的链上入口，它的战略价值远超收入本身。

真正的开放问题是：当 Privy 的产品路线开始服从 Stripe 的稳定币战略，那些不想进 Stripe 生态的开发者会不会开始迁移——以及在 EOA 架构下，迁移的成本会有多痛。

## 数据来源

- [Privy 安全架构文档](https://docs.privy.io/security/wallet-infrastructure/architecture)（官方）
- [Privy 定价页](https://www.privy.io/pricing)（官方）
- [Privy：嵌入式钱包如何工作](https://www.privy.io/blog/how-privy-embedded-wallets-work)（官方）
- [Privy：低层密钥管理驱动可编程钱包](https://blog.privy.io/blog/powering-programmable-wallets-with-low-level-key-management)（官方）
- [Privy：从设备端迁移到 TEE](https://docs.privy.io/recipes/tee-wallet-migration-guide)（官方）
- [Privy：多区域部署降低延迟](https://blog.privy.io/blog/lowering-latency-with-multi-region-wallet-infrastructure)（官方）
- [Privy：宣布被 Stripe 收购](https://privy.io/blog/announcing-our-acquisition-by-stripe)（官方）
- [privy-io/shamir-secret-sharing](https://github.com/privy-io/shamir-secret-sharing)（开源库）
- [Allium：Privy 如何支撑 1.2 亿钱包的链上上下文](https://www.allium.so/blog/how-privy-powers-onchain-context-for-120-million-wallets/)（第三方）
- [AInvest：Privy 的 1.2 亿钱包重组风险](https://www.ainvest.com/news/privy-120m-wallet-reconstitution-risk-shows-embedded-crypto-power-sits-2607/)（第三方，批评视角）
- [PANews：Stripe 收购 Privy 的战略分析](https://www.panewslab.com/en/articles/9n42sapi)（第三方）
- [Crossmint：Privy vs Crossmint](https://crossmint.com/learn/privy-vs-crossmint)（**有立场**，竞品对比）

内容经改写以符合来源许可要求。
