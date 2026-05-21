# 本地存储与加密：Keystore、Secure Enclave、TEE

## 一句话总结

私钥必须存储在某个地方——如何在"可用性"和"不可提取性"之间找到平衡，是钱包本地安全的核心问题。

## 存储方案光谱

```
安全性低 ←─────────────────────────────────→ 安全性高
便利性高                                      便利性低

明文存储 → 密码加密 → 系统Keychain → Secure Enclave → 硬件钱包
(最危险)   (Keystore)  (OS保护)     (硬件隔离)      (物理隔离)
```

## 方案一：Keystore 文件（密码加密）

### 以太坊 Keystore V3 格式

```json
{
  "version": 3,
  "id": "uuid",
  "address": "0x...",
  "crypto": {
    "cipher": "aes-128-ctr",
    "ciphertext": "加密后的私钥（十六进制）",
    "cipherparams": {
      "iv": "初始化向量"
    },
    "kdf": "scrypt",
    "kdfparams": {
      "dklen": 32,
      "n": 262144,
      "p": 1,
      "r": 8,
      "salt": "随机盐值"
    },
    "mac": "消息认证码（验证密码正确性）"
  }
}
```

### 加密流程

```
用户密码
    │
    │ KDF（密钥派生函数：scrypt/argon2）
    │ 参数：salt, n=262144, r=8, p=1
    │ 目的：让暴力破解变慢
    ▼
派生密钥（32 字节）
    │
    ├──→ 前 16 字节：AES 加密密钥
    │         │
    │         │ AES-128-CTR 加密
    │         ▼
    │    私钥 → 密文（ciphertext）
    │
    └──→ 后 16 字节 + 密文 → Keccak256 → MAC
                                          │
                                          ▼
                                    验证密码正确性
```

### 解密流程

```
1. 用户输入密码
2. 用相同的 KDF 参数派生密钥
3. 用派生密钥的后半部分 + 密文计算 MAC
4. 比较计算的 MAC 与存储的 MAC
   - 不匹配 → 密码错误
   - 匹配 → 用前半部分解密得到私钥
```

### KDF 的作用

KDF（Key Derivation Function）的目的是让暴力破解变慢：

| KDF | 特点 | 单次计算时间 |
|-----|------|-------------|
| scrypt (n=262144) | 内存密集型 | ~1 秒 |
| Argon2id | 更现代，抗 GPU/ASIC | ~1 秒 |
| PBKDF2 (100K rounds) | CPU 密集型 | ~0.5 秒 |

如果暴力破解每次尝试需要 1 秒：
- 6 位数字密码（10^6）：~11 天可破解 ❌
- 8 位混合密码（62^8）：~7000 亿年 ✅

### Keystore 的局限

- 安全性完全依赖密码强度
- 私钥在解密时以明文存在于内存中
- 恶意软件可以在解密瞬间截获
- 用户可能设置弱密码

## 方案二：操作系统 Keychain/Keystore

### iOS Keychain

```
┌─────────────────────────────────────┐
│           iOS Keychain               │
├─────────────────────────────────────┤
│  访问控制：                          │
│  - 设备解锁后可访问                  │
│  - 需要生物识别（Face ID/Touch ID）  │
│  - App 沙箱隔离                     │
│                                      │
│  存储特性：                          │
│  - 硬件加密（与设备绑定）            │
│  - 备份到 iCloud Keychain（可选）    │
│  - 设备锁定时数据不可访问            │
│                                      │
│  API：                               │
│  SecItemAdd / SecItemCopyMatching    │
└─────────────────────────────────────┘
```

```swift
// iOS 存储私钥到 Keychain
let query: [String: Any] = [
    kSecClass: kSecClassGenericPassword,
    kSecAttrAccount: "wallet_private_key",
    kSecValueData: privateKeyData,
    kSecAttrAccessible: kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    // 只有设备解锁时可访问，不会备份到其他设备
    kSecAttrAccessControl: SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        .biometryCurrentSet, // 需要当前注册的生物识别
        nil
    )
]
SecItemAdd(query as CFDictionary, nil)
```

### Android Keystore

```
┌─────────────────────────────────────┐
│         Android Keystore             │
├─────────────────────────────────────┤
│  硬件支持（如果设备有 TEE/SE）：     │
│  - 密钥在硬件中生成和使用            │
│  - 密钥材料不可导出                  │
│  - 签名操作在安全环境中完成          │
│                                      │
│  访问控制：                          │
│  - 需要用户认证（指纹/PIN）          │
│  - 密钥使用有时间限制               │
│  - App 隔离                         │
│                                      │
│  API：                               │
│  KeyStore / KeyGenerator             │
└─────────────────────────────────────┘
```

```kotlin
// Android 在 Keystore 中生成密钥
val keyGenerator = KeyGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
)
keyGenerator.init(
    KeyGenParameterSpec.Builder("wallet_key", 
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
        .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
        .setUserAuthenticationRequired(true) // 需要用户认证
        .setUserAuthenticationValidityDurationSeconds(30) // 认证有效 30 秒
        .build()
)
keyGenerator.generateKey()
```

### 系统 Keychain 的优势与局限

**优势**：
- 操作系统级保护，比应用层加密更安全
- 生物识别集成
- 硬件加密支持（如果设备有 TEE）
- 应用沙箱隔离

**局限**：
- 不同平台 API 不同（跨平台开发复杂）
- 密钥可能随设备备份被复制（需要正确配置）
- Root/越狱设备上保护失效
- 曲线支持有限（可能不支持 secp256k1）

## 方案三：Secure Enclave / TEE

### Apple Secure Enclave

```
┌─────────────────────────────────────────────────┐
│                 主处理器                          │
│  ┌───────────────────────────────────────────┐  │
│  │  应用层                                    │  │
│  │  钱包 App → 请求签名                       │  │
│  └───────────────────────────────────────────┘  │
│                      │                           │
│                      │ 签名请求                   │
│                      ▼                           │
│  ┌───────────────────────────────────────────┐  │
│  │  Secure Enclave（独立处理器）              │  │
│  │                                            │  │
│  │  - 密钥在此生成，永不离开                  │  │
│  │  - 签名在此完成                            │  │
│  │  - 支持 P-256 (secp256r1) 曲线            │  │
│  │  - 不支持 secp256k1 ❌                     │  │
│  │  - 防物理篡改                              │  │
│  │                                            │  │
│  │  返回：签名结果（不是私钥）                │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

**关键限制**：Apple Secure Enclave 只支持 P-256 曲线，不支持比特币/以太坊使用的 secp256k1。

**解决方案**：
1. 用 Secure Enclave 的 P-256 密钥加密 secp256k1 私钥（间接保护）
2. 使用 AA 钱包 + P-256 签名验证（Passkey 方案）
3. 等待硬件支持更多曲线

### Android TEE（Trusted Execution Environment）

```
┌─────────────────────────────────────────────────┐
│  Normal World (REE)        │  Secure World (TEE) │
│                            │                     │
│  Android OS               │  Trusted OS          │
│  钱包 App                 │  Trusted App         │
│                            │                     │
│  请求签名 ──────────────→ │  执行签名            │
│  ←────────────────────── │  返回结果            │
│                            │                     │
│  特点：                    │  特点：              │
│  - 可能被恶意软件攻击     │  - 隔离执行环境      │
│  - 内存可被读取           │  - 内存不可被外部访问 │
│                            │  - 密钥不可导出      │
└─────────────────────────────────────────────────┘
```

### StrongBox（Android 硬件安全模块）

部分 Android 设备有独立的安全芯片（类似 Secure Enclave）：

```kotlin
val keyGenerator = KeyGenerator.getInstance(
    KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore"
)
keyGenerator.init(
    KeyGenParameterSpec.Builder("wallet_key", ...)
        .setIsStrongBoxBacked(true) // 要求使用 StrongBox 硬件
        .build()
)
```

## 方案四：浏览器插件的存储

### MetaMask 的存储方式

```
┌─────────────────────────────────────────────────┐
│  浏览器插件存储                                   │
│                                                  │
│  chrome.storage.local / IndexedDB                │
│  ┌──────────────────────────────────────────┐   │
│  │  加密的 Vault                             │   │
│  │  {                                        │   │
│  │    "data": "AES-GCM 加密的密钥数据",      │   │
│  │    "iv": "...",                           │   │
│  │    "salt": "..."                          │   │
│  │  }                                        │   │
│  │                                           │   │
│  │  解密密钥 = PBKDF2(用户密码, salt)        │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  安全边界：                                      │
│  - 浏览器沙箱（其他网页不能访问）                │
│  - 插件隔离（其他插件不能访问）                  │
│  - 但：恶意插件可能绕过隔离                     │
│  - 但：浏览器漏洞可能暴露数据                   │
└─────────────────────────────────────────────────┘
```

### 浏览器存储的风险

| 风险 | 说明 |
|------|------|
| 恶意浏览器扩展 | 可能读取其他扩展的存储 |
| 浏览器漏洞 | 沙箱逃逸 |
| 物理访问 | 电脑未锁定时可以导出 |
| 内存攻击 | 解密后私钥在内存中 |
| 备份泄露 | 浏览器配置文件备份可能包含加密数据 |

## Passkey 存储模型

### WebAuthn / FIDO2

```
┌─────────────────────────────────────────────────┐
│  Passkey 存储                                    │
│                                                  │
│  密钥生成和存储位置：                            │
│  - Apple: Secure Enclave + iCloud Keychain 同步  │
│  - Android: TEE/StrongBox + Google Password Mgr  │
│  - Windows: TPM + Windows Hello                  │
│                                                  │
│  特点：                                          │
│  - 私钥永不离开安全硬件                          │
│  - 用生物识别授权使用                            │
│  - 跨设备同步（通过平台云服务）                  │
│  - 支持 P-256 和 Ed25519                        │
│  - 不支持 secp256k1                             │
│                                                  │
│  用于钱包：                                      │
│  - 需要 AA 钱包（合约验证 P-256 签名）          │
│  - 或用 Passkey 加密/解锁传统私钥               │
└─────────────────────────────────────────────────┘
```

## 内存安全

### 私钥在内存中的风险

即使存储是加密的，使用时私钥必须解密到内存中：

```
攻击窗口：
1. 用户输入密码 → 私钥解密到内存
2. 签名操作使用私钥
3. 签名完成 → 私钥应该从内存中清除

在步骤 1-3 之间，恶意软件可能读取内存
```

### 防御措施

```typescript
// 1. 使用后立即清零
function signAndClear(privateKey: Uint8Array, message: Uint8Array): Signature {
  try {
    return sign(privateKey, message);
  } finally {
    // 将私钥内存清零
    privateKey.fill(0);
  }
}

// 2. 最小化私钥在内存中的时间
// 不要在应用启动时就解密所有私钥
// 只在需要签名时才解密，签名后立即清除

// 3. 使用安全内存分配（如果平台支持）
// mlock() 防止内存被 swap 到磁盘
// mprotect() 设置内存页权限
```

### JavaScript 的特殊挑战

JavaScript（浏览器插件钱包）的内存管理是自动的（GC），无法精确控制：

- 无法保证 `privateKey = null` 后内存立即被清除
- 字符串是不可变的，无法原地清零
- GC 时机不可控
- 无法使用 mlock 等系统调用

部分缓解措施：
- 使用 `Uint8Array` 而非字符串存储密钥（可以 fill(0)）
- 尽量缩短密钥在内存中的生命周期
- 使用 Web Crypto API（密钥可能在更安全的环境中处理）

## 各方案对比总结

| 方案 | 安全级别 | 平台 | 曲线支持 | 可导出 |
|------|---------|------|---------|--------|
| 明文文件 | ⭐ | 全部 | 全部 | 是 |
| Keystore（密码加密） | ⭐⭐ | 全部 | 全部 | 是 |
| 系统 Keychain | ⭐⭐⭐ | iOS/Android/macOS | 有限 | 可配置 |
| TEE/Secure Enclave | ⭐⭐⭐⭐ | 移动设备 | P-256, Ed25519 | 否 |
| 硬件钱包 | ⭐⭐⭐⭐⭐ | 独立设备 | 全部 | 否 |

## 小结

本地存储安全是一个层层递进的防御体系：

1. **最外层**：应用层加密（Keystore），防止文件被直接读取
2. **中间层**：系统 Keychain，利用 OS 的安全机制
3. **最内层**：安全硬件（SE/TEE），密钥永不离开芯片

理想的方案是让私钥永远不以完整形态存在于通用处理器的内存中——这正是 Secure Enclave 和硬件钱包的价值。但由于曲线支持的限制（secp256k1 不被大多数安全硬件原生支持），实际产品中往往需要组合多种方案。

AA 钱包 + Passkey 的组合正在改变这个局面——通过在合约层支持 P-256 验证，让安全硬件可以直接参与签名，不再需要 secp256k1 的妥协。
