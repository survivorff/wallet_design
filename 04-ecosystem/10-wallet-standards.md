# 钱包标准与协议：EIP-1193、EIP-6963、WalletConnect v2

## 一句话总结

标准化协议让"任何钱包都能连接任何 DApp"成为现实——理解这些标准，就理解了钱包生态如何避免碎片化。

## 主要标准概览

| 标准 | 作用 | 状态 |
|------|------|------|
| EIP-1193 | Provider API | 广泛采用 |
| EIP-1474 | JSON-RPC 标准方法 | 广泛采用 |
| EIP-6963 | 多 Provider 发现 | 普及中 |
| EIP-3326 | 切换网络 | 广泛采用 |
| EIP-3085 | 添加网络 | 广泛采用 |
| EIP-4361 (SIWE) | 钱包登录 | 普及中 |
| EIP-712 | 类型化数据签名 | 广泛采用 |
| EIP-2255 | 权限请求 | 部分采用 |
| WalletConnect v2 | 跨设备连接 | 广泛采用 |
| ERC-4337 | 账户抽象 | 普及中 |

## EIP-1193：Provider API

### 核心接口

```typescript
interface EIP1193Provider {
  request(args: { method: string; params?: any[] }): Promise<any>;
  on(event: string, listener: Function): void;
  removeListener(event: string, listener: Function): void;
  
  // 已废弃但仍存在
  send?(method: string, params?: any[]): Promise<any>;
  sendAsync?(payload: any, callback: Function): void;
}
```

### 标准事件

```typescript
// 账户变更
provider.on('accountsChanged', (accounts: string[]) => {
  // 用户切换了账户或断开连接
});

// 网络变更
provider.on('chainChanged', (chainId: string) => {
  // 用户切换了网络（DApp 通常需要刷新）
});

// 连接事件
provider.on('connect', ({ chainId }: { chainId: string }) => {
  // 钱包连接成功
});

// 断开事件
provider.on('disconnect', ({ code, message }: { code: number; message: string }) => {
  // 钱包断开
});

// 消息事件
provider.on('message', (message: ProviderMessage) => {
  // 钱包推送消息（如订阅事件）
});
```

### 标准方法

```typescript
// 请求账户
const accounts = await provider.request({
  method: 'eth_requestAccounts'
});

// 获取链 ID
const chainId = await provider.request({
  method: 'eth_chainId'
});

// 发送交易
const txHash = await provider.request({
  method: 'eth_sendTransaction',
  params: [{ from, to, value, data, gas, ... }]
});

// 个人签名
const signature = await provider.request({
  method: 'personal_sign',
  params: [message, address]
});

// EIP-712 签名
const signature = await provider.request({
  method: 'eth_signTypedData_v4',
  params: [address, JSON.stringify(typedData)]
});
```

## EIP-6963：多钱包发现

### 解决的问题

```
传统方式：钱包注入 window.ethereum
问题：多个钱包冲突，后注入的覆盖先注入的

EIP-6963 方式：钱包通过事件宣告自己
DApp 监听事件，发现所有可用钱包
```

### 协议流程

```typescript
// 钱包：宣告自己
window.dispatchEvent(
  new CustomEvent('eip6963:announceProvider', {
    detail: Object.freeze({
      info: {
        uuid: '350670db-19fa-4704-a166-e52e178b59d2',
        name: 'MetaMask',
        icon: 'data:image/svg+xml;base64,...',
        rdns: 'io.metamask',
      },
      provider: providerInstance,
    })
  })
);

// DApp：监听并收集所有钱包
const wallets: ProviderDetail[] = [];

window.addEventListener('eip6963:announceProvider', (event) => {
  wallets.push(event.detail);
});

// DApp：请求钱包宣告
window.dispatchEvent(new Event('eip6963:requestProvider'));
```

### 实际效果

```
用户在浏览器装了 MetaMask、Rabby、Phantom：

旧方式：
- window.ethereum 只有一个（取决于安装顺序）
- 用户无法选择

EIP-6963：
- DApp 显示三个钱包供选择
- 用户自由选择使用哪个
- 切换钱包不需要禁用其他扩展
```

## EIP-712：结构化数据签名

### 为什么需要

```
传统签名（personal_sign）：
"Sign this message: 0xabc123..."
用户：???（看不懂十六进制）

EIP-712 签名：
{
  "types": { ... },
  "primaryType": "Order",
  "domain": { name: "Uniswap", chainId: 1, verifyingContract: "0x..." },
  "message": {
    "tokenIn": "USDC",
    "tokenOut": "ETH",
    "amount": "1000",
    ...
  }
}

钱包可以展示结构化的数据，用户更容易理解
```

### 关键特性

```
1. 域分隔符（Domain Separator）
   - 防止跨应用重放
   - 包含 chainId、合约地址、版本

2. 类型化数据
   - 定义数据结构
   - 钱包可以按类型展示

3. 主类型（Primary Type）
   - 标识签名的目的
   - "Order"、"Permit"、"Transfer" 等

4. 哈希计算
   - 确定性的哈希算法
   - 链上合约可验证签名
```

### 应用场景

```
- Gasless 授权（Permit）
- 链下订单（Uniswap、OpenSea）
- 跨应用授权
- DApp 登录（SIWE）
```

## EIP-4361（SIWE）：钱包登录

### 标准消息格式

```
${domain} wants you to sign in with your Ethereum account:
${address}

${statement}

URI: ${uri}
Version: ${version}
Chain ID: ${chainId}
Nonce: ${nonce}
Issued At: ${issuedAt}
${expirationTime ? `Expiration Time: ${expirationTime}` : ''}
${notBefore ? `Not Before: ${notBefore}` : ''}
${requestId ? `Request ID: ${requestId}` : ''}
${resources ? `Resources:\n${resources.join('\n')}` : ''}
```

### 关键字段

```
domain：发起请求的网站（防钓鱼）
address：用户地址
statement：可选的人类可读说明
URI：完整 URL
version：协议版本
chainId：用户期望的链
nonce：随机字符串，防重放
issuedAt：签名时间
expirationTime：过期时间（防止旧签名滥用）
```

## WalletConnect v2

### 关键改进 vs v1

| | v1 | v2 |
|---|----|----|
| 多链 | ❌ | ✅ Namespaces |
| 多会话 | ❌ | ✅ |
| 推送通知 | ❌ | ✅ |
| Relay 网络 | 单一 Bridge | 去中心化 |
| 加密 | AES | X25519 + ChaCha20 |
| 配对 | 一次性 | 可复用 |

### Namespaces

```typescript
// DApp 请求会话
{
  requiredNamespaces: {
    eip155: {
      methods: ['eth_sendTransaction', 'personal_sign'],
      chains: ['eip155:1', 'eip155:137'],
      events: ['accountsChanged', 'chainChanged'],
    },
  },
  optionalNamespaces: {
    eip155: {
      methods: ['eth_signTypedData'],
      chains: ['eip155:42161'],
      events: [],
    },
  },
}

// 钱包返回支持的部分
```

## ERC-4337：账户抽象

### 标准化的价值

```
没有 ERC-4337 之前：
- 各种合约钱包实现不互通
- DApp 需要为每种合约钱包做适配
- Bundler/Paymaster 难以建立

ERC-4337 之后：
- 统一的 UserOperation 格式
- 统一的 EntryPoint 合约
- 任何 Bundler 可以处理任何合约钱包的 UserOp
- 任何 Paymaster 可以为任何合约钱包代付
```

## 标准化的趋势

### 模块化标准

```
ERC-7579（最小化模块账户）：
- 标准化合约钱包的模块接口
- 不同 AA 实现可以共享模块
- 用户可以在不同 AA 钱包间切换

ERC-6900（模块化账户）：
- 更复杂的模块化标准
- 由 Alchemy 主导
```

### 跨链标准

```
ERC-7683（跨链意图）：
- 标准化跨链订单格式
- 不同跨链协议可以互通
- 钱包可以聚合多个跨链方案
```

## 标准的局限

### 标准化的代价

```
1. 演进缓慢
   - EIP 流程需要时间
   - 需要多方共识
   - 创新被标准约束

2. 最低公分母
   - 标准只能涵盖通用功能
   - 钱包独特功能无法通过标准暴露

3. 实现差异
   - 同一标准不同钱包实现可能有差异
   - 边界情况处理不一致
   - 仍需要测试和适配
```

## 小结

钱包标准化协议是 Web3 生态健康发展的基石。它们让：
- 用户可以自由选择钱包
- 开发者可以一次集成支持所有钱包
- 钱包之间公平竞争（靠产品力而非垄断）
- 创新可以在标准之上构建

理解这些标准对所有 Web3 从业者都是必要的——无论你是钱包开发者、DApp 开发者还是产品经理，标准都决定了你能做什么、不能做什么。
