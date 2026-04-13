---
title: NextCloud集成LDAP
date: 2026-04-13
description: 如何集成LDAP认证登录。
slug: nextcloud-ladp
toc: true
categories:
    - IT
tags:
    - NextCloud
    - LDAP
---






## LDAP 相关概念

### 🔹 User DN（用户专有名称）  
**中文含义**：用户的 Distinguished Name（专有名称）  
**作用**：唯一标识 LDAP 目录中某个具体用户的完整路径。

**典型格式**：  
```
uid=johndoe,ou=users,dc=example,dc=com
```

**使用场景**：  
在身份认证过程中，系统通常会将用户输入的用户名（如 `johndoe`）拼接成完整的 User DN，然后使用该 DN 与用户提供的密码尝试绑定（Bind）到 LDAP 服务器，从而完成验证。

> ✅ **User DN = 用户的“完整门牌号”** —— 精确指向目录中的一个实体。

---

### 🔹 Base DN（基准专有名称）  
**中文含义**：搜索操作的起始根节点  
**作用**：定义 LDAP 查询的**搜索范围起点**，所有子树查询均从此节点向下展开。

**典型格式**：  
- `dc=example,dc=com` 　　　　→ 对应域名 `example.com`  
- `ou=users,dc=example,dc=com` → 限定在 `users` 组织单元下

**类比理解**：  
就像在文件系统中设置搜索路径为 `C:\Users` —— 你不会去扫描 `C:\Windows`，因为你的“搜索根目录”已限定在 `Users`。

**为何关键**？  
- **设得太宽**（如 `dc=com`）→ 搜索效率低，遍历无关数据；  
- **设得太窄**（如直接指定某用户）→ 可能找不到任何结果。  

> ✅ **Base DN = 搜索的“辖区范围”** —— 决定“我在哪片区域找人”。

---

## NextCloud集成LDAP

> 注意:社区版需要单独安装LDAP插件

### 服务器

![服务器](https://cdn.jsdelivr.net/gh/handloong/image-bed@main/hugo/2026/04/13/1afbf6dd2c531f46b794a9427a0746c1-1.webp)



1: 填写LDAPIP，例如:198.168.1.2

2: 填写用户DN例如：`CN=doer-ldap,OU=Service Accounts,OU=Special Accounts,DC=doer,DC=com,DC=cn`

3:填写doer-ldap用户对应的密码

4：填写BaseDB,例如:`OU=BU_doer,OU=KS-FileSystem,OU=KS-Site,OU=DFS,OU=Group Accounts,DC=doer,DC=com,DC=cn`


### 用户

![用户](https://cdn.jsdelivr.net/gh/handloong/image-bed@main/hugo/2026/04/13/eecbd4cded539c588467171f45790bc0-2.webp)

可以直接写LDAP查询: `(&(|(objectclass=user)))`

### 登录属性

![登录属性](https://cdn.jsdelivr.net/gh/handloong/image-bed@main/hugo/2026/04/13/06df75f84ea4517cb80aee8182b1da88-3.webp)

这里主要控制登录的时候从指定的表达式查询,其中`%uid`是NextCloud登录页面输入的用户名的占位符。

例如： `(&(&(|(objectclass=user)))(samaccountname=%uid)(department=*BU_doer*))`

> 注意：这里加了一个用户属性department的限制，我不知道为啥BaseDN限制不住，这里使用登录表达式限制指定部门登录

### 群组

![群组](https://cdn.jsdelivr.net/gh/handloong/image-bed@main/hugo/2026/04/13/dc9446568674a5eebb0aebd827be3d0a-4.webp)



这里是限制NextCloud分配某个文件夹给组用的，例如：`(&(objectClass=group)(objectCategory=group))`

### 高级

![高级](https://cdn.jsdelivr.net/gh/handloong/image-bed@main/hugo/2026/04/13/ac2e83a8b302de3d004a76c59c5fcbe3-5.webp)

这里主要设置NextCloud用户相关的一些设定。比如用户名，账号，邮箱，上图中

1表示使用用户属性的sn字段显示

2表示是否显示其他的，如上配置在nextcloud中会显示`张三(1008611)`

3表示按照什么搜索用户，我这里写的是工号和姓名搜索，多个使用分号分隔