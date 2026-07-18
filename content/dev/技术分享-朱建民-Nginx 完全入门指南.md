---
title: Nginx 完全入门指南
date: 2026-07-18
description: 一份直白、实用、面向IT运维与开发的 Nginx 参考手册
toc: true
slug: complete-guide-to-nginx
categories:
    - Dev
    - 技术分享
tags:
    - Nginx
---


## 🧭 Nginx 是什么？

用一句话概括：**Nginx 是一个高性能的 HTTP 服务器和反向代理服务器**。

你可以把它想象成一个 **"非常聪明的门卫"** —— 所有用户的请求先到达它这里，由它决定如何处理：

| 角色 | 说明 | 类比 |
|------|------|------|
| **Web 服务器** | 直接托管静态文件（HTML/CSS/JS/图片），高效返回给用户 | 像超市货架，用户直接取走商品 |
| **反向代理** | 接收用户请求，转发给后端应用服务器（如 Java、Python、Node.js），再将结果返回 | 像餐厅服务员，帮顾客向后厨点菜 |
| **负载均衡** | 将大量请求分发到多台后端服务器，分摊压力 | 像快递分拣中心，把包裹分配给不同的快递员 |

**核心优势**：
- 占用内存少（能处理数万并发连接）
- 稳定性极高（7x24 小时运行）
- 配置灵活（支持热重载，不中断服务）

---

## 🎯 Nginx 能解决哪些实际问题？

| 序号 | 场景 | 说明 |
|:----:|------|------|
| 1 | **前后端分离部署** | 前端代码由 Nginx 托管，后端 API 请求通过反向代理转发给应用服务器，解决跨域问题 |
| 2 | **多站点共用一台服务器** | 通过虚拟主机配置，在同一台机器上使用不同域名或端口部署多个网站 |
| 3 | **HTTPS 加密访问** | 轻松配置 SSL 证书，将 HTTP 升级为 HTTPS，提升安全性 |
| 4 | **流量治理与灰度发布** | 通过负载均衡权重控制，将部分流量引入新版本服务，实现平滑上线 |
| 5 | **访问控制与安全防护** | 限制 IP 访问、设置连接数限制、防 DDoS 攻击 |
| 6 | **动静分离** | 动态请求转发后端，静态资源由 Nginx 直接返回，提升响应速度 |

---

## 🏗️ 核心概念与配置文件结构

### 配置文件位置

```bash
/etc/nginx/nginx.conf          # 主配置文件
/etc/nginx/conf.d/             # 额外配置目录
/etc/nginx/sites-available/    # 可用站点配置（Debian/Ubuntu）
/etc/nginx/sites-enabled/      # 已启用站点配置（软链接）
/usr/share/nginx/html/         # 默认网站根目录
/var/log/nginx/access.log      # 访问日志
/var/log/nginx/error.log       # 错误日志
```

### 核心指令速查

```bash
指令	           作用	         示例
server_name	 设置域名	         server_name example.com;
listen	           监听端口	listen 80;
root	           网站根目录	root /var/www/html;
index	           默认首页	index index.html;
location	           URL 路由匹配	location /api/ { ... }
proxy_pass	  反向代理转发	proxy_pass http://backend;
try_files	  按顺序尝试文件	try_files $uri $uri/ /index.html;
return	           返回状态码或重定向	return 301 https://$server_name$request_uri;
rewrite	           URL 重写	rewrite ^/old/(.*)$ /new/$1 permanent;
alias	           路径别名	alias /var/www/static/;
```
### 📦 安装与基础环境

```bash
# 更新软件包索引
sudo apt update

# 安装 Nginx
sudo apt install nginx -y

# 查看版本
nginx -v

# 查看编译参数（包含已编译的模块）
nginx -V

# 启动服务
sudo systemctl start nginx

# 设置开机自启
sudo systemctl enable nginx
CentOS / RHEL / Fedora

```bash
# 安装 EPEL 仓库（CentOS 7）
sudo yum install epel-release -y

# 安装 Nginx
sudo yum install nginx -y   # 或 dnf install nginx -y

# 启动服务
sudo systemctl start nginx
sudo systemctl enable nginx
```

验证安装：在浏览器输入服务器 IP，看到 "Welcome to nginx!" 页面即表示安装成功。

### 🖥️ 服务管理命令（systemctl）

```bash
# 现代 Linux 系统使用 systemctl 管理 Nginx服务：

sudo systemctl start nginx      # 启动 Nginx
sudo systemctl stop nginx      # 停止 Nginx
sudo systemctl restart nginx   # 重启 Nginx（先停后启）
sudo systemctl reload nginx    # 重新加载配置（推荐）
sudo systemctl status nginx    # 查看运行状态
sudo systemctl enable nginx    # 设置开机自启
sudo systemctl disable nginx   # 取消开机自启
sudo systemctl is-active nginx # 检查是否运行中
sudo systemctl is-enabled nginx # 检查是否开机自启
```

> 💡 推荐做法：修改配置后优先使用 `reload`，而非 `restart`，避免服务中断。
### ⚡ Nginx 程序控制命令
通过 nginx 可执行文件发送更细粒度的信号：

```
# 语法检查（不启动服务）
sudo nginx -t

# 检查并显示所有加载的配置（含 include 文件）
sudo nginx -T

# 快速停止服务（立即终止）
sudo nginx -s stop

# 优雅停止（处理完当前请求后退出）
sudo nginx -s quit

# 重新加载配置文件（等效于 systemctl reload）
sudo nginx -s reload

# 重新打开日志文件（用于日志切割）
sudo nginx -s reopen

# 查看版本信息
nginx -v

# 查看版本及编译参数（包含模块信息）
nginx -V
```

### 📝 常用配置场景示例

#### 场景1：托管静态网站

```nginx
server {
    listen 80;
    server_name www.mywebsite.com;

    root /var/www/mywebsite;
    index index.html index.htm;

    # 处理 404 页面
    error_page 404 /404.html;
    location = /404.html {
        internal;
    }
}
```

#### 场景2：反向代理（前后端分离）

```nginx
server {
    listen 80;
    server_name api.myapp.com;

    # 所有 /api/ 请求转发到后端服务
    location /api/ {
        proxy_pass http://localhost:3000/;

        # 传递真实客户端信息
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 超时设置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # 静态文件直接由 Nginx 处理
    location /static/ {
        root /var/www/myapp;
        expires 7d;
        add_header Cache-Control "public, immutable";
    }
}
```

#### 场景3：负载均衡

```nginx
# 定义后端服务器组
upstream backend_servers {
    # 负载均衡策略：默认轮询
    server 192.168.1.10:8080 weight=3;   # 权重3，处理更多请求
    server 192.168.1.11:8080;             # 权重默认为1
    server 192.168.1.12:8080 backup;      # 备用服务器，仅在其他都故障时启用

    # 可选策略（取消注释使用）：
    # least_conn;      # 最少连接
    # ip_hash;         # 基于客户端 IP 的哈希，保持会话一致性
}

server {
    listen 80;
    server_name app.mycompany.com;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### 场景4：配置 HTTPS（SSL/TLS）

```nginx
server {
    listen 443 ssl http2;
    server_name secure.example.com;

    # SSL 证书配置
    ssl_certificate /etc/nginx/ssl/example.com.crt;
    ssl_certificate_key /etc/nginx/ssl/example.com.key;

    # SSL 安全优化
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    root /var/www/secure;
    index index.html;
}

# HTTP 自动跳转 HTTPS
server {
    listen 80;
    server_name secure.example.com;
    return 301 https://$server_name$request_uri;
}
```

### 📋 日志管理与问题排查

#### 日志位置

```bash
/var/log/nginx/access.log    # 访问日志（所有请求记录）
/var/log/nginx/error.log      # 错误日志（调试问题首选）
```

#### 自定义日志格式

```nginx
http {
    # 定义日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    # 应用日志格式
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;
}
```

#### 实时查看日志

```bash
# 实时查看访问日志
sudo tail -f /var/log/nginx/access.log

# 实时查看错误日志
sudo tail -f /var/log/nginx/error.log

# 过滤特定关键词
sudo tail -f /var/log/nginx/error.log | grep "error"

# 查看最近100行错误
sudo tail -n 100 /var/log/nginx/error.log

# 统计访问最多的 IP
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10
```

### 📝 日常运维速查卡
```
# 📌 配置测试（每次改配置必备）
sudo nginx -t && sudo systemctl reload nginx

# 📌 查看服务状态
sudo systemctl status nginx

# 📌 查看错误日志（最重要）
sudo tail -f /var/log/nginx/error.log

# 📌 查看访问日志
sudo tail -f /var/log/nginx/access.log

# 📌 端口检查
sudo ss -tlnp | grep nginx

# 📌 重载配置（不中断服务）
sudo nginx -s reload

# 📌 完全重启
sudo systemctl restart nginx

# 📌 查看版本及模块
nginx -V

# 📌 检查哪些配置文件被加载
sudo nginx -T | grep "server_name"

# 📌 测试配置并自动重载（一键命令）
sudo nginx -t && sudo systemctl reload nginx

# 📌 查看 Nginx 进程
ps aux | grep nginx

# 📌 统计日志中 500 错误的数量
sudo grep " 500 " /var/log/nginx/access.log | wc -l
```