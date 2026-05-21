# 钱包与 DApp 的关系：连接协议与标准

## 一句话总结

钱包和 DApp 之间是一种"互相依赖又互相博弈"的关系——DApp 需要钱包让用户操作，钱包需要 DApp 让用户使用，但谁是主导方决定了价值如何分配。

## 关系演进

### 阶段一：钱包是配件（2015-2017）

```
DApp 是主角，钱包是必需的配件
- DApp 在自己网站上引导用户安装 MetaMask
- 钱包仅仅是签名工具
- 用户对钱包的认知：DApp 要我装的东西
```

### 阶段二：钱包是入口（2018-2021）

```
随着 MetaMask 用户量增长：
- 钱包开始有"DApp 浏览器"
- 用户从钱包发现 DApp
- DApp 的获客依赖钱包的推荐
```

### 阶段三：钱包是平台（2022-至今）

```
钱包内置越来越多功能：
- 内置 Swap 替代 DEX 前端
- 内置 NFT 市场替代 OpenSea
- 内置 Bridge 替代独立桥前端
- DApp 的"前端"被钱包蚕食
```

### 阶段四：钱包是渠道（未来）

```
嵌入式钱包模式下：
- DApp 集成钱包 SDK
- 钱包变成"水电"基础设施
- DApp 重新成为前台主角
- 但钱包通过 SDK 收费/分润
```

## 连接协议标准

### EIP-1193：Provider API

```typescript
// DApp 通过统一接口与任何钱包通信
interface Provider {
  request(args: { method: string; params?: any[] }): Promise<any>;
  on(event: string, listener: (...args: any[]) => void): void;
}

// 标准方法：
- eth_requestAccounts: 请求连接
- eth_chainId: 获取当前链
- eth_sendTransaction: 发送交易
- personal_sign: 签名消息
- wallet_switchEthereumChain: 切换网络
- wallet_addEthereumChain: 添加网络
```

### EIP-6963：多钱包发现

```typescript
// 解决"window.ethereum 冲突"问题
// 钱包通过事件宣布自己

window.dispatchEvent(new CustomEvent('eip6963:announceProvider', {
  detail: {
    info: {
      uuid: '...',
      name: 'MetaMask',
      icon: 'data:image/...',
      rdns: 'io.metamask',
    },
    provider: provider,
  }
}));
```

### EIP-4361：Sign-In with Ethereum (SIWE)

```
标准化的"用钱包登录"消息格式：

example.com wants you to sign in with your Ethereum account:
0xUserAddress

Optional statement here.

URI: https://example.com
Version: 1
Chain ID: 1
Nonce: 32891756
Issued At: 2024-01-15T12:00:00Z
```

### WalletConnect 协议

```
跨设备/跨平台的连接协议：
- v1（已弃用）：单链、单会话
- v2：多链、多会话、推送通知

核心概念：
- Pairing：建立钱包-DApp 信任
- Session：长期连接（默认 7 天）
- Namespaces：定义会话支持的链和方法
- Relay：消息中继网络
```

## 钱包-DApp 的博弈

### 价值分配的争夺

```
谁能向用户收钱？

DApp 视角：
- 我们提供服务（DEX、借贷、NFT 市场）
- 用户因为我们的产品才来
- 我们应该收手续费

钱包视角：
- 我们控制用户入口
- 没有钱包用户连不上 DApp
- 我们也应该收一份

实际情况：
- DApp 收基础手续费（如 Uniswap 的 LP 费）
- 钱包通过聚合 + 加价收费（如 MetaMask Swap 的 0.875%）
- 用户可能为同一笔操作"被收两次费"
```

### 用户归属的争夺

```
谁拥有用户关系？

钱包：
- 用户的资产在钱包里
- 用户的交易历史在钱包里
- 用户的身份在钱包里
- 钱包可以推送通知给用户

DApp：
- DApp 提供具体的服务和价值
- 用户在 DApp 中花费时间
- DApp 有自己的社区和品牌

结论：钱包拥有"账户层"用户关系，DApp 拥有"服务层"用户关系
```

## 标准化的好处与代价

### 好处

```
对用户：
- 任何钱包都能连接任何 DApp
- 切换钱包不丢失 DApp 访问
- 一致的连接体验

对 DApp：
- 不需要为每个钱包做适配
- 一次集成支持所有钱包
- 降低开发成本

对钱包：
- 不需要 DApp 主动支持
- 可以无缝替代现有钱包
- 公平竞争
```

### 代价

```
对头部钱包：
- 失去专有优势（"只有 MetaMask 能连接 Uniswap"不再成立）
- 网络效应被削弱
- 必须靠产品力竞争

对创新：
- 标准协议演进慢
- 创新功能可能被标准限制
- 需要发起 EIP 才能让所有钱包支持新功能
```

## 钱包应该如何与 DApp 协作

### 不破坏 DApp 体验

```
✅ 好的实践：
- 内置 Swap 时，跳转到 DApp 是另一个选项
- 不强制用户在钱包内完成操作
- 提供 DApp 链接（用户可以选择去原生 DApp）

❌ 差的实践：
- 完全替代 DApp 前端，让用户离不开钱包
- 修改 DApp 显示的数据
- 隐瞒 DApp 信息（如真实费率）
```

### 增值而非取代

```
钱包可以为 DApp 增加价值：
- 安全增强（交易模拟、风险提示）
- 用户体验（更好的 Gas 估算）
- 数据展示（聚合多个 DApp 的头寸）
- 自动化（基于 DApp 状态的提醒）

而不是简单地复制 DApp 的功能
```

## 小结

钱包和 DApp 是 Web3 生态中最重要的两个角色，它们的关系决定了价值如何在生态中分配。最健康的状态是合作共赢：

- DApp 专注于核心服务和创新
- 钱包专注于用户体验和安全
- 通过标准协议互相协作
- 在收入分配上找到平衡

但实际上，两者的边界在不断模糊——钱包越来越像平台，DApp 越来越想直接对接用户（嵌入式钱包）。这种博弈会持续下去，最终的平衡点取决于哪一方更能持续给用户创造价值。
