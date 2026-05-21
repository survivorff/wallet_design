# 多链架构设计：如何支持 EVM/非 EVM/Move 链

## 一句话总结

多链钱包的技术挑战不在于"支持更多链"，而在于如何用一套统一的架构优雅地处理各链之间的根本性差异。

## 架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                        应用层                                 │
│  UI 组件 | 路由逻辑 | 状态管理 | 通知系统                    │
├─────────────────────────────────────────────────────────────┤
│                      抽象适配层                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  统一接口定义                                        │    │
│  │  - getBalance(chain, address)                       │    │
│  │  - sendTransaction(chain, tx)                       │    │
│  │  - signMessage(chain, message)                      │    │
│  │  - getTransactionHistory(chain, address)            │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                      链适配器层                               │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐  │
│  │  EVM   │ │Solana  │ │Bitcoin │ │Cosmos  │ │ Move   │  │
│  │Adapter │ │Adapter │ │Adapter │ │Adapter │ │Adapter │  │
│  └────────┘ └────────┘ └────────┘ └────────┘ └────────┘  │
├─────────────────────────────────────────────────────────────┤
│                      密钥管理层                               │
│  HD 派生引擎 | 多曲线支持 | 签名器接口                       │
├─────────────────────────────────────────────────────────────┤
│                      网络/数据层                              │
│  RPC 管理 | 数据索引 | 缓存策略 | 价格聚合                   │
└─────────────────────────────────────────────────────────────┘
```

## 链适配器模式

### 统一接口定义

```typescript
interface ChainAdapter {
  // 基础信息
  chainId: string;
  chainName: string;
  nativeToken: TokenInfo;
  
  // 地址相关
  deriveAddress(publicKey: Uint8Array): string;
  validateAddress(address: string): boolean;
  
  // 余额查询
  getBalance(address: string): Promise<BigNumber>;
  getTokenBalances(address: string): Promise<TokenBalance[]>;
  
  // 交易相关
  buildTransaction(params: TxParams): Promise<UnsignedTx>;
  signTransaction(unsignedTx: UnsignedTx, signer: Signer): Promise<SignedTx>;
  broadcastTransaction(signedTx: SignedTx): Promise<TxHash>;
  getTransactionStatus(txHash: string): Promise<TxStatus>;
  
  // Gas 估算
  estimateFee(tx: UnsignedTx): Promise<FeeEstimate>;
  
  // 历史记录
  getTransactionHistory(address: string, options?: QueryOptions): Promise<Transaction[]>;
}
```

### EVM 适配器

```typescript
class EVMAdapter implements ChainAdapter {
  private provider: JsonRpcProvider;
  private chainConfig: EVMChainConfig;
  
  deriveAddress(publicKey: Uint8Array): string {
    // Keccak256(publicKey)[12:32] → 0x 前缀
    const hash = keccak256(publicKey.slice(1)); // 去掉 04 前缀
    return '0x' + hash.slice(-40);
  }
  
  async buildTransaction(params: TxParams): Promise<UnsignedTx> {
    const nonce = await this.provider.getTransactionCount(params.from);
    const feeData = await this.provider.getFeeData();
    
    return {
      to: params.to,
      value: params.value,
      data: params.data || '0x',
      nonce,
      chainId: this.chainConfig.chainId,
      maxFeePerGas: feeData.maxFeePerGas,
      maxPriorityFeePerGas: feeData.maxPriorityFeePerGas,
      gasLimit: await this.estimateGas(params),
    };
  }
  
  // 所有 EVM 链共享这套逻辑，只是 chainId 和 RPC 不同
}
```

### Solana 适配器

```typescript
class SolanaAdapter implements ChainAdapter {
  private connection: Connection;
  
  deriveAddress(publicKey: Uint8Array): string {
    // Ed25519 公钥直接 Base58 编码
    return bs58.encode(publicKey);
  }
  
  async buildTransaction(params: TxParams): Promise<UnsignedTx> {
    const recentBlockhash = await this.connection.getLatestBlockhash();
    
    // Solana 交易是指令（Instruction）的集合
    const transaction = new Transaction();
    transaction.recentBlockhash = recentBlockhash.blockhash;
    transaction.feePayer = new PublicKey(params.from);
    
    // 添加指令（转账、合约调用等）
    transaction.add(instruction);
    
    return transaction;
  }
  
  // Solana 特有：需要处理 Versioned Transactions、Address Lookup Tables 等
}
```

### Bitcoin 适配器

```typescript
class BitcoinAdapter implements ChainAdapter {
  private network: bitcoin.Network;
  
  deriveAddress(publicKey: Uint8Array): string {
    // 支持多种地址格式
    // P2PKH: 1xxx (Legacy)
    // P2SH-P2WPKH: 3xxx (Nested SegWit)
    // P2WPKH: bc1qxxx (Native SegWit)
    // P2TR: bc1pxxx (Taproot)
    return bitcoin.payments.p2wpkh({ pubkey: publicKey, network }).address;
  }
  
  async buildTransaction(params: TxParams): Promise<UnsignedTx> {
    // Bitcoin 使用 UTXO 模型，需要：
    // 1. 查询可用 UTXO
    const utxos = await this.getUTXOs(params.from);
    
    // 2. 选择 UTXO（coin selection）
    const selected = this.selectCoins(utxos, params.value + estimatedFee);
    
    // 3. 构建交易（输入 + 输出 + 找零）
    const psbt = new bitcoin.Psbt({ network: this.network });
    selected.forEach(utxo => psbt.addInput(utxo));
    psbt.addOutput({ address: params.to, value: params.value });
    psbt.addOutput({ address: params.from, value: change }); // 找零
    
    return psbt;
  }
  
  // Bitcoin 特有：UTXO 选择算法、SegWit 处理、Taproot 签名
}
```

### Cosmos 适配器

```typescript
class CosmosAdapter implements ChainAdapter {
  private client: StargateClient;
  private chainInfo: CosmosChainInfo;
  
  deriveAddress(publicKey: Uint8Array): string {
    // SHA256 + RIPEMD160 + Bech32 编码
    const hash = ripemd160(sha256(publicKey));
    return bech32.encode(this.chainInfo.bech32Prefix, bech32.toWords(hash));
  }
  
  async buildTransaction(params: TxParams): Promise<UnsignedTx> {
    // Cosmos 使用 Protobuf 编码的消息
    const msg = {
      typeUrl: '/cosmos.bank.v1beta1.MsgSend',
      value: {
        fromAddress: params.from,
        toAddress: params.to,
        amount: [{ denom: 'uatom', amount: params.value.toString() }],
      },
    };
    
    return { messages: [msg], fee: estimatedFee, memo: params.memo };
  }
  
  // Cosmos 特有：IBC 转账、Staking、Governance 消息类型
}
```

## 密钥管理的多链挑战

### 多曲线支持

```typescript
interface Signer {
  curve: 'secp256k1' | 'ed25519' | 'secp256r1';
  sign(message: Uint8Array): Promise<Signature>;
  getPublicKey(): Uint8Array;
}

class HDKeyManager {
  private seed: Uint8Array;
  
  deriveSigner(chain: ChainType, accountIndex: number): Signer {
    const path = this.getDerivationPath(chain, accountIndex);
    
    switch (chain) {
      case 'evm':
      case 'bitcoin':
      case 'cosmos':
        // secp256k1 曲线
        return new Secp256k1Signer(derivePath(this.seed, path));
        
      case 'solana':
      case 'aptos':
      case 'sui':
        // Ed25519 曲线
        return new Ed25519Signer(derivePath(this.seed, path));
        
      case 'passkey':
        // P-256 曲线（WebAuthn）
        return new P256Signer(/* from Passkey API */);
    }
  }
  
  private getDerivationPath(chain: ChainType, account: number): string {
    const coinTypes: Record<string, number> = {
      bitcoin: 0,
      evm: 60,
      solana: 501,
      cosmos: 118,
      aptos: 637,
      sui: 784,
    };
    return `m/44'/${coinTypes[chain]}'/${account}'/0/0`;
  }
}
```

### 同一助记词，不同链的地址

```
助记词: "abandon ability able about above absent..."

Bitcoin (secp256k1, m/44'/0'/0'/0/0):
  → bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4

Ethereum (secp256k1, m/44'/60'/0'/0/0):
  → 0x71C7656EC7ab88b098defB751B7401B5f6d8976F

Solana (ed25519, m/44'/501'/0'/0'):
  → 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU

Cosmos (secp256k1, m/44'/118'/0'/0/0):
  → cosmos1qypqxpq9qcrsszg2pvxq6rs0zqg3yyc5lzv7xu
```

## 数据层架构

### RPC 管理

```typescript
class RPCManager {
  private endpoints: Map<string, RPCEndpoint[]>;
  private healthChecker: HealthChecker;
  
  async call(chain: string, method: string, params: any[]): Promise<any> {
    const endpoints = this.endpoints.get(chain);
    
    // 负载均衡 + 故障转移
    for (const endpoint of this.sortByHealth(endpoints)) {
      try {
        const result = await endpoint.call(method, params);
        this.healthChecker.recordSuccess(endpoint);
        return result;
      } catch (error) {
        this.healthChecker.recordFailure(endpoint);
        continue; // 尝试下一个节点
      }
    }
    
    throw new Error(`All RPC endpoints for ${chain} are unavailable`);
  }
}
```

### 数据索引策略

不同链的数据获取方式差异很大：

| 链 | 余额查询 | 代币余额 | 交易历史 |
|---|---|---|---|
| EVM | eth_getBalance | 逐个合约查询 / Multicall | 事件日志扫描 / 索引服务 |
| Solana | getBalance | getTokenAccountsByOwner | getSignaturesForAddress |
| Bitcoin | 需要 UTXO 索引 | N/A（只有 BTC） | 地址交易索引 |
| Cosmos | bank/balances | bank/balances（原生多资产） | tx/search |

### 缓存策略

```typescript
class DataCache {
  // 余额：短缓存（10-30秒），因为可能随时变化
  // 代币列表：中缓存（5-15分钟）
  // 交易历史：长缓存（直到有新交易）
  // 价格数据：短缓存（30-60秒）
  // 链配置：长缓存（很少变化）
  
  private caches: Map<string, CacheEntry> = new Map();
  
  async getWithCache<T>(
    key: string, 
    ttl: number, 
    fetcher: () => Promise<T>
  ): Promise<T> {
    const cached = this.caches.get(key);
    if (cached && Date.now() - cached.timestamp < ttl) {
      return cached.data as T;
    }
    const data = await fetcher();
    this.caches.set(key, { data, timestamp: Date.now() });
    return data;
  }
}
```

## EVM 多链的特殊优势

EVM 兼容链共享大量基础设施：

```
共享的部分：
✅ 地址格式（0x + 40 hex）
✅ 交易格式（RLP 编码）
✅ 签名算法（secp256k1 ECDSA）
✅ ABI 编码
✅ RPC 接口（eth_* 方法）
✅ 合约标准（ERC-20、ERC-721...）

不同的部分：
❌ Chain ID
❌ Gas 参数（不同链的 Gas 机制可能有差异）
❌ 区块时间
❌ 预编译合约
❌ 部分 opcode 行为
```

这意味着一个 EVM 适配器可以通过配置支持几十条链，而非 EVM 链每条都需要独立的适配器。

## 跨链操作的技术实现

### 跨链桥集成

```typescript
interface BridgeAdapter {
  // 查询可用路由
  getRoutes(params: {
    fromChain: string;
    toChain: string;
    token: string;
    amount: BigNumber;
  }): Promise<BridgeRoute[]>;
  
  // 构建跨链交易
  buildBridgeTx(route: BridgeRoute): Promise<UnsignedTx>;
  
  // 查询跨链状态
  getBridgeStatus(txHash: string): Promise<BridgeStatus>;
}

// 聚合多个桥
class BridgeAggregator {
  private bridges: BridgeAdapter[]; // LI.FI, Socket, Across, Stargate...
  
  async getBestRoute(params: RouteParams): Promise<BridgeRoute> {
    const allRoutes = await Promise.all(
      this.bridges.map(b => b.getRoutes(params))
    );
    // 按费用、速度、安全性排序
    return this.rankRoutes(allRoutes.flat());
  }
}
```

## 测试策略

多链架构的测试特别复杂：

```
单元测试：
- 每个适配器的地址派生
- 交易构建逻辑
- 签名验证

集成测试：
- 各链测试网的端到端交易
- RPC 故障转移
- 缓存一致性

模拟测试：
- Fork 主网状态进行测试（Hardhat/Foundry for EVM）
- 模拟网络延迟和故障
```

## 小结

多链钱包的架构核心是**适配器模式**——用统一的接口抽象不同链的差异，让上层应用代码不需要关心底层链的具体实现。

关键的工程决策：
1. **接口设计**：足够通用以覆盖所有链，又足够具体以暴露各链特性
2. **代码复用**：EVM 链之间最大化复用，非 EVM 链独立实现
3. **数据策略**：不同链用不同的数据获取和缓存策略
4. **错误处理**：优雅地处理各链的特殊错误和边界情况

这是一个持续演进的架构——每当新链出现，就需要新的适配器。好的架构设计让添加新链的成本尽可能低。
