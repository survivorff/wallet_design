# Fireblocks Embedded Wallet 剖析：机构托管巨头下沉消费级

## 一句话总结

Fireblocks 用 MPC-CMP 这套机构级 TSS 实现，加上 2025 年收购的 Dynamic 作为前端和登录层，把"托管 $10T+ 交易额的机构基础设施"打包成嵌入式钱包卖给消费级应用——技术和资本都最厚，代价是两套技术栈的整合还没走完。

## 基本档案

| 项目 | 信息 |
|---|---|
| 定位 | 嵌入式钱包（前身 Non-Custodial Wallet / NCW），机构级数字资产基础设施的下沉产品 |
| 母公司规模 | 2,000+ 企业客户，累计保护 $10T+ 数字资产交易额 |
| 估值 | $8B（2025 年 10 月 E 轮 $550M），自称估值最高的数字资产基础设施公司 |
| 关键并购 | 2025 年 10 月收购 **Dynamic**；Dynamic 累计 onboard 50M+ 用户 |
| 核心技术 | MPC-CMP（TSS），签名速度比前代快约 8 倍，支持离线/冷钱包签名 |
| 链支持 | 全部 EVM 网络、全部 SVM 网络、Bitcoin、Sui、TON 等（Dynamic 口径） |
| 可用性 | API / 控制台 / 交易引擎 / 节点月度可用率 99.97%+ |

## 技术架构

### 核心思路：真 MPC，私钥从不完整存在

这是 Fireblocks 和 Privy / Turnkey 的根本区别。

```
TEE 路线（Privy、Turnkey）：
  私钥或种子在飞地内是完整存在的（Privy 是临时重组，
  Turnkey 是种子常驻飞地）
  安全边界 = 硬件飞地 + 代码约束

MPC / TSS 路线（Fireblocks）：
  私钥从生成起就没有完整形态，
  各方各持一个 share，通过多轮通信协作产出签名
  安全边界 = 数学
```

Fireblocks 自己的说法很直白：完整密钥在任何时刻都不会在单一位置被组装出来，所以单个被攻破的设备、内部人或攻击者都拿不到资金。

**这是嵌入式钱包场景下 Fireblocks 最硬的卖点**，也是它对 Privy"2-of-2 可重组"质疑的天然回应。

### MPC-CMP：让 MPC 快到能用

MPC 一直有个工程问题：签名要跑多轮通信，慢。Fireblocks 的 MPC-CMP 解决的就是这个：

```
效果：签名速度比前代提升约 800%
额外能力：支持离线 / 冷钱包签名

意义：
  在 MPC-CMP 之前，MPC 钱包的签名延迟让消费级体验很难做
  之后，MPC 才具备和 TEE 路线竞争体验的资格
```

### 密钥生成与存储

```
MPC 密钥生成流程：
  用户设备 ←──多轮通信──→ Fireblocks
  逐步协作生成各自的 key share

用户 share 的存储可以自选：
  - 移动端硬件飞地（iOS Secure Enclave / Android Keystore）
  - 生物识别保护
  - 2FA
  - 其它自定义方案

Device ID：
  代表"持有用户 key share 且能参与 MPC 运算"的逻辑标识
  每个 SDK 实例一个唯一 ID
  → 多设备就是多 Device ID
```

关键点：**用户 share 的保管方式交给集成方决定**。这带来灵活性，也带来责任——用户 share 存哪、怎么备份，是集成方的设计题。

### 备份与灾难恢复

Fireblocks 在这一层有两个机制，都体现了机构级基因：

```
1. 强制备份
   除非 CSM 和支持团队特批，
   每个钱包必须先完成备份流程才能接收资产
   → 强制而非建议，把"用户没备份就丢币"的场景堵死

2. 灾难恢复（Disaster Recovery）
   允许企业重新生成 Fireblocks 侧管理的 shares
   → 保证业务连续性：即使 Fireblocks 侧出问题，
     企业也能恢复运营
```

Fireblocks 把这一点称为"真正独立的恢复，不依赖单一供应商"。对合规和业务连续性审查来说，这是必答题。

### Dynamic：补上前端和登录层

```
Fireblocks 缺什么：
  ❌ 消费级登录体验（邮箱、社交、Passkey）
  ❌ 前端 SDK 的开发者体验
  ❌ 消费级市场的品牌和客户名单

Dynamic 有什么：
  ✅ 50M+ 累计 onboard 用户
  ✅ 钱包连接 + 嵌入式钱包双能力
  ✅ Headless 设计，UI 完全可定制
  ✅ 广泛的链支持：全 EVM、全 SVM、Bitcoin、Sui、TON

2025 年 10 月收购后：
  Fireblocks 的嵌入式钱包能力"由 Dynamic 提供"
```

### 整合现状：两套栈并存

这是选型时必须知道的事实。

```
Dynamic 的文档里有一篇叫「Migrate from Fireblocks」，
描述如何把 Fireblocks NCW SDK 的钱包接管过来：

  接管过程中，跑在 iframe 里的 Fireblocks NCW SDK
  与 Fireblocks 交换 MPC 协议消息
  （不是直接调 Fireblocks API）

含义：
  老客户在 Fireblocks NCW SDK 上
  新客户被引导到 Dynamic
  官方提供迁移路径
```

**判断**：这说明整合还在进行中。选型时要明确问清"我接的是哪一套，长期支持哪一套，迁移成本谁承担"。收购后 12 个月内的产品线通常是最不稳定的时期。

## 业务架构

### 商业逻辑：嵌入式钱包是入口，不是主业

```
Fireblocks 的收入结构（推断，未公开披露明细）：

  主业：机构托管与交易基础设施
    - 交易所、银行、资管、做市商
    - 按资产规模 / 交易量 / 席位收费
    - 客单价极高（年费六位数美元起）

  嵌入式钱包：
    - 客单价低得多
    - 战略价值在于：
      a) 拿下"未来会变成机构"的高增长 Fintech
      b) 防守——不做的话客户会流向 Privy/Turnkey，
         然后连托管一起被替换
      c) 交叉销售：EW 客户长大后需要托管、清算、
         做市连接，全在 Fireblocks 体系内
```

这解释了为什么 Fireblocks 愿意花钱买 Dynamic：**它买的不是收入，是漏斗上游**。

### 定位与销售模式

```
面向：
  已有合规负担的 Fintech、银行、券商、交易所
  需要走采购/审计流程的企业

不面向：
  周末想上线一个 DApp 的独立开发者

体现在：
  - 定价不完全自助，企业级议价
  - 强制备份流程需要 CSM 介入
  - 提供 Buyer's Guide 这类采购导向的材料
  - 强调 99.97%+ SLA
```

### 内容营销策略值得注意

Fireblocks 自己出了一份《Fireblocks vs. Privy vs. Turnkey》嵌入式钱包基础设施对比报告，还有《Web3 公司采购指南》和《90 天上线计划》。

```
这套内容的作用：
  把"选谁"的框架定义权拿到自己手里

它的核心论点：
  "对嵌入式钱包来说，TSS-MPC 提供了不牺牲速度的
   最强安全性，加上真正独立的恢复，不依赖单一供应商"

这个论点技术上站得住，但显然是围绕
自己的优势项（MPC + 独立恢复）设计的评价维度。

读这份报告要意识到：它选的评分维度本身就是结论。
```

同理，Turnkey 的 vs Privy 页面和 Crossmint 的 alternatives 页面也是一样的操作。**这个赛道的"客观对比"基本都是竞品内容营销**，交叉对读才有意义。

## 批判性分析

### 优势一：安全模型上限最高

```
真 MPC + 用户 share 存设备硬件飞地
= 私钥数学上从未完整存在
+ Fireblocks 单方无法动用资产

这是四家里唯一能给出这个组合保证的。
```

### 优势二：资本和信任厚度无人能比

```
$8B 估值、$10T+ 累计交易额、2,000+ 企业客户
→ 供应商风险接近零
→ 过合规审查最容易
→ 保险、审计、司法区覆盖是现成的
```

对一家需要向银行合作方或监管解释"你们的密钥谁管"的 Fintech 来说，回答"Fireblocks"的沟通成本最低。

### 优势三：独立恢复是被低估的差异点

```
其它路线的隐含风险：
  供应商消失 → 用户资产能不能拿回来？

Privy：可导出私钥，但要求 Privy 服务还活着才能导
Turnkey：可导出种子，同上
Crossmint：合约在链上，理论上供应商消失也能操作
Fireblocks：企业可重新生成 Fireblocks 侧 shares

Fireblocks 和 Crossmint 是两种不同思路的"活得下去"方案：
  Fireblocks = 你能重建我这一半
  Crossmint = 我这一半根本不必要
```

### 局限一：整合期风险

前面说过的两套栈问题，是当下最实际的风险。收购刚过一年不到，产品边界、路线图、定价都可能变。

### 局限二：MPC 的工程复杂度转嫁给了集成方

```
你需要自己决定和实现：
  - 用户 share 存哪（设备飞地？生物识别？2FA？）
  - 多设备怎么同步（多个 Device ID 怎么管）
  - 备份流程的 UX（还是强制的，不能跳过）
  - 恢复流程的 UX

对比 Privy：这些基本是默认帮你做完的。

结果：
  Fireblocks EW 的"接入完成"到"体验做好"之间，
  工程量比 Privy 大不少。
```

强制备份尤其是个 UX 难题——它安全上正确，但在 Onboarding 漏斗里加了一步硬门槛，而嵌入式钱包的全部卖点就是消除 Onboarding 摩擦。这是一个真实的张力。

### 局限三：嵌入式钱包在内部的优先级

```
一个不好回答但必须问的问题：

  当 Fireblocks 的机构主业和嵌入式钱包业务
  在资源上冲突时，哪边赢？

历史上，被巨头收购的开发者工具经常慢下来。
Dynamic 作为独立公司时迭代很快，
成为"a Fireblocks company"之后能不能保持，还要观察。
```

### 局限四：50M+ 用户这个数字的口径

```
50M+ 是 Dynamic 累计 onboard 的用户数，
不是 Fireblocks EW 当前的活跃钱包数。

和 Privy 的 120M+ 账户一样，
是"累计创建"而非"月活"。

Fireblocks 的 $10T+ 交易额和 2,000+ 客户
则是全公司口径，绝大部分来自机构托管主业，
不能算作嵌入式钱包的业绩。
```

## 什么情况下该选 Fireblocks EW

```
适合：
  ✅ 你是受监管的 Fintech / 银行 / 券商，
     需要向合规和审计解释密钥管理
  ✅ 需要"数学上不可重组"的安全保证
  ✅ 需要独立于供应商的恢复能力（业务连续性要求硬）
  ✅ 未来会需要机构托管、清算、做市连接
     （在同一体系内扩展）
  ✅ 有工程团队能处理 MPC 的集成复杂度
  ✅ 单用户价值高，能摊销较高的接入成本

不适合：
  ❌ 小团队快速验证想法（销售流程和接入成本都重）
  ❌ Onboarding 转化率是唯一 KPI
     （强制备份流程和你的目标冲突）
  ❌ 想避开整合期不确定性
  ❌ 需要"换供应商不换地址"
  ❌ 预算敏感的消费级长尾应用
```

## 小结

Fireblocks Embedded Wallet 是这个赛道里**技术上限最高、资本最厚、合规最稳**的选项，也是唯一能用"真 MPC + 独立恢复"这个组合回答所有安全质疑的一家。收购 Dynamic 补上了它唯一的短板——消费级前端。

但它的定位注定了它赢不了整个市场。嵌入式钱包对 Fireblocks 是漏斗上游而不是主业，销售流程、接入成本、强制备份这些机构级基因，在"周末上线一个 App"的场景里全是摩擦。

它真正要抢的是那一批**从 Fintech 起步、注定要变成受监管金融机构**的客户。这批客户数量不多但单客户价值极高，而且一旦进来就很难走——因为托管、清算、EW 会一起绑在一个体系里。

看它值不值得选，问自己一个问题就够了：**你三年后需要向监管机构解释你的密钥架构吗？** 需要，Fireblocks 的溢价就是值的。

## 数据来源

- [Fireblocks 嵌入式钱包产品页](https://www.fireblocks.com/products/embedded-wallets)（官方）
- [Fireblocks 开发者文档：创建嵌入式钱包](https://developers.fireblocks.com/docs/create-embedded-wallets)（官方）
- [Fireblocks：MPC 密钥生成](https://developers.fireblocks.com/docs/embedded-wallet-mpc-key-generation)（官方）
- [Fireblocks：嵌入式钱包灾难恢复](https://developers.fireblocks.com/docs/embedded-wallet-disaster-recovery)（官方）
- [Fireblocks：资产管理与强制备份](https://developers.fireblocks.com/docs/embedded-wallet-asset-management)（官方）
- [Fireblocks：MPC-CMP 提速 8 倍](https://www.fireblocks.com/blog/pushing-mpc-wallet-signing-speeds-8x-with-mpc-cmp-9)（官方）
- [Fireblocks：什么是 MPC](https://www.fireblocks.com/what-is-mpc/)（官方）
- [Fireblocks：嵌入式钱包 90 天上线计划](https://www.fireblocks.com/blog/embedded-wallets-90-day-rollout-plan)（官方，含 Dynamic 收购时间与用户数）
- [Fireblocks：$550M E 轮，估值 $80 亿](https://www.fireblocks.com/blog/550m-series-e-zero-to-crypto)（官方）
- [Fireblocks：安全创新与平台规模](https://www.fireblocks.com/blog/fireblocks-security-innovations-digital-asset-infrastructure/)（官方）
- [Fireblocks 定价页](https://www.fireblocks.com/pricing)（官方，含 SLA）
- [Dynamic 文档：从 Fireblocks 迁移](https://dynamic-docs.mintlify.app/overview/wallets/embedded-wallets/mpc/migrate-from-fireblocks)（官方）
- [Fireblocks：Web3 公司采购指南](https://www.fireblocks.com/report/buyers-guide-web3-companies)（**有立场**）
- [Fireblocks：嵌入式钱包基础设施对比报告](https://www.fireblocks.com/report/compare-embedded-wallet-infrastructure)（**有立场**，自制 vs Privy/Turnkey 对比）

内容经改写以符合来源许可要求。
