# 交易构建与广播：从用户意图到链上确认

## 一句话总结

用户点击"确认"到交易上链之间，钱包需要完成一系列复杂的步骤——构建、签名、广播、追踪——每一步都可能出错。

## 交易的完整生命周期

```
用户意图 → 参数收集 → 交易构建 → Gas 估算 → 用户确认 → 签名 → 广播 → 等待确认 → 结果展示
   │          │          │          │          │        │       │         │          │
   ▼          ▼          ▼          ▼          ▼        ▼       ▼         ▼          ▼
"转100U"   解析目标   编码数据   预估费用   展示详情  密钥签名  发送节点  监听状态   通知用户
给Alice    地址/金额  calldata  Gas Price  确认页    ECDSA    RPC      Pending    成功/失败
```

## 阶段一：交易构建

### EVM 交易结构（EIP-1559）

```typescript
interface EVMTransaction {
  // 基础字段
  chainId: number;           // 链 ID（防重放）
  nonce: number;             // 发送者的交易序号
  to: string;                // 目标地址（合约或 EOA）
  value: bigint;             // 转账的 ETH 数量（wei）
  data: string;              // 调用数据（合约交互时）
  
  // Gas 相关（EIP-1559）
  maxFeePerGas: bigint;      // 愿意支付的最大 Gas 单价
  maxPriorityFeePerGas: bigint; // 给验证者的小费
  gasLimit: bigint;          // 最大 Gas 用量
  
  // 访问列表（EIP-2930，可选）
  accessList?: AccessListItem[];
}
```

### 构建过程

```typescript
async function buildTransaction(intent: UserIntent): Promise<EVMTransaction> {
  // 1. 解析用户意图
  const { to, value, data } = parseIntent(intent);
  
  // 2. 获取 nonce
  const nonce = await provider.getTransactionCount(from, 'pending');
  // 注意：用 'pending' 而非 'latest'，避免 nonce 冲突
  
  // 3. 估算 Gas Limit
  const gasEstimate = await provider.estimateGas({ from, to, value, data });
  const gasLimit = gasEstimate * 120n / 100n; // 加 20% 安全边际
  
  // 4. 获取 Gas Price 建议
  const feeData = await provider.getFeeData();
  // 或使用更精确的 fee history
  const feeHistory = await provider.send('eth_feeHistory', ['0x5', 'latest', [25, 50, 75]]);
  
  // 5. 组装交易
  return {
    chainId: await provider.getNetwork().then(n => n.chainId),
    nonce,
    to,
    value,
    data,
    maxFeePerGas: feeData.maxFeePerGas,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas,
    gasLimit,
  };
}
```

### Calldata 编码

当用户与智能合约交互时，需要将函数调用编码为 calldata：

```typescript
// 用户想在 Uniswap 上 swap
// 函数：swapExactETHForTokens(uint amountOutMin, address[] path, address to, uint deadline)

const iface = new ethers.Interface(UniswapRouterABI);
const data = iface.encodeFunctionData('swapExactETHForTokens', [
  amountOutMin,           // 最小输出量
  [WETH_ADDRESS, USDC_ADDRESS], // 交换路径
  userAddress,            // 接收地址
  Math.floor(Date.now() / 1000) + 900, // 15分钟截止
]);

// data = "0x7ff36ab5000000000000000000000000000000000000000000..."
```

### Nonce 管理

Nonce 管理是钱包开发中最容易出问题的地方之一：

```typescript
class NonceManager {
  private localNonce: Map<string, number> = new Map();
  
  async getNextNonce(address: string): Promise<number> {
    // 获取链上确认的 nonce
    const confirmedNonce = await provider.getTransactionCount(address, 'latest');
    // 获取 pending 池中的 nonce
    const pendingNonce = await provider.getTransactionCount(address, 'pending');
    // 本地追踪的 nonce
    const localNonce = this.localNonce.get(address) || 0;
    
    // 取最大值，避免冲突
    const nextNonce = Math.max(confirmedNonce, pendingNonce, localNonce);
    this.localNonce.set(address, nextNonce + 1);
    return nextNonce;
  }
}

// 常见问题：
// 1. 快速连续发送多笔交易 → nonce 冲突
// 2. 交易被 drop 但 nonce 已递增 → nonce gap（后续交易卡住）
// 3. 不同设备同时发送 → nonce 竞争
```

## 阶段二：签名

### RLP 编码与签名

```typescript
async function signTransaction(tx: EVMTransaction, privateKey: Uint8Array): Promise<string> {
  // 1. RLP 编码交易数据（不含签名）
  const unsignedTxRLP = rlpEncode([
    tx.chainId,
    tx.nonce,
    tx.maxPriorityFeePerGas,
    tx.maxFeePerGas,
    tx.gasLimit,
    tx.to,
    tx.value,
    tx.data,
    tx.accessList || [],
  ]);
  
  // 2. 计算交易哈希
  const txHash = keccak256(Buffer.concat([Buffer.from([0x02]), unsignedTxRLP]));
  // 0x02 前缀表示 EIP-1559 交易类型
  
  // 3. ECDSA 签名
  const { r, s, v } = ecdsaSign(txHash, privateKey);
  
  // 4. 将签名附加到交易中
  const signedTxRLP = rlpEncode([
    tx.chainId, tx.nonce, tx.maxPriorityFeePerGas, tx.maxFeePerGas,
    tx.gasLimit, tx.to, tx.value, tx.data, tx.accessList || [],
    v, r, s
  ]);
  
  return '0x02' + signedTxRLP.toString('hex');
}
```

### 不同签名场景

| 场景 | 签名方式 | 延迟 |
|------|---------|------|
| 本地私钥 | 直接签名 | <1ms |
| 硬件钱包 | USB/蓝牙通信 + 用户确认 | 5-30s |
| MPC | 多方协作计算 | 200ms-5s |
| 合约钱包 | 构建 UserOp + Bundler | 1-10s |

## 阶段三：广播

### 发送交易

```typescript
async function broadcastTransaction(signedTx: string): Promise<string> {
  try {
    const txHash = await provider.send('eth_sendRawTransaction', [signedTx]);
    return txHash;
  } catch (error) {
    // 常见错误处理
    if (error.message.includes('nonce too low')) {
      // Nonce 已被使用，需要重新获取
      throw new NonceConflictError();
    }
    if (error.message.includes('insufficient funds')) {
      // 余额不足（包括 Gas 费）
      throw new InsufficientFundsError();
    }
    if (error.message.includes('replacement transaction underpriced')) {
      // 相同 nonce 的交易已存在，新交易的 Gas Price 不够高
      throw new ReplacementUnderpricedError();
    }
    throw error;
  }
}
```

### 多节点广播

为了提高交易被打包的概率：

```typescript
async function broadcastToMultipleNodes(signedTx: string): Promise<string> {
  const nodes = getRPCEndpoints(chainId);
  
  // 并行发送到多个节点
  const results = await Promise.allSettled(
    nodes.map(node => node.send('eth_sendRawTransaction', [signedTx]))
  );
  
  // 只要有一个成功就行
  const success = results.find(r => r.status === 'fulfilled');
  if (success) return success.value;
  
  throw new BroadcastFailedError(results);
}
```

## 阶段四：状态追踪

### 交易状态机

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Created │ ──→ │ Pending  │ ──→ │Confirmed │
└──────────┘     └──────────┘     └──────────┘
                      │                  │
                      │                  ▼
                      │            ┌──────────┐
                      │            │  Final   │
                      │            │(N blocks)│
                      │            └──────────┘
                      ▼
                ┌──────────┐
                │ Dropped/ │
                │ Replaced │
                └──────────┘
```

### 状态监听

```typescript
class TransactionTracker {
  async trackTransaction(txHash: string): Promise<TxResult> {
    // 方式 1：轮询
    while (true) {
      const receipt = await provider.getTransactionReceipt(txHash);
      if (receipt) {
        return {
          status: receipt.status === 1 ? 'success' : 'failed',
          blockNumber: receipt.blockNumber,
          gasUsed: receipt.gasUsed,
          logs: receipt.logs,
        };
      }
      await sleep(3000); // 每 3 秒查询一次
    }
    
    // 方式 2：WebSocket 订阅（更高效）
    // provider.on(txHash, (receipt) => { ... });
    
    // 方式 3：过滤器
    // 监听特定地址的 pending 交易和确认
  }
  
  // 检测交易是否被 drop
  async checkIfDropped(txHash: string, nonce: number, from: string): Promise<boolean> {
    const currentNonce = await provider.getTransactionCount(from, 'latest');
    if (currentNonce > nonce) {
      // nonce 已被使用，但不是这笔交易 → 被替换了
      const receipt = await provider.getTransactionReceipt(txHash);
      if (!receipt) return true; // 被 drop 或替换
    }
    return false;
  }
}
```

## 异常处理

### 交易卡住（Stuck Transaction）

```typescript
// 加速：用更高的 Gas Price 重新发送相同 nonce 的交易
async function speedUpTransaction(originalTx: EVMTransaction): Promise<string> {
  const newTx = {
    ...originalTx,
    maxFeePerGas: originalTx.maxFeePerGas * 130n / 100n, // +30%
    maxPriorityFeePerGas: originalTx.maxPriorityFeePerGas * 130n / 100n,
  };
  return signAndBroadcast(newTx);
}

// 取消：发送一笔 0 ETH 给自己的交易，相同 nonce，更高 Gas
async function cancelTransaction(originalTx: EVMTransaction): Promise<string> {
  const cancelTx = {
    chainId: originalTx.chainId,
    nonce: originalTx.nonce, // 相同 nonce
    to: originalTx.from,     // 发给自己
    value: 0n,
    data: '0x',
    maxFeePerGas: originalTx.maxFeePerGas * 150n / 100n, // +50%
    maxPriorityFeePerGas: originalTx.maxPriorityFeePerGas * 150n / 100n,
    gasLimit: 21000n,
  };
  return signAndBroadcast(cancelTx);
}
```

### 交易失败（Revert）

```typescript
// 获取失败原因
async function getRevertReason(txHash: string): Promise<string> {
  const tx = await provider.getTransaction(txHash);
  try {
    await provider.call({
      to: tx.to,
      data: tx.data,
      value: tx.value,
      from: tx.from,
    }, tx.blockNumber);
  } catch (error) {
    // 解析 revert reason
    // "execution reverted: Insufficient balance"
    return decodeRevertReason(error.data);
  }
}
```

## 批量交易（Batching）

### ERC-4337 批量操作

```typescript
// 一个 UserOp 执行多个操作
const callData = account.interface.encodeFunctionData('executeBatch', [
  [uniswapRouter, aavePool, transferTarget],  // 目标地址数组
  [0, 0, amount],                              // value 数组
  [swapCalldata, depositCalldata, transferCalldata], // calldata 数组
]);

// 链上效果：一笔交易完成 swap + deposit + transfer
// 好处：省 Gas（共享基础开销）、原子性（全部成功或全部失败）
```

### Multicall 模式

```typescript
// 使用 Multicall 合约批量读取数据
const multicall = new Contract(MULTICALL_ADDRESS, MulticallABI);
const calls = tokens.map(token => ({
  target: token.address,
  callData: erc20Interface.encodeFunctionData('balanceOf', [userAddress]),
}));

const results = await multicall.aggregate(calls);
// 一次 RPC 调用获取所有代币余额
```

## 不同链的交易差异

| 维度 | EVM | Solana | Bitcoin |
|------|-----|--------|---------|
| 交易模型 | 账户 + Nonce | 无 Nonce，用 Blockhash | UTXO |
| 确认时间 | 12s (L1) / 2s (L2) | 400ms | 10min |
| 确定性 | 概率性（需等多个区块） | 确定性（一次确认即最终） | 概率性（6 确认） |
| 并行性 | 串行（Nonce 顺序） | 并行（无 Nonce 依赖） | 并行（不同 UTXO） |
| 失败处理 | Revert（Gas 照扣） | 失败不扣费 | 不会被打包 |

## 小结

交易生命周期管理是钱包最核心的工程挑战之一。用户看到的是简单的"发送"按钮，背后是：

1. **构建**：正确编码用户意图为链上可执行的数据
2. **估算**：预测合理的 Gas 费用
3. **签名**：安全地使用私钥生成签名
4. **广播**：可靠地将交易送达网络
5. **追踪**：实时监控交易状态并处理异常

每一步都有大量的边界情况和错误场景需要处理。好的钱包不只是能发送交易，而是能在各种异常情况下给用户清晰的反馈和可操作的解决方案。
