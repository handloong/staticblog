---
title: "Oracle PGA 内存增高排查"
date: 2026-07-18
description: 排查 .NET 应用连接 Oracle 数据库时 PGA 内存持续增高的根因与解决方案
toc: true
slug: oracle-pga-memory-investigation
categories:
    - Dev
    - 技术分享
    - 故障排查
tags:
    - oracle
    - pga
    - sqlclient
    - connection-lifetime
---

# Oracle PGA 内存增高排查

## 问题现象

生产环境运行一段时间后，Oracle 数据库服务器上的 **PGA（Program Global Area）内存占用持续增长**，最终触发数据库性能告警甚至 OOM。

初步排查特征：

- 应用端使用 **ADO.NET / SqlClient** 连接 Oracle 数据库
- 业务高峰期 PGA 增长明显，低峰期未见明显释放
- 重启应用后 PGA 回落，随后再次缓慢攀升
- 数据库层面未发现明显的 SQL 泄漏或大结果集问题

---

## 排查过程

### 第一阶段：定位问题方向

通过 AWR 报告和 ASH 等待事件分析，发现：

| 指标 | 观察值 |
|------|--------|
| PGA 总使用量 | 持续上升，峰值远超预期 |
| 主要消耗来源 | 大量空闲会话仍持有 PGA 分配 |
| 会话数 | 正常范围内 |

**关键线索**：PGA 并非被活跃 SQL 占用，而是被大量**看似空闲的数据库会话**所持有。

### 第二阶段：分析连接池行为

.NET 应用的数据库连接由 **SqlClient 连接池** 管理。连接池的核心机制是复用物理连接，减少频繁创建/销毁的开销。但这里存在一个关键问题——**连接池中的"闲置连接"在 Oracle 服务端是否真正释放了资源？**

查阅微软官方文档和 GPT 辅助分析后，聚焦到两个参数：

#### Connection Lifetime（别名：Load Balance Timeout）

```
参数名称   ：Connection Lifetime / LoadBalanceTimeout
含义       ：连接在池中保留的最小存活时间（单位：秒）
类型       ：整数值
默认值     ：0（表示不启用此机制）
验证规则   ：必须 ≥ 0，否则抛出异常
作用时机   ：当连接从池中被取出使用时，检查其创建时间，若已超过 Connection Lifetime，则直接销毁重建
```

> **核心逻辑**：这个参数控制的是**连接池内连接的最大生命周期**，而不是空闲超时。当连接在池中停留时间超过该阈值且被取出复用时，才会被标记为过期并重新创建。

### 第三阶段：查看源码验证

查看 **Microsoft.Data.SqlClient（net8.0）** 源码：

```csharp
// SqlClient 内部常量定义
internal const bool IntegratedSecurity = false;
internal const SqlConnectionIPAddressPreference IPAddressPreference = SqlConnectionIPAddressPreference.IPv4First;

internal const int LoadBalanceTimeout = 0;  // ← 默认值为 0！

internal const int MaxPoolSize = 100;        // 默认最大连接数 100
internal const int MinPoolSize = 0;          // 默认最小连接数 0
internal const bool MultipleActiveResultSets = false;
```

**关键发现**：`LoadBalanceTimeout` 的默认值是 **0**，意味着**永远不会因为生命周期而主动回收旧连接**。

进一步查看连接字符串关键字映射（`SqlConnectionOptions.cs`）：

```csharp
// 第 197 行左右
AddKeywordToMap(DbConnectionStringKeywords.LoadBalanceTimeout,
                DbConnectionStringSynonyms.ConnectionLifetime);  // ← 同义词映射！
```

这证实了 **`Connection Lifetime` 和 `LoadBalanceTimeout` 是同一个参数的两个名字**。

### 第四阶段：根因确认

结合以上信息，完整的问题链路如下：

```
┌──────────────────────────────────────────────────────────────┐
│                     问题链路图                                │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 应用启动，连接池初始化                                     │
│     ↓                                                        │
│  2. 业务请求到来，创建物理连接到 Oracle                          │
│     ↓                                                        │
│  3. 请求结束，连接归还到池中                                   │
│     ↓                                                        │
│  4. Oracle 服务端会话进入 IDLE 状态，但 PGA 内存仍然分配        │
│     ↓                                                        │
│  5. 由于 LoadBalanceTimeout=0（默认），                        │
│     池中连接永不过期                                          │
│     ↓                                                        │
│  6. 即使长时间不用，这些连接一直占着 Oracle 的 PGA              │
│     ↓                                                        │
│  7. 新的业务波动 → 新连接创建 → 更多 PGA 被锁定               │
│     ↓                                                        │
│  8. PGA 只增不减 → 最终告警/OOM                               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**根本原因**：SqlClient 连接池默认不会主动淘汰旧连接，导致 Oracle 端大量空闲会话长期持有 PGA 内存不被释放。

---

## 解决方案

### 在连接字符串中添加 `Connection Lifetime`

```xml
<!-- 修改前 -->
<add name="OracleConn"
     connectionString="Data Source=ORCL;User Id=myUser;Password=myPwd;"
     providerName="Oracle.ManagedDataAccess.Client" />

<!-- 修改后 -->
<add name="OracleConn"
     connectionString="Data Source=ORCL;User Id=myUser;Password=myPwd;
                    Connection Lifetime=300;"
     providerName="Oracle.ManagedDataAccess.Client" />
```

或通过代码方式设置：

```csharp
var connectionString = "Server=myServer;Database=myDB;" +
                      "User Id=myUser;Password=myPwd;" +
                      "Connection Lifetime=300;";   // 单位：秒

// 或使用 SqlConnectionStringBuilder
var builder = new SqlConnectionStringBuilder
{
    DataSource = "myServer",
    InitialCatalog = "myDB",
    UserID = "myUser",
    Password = "myPwd",
    ConnectTimeout = 30,
    // 关键参数：连接在池中最长存活 5 分钟
    LoadBalanceTimeout = 300      // 等价于 Connection Lifetime
};
string connStr = builder.ConnectionString;
```

### 参数调优建议

| 场景 | 推荐值 | 说明 |
|------|--------|------|
| 一般业务系统 | **180 ~ 300**（3~5 分钟） | 兼顾连接复用率和资源释放 |
| 高并发短连接场景 | **60 ~ 120**（1~2 分钟） | 加快旧连接淘汰速度 |
| 低频长事务场景 | **600 ~ 900**（10~15 分钟） | 减少不必要的连接重建 |
| 不确定 | 从 **300** 开始，根据监控调整 | 观察 PGA 变化趋势后再微调 |

---

## 验证修复效果

### 1. 应用层验证

修改连接串部署后，观察以下指标：

```sql
-- 查看 Oracle 当前会话及 PGA 使用情况
SELECT s.sid,
       s.serial#,
       s.username,
       s.status,
       s.logon_time,
       ROUND(p.pga_used_mem / 1024 / 1024, 2) AS pga_used_mb,
       ROUND(p.pga_alloc_mem / 1024 / 1024, 2) AS pga_alloc_mb,
       s.program,
       s.machine
FROM v$session s
JOIN v$process p ON s.paddr = p.addr
WHERE s.username IS NOT NULL
ORDER BY p.pga_alloc_mem DESC;
```

### 2. 监控对比项

| 指标 | 修复前 | 修复后（预期） |
|------|--------|----------------|
| PGA 峰值 | 持续攀升 | 在一定范围内波动 |
| 空闲会话 PGA | 长期占用 | 定期释放 |
| 连接重建频率 | 极低 | 轻微增加（可接受） |

---

## 相关参数速查

| 连接字符串参数 | 别名 | 默认值 | 说明 |
|---------------|------|--------|------|
| `Connection Lifetime` | `Load Balance Timeout` | **0**（永不过期） | 连接池中连接的最大存活秒数 |
| `Max Pool Size` | — | **100** | 连接池最大连接数 |
| `Min Pool Size` | — | **0** | 连接池最小保持连接数 |
| `Connect Timeout` | `Connection Timeout` | **15** | 连接超时（秒） |

---

## 总结

本次问题的根因是 **SqlClient 连接池的 `LoadBalanceTimeout`（即 `Connection Lifetime`）默认值为 0**，导致连接池中的物理连接永不主动回收，进而造成 Oracle 端大量空闲会话持续占用 PGA 内存。解决方案非常简单——**在连接串中加入 `Connection Lifetime=xxx`** 即可让连接池定期淘汰旧连接，从而间接释放 Oracle 端的 PGA 资源。

**经验教训**：在使用连接池访问 Oracle 时，不要忽略连接生命周期参数的配置，尤其是在长时间运行的生产环境中。
