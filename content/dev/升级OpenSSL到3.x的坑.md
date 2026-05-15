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

在更新系统的时候，由于系统使用的是centos 7，和sftp连接的时候用的openssl加密方式是SHA1，升级到最新系统报错无法连接

## 解决方案

临时启用sha1，最好sftp服务器启用更安全的加密方式

```shell
 export OPENSSL_ENABLE_SHA1_SIGNATURES=1
```

[参考资料](https://fedoraproject.org/wiki/SHA1SignaturesGuidance)