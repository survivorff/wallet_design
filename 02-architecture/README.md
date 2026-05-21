# 02 技术架构深度解析

## 本章内容

把钱包底层的技术栈拆开，从密码学原语到系统架构，逐层讲透。

## 文章列表

- [x] [密钥管理基础：私钥、公钥、地址的关系](./01-keys-addresses.md)
- [x] [助记词与 HD 钱包：BIP-32/39/44 详解](./02-hd-wallets-bip32-39-44.md)
- [x] [签名方案对比：ECDSA vs Schnorr vs EdDSA](./03-signature-schemes.md)
- [x] [MPC/TSS 技术原理：密钥不在一处的安全哲学](./04-mpc-tss-technical.md)
- [x] [账户抽象（AA）：ERC-4337 架构全解](./05-account-abstraction-erc4337.md)
- [ ] 智能合约钱包的技术实现：Proxy 模式与可升级性
- [ ] 多链架构设计：如何支持 EVM/非 EVM/Move 链
- [ ] 交易构建与广播：从用户意图到链上确认
- [ ] RPC 节点与数据索引：钱包的"眼睛"
- [ ] 本地存储与加密：Keystore、Secure Enclave、TEE

## 核心问题

1. 钱包的安全边界在哪里？
2. MPC 和 AA 是互补还是替代关系？
3. 多链支持的技术复杂度在哪里？
4. 去中心化与用户体验如何平衡？
