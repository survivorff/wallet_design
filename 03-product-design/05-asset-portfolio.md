# 资产展示与 Portfolio：余额、NFT、DeFi 头寸

## 一句话总结

钱包的资产展示不只是"显示余额"——它需要把分散在多链、多协议、多资产类型中的复杂状态，翻译成用户一眼能看懂的 Portfolio 视图。

## 资产展示的复杂度

### 用户资产的真实状态

一个活跃的 DeFi 用户可能同时拥有：

```
原生代币：
- 3.2 ETH (Ethereum)
- 0.5 ETH (Arbitrum)
- 100 SOL (Solana)

ERC-20 代币：
- 5000 USDC (Ethereum)
- 2000 USDC (Polygon)
- 1000 UNI (Ethereum)
- 500 AAVE (Ethereum)

NFT：
- 2 个 BAYC (Ethereum)
- 5 个 Art Blocks (Ethereum)
- 10 个游戏道具 NFT (Polygon)

DeFi 头寸：
- Aave: 存入 10000 USDC，借出 5000 DAI
- Uniswap V3: ETH/USDC LP，范围 1800-2200
- Lido: 质押 10 ETH → 10 stETH
- Convex: 锁定 CRV，获得 cvxCRV
- Eigenlayer: 再质押 5 ETH

Staking：
- 32 ETH 验证者节点
- 1000 ATOM 质押给验证者

Vesting/锁仓：
- 10000 TOKEN 线性释放（还剩 8000 未释放）
```

钱包需要把这些全部聚合成一个清晰的视图。

## 设计层次

### 层次一：总资产概览

```
┌─────────────────────────────────────┐
│  总资产                              │
│  $127,450.32                        │
│  ↑ $2,340.50 (+1.87%) 24h           │
│                                      │
│  ┌─────────────────────────────┐    │
│  │ ████████████░░░░ 资产分布    │    │
│  │ ETH 45% | 稳定币 30% |      │    │
│  │ DeFi 15% | NFT 10%          │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
```

关键设计决策：
- 是否包含 DeFi 头寸的净值？
- NFT 用地板价还是最后成交价？
- 锁仓/Vesting 的代币算不算？
- 借贷的负债要不要扣除？

### 层次二：代币列表

```
┌─────────────────────────────────────────────┐
│  代币                          余额    价值   │
├─────────────────────────────────────────────┤
│  🔷 ETH                      3.7    $7,400  │
│     Ethereum: 3.2 | Arbitrum: 0.5           │
│                                              │
│  💲 USDC                     7,000   $7,000  │
│     Ethereum: 5,000 | Polygon: 2,000        │
│                                              │
│  🦄 UNI                      1,000  $6,500  │
│     Ethereum                                 │
│                                              │
│  👻 AAVE                      500   $45,000 │
│     Ethereum                                 │
│                                              │
│  [显示小额代币 (12)]                         │
└─────────────────────────────────────────────┘
```

设计要点：
- 同一代币跨链聚合显示
- 展开可看各链分布
- 小额代币折叠（避免垃圾代币干扰）
- 排序：按价值 / 按涨跌幅 / 自定义

### 层次三：NFT 展示

```
┌─────────────────────────────────────────────┐
│  NFT (17)                    估值: $180,000  │
├─────────────────────────────────────────────┤
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐         │
│  │BAYC │ │BAYC │ │Art  │ │Art  │ ...      │
│  │#1234│ │#5678│ │#001 │ │#002 │          │
│  │30ETH│ │30ETH│ │5ETH │ │3ETH │          │
│  └─────┘ └─────┘ └─────┘ └─────┘         │
│                                              │
│  按集合分组 | 按价值排序 | 隐藏垃圾 NFT     │
└─────────────────────────────────────────────┘
```

NFT 展示的特殊挑战：
- 图片加载（IPFS 可能很慢）
- 估值困难（非同质化，没有统一价格）
- 垃圾 NFT 过滤（空投的诈骗 NFT）
- 元数据解析（不同标准、不同存储方式）

### 层次四：DeFi 头寸

```
┌─────────────────────────────────────────────┐
│  DeFi 头寸                   净值: $35,000   │
├─────────────────────────────────────────────┤
│                                              │
│  📊 Aave (Ethereum)                         │
│  ├── 存入: 10,000 USDC (+$120 利息)         │
│  └── 借出: 5,000 DAI (利率 3.5%)            │
│      健康因子: 1.85 ✅                       │
│                                              │
│  🦄 Uniswap V3 (Ethereum)                   │
│  └── ETH/USDC LP                            │
│      范围: $1,800 - $2,200                   │
│      当前价: $2,050 (在范围内 ✅)            │
│      价值: $8,500 | 未领取费用: $230         │
│                                              │
│  🌊 Lido (Ethereum)                         │
│  └── 质押: 10 stETH ($20,000)               │
│      收益率: 3.8% APR                        │
│                                              │
└─────────────────────────────────────────────┘
```

DeFi 头寸展示的挑战：
- 需要理解每个协议的合约结构
- 头寸状态是动态的（利率变化、价格变化）
- 健康因子等风险指标需要实时计算
- 协议数量巨大（几百个），无法全部支持

## 技术实现

### 代币发现

如何知道用户持有哪些代币？

```
方案 1：已知代币列表扫描
- 维护一个"已知代币"列表（如 CoinGecko 的代币列表）
- 逐个查询用户在这些代币合约上的余额
- 问题：无法发现新代币

方案 2：Transfer 事件扫描
- 扫描所有 Transfer 事件中 to = 用户地址的记录
- 可以发现所有曾经收到过的代币
- 问题：需要索引服务，扫描慢

方案 3：索引服务 API
- 使用 Alchemy/Moralis 等服务的 getTokenBalances API
- 一次调用返回所有非零余额的代币
- 问题：依赖第三方服务
```

### 价格数据

```
数据源优先级：
1. DEX 实时价格（链上数据，最准确）
2. CoinGecko / CoinMarketCap API（覆盖广）
3. 预言机价格（Chainlink，链上可验证）
4. 用户自定义价格（对于无流动性的代币）

刷新策略：
- 主要代币：每 30 秒刷新
- 次要代币：每 5 分钟刷新
- NFT 地板价：每 15 分钟刷新
- DeFi 头寸：每 1 分钟刷新
```

### DeFi 头寸解析

```typescript
// 需要为每个协议实现专门的解析器
interface DeFiProtocolAdapter {
  protocolName: string;
  supportedChains: string[];
  
  // 检测用户是否有头寸
  hasPosition(address: string): Promise<boolean>;
  
  // 获取头寸详情
  getPositions(address: string): Promise<DeFiPosition[]>;
  
  // 计算头寸价值
  getPositionValue(position: DeFiPosition): Promise<USD>;
}

// 示例：Aave V3 适配器
class AaveV3Adapter implements DeFiProtocolAdapter {
  async getPositions(address: string): Promise<DeFiPosition[]> {
    const userData = await aavePool.getUserAccountData(address);
    const reserves = await aavePool.getUserReserveData(address);
    
    return reserves.map(reserve => ({
      type: reserve.currentATokenBalance > 0 ? 'supply' : 'borrow',
      token: reserve.underlyingAsset,
      amount: reserve.currentATokenBalance || reserve.currentVariableDebt,
      apy: reserve.currentLiquidityRate,
      healthFactor: userData.healthFactor,
    }));
  }
}
```

### 垃圾代币/NFT 过滤

```typescript
// 垃圾代币的特征
function isSpamToken(token: TokenInfo): boolean {
  return (
    token.holders < 100 ||           // 持有者太少
    token.totalSupply === 0 ||       // 无效供应量
    token.name.includes('Visit') ||  // 名称包含钓鱼链接
    token.isHoneypot ||              // 蜜罐代币（只能买不能卖）
    !token.hasLiquidity ||           // 无流动性
    token.isAirdropSpam              // 已知垃圾空投
  );
}

// 垃圾 NFT 的特征
function isSpamNFT(nft: NFTInfo): boolean {
  return (
    nft.collection.isReported ||     // 被举报的集合
    nft.metadata?.name?.includes('claim') || // 钓鱼 NFT
    nft.collection.floorPrice === 0 || // 无价值
    nft.isUnsolicitedAirdrop         // 未请求的空投
  );
}
```

## 产品设计最佳实践

### 1. 信息密度平衡

```
新手用户需要：简洁、重点突出
- 总资产金额
- 主要代币列表
- 简单的涨跌指示

高级用户需要：详细、可操作
- DeFi 头寸详情
- 健康因子警告
- 未领取的奖励
- 授权管理
```

### 2. 加载状态处理

```
多链数据加载是异步的，需要优雅处理：

✅ 好的做法：
- 先显示已缓存的数据
- 逐步加载各链数据（显示加载进度）
- 数据到达后平滑更新（不要闪烁）

❌ 差的做法：
- 等所有链数据都加载完才显示
- 加载时显示空白页面
- 数据更新时整个列表重新渲染
```

### 3. 资产安全标记

```
在资产列表中标记风险：
- 🔴 代币合约有已知漏洞
- 🟡 代币流动性极低
- ⚠️ 无限授权给未知合约
- 🔒 资产在锁仓中
- 📉 DeFi 头寸接近清算
```

## 小结

资产展示看似简单（不就是显示余额吗？），实际上是钱包产品中工程量最大的模块之一。它需要：

1. 聚合多链、多协议、多资产类型的数据
2. 实时更新且保持一致性
3. 过滤垃圾信息
4. 在信息完整性和界面简洁性之间平衡
5. 为不同层次的用户提供不同深度的视图

做好资产展示的钱包（如 Zerion、DeBank）往往能获得很高的用户粘性——因为用户每天打开钱包的第一件事就是看自己的资产。
