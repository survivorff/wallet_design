# 技术架构横向对比：TEE、MPC、合约钱包三条路线的真实取舍

## 一句话总结

四家基础设施厂商其实在回答两个不同的问题——Privy/Turnkey/Fireblocks 在比"私钥怎么保管得更安全"，Crossmint 在说"钱包根本不该等于私钥"，选谁的关键不是哪家密码学更漂亮，而是你能接受哪一种信任假设。

## 一张表看清四家的架构

| | **Privy** | **Turnkey** | **Fireblocks EW** | **Crossmint** |
|---|---|---|---|---|
| **钱包本体** | EOA（BIP-39 HD） | EOA（飞地内 HD 种子） | EOA（MPC 密钥） | 智能合约钱包 |
| **私钥形态** | 加密分片，签名时在 TEE 内临时重组 | 种子常驻飞地，不出飞地 | 从不完整存在，各方持 share | 无单一私钥，签名器可换 |
| **安全边界** | AWS Nitro Enclaves + 2-of-2 Shamir | 安全飞地 + QuorumOS 远程证明 | MPC-CMP（TSS）数学保证 | 链上合约 + 设备硬件飞地 / 云 HSM |
| **签名在哪发生** | 云端 TEE | 云端飞地 | 用户设备 + 服务商多轮通信 | 用户设备（Secure Enclave / Keystore）或云 KMS |
| **签名延迟** | < 20ms（多区域优化后） | 需一次网络往返 | 需多轮通信（MPC-CMP 已提速约 8 倍） | 设备端无网络往返 |
| **策略执行位置** | TEE 内（钱包策略） | **飞地内策略引擎**（ALLOW/DENY/REQUIRES_CONSENSUS） | 平台侧 | **链上合约**（可审计） |
| **可验证性** | Attestation + 部署治理流程 | **QuorumOS 开源 + 远程证明** | MPC 数学性质 | **合约代码链上公开** |
| **换供应商** | 导出私钥，地址体系要迁移 | 导出种子，同上 | 导出/重建 shares | **换签名器，地址不变** |
| **服务商消失** | 需 Privy 服务在线才能导出 | 需 Turnkey 服务在线 | 企业可重建服务商侧 shares | **合约仍在链上，可直接操作** |
| **单方能否重组私钥** | 理论可能（两片都在服务商域内，靠代码+流程阻止） | 靠飞地约束 + 子组织隔离阻止 | **数学上不可能** | 无完整私钥可重组 |
| **Gas 成本** | EOA，最低 | EOA，最低 | EOA，最低 | 合约钱包，UserOperation 开销 |
| **链支持速度** | 快（只需适配签名算法） | 快，50+ 链 | 快，全 EVM/SVM + BTC/Sui/TON | 慢（每链要合约实现） |
| **新增风险面** | TEE 部署流程 | root quorum 配置错误 | 集成方的 share 保管方案 | 合约 bug + ERC-7579 模块组合 |

## 三条路线的本质差别

### 路线一：云 TEE（Privy、Turnkey）

```
核心假设：
  硬件飞地是可信的，飞地里的代码是受约束的

Privy 的具体做法：
  私钥拆成 Enclave Share + Auth Share（2-of-2 Shamir）
  签名时在 TEE 内存里临时拼回完整私钥，用完销毁

Turnkey 的具体做法：
  HD 种子常驻飞地，从不出来
  策略引擎也跑在飞地内，签名前先评估

共同的信任链：
  你相信 AWS Nitro（或同类）的硬件隔离
  + 你相信飞地里跑的是声称的代码
  + 你相信那份代码是善意的

Privy 用"严格的部署治理"保证第三条
Turnkey 用"QuorumOS 开源 + 远程证明"让你自己验第三条
  ← 这是两家最实质的差别
```

**这条路线的真正攻击面不是飞地ï¼是部署流程。** TEE 能证明"跑的是代码 X",证明不了"代码 X 没有后门"。所以谁能让客户自己验证代码 X 是什么ï¼谁的可验证性就更强。

### 路线二：MPC / TSS（Fireblocks）

```
核心假设：
  不需要相信任何单一方，因为完整私钥从不存在

做法：
  密钥生成时就是分布式的，各方各持 share
  签名通过多轮通信协作产出
  用户 share 存在设备硬件飞地 / 生物识别保护下

优势（唯一性的）：
  数学上单方无法重组私钥
  → 不需要相信 Fireblocks 的代码或流程

代价：
  1. 工程复杂度高（多轮通信、Device ID 管理、多设备同步）
  2. 历史上慢（MPC-CMP 提速约 8 倍后才能做消费级）
  3. 用户 share 的保管方案要集成方自己设计
  4. 强制备份流程 → Onboarding 摩擦
```

**这条路线的取舍很清晰：用工程复杂度和 Onboarding 摩擦，换掉对服务商的信任依赖。**

### 路线三：合约钱包 + 可换签名器（Crossmint）

```
核心思路：
  换个命题。钱包不该是私钥，钱包该是链上的一个合约。
  私钥（签名器）只是"能授权这个合约的钥匙之一"。

做法：
  EVM: ERC-4337 + ERC-7579 模块化合约钱包
  Solana: PDA
  Stellar: Soroban
  签名器：设备硬件飞地 / Passkey / 外部钱包 /
         AWS KMS / Azure KV / GCP HSM / 邮箱手机社交（恢复）

带来的独特能力：
  ✅ 换签名器地址不变 → 无供应商锁定
  ✅ 服务商消失也能用（合约在链上）
  ✅ 权限规则链上可审计（不是服务商侧策略引擎）
  ✅ 设备端签名无网络往返
  ✅ 后量子迁移不用换地址

代价：
  ⚠️ Gas 成本（UserOperation + Paymaster）
  ⚠️ 部署成本（每个钱包是一个合约）
  ⚠️ 每条链都要有合约实现 → 新链慢
  ⚠️ 合约 bug 风险 + 模块组合风险（EOA 没有这一层）
  ⚠️ 部分协议对合约钱包的兼容性仍有坑
```

## 关键维度深入

### 维度一："谁能动我的钱"的真实答案

这是最该问清的一个问题ï¼四家的答案实质不同。

```
Privy：
  两片（Enclave Share + Auth Share）都在 Privy 控制域内。
  阻止 Privy 重组的是 TEE 代码约束 + 部署治理流程。
  → 性质：流程性不可能，不是数学性不可能

Turnkey：
  种子在飞地内，客户是子组织，父组织（应用方）只读。
  阻止 Turnkey 的是飞地约束 + 远程证明。
  阻止应用方的是子组织隔离（这条是架构事实）。
  → 对应用方：数学/架构性不可能
  → 对 Turnkey 本身：可验证的流程性不可能

Fireblocks：
  真 MPC，用户 share 在用户设备。
  → 数学性不可能

Crossmint：
  合约钱包，签名器集合链上可查。
  Crossmint 是否在签名器集合里，链上可验。
  → 可公开验证
```

**实务建议**：如果你要向监管或审计解释这件事ï¼答案的"可验证性"比"安全性"更重要。Fireblocks 和 Crossmint 的答案最容易说清ï¼Turnkey 次之(有远程证明),Privy 需要解释流程。

### 维度二：延迟

```
签名延迟从低到高：

  Crossmint（设备端签名）
    → 无网络往返，理论最低
    但：合约钱包的 UserOperation 有链上额外开销

  Privy（云 TEE，多区域）
    → < 20ms，一次网络往返

  Turnkey（云飞地）
    → 一次网络往返 + 策略引擎评估

  Fireblocks（MPC）
    → 多轮通信，MPC-CMP 提速后可用但仍最高
```

对高频场景(交易终端、游戏、Agent),这个排序有实际影响。但注意 Crossmint 省的是签名延迟ï¼链上确认时间四家都一样。

### 维度三：策略与权限控制

这是四家产品化程度差异最大的一块。

| | 策略执行位置 | 表达能力 | 可审计性 |
|---|---|---|---|
| Privy | TEE 内钱包策略 | 中 | 需信任 Privy |
| **Turnkey** | **飞地内策略引擎** | **强**：ALLOW / DENY / REQUIRES_CONSENSUS，链无关语言，按 user tag 做 RBAC | 远程证明 |
| Fireblocks | 平台侧 | 强（机构级） | 审计报告 |
| **Crossmint** | **链上合约** | 中强：多签名器 + scoped permissions | **链上公开可验** |

Turnkey 的一个独特优势值得强调：

```
策略语言与链无关。

"需要 2 个以上审批者"这条规则：
  链上多签 → 每条链部署合约，规则受合约实现限制
  Turnkey  → 一次定义，50+ 链全部生效

对多链支付/交易平台，这是实质性的运维简化。
```

### 维度四：迁移与退出成本

这是 Crossmint 唯一的、也是最强的差异点。

```
假设三年后你要换供应商，四种情况：

Privy / Turnkey（EOA）：
  1. 导出私钥/种子（需要供应商服务在线）
  2. 导入新供应商
  3. 通知所有用户地址体系变更
  4. 更新所有集成方、白名单、审计记录
  5. 处理用户没来迁移的长尾
  → 对 150 万钱包量级，这是一个季度级的项目

Fireblocks（MPC）：
  可重建服务商侧 shares（业务连续性有保障）
  但换到别家仍要迁移

Crossmint（合约钱包）：
  更新 recovery signers
  → 地址、余额、交易历史全部保留
  → 用户完全无感
```

**这个差别在决策时经常被低估ï¼因为它的成本在三年后才发生。** 但对企业采购和受监管机构来说ï¼它恰好是尽调清单上的必答题——这也解释了为什么 Crossmint 能拿下 MoneyGram、Western Union 这个量级的客户。

### 维度五：新增的风险面

每条路线都引入了自己独有的失败模式。选型时要问"我的团队能不能管住这个新风险"。

```
Privy 的新风险：TEE 部署流程被攻破
  → 你管不了（这是供应商侧）
  → 缓解：看它的审计报告和治理披露

Turnkey 的新风险：root quorum 配置错误
  → 你要管（这是客户侧责任）
  → root user 能绕过策略引擎
  → 如果 root quorum = 1 且凭证泄露，所有策略是装饰
  → 这是 Turnkey 架构下最容易踩的坑

Fireblocks 的新风险：用户 share 保管方案设计不当
  → 你要管
  → 存哪、多设备怎么同步、备份 UX 怎么做，都是你的题

Crossmint 的新风险：合约 bug + ERC-7579 模块组合
  → 部分你要管（选哪些模块）
  → 审计范围从"一个合约"变成"合约 × 模块组合"
```

**注意 Turnkey 和 Fireblocks 的风险都在客户侧。** 这意味着它们的安全性上限更高ï¼但实际安全水平取决于你的团队。Privy 把更多东西替你决定了ï¼所以你踩坑的机会少ï¼但你也验证不了它。

## 决策树

```
你的首要约束是什么？

├─ 上线速度 / Onboarding 转化率
│   → Privy
│     （全栈、几行代码、出入金发卡都有）
│
├─ 要向审计/监管证明密钥管理
│   ├─ 需要"数学上不可重组" → Fireblocks EW
│   └─ 需要"可自行验证代码"   → Turnkey
│
├─ vendor lock-in 是硬否决项
│   → Crossmint
│     （唯一能做到换供应商不换地址）
│
├─ 需要复杂权限控制（多人审批、限额、角色）
│   ├─ 规则要跨链统一、链下执行 → Turnkey
│   └─ 规则要链上公开可验       → Crossmint
│
├─ 高频签名，成本敏感
│   ├─ 钱包多但签名少 → Turnkey（钱包不限量）
│   └─ 签名极多       → Privy 企业档议价
│     （详见 09-business-model-comparison）
│
├─ 未来会需要机构托管/清算/做市
│   → Fireblocks（同一体系内扩展）
│
└─ AI Agent 场景（要自主签名 + 硬约束）
    → Turnkey（策略引擎 + 委托访问最贴合）
    或 Crossmint（链上权限 + Agent 钱包产品线）
```

## 一个容易被忽略的结论：可以混用

实践中最好的方案常常不是选一家。

```
常见组合一：Turnkey 作为合约钱包的签名器
  合约钱包（自建或用现成实现）
    ← 签名器 = Turnkey
  → 拿到 Turnkey 的策略引擎 + 合约钱包的可迁移性

  Crossmint 自己也承认这种组合方式，
  它把 Turnkey / Privy / Alchemy 描述为
  "可以组合进智能合约钱包方案的签名器层"

常见组合二：分层用不同供应商
  终端用户钱包 → Privy（体验优先）
  企业资金钱包 → Fireblocks 或 Crossmint + KMS（安全优先）

常见组合三：多供应商防锁定
  主用一家，保留第二家的接入能力
  → 成本高，但对大规模业务是必要的保险
```

**如果你的业务规模够大ï¼单一供应商本身就是风险。** AllScale 对 Turnkey 的深度依赖(见 [05-allscale](./05-allscale.md))就是一个现实的例子。

## 小结

把四家的架构摊开看ï¼会发现一个规律：**安全性的差距在收窄ï¼可验证性和可迁移性的差距在拉大。**

TEE、MPC、合约钱包三条路线现在都能做到"服务商不能随便动用户的钱"。真正区分它们的是两个更难的问题：**你能不能自己验证这一点**(Turnkey 的 QuorumOS、Crossmint 的链上合约、Fireblocks 的数学性质),以及**三年后你能不能带着地址走**(只有 Crossmint 能)。

所以选型时最有价值的问题不是"哪家更安全",而是这三个：

1. 我需要向谁证明安全性ï¼那个人接受什么形式的证明？
2. 这个架构新增的风险面在谁那边ï¼我的团队管得住吗？
3. 三年后我要换供应商ï¼成本是一个季度的项目还是一次配置更新？

技术选型的成本大部分不在接入那一天ï¼在这三个问题的答案上。

## 延伸阅读

本章之外，架构层面的基础知识在这些篇目：

- [02-architecture/04-mpc-tss-technical](../02-architecture/04-mpc-tss-technical.md) — MPC/TSS 的密码学原理
- [02-architecture/05-account-abstraction-erc4337](../02-architecture/05-account-abstraction-erc4337.md) — ERC-4337 机制
- [02-architecture/06-smart-contract-wallet-implementation](../02-architecture/06-smart-contract-wallet-implementation.md) — 合约钱包实现与兼容性坑
- [02-architecture/11-self-custody-spectrum](../02-architecture/11-self-custody-spectrum.md) — 自托管谱系与 TEE 托管深度分析
- [02-architecture/13-modular-accounts-and-new-paradigms](../02-architecture/13-modular-accounts-and-new-paradigms.md) — ERC-7579 / 6900 模块化账户

## 数据来源

架构信息全部来自各厂商官方文档，具体链接见各厂商单篇的数据来源一节：

- [Privy 安全架构](https://docs.privy.io/security/wallet-infrastructure/architecture)
- [Turnkey 核心概念](https://docs.turnkey.com/get-started/about-turnkey)、[安全飞地](https://docs.turnkey.com/security/secure-enclaves)、[QuorumOS 开源](https://www.turnkey.com/blog/quorumos-is-now-open-source)
- [Fireblocks MPC 密钥生成](https://developers.fireblocks.com/docs/embedded-wallet-mpc-key-generation)、[MPC-CMP](https://www.fireblocks.com/blog/pushing-mpc-wallet-signing-speeds-8x-with-mpc-cmp-9)
- [Crossmint 钱包架构](https://docs.crossmint.com/wallets/v0/architecture)

对比结论为本知识库基于上述官方文档的分析，非厂商观点。带立场的厂商对比材料（Fireblocks 对比报告、Turnkey vs Privy 页面、Crossmint alternatives 系列）已在各单篇中标注。内容经改写以符合来源许可要求。
