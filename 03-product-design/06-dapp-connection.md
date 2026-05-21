# DApp 浏览器与 WalletConnect：连接体验设计

## 一句话总结

钱包与 DApp 的连接是 Web3 交互的起点——连接体验的好坏直接决定了用户能否顺利进入链上世界。

## 连接方式演进

```
2016: Provider 注入（MetaMask 定义）
2018: WalletConnect v1（二维码连接）
2022: WalletConnect v2（多链、多会话）
2023: EIP-6963（多钱包发现）
2024: 深度链接 + Universal Links 成熟
未来: 无感连接（嵌入式钱包不需要"连接"步骤）
```

## 方式一：Provider 注入（浏览器插件）

### 工作原理

```
┌─────────────────────────────────────────────┐
│  浏览器                                      │
│                                              │
│  ┌──────────────┐    ┌──────────────────┐   │
│  │  DApp 网页    │    │  钱包插件         │   │
│  │              │    │                  │   │
│  │  检测        │    │  注入            │   │
│  │  window.     │←───│  window.ethereum │   │
│  │  ethereum    │    │                  │   │
│  │              │───→│  处理请求         │   │
│  │  发送请求    │    │  弹窗确认         │   │
│  │              │←───│  返回结果         │   │
│  └──────────────┘    └──────────────────┘   │
└─────────────────────────────────────────────┘
```

### EIP-1193 标准接口

```typescript
interface EIP1193Provider {
  // 发送请求
  request(args: { method: string; params?: any[] }): Promise<any>;
  
  // 事件监听
  on(event: string, listener: (...args: any[]) => void): void;
  removeListener(event: string, listener: (...args: any[]) => void): void;
}

// DApp 使用示例
const accounts = await window.ethereum.request({ 
  method: 'eth_requestAccounts' 
});

const chainId = await window.ethereum.request({ 
  method: 'eth_chainId' 
});

// 监听账户切换
window.ethereum.on('accountsChanged', (accounts) => {
  console.log('用户切换了账户:', accounts[0]);
});

// 监听网络切换
window.ethereum.on('chainChanged', (chainId) => {
  console.log('网络切换到:', chainId);
  window.location.reload(); // 通常需要刷新页面
});
```

### EIP-6963：多钱包共存

**问题**：多个钱包插件都想注入 `window.ethereum`，导致冲突。

```typescript
// 旧方式：只有一个 window.ethereum，后安装的覆盖先安装的
// 用户装了 MetaMask 和 Rabby，只有一个能生效

// EIP-6963 新方式：钱包通过事件宣布自己的存在
window.addEventListener('eip6963:announceProvider', (event) => {
  const { info, provider } = event.detail;
  // info: { uuid, name, icon, rdns }
  // provider: EIP-1193 Provider 实例
  
  // DApp 可以展示所有可用钱包让用户选择
  availableWallets.push({ info, provider });
});

// DApp 请求发现钱包
window.dispatchEvent(new Event('eip6963:requestProvider'));
```

效果：用户可以在连接时看到所有已安装的钱包，自由选择。

## 方式二：WalletConnect

### 为什么需要 WalletConnect

```
场景：用户在电脑浏览器上使用 DApp，但钱包在手机上
场景：用户在手机浏览器上使用 DApp，钱包是另一个 App
场景：DApp 是桌面应用，不是网页

这些场景下 Provider 注入不可用 → 需要跨设备/跨应用的连接方案
```

### WalletConnect v2 架构

```
┌──────────┐         ┌──────────────┐         ┌──────────┐
│  DApp    │ ──WSS──→│  Relay Server │←──WSS── │  Wallet  │
│          │         │  (中继服务器)  │         │          │
│  生成    │         │              │         │  扫描    │
│  二维码  │         │  转发加密消息 │         │  二维码  │
│          │←────────│              │────────→│          │
│  收到    │         │              │         │  签名    │
│  结果    │         │              │         │  确认    │
└──────────┘         └──────────────┘         └──────────┘

通信流程：
1. DApp 生成配对 URI（包含加密密钥）
2. 用户用钱包扫描二维码（获取配对信息）
3. 钱包与 DApp 通过 Relay 建立加密通道
4. DApp 发送请求 → Relay 转发 → 钱包处理 → 返回结果
```

### v2 相比 v1 的改进

| 特性 | v1 | v2 |
|------|----|----|
| 多链支持 | 单链会话 | 多链会话 |
| 会话管理 | 一个 DApp 一个会话 | 灵活的会话命名空间 |
| 中继 | 单一 Bridge 服务器 | 去中心化 Relay 网络 |
| 加密 | 对称加密 | X25519 + ChaCha20 |
| 配对 | 每次新连接 | 可复用配对 |
| 推送通知 | 不支持 | 支持 |

### 连接体验的痛点

```
当前 WalletConnect 的用户体验问题：

1. 连接步骤多
   打开 DApp → 点击连接 → 选择 WalletConnect → 
   显示二维码 → 打开钱包 App → 扫码 → 确认连接
   = 7 步操作

2. 连接不稳定
   - WebSocket 断连
   - 手机锁屏后会话丢失
   - 需要重新连接

3. 签名延迟
   - DApp 发送请求 → 等待推送通知 → 用户打开钱包 → 确认
   - 整个流程可能需要 10-30 秒

4. 多链切换
   - 切换网络需要钱包端确认
   - 有时候切换失败需要重新连接
```

## 方式三：Deep Link / Universal Link

### 移动端原生连接

```
流程：
1. DApp 构造一个特殊 URL
   phantom://connect?redirect_url=...&app_url=...
   
2. 系统打开对应的钱包 App

3. 钱包 App 处理请求，用户确认

4. 钱包通过 redirect_url 返回结果给 DApp

优势：
- 原生体验，不需要扫码
- 速度快（直接唤起 App）
- 不依赖中继服务器

劣势：
- 需要 DApp 知道钱包的 URL Scheme
- 不同钱包有不同的 Scheme
- 跨平台兼容性问题
```

## 方式四：嵌入式钱包（无需连接）

```
传统模式：
用户 → 安装钱包 → 打开 DApp → 连接钱包 → 使用

嵌入式模式：
用户 → 打开 App → 使用（钱包在后台自动创建和管理）

技术实现：
- App 集成 Privy/Dynamic/Particle SDK
- 用户登录时自动创建钱包
- 签名由 SDK 在后台处理
- 用户可能完全不知道"钱包"的存在
```

这是"连接"问题的终极解决方案——消除连接步骤本身。

## 连接体验的设计原则

### 1. 减少步骤

```
❌ 差的体验（7 步）：
点击连接 → 弹出模态框 → 选择连接方式 → 选择钱包 → 
显示二维码 → 扫码 → 确认

✅ 好的体验（2-3 步）：
点击连接 → 选择钱包（自动检测已安装的）→ 确认
```

### 2. 记住用户选择

```
首次连接：展示所有选项
再次访问：自动连接上次使用的钱包（或一键重连）
```

### 3. 状态清晰

```
连接状态应该随时可见：
- 当前连接的钱包
- 当前网络
- 当前账户地址
- 连接是否健康（WalletConnect 会话是否活跃）
```

### 4. 优雅降级

```
如果首选连接方式失败：
- WalletConnect 断连 → 提示重新扫码
- 插件钱包未安装 → 引导安装或提供替代方案
- 网络不匹配 → 自动请求切换网络
```

## DApp 浏览器设计

### 移动端钱包内置浏览器

```
┌─────────────────────────────────────┐
│  钱包 App                            │
│  ┌─────────────────────────────┐    │
│  │  内置 WebView                │    │
│  │                              │    │
│  │  ┌──────────────────────┐   │    │
│  │  │  DApp 网页            │   │    │
│  │  │                      │   │    │
│  │  │  window.ethereum     │   │    │
│  │  │  (由钱包注入)        │   │    │
│  │  └──────────────────────┘   │    │
│  │                              │    │
│  │  [← ] [→] [🔄] [URL 栏]    │    │
│  └─────────────────────────────┘    │
│                                      │
│  [首页] [浏览器] [Swap] [设置]       │
└─────────────────────────────────────┘
```

设计考量：
- URL 栏需要显示安全状态（是否 HTTPS、是否已知 DApp）
- 收藏夹/历史记录
- DApp 推荐/发现页面
- 网络切换控件
- 连接状态指示

### 安全考量

```
内置浏览器的风险：
- WebView 可能有安全漏洞
- 恶意网页可能尝试攻击 WebView
- 注入的 Provider 可能被恶意脚本利用

防御措施：
- 使用最新版本的 WebView 引擎
- 限制 WebView 的权限（不允许访问本地文件等）
- 对注入的 Provider 做权限控制
- 恶意网站黑名单
- 每次签名都需要用户在原生 UI 中确认（不在 WebView 中）
```

## 小结

DApp 连接是钱包产品体验的"第一印象"——如果连接都搞不定，后续的一切都无从谈起。

演进方向很清晰：
1. **短期**：EIP-6963 改善多钱包共存、WalletConnect v2 改善跨设备体验
2. **中期**：Deep Link + Universal Link 让移动端连接更流畅
3. **长期**：嵌入式钱包消除"连接"步骤本身

最终目标是让"连接钱包"这个动作变得像"登录"一样自然——甚至比登录更简单。
