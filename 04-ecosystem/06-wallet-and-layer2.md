# 钱包与 Layer2：Rollup 生态的钱包适配

## 一句话总结

L2 不是"另一条链"那么简单——它有独特的费用模型、确认机制和提取流程，钱包需要专门适配才能给用户最佳体验。

## L2 与 L1 的关键差异

### Gas 模型

```
L1 (Ethereum)：
gas_cost = gas_used × gas_price
其中 gas_price = base_fee + priority_fee

L2 (Optimistic Rollup, e.g. Optimism, Arbitrum)：
gas_cost = L2_execution_fee + L1_data_fee
- L2 执行费：很低（几乎免费）
- L1 数据费：取决于 L1 当前 gas_price + 交易数据大小

L2 (ZK Rollup, e.g. zkSync, Starknet)：
gas_cost 类似但 L1 数据更压缩
```

钱包需要理解这种双层费用结构，准确估算和展示。

### 确认机制

```
L1 确认：
- 区块时间 12 秒
- 12-18 个区块后视为最终确认
- ~3 分钟达到经济最终性

L2 软确认（Soft Finality）：
- L2 排序器（Sequencer）打包
- 几秒钟内"软确认"
- 但还没写入 L1，理论上可能被回滚

L2 硬确认（Hard Finality）：
- 数据提交到 L1
- L1 确认后才算"真正最终"
- Optimistic Rollup：~7 天挑战期
- ZK Rollup：~1 小时（取决于证明频率）
```

## 提取（Withdraw）的复杂性

### Optimistic Rollup 的挑战期

```
用户从 Optimism 提取 ETH 到 Ethereum：

Day 1: 在 L2 发起提取（触发消息到 L1）
Day 1-7: 挑战期（任何人可以提交欺诈证明）
Day 8: 在 L1 完成提取，资产到账

钱包的展示：
- 提取后立即显示"处理中"
- 显示预计到账时间
- 7 天后提醒用户来 L1 完成 claim
```

### 快速桥（替代方案）

```
用户不想等 7 天 → 使用第三方快速桥

工作原理：
1. 流动性提供者在两边都有资金
2. 用户在 L2 提交交易
3. LP 在 L1 立即给用户支付
4. LP 之后通过原生桥拿回资金
5. LP 收取一定费用作为报酬

代表：
- Hop Protocol
- Across
- Stargate
- LayerZero
```

钱包应该在"原生桥（慢但便宜）"和"快速桥（快但有费用）"之间提供清晰选择。

## 主要 L2 的特点

### Optimism / OP Stack

```
特点：
- 最早的 Optimistic Rollup
- 通用 EVM 兼容
- OP Stack 衍生出 Base、Worldchain 等

钱包适配要点：
- 标准 EVM RPC（兼容性好）
- 注意 L1 数据费的波动
- Sequencer 中心化风险
```

### Arbitrum

```
特点：
- 最大的 L2（TVL 第一）
- Arbitrum Stylus 支持非 EVM 语言（Rust 等）
- Arbitrum Nova 用 AnyTrust 优化成本

钱包适配要点：
- 标准 EVM 兼容
- Nova 链是独立的，需要特别处理
```

### Base

```
特点：
- Coinbase 推出
- 基于 OP Stack
- 用户增长快（Coinbase 流量加持）

钱包适配要点：
- 与 Optimism 类似
- 注意 Coinbase 的合规要求
- 集成 Coinbase Smart Wallet 是优势
```

### zkSync

```
特点：
- ZK Rollup
- 较快的最终性（相比 OP Rollup）
- Native Account Abstraction

钱包适配要点：
- 账户抽象是原生的
- 智能合约钱包是默认
- Paymaster 集成简化（链原生支持）
- 地址派生与传统 EVM 不同（用 keccak256 + 部署者地址）
```

### Starknet

```
特点：
- ZK Rollup
- 用 Cairo 语言（非 EVM）
- 账户抽象原生

钱包适配要点：
- 完全不同的虚拟机
- 不能用标准 EVM 工具
- 需要专门的 Starknet 钱包（Argent X、Braavos）
- 地址格式不同
```

### Polygon zkEVM / Linea / Scroll

```
特点：
- 与 Ethereum 字节码兼容的 ZK Rollup
- 大多数 EVM 工具直接可用
- 各自有不同的优化重点

钱包适配要点：
- 基本可以当成 EVM 链处理
- 但 ZK 证明的 finality 模型需要理解
- L1 数据费的计算方式可能不同
```

## 钱包的 L2 体验设计

### 网络添加和管理

```
当前痛点：
- 用户需要手动添加每条 L2 网络
- 输入 RPC URL、Chain ID、浏览器 URL 等
- 容易输错或被恶意 RPC 欺骗

更好的设计：
- 内置主流 L2 的预设
- 用户一键添加（类似 chainlist.org）
- 验证 RPC 真实性
- 自动选择健康的 RPC 节点
```

### Gas 估算

```
L2 Gas 估算的特殊性：
- 需要考虑 L1 数据费的波动
- 当 L1 gas 高时，L2 操作成本也会上升
- 钱包需要展示"L2 + L1 数据费"的总和

最佳实践：
- 显示总费用（美元 + Gas Token）
- 解释费用构成（L2 执行 + L1 数据）
- 当 L1 拥堵时提示"现在 L1 拥堵，建议稍后再操作"
```

### 跨 L2 资产管理

```
用户场景：
- 资产在 Ethereum、Optimism、Arbitrum、Base 等多个 L2
- 想要统一视图
- 需要在 L2 之间移动资产

钱包功能：
- 多链余额聚合
- L2 ↔ L2 跨链（不需要回到 L1）
- 智能路由（找到最优桥）
```

## 未来趋势

### Superchain / Hyperchain

```
共享同一安全和互操作层的多条 L2：
- OP Superchain：Optimism、Base、Mode、Worldchain...
- Polygon Aggregation Layer：多条 zkEVM 链
- Arbitrum Orbit：自定义 L3 链

对钱包的挑战：
- 数百条同构链
- 用户难以区分
- 需要更强的链抽象能力
```

### 统一桥接

```
未来 L2 之间的桥接可能：
- 协议层原生支持（不需要第三方桥）
- 几乎免费
- 几乎实时
- 安全等同于 L1

钱包可以彻底隐藏"链"的概念
```

## 小结

L2 是钱包必须重点支持的领域，因为：
1. L1 高 Gas 让 L2 成为大多数用户的实际选择
2. L2 的数量在快速增长（数百条）
3. L2 的特殊性（费用、确认、提取）需要专门处理

好的钱包应该在 L2 上提供与 L1 同等甚至更好的体验，并通过聚合和抽象，让用户从"管理多条链"中解放出来。
