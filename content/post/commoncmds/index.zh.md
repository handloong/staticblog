---
title: 常用Shell命令
date: 2026-04-14
description: 记录本人常用的linux/windows命令
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
``` shell
tail -100f mylog-2026-04.log
```

查找指定文件夹下文件(*.log)包含doer.ltd的
``` shell
find /home/logs/config -type f -exec grep 'doer.ltd' {} +
find /home/logs/config -type f -name "*.log" -exec grep -H 'doer.ltd' {} +
```

