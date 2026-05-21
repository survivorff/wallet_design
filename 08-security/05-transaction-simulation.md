# 交易模拟与预执行：让用户"预见"结果

## 一句话总结

交易模拟是在用户签名之前，在本地或服务端模拟执行交易，展示"如果你确认，会发生什么"——这是对抗钓鱼攻击最有效的产品手段。

## 为什么交易模拟如此重要

### 没有模拟时的用户体验

```
用户看到的：
┌─────────────────────────────┐
│ 确认交易                     │
│                              │
│ To: 0x7a25...488D            │
│ Data: 0x7ff36ab5000000...    │
│ Value: 1 ETH                 │
│                              │
│ [拒绝]        [确认]         │
└─────────────────────────────┘

用户的判断依据：几乎为零
结果：要么盲目确认，要么因恐惧拒绝所有操作
```

### 有模拟时的用户体验

```
用户看到的：
┌─────────────────────────────┐
│ 交易预览                     │
│                              │
│ 📊 资产变化：                │
│   - 1.0 ETH ($2,000)       │
│   + 1,985 USDC ($1,985)    │
│                              │
│ ⚠️ 授权变化：无              │
│ ✅ 风险评估：低风险          │
│                              │
│ [拒绝]        [确认]         │
└─────────────────────────────┘

用户的判断依据：清晰的资产变化预览
结果：能识别异常（如"为什么我会损失所有 NFT？"）
```

## 技术实现

### 方式一：本地模拟（eth_call）

```typescript
async function simulateTransaction(tx: Transaction): Promise<SimulationResult> {
  // 使用 eth_call 模拟执行（不上链）
  try {
    const result = await provider.call({
      from: tx.from,
      to: tx.to,
      data: tx.data,
      value: tx.value,
    });
    
    return { success: true, returnData: result };
  } catch (error) {
    return { success: false, revertReason: decodeRevertReason(error) };
  }
}
```

局限：
- 只能知道"成功/失败"，不能看到状态变化
- 不能看到内部调用的 Transfer 事件
- 不能看到余额变化的具体数值

### 方式二：Trace 模拟（debug_traceCall）

```typescript
async function traceTransaction(tx: Transaction): Promise<TraceResult> {
  // 使用 debug_traceCall 获取完整执行轨迹
  const trace = await provider.send('debug_traceCall', [
    { from: tx.from, to: tx.to, data: tx.data, value: tx.value },
    'latest',
    { tracer: 'callTracer' }  // 或 'prestateTracer'
  ]);
  
  // trace 包含：
  // - 所有内部调用（call, delegatecall, staticcall）
  // - 每次调用的输入输出
  // - Gas 消耗
  // - 状态变化
  
  return parseTrace(trace);
}
```

可以获得：
- 完整的调用链（哪些合约被调用了）
- 状态变化（哪些存储槽被修改了）
- 事件日志（Transfer、Approval 等）
- 精确的 Gas 消耗

局限：
- 不是所有 RPC 节点都支持 debug_traceCall
- 计算量大，响应慢
- 需要专门的基础设施

### 方式三：状态差异模拟（Tenderly/Alchemy）

```typescript
// 使用 Tenderly Simulation API
async function simulateWithTenderly(tx: Transaction): Promise<DetailedSimulation> {
  const response = await fetch('https://api.tenderly.co/api/v1/simulate', {
    method: 'POST',
    body: JSON.stringify({
      network_id: tx.chainId,
      from: tx.from,
      to: tx.to,
      input: tx.data,
      value: tx.value,
      save: false,
    }),
  });
  
  const result = await response.json();
  
  // 返回：
  // - 资产变化（ERC-20 转入转出）
  // - NFT 变化
  // - 授权变化
  // - 内部交易
  // - Gas 使用
  // - 成功/失败及原因
  
  return parseSimulationResult(result);
}
```

### 方式四：Fork 模拟

```typescript
// 在主网 Fork 上执行交易
async function forkAndSimulate(tx: Transaction): Promise<SimulationResult> {
  // 1. 创建主网 Fork（在最新区块状态上）
  const fork = await createFork(tx.chainId, 'latest');
  
  // 2. 在 Fork 上执行交易
  const receipt = await fork.sendTransaction(tx);
  
  // 3. 比较执行前后的状态差异
  const balanceBefore = await fork.getBalance(tx.from, 'pre');
  const balanceAfter = await fork.getBalance(tx.from, 'post');
  
  // 4. 解析所有事件日志
  const events = parseEvents(receipt.logs);
  
  return { balanceChanges, events, gasUsed: receipt.gasUsed };
}
```

## 模拟结果的解析与展示

### 资产变化解析

```typescript
interface AssetChange {
  type: 'native' | 'erc20' | 'erc721' | 'erc1155';
  direction: 'in' | 'out';
  token: TokenInfo;
  amount: BigNumber;
  valueUSD: number;
  from?: string;  // 来源地址
  to?: string;    // 目标地址
}

function parseAssetChanges(logs: Log[]): AssetChange[] {
  const changes: AssetChange[] = [];
  
  for (const log of logs) {
    // ERC-20 Transfer
    if (log.topics[0] === TRANSFER_TOPIC) {
      const from = decodeAddress(log.topics[1]);
      const to = decodeAddress(log.topics[2]);
      const amount = decodeBigNumber(log.data);
      
      if (to === userAddress) {
        changes.push({ type: 'erc20', direction: 'in', amount, ... });
      }
      if (from === userAddress) {
        changes.push({ type: 'erc20', direction: 'out', amount, ... });
      }
    }
    
    // ERC-20 Approval
    if (log.topics[0] === APPROVAL_TOPIC) {
      // 记录授权变化
    }
    
    // ERC-721 Transfer
    if (log.topics[0] === TRANSFER_TOPIC && log.topics.length === 4) {
      // NFT 转移
    }
  }
  
  return changes;
}
```

### 风险评估

```typescript
interface RiskAssessment {
  level: 'low' | 'medium' | 'high' | 'critical';
  reasons: string[];
  suggestions: string[];
}

function assessRisk(simulation: SimulationResult, context: TxContext): RiskAssessment {
  const risks: string[] = [];
  
  // 检查：是否有大额资产流出
  if (simulation.totalOutflowUSD > 10000) {
    risks.push('大额资产转出');
  }
  
  // 检查：是否授权了无限额度
  if (simulation.approvals.some(a => a.amount === MAX_UINT256)) {
    risks.push('无限额度授权');
  }
  
  // 检查：目标合约是否在黑名单
  if (isBlacklisted(context.to)) {
    risks.push('目标地址在黑名单中');
  }
  
  // 检查：合约是否刚部署
  if (context.contractAge < 7 * 24 * 3600) {
    risks.push('合约部署不到 7 天');
  }
  
  // 检查：是否是首次交互
  if (!context.hasInteractedBefore) {
    risks.push('首次与此合约交互');
  }
  
  // 检查：模拟结果与预期是否一致
  if (simulation.unexpectedOutflows.length > 0) {
    risks.push('存在非预期的资产流出');
  }
  
  return {
    level: calculateRiskLevel(risks),
    reasons: risks,
    suggestions: generateSuggestions(risks),
  };
}
```

## 模拟的局限性

### 不能模拟的情况

| 情况 | 原因 | 影响 |
|------|------|------|
| MEV 攻击 | 模拟不考虑交易排序 | 实际执行可能被夹 |
| 时间依赖 | 模拟时间与执行时间不同 | 时间锁相关逻辑可能不同 |
| 状态变化 | 模拟后到执行前状态可能变化 | 价格变动、流动性变化 |
| 多交易依赖 | 只模拟单笔交易 | 前序交易可能影响结果 |
| 链下条件 | 预言机更新、Keeper 操作 | 模拟时的价格可能过时 |

### 模拟可能被绕过

```
攻击者的对策：
1. 合约检测是否在模拟环境中
   - 检查 block.number 是否异常
   - 检查 tx.origin 是否是已知的模拟器
   - 在模拟时表现正常，实际执行时作恶

2. 延迟攻击
   - 第一次交互正常（建立信任）
   - 后续交互才触发恶意逻辑

3. 条件触发
   - 只在特定条件下（如余额超过阈值）才触发恶意行为
   - 模拟时条件不满足，看起来安全

防御：
- 不仅模拟当前交易，还分析合约代码
- 检查合约是否有可疑的条件分支
- 结合链上历史数据（这个合约以前有没有恶意行为）
```

## 行业方案对比

| 产品 | 实现方式 | 特点 |
|------|---------|------|
| Rabby | 内置模拟 | 每笔交易自动预执行，显示资产变化 |
| Blowfish | API 服务 | 提供模拟 API，被多个钱包集成 |
| Pocket Universe | 浏览器插件 | 拦截交易请求，模拟后展示风险 |
| Fire | 浏览器插件 | 交易模拟 + 风险评分 |
| Tenderly | 开发工具 | 专业级模拟，支持 Fork |
| BlockAid | API 服务 | 实时威胁检测 + 模拟 |

## 产品设计建议

### 模拟结果的展示层次

```
第一层（所有用户看到）：
✅ 你将收到：1,985 USDC
❌ 你将支付：1.0 ETH + $3.50 Gas
净变化：-$15（滑点 + Gas）

第二层（点击展开）：
- 通过 Uniswap V3 Router 执行
- 路径：ETH → WETH → USDC
- 滑点保护：2%
- 合约已验证 ✅

第三层（高级用户）：
- 完整的内部调用链
- 状态变化详情
- 原始 calldata 解码
```

### 异常情况的处理

```
模拟失败时：
⚠️ "无法预览交易结果。可能原因：网络拥堵、合约逻辑复杂。"
[仍然发送（风险自担）] [取消]

模拟显示高风险时：
🔴 "检测到异常：此交易将转出你所有的 BAYC NFT"
[我确定要继续（需要输入 CONFIRM）] [取消]

模拟超时时：
⏱️ "模拟耗时较长，正在处理..."
[跳过模拟直接发送] [继续等待]
```

## 小结

交易模拟是当前对抗钓鱼攻击最有效的产品手段——它把"用户需要理解复杂的技术细节"转变为"用户只需要看懂资产变化"。

关键原则：
1. **默认开启**：每笔交易都应该模拟，不需要用户主动触发
2. **结果清晰**：用"你会得到什么/失去什么"的语言展示
3. **风险突出**：异常情况用强视觉信号提醒
4. **不阻塞**：模拟应该快速（<2 秒），不能让用户等太久
5. **承认局限**：告诉用户模拟不是 100% 准确的

Rabby 通过将交易预执行作为核心功能，在安全体验上建立了显著的差异化——这证明了交易模拟不只是"锦上添花"，而是可以成为钱包的核心竞争力。
