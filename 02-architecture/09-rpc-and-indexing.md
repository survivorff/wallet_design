# RPC 节点与数据索引：钱包的"眼睛"

## 一句话总结

钱包本身不存储区块链数据——它通过 RPC 节点和索引服务"看到"链上世界，这些基础设施的质量直接决定了钱包的响应速度和数据准确性。

## 钱包需要什么数据

### 核心数据需求

| 数据类型 | 用途 | 实时性要求 |
|---------|------|-----------|
| 原生代币余额 | 显示 ETH/SOL 等余额 | 高（秒级） |
| 代币余额 | 显示 ERC-20 等代币 | 高 |
| NFT 持有 | 显示用户的 NFT | 中（分钟级） |
| 交易历史 | 活动记录 | 中 |
| Gas 价格 | 费用估算 | 高 |
| 代币价格 | 资产估值 | 中 |
| 合约 ABI | 交易解析 | 低（缓存） |
| 代币元数据 | 名称、符号、精度、图标 | 低 |
| DeFi 头寸 | Portfolio 展示 | 中 |
| 授权状态 | 安全管理 | 中 |

### 数据获取的挑战

```
问题 1：RPC 节点只提供"当前状态"，不提供"历史聚合"
  - eth_getBalance 只返回当前余额
  - 要获取交易历史需要扫描所有区块的日志

问题 2：代币余额需要逐个查询
  - 以太坊没有"获取某地址所有代币余额"的原生方法
  - 需要知道代币合约地址，然后逐个调用 balanceOf

问题 3：数据分散在多条链上
  - 用户的资产可能在 10+ 条链上
  - 每条链需要独立的数据源
```

## RPC 节点

### 什么是 RPC

RPC（Remote Procedure Call）是钱包与区块链节点通信的接口：

```
钱包 ──HTTP/WebSocket──→ RPC 节点 ──→ 区块链网络

常用方法：
eth_getBalance          获取余额
eth_getTransactionCount 获取 nonce
eth_estimateGas         估算 Gas
eth_sendRawTransaction  发送交易
eth_call                只读调用（不上链）
eth_getLogs             获取事件日志
eth_getBlockByNumber    获取区块信息
```

### RPC 提供商

| 提供商 | 特点 | 免费额度 |
|--------|------|---------|
| Infura | 最老牌，MetaMask 默认 | 100K 请求/天 |
| Alchemy | 功能丰富，增强 API | 300M CU/月 |
| QuickNode | 高性能，多链 | 有限免费 |
| Ankr | 去中心化 RPC | 有限免费 |
| Chainstack | 企业级 | 有限免费 |
| 公共 RPC | 免费但不稳定 | 无限（有限速） |

### 自建节点 vs 第三方服务

| | 自建节点 | 第三方 RPC |
|---|---|---|
| 成本 | 高（服务器 + 维护） | 按量付费 |
| 延迟 | 最低（本地） | 取决于地理位置 |
| 可靠性 | 自己负责 | SLA 保证 |
| 隐私 | 最好（数据不经第三方） | 提供商可以看到请求 |
| 扩展性 | 需要自己扩容 | 弹性扩展 |
| 增强功能 | 无 | 索引、Webhook、调试 API |

大多数钱包使用第三方 RPC 服务，因为自建节点的运维成本太高。但这引入了中心化依赖——Infura 宕机时 MetaMask 就不可用。

### RPC 的局限

标准 RPC 接口有很多钱包需要但不提供的功能：

```
❌ 获取地址的所有代币余额（需要逐个查询）
❌ 获取完整交易历史（需要扫描所有区块）
❌ 获取代币转账事件（需要过滤大量日志）
❌ 获取 NFT 元数据（需要额外的 IPFS/HTTP 请求）
❌ 获取 DeFi 头寸（需要理解每个协议的合约）
❌ 实时价格数据（不是链上数据）
```

这就是为什么需要**索引服务**。

## 数据索引服务

### 索引的原理

```
区块链节点（原始数据）
    │
    │ 持续同步
    ▼
索引服务（处理 + 存储）
    │
    │ 结构化查询
    ▼
钱包（快速获取所需数据）

索引服务做的事：
1. 监听新区块和交易
2. 解析事件日志（Transfer、Approval 等）
3. 维护地址 → 代币余额的映射
4. 维护地址 → 交易历史的索引
5. 解析 NFT 元数据
6. 追踪 DeFi 协议的用户头寸
```

### 主要索引服务

| 服务 | 类型 | 提供的数据 |
|------|------|-----------|
| Alchemy Enhanced APIs | 商业 | 代币余额、NFT、交易历史 |
| Moralis | 商业 | 代币、NFT、DeFi、跨链 |
| Covalent | 商业 | 统一多链数据 API |
| The Graph | 去中心化 | 自定义 Subgraph 查询 |
| Dune Analytics | 分析 | SQL 查询链上数据 |
| Etherscan API | 商业 | 交易历史、合约验证 |
| DeBank API | 商业 | DeFi 头寸、Portfolio |
| Zapper API | 商业 | DeFi 头寸聚合 |
| SimpleHash | 商业 | NFT 数据聚合 |

### 增强 API 示例

```typescript
// 标准 RPC：获取一个代币余额需要一次调用
const balance = await provider.call({
  to: tokenAddress,
  data: erc20.encodeFunctionData('balanceOf', [userAddress])
});

// Alchemy Enhanced API：一次获取所有代币余额
const response = await alchemy.core.getTokenBalances(userAddress);
// 返回：[{ token: "0x...", balance: "1000000" }, ...]

// 标准 RPC：获取交易历史需要扫描所有区块
// Alchemy：直接查询
const history = await alchemy.core.getAssetTransfers({
  fromAddress: userAddress,
  category: ['erc20', 'erc721', 'external'],
});
```

## 钱包的数据架构

### 典型的数据流

```
┌─────────────────────────────────────────────────────────┐
│                      钱包客户端                           │
├─────────────────────────────────────────────────────────┤
│  本地缓存层                                              │
│  - 余额缓存（TTL: 15s）                                 │
│  - 代币列表缓存（TTL: 5min）                             │
│  - 交易历史缓存（增量更新）                               │
│  - 价格缓存（TTL: 30s）                                  │
├─────────────────────────────────────────────────────────┤
│  数据聚合层                                              │
│  - 多源数据合并                                          │
│  - 数据一致性校验                                        │
│  - 错误重试和降级                                        │
├─────────────────────────────────────────────────────────┤
│  数据源层                                                │
│  ┌──────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │ RPC  │ │ 索引服务  │ │ 价格API  │ │ 元数据   │      │
│  │节点  │ │(Alchemy) │ │(CoinGecko)│ │(IPFS)   │      │
│  └──────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────┘
```

### 实时更新策略

```typescript
class BalanceUpdater {
  // 策略 1：定时轮询
  startPolling(address: string, interval: number = 15000) {
    setInterval(async () => {
      const balance = await this.fetchBalance(address);
      this.updateUI(balance);
    }, interval);
  }
  
  // 策略 2：WebSocket 订阅
  subscribeToUpdates(address: string) {
    const ws = new WebSocket(RPC_WS_URL);
    ws.send(JSON.stringify({
      method: 'eth_subscribe',
      params: ['newPendingTransactions'], // 或 'logs' 过滤特定地址
    }));
    ws.onmessage = (event) => {
      // 检查是否与用户地址相关
      this.handleUpdate(event.data);
    };
  }
  
  // 策略 3：交易后主动刷新
  onTransactionConfirmed(txHash: string) {
    // 交易确认后立即刷新余额
    setTimeout(() => this.fetchBalance(address), 2000);
  }
}
```

### 多链数据聚合

```typescript
class PortfolioAggregator {
  async getFullPortfolio(addresses: ChainAddress[]): Promise<Portfolio> {
    // 并行查询所有链的数据
    const results = await Promise.all(
      addresses.map(({ chain, address }) => 
        this.getChainPortfolio(chain, address)
      )
    );
    
    // 聚合
    return {
      totalValueUSD: results.reduce((sum, r) => sum + r.valueUSD, 0),
      tokens: this.mergeTokens(results), // 合并同一代币在不同链的余额
      nfts: results.flatMap(r => r.nfts),
      defiPositions: results.flatMap(r => r.defi),
    };
  }
}
```

## 隐私考量

### RPC 请求暴露的信息

每次 RPC 请求都会暴露：
- 用户的 IP 地址
- 查询的钱包地址
- 查询的时间和频率
- 交互的合约和 DApp

RPC 提供商理论上可以：
- 关联 IP 和钱包地址
- 追踪用户的链上活动模式
- 审查或延迟特定交易

### 隐私保护方案

| 方案 | 说明 | 代价 |
|------|------|------|
| 自建节点 | 数据不经第三方 | 高成本 |
| Tor/VPN | 隐藏 IP | 延迟增加 |
| 多 RPC 轮换 | 分散数据 | 实现复杂 |
| 本地轻客户端 | 直连 P2P 网络 | 同步时间 |
| 去中心化 RPC（Pocket Network） | 请求分散到多个节点 | 延迟不稳定 |

## 可靠性设计

### 故障转移

```typescript
class ReliableRPC {
  private endpoints: RPCEndpoint[];
  private currentIndex: number = 0;
  
  async call(method: string, params: any[]): Promise<any> {
    const maxRetries = this.endpoints.length;
    
    for (let i = 0; i < maxRetries; i++) {
      const endpoint = this.endpoints[(this.currentIndex + i) % this.endpoints.length];
      try {
        const result = await endpoint.call(method, params, { timeout: 5000 });
        return result;
      } catch (error) {
        if (i === maxRetries - 1) throw error;
        // 标记当前节点不健康，尝试下一个
        endpoint.markUnhealthy();
      }
    }
  }
}
```

### 数据一致性

不同数据源可能返回不一致的数据：

```typescript
// 余额不一致的处理
async function getReliableBalance(address: string): Promise<bigint> {
  const [rpcBalance, indexerBalance] = await Promise.all([
    rpcProvider.getBalance(address),      // 直接从节点获取
    indexerService.getBalance(address),   // 从索引服务获取
  ]);
  
  // 如果差异超过阈值，以 RPC 为准（更实时）
  if (Math.abs(Number(rpcBalance - indexerBalance)) > threshold) {
    console.warn('Balance mismatch, using RPC value');
    return rpcBalance;
  }
  
  return rpcBalance;
}
```

## 小结

RPC 和索引服务是钱包的"感官系统"——没有它们，钱包就是一个盲人。

关键的架构决策：
1. **数据源选择**：在成本、速度、隐私之间权衡
2. **缓存策略**：平衡实时性和性能
3. **可靠性**：多源冗余 + 故障转移
4. **隐私**：意识到 RPC 请求暴露的信息

对于钱包产品来说，数据层的质量直接影响用户体验——余额显示延迟、交易历史缺失、NFT 图片加载慢，这些都是数据层的问题。好的钱包需要在这一层投入大量工程努力。
