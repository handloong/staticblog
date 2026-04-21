---
title: 常用Shell命令
date: 2026-04-14
description: 记录本人常用的linux/windows/tools命令
toc: true
categories:
    - IT
tags:
    - cmd
    - shell
---

## linux

### 网络

防火墙放行linux指定端口
``` shell
firewall-cmd --zone=public --add-port=8880/tcp --permanent
firewall-cmd --reload
```

### 文本

监控文件后100行变化
```shell
tail -100f mylog-2026-04.log
```

查找指定文件夹下文件(*.log)包含doer.ltd的
```shell
find /home/logs/config -type f -exec grep 'doer.ltd' {} +
find /home/logs/config -type f -name "*.log" -exec grep -H 'doer.ltd' {} +
```

### 其他
pm2启动程序
```shell
 pm2 start  "dotnet /app/MyBlog/MyBlog.WebApi.dll"  --name MyBlog.WebApi
 pm2 save
```
更新系统
```
unzip -o /tmp/1.8.0.3.webapi.zip -d /apps/DoerPPP
pm2 restart 0
```


## windows

### 网络

查看指定已连接过的wifi密码
``` shell
netsh wlan show profile name="MyBlog-MIFI" key=clear
```


## Tools

### 网络

Wireshark过滤器
```
(tcp.port == 4051 ||tcp.port == 4045||tcp.port == 4055) && tcp.len>50
```
### 文本
`notepad--`中正则搜索既包含A又包含B又包含C又包含D的
```
^(?=.*A)(?=.*B)(?=.*C)(?=.*D).+$
```
