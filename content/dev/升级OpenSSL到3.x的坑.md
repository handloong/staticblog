---
title: 升级OpenSSL到3.x的坑
date: 2026-05-06
description: 
toc: true
slug: pitfalls-upgrading-openssl-to-3-x
categories:
    - Dev
    - 技术分享
tags:
    - openssl
---

## 背景

从 CentOS 7 升级到新系统（如 Rocky Linux 9 / CentOS Stream 9）后，系统自带的 OpenSSL 从 1.x 升级到了 3.x。OpenSSL 3.0 出于安全考虑，默认禁用了 SHA1 签名算法——因为 SHA1 已被证明存在碰撞攻击风险，不再被认为是安全的签名算法。

然而，如果 SFTP 服务器的 SSH 主机密钥或客户端证书仍在使用 SHA1 签名，升级后连接时就会报错类似：

```
signature algorithm sha1 not in peer list
```

## 解决方案

### 临时方案

在客户端设置环境变量，允许 OpenSSL 3.x 使用 SHA1 签名：

```shell
export OPENSSL_ENABLE_SHA1_SIGNATURES=1
```

**注意**：这仅应在过渡期使用，SHA1 签名的安全性已被攻破。

### 根本方案

在 **SFTP 服务器端** 升级密钥和证书，改用更安全的签名算法（如 SHA-256、SHA-512）：

```shell
# 例如重新生成 SSH 主机密钥
ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
```

## 参考资料

- [Fedora SHA1 Signatures Guidance](https://fedoraproject.org/wiki/SHA1SignaturesGuidance)