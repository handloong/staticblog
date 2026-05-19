---
title: "PM2 入门教程"
date: 2026-05-19
draft: false
tags: ["pm2", "nodejs", "deployment", "devops"]
categories: ["开发工具", "部署"]
description: "PM2 是一个生产级 Node.js 进程管理器，本教程介绍 PM2 的常用命令和部署方法"
---

# PM2 入门教程

PM2 是一个强大的 Node.js 进程管理器，可以帮助你：

- 保持应用始终运行
- 自动重启崩溃的应用
- 零停机部署
- 实时监控应用性能
- 日志管理

## 1. 安装 PM2

### 全局安装

```bash
npm install -g pm2
```

### 验证安装

```bash
pm2 --version
```

## 2. 常用命令

### 启动应用

```bash
# 启动单个应用
pm2 start app.js

# 启动 TypeScript/ES6 应用
pm2 start npm --name "my-app" start

# 启动多个进程（集群模式）
pm2 start ecosystem.config.js

# 指定端口和环境变量
pm2 start app.js --name "production-app" --env production
```

### 停止应用

```bash
# 停止单个应用
pm2 stop <app-name|app-id>

# 停止所有应用
pm2 stop all
```

### 重启应用

```bash
# 重启单个应用
pm2 restart <app-name|app-id>

# 重启所有应用
pm2 restart all
```

### 删除应用

```bash
# 从 PM2 管理中删除应用
pm2 delete <app-name|app-id>

# 删除所有应用
pm2 delete all
```

### 查看应用状态

```bash
# 查看所有应用状态
pm2 list

# 查看应用详情
pm2 show <app-name|app-id>

# 查看实时日志
pm2 logs <app-name|app-id>

# 查看监控图表
pm2 monit
```

### 生成启动脚本

```bash
# 生成启动脚本（让应用开机自启）
pm2 startup

# 保存当前应用列表（用于恢复）
pm2 save
```

## 3. 零停机部署

PM2 的一个核心特性是零停机部署（Zero Downtime Deployments）。

### 使用 reload 命令

```bash
# 重新加载应用（不会中断服务）
pm2 reload <app-name>
```

### 使用 deploy 流程

```bash
# 1. 部署新版本代码
# git pull 或 npm install 等

# 2. 重新加载应用（零停机）
pm2 reload all

# 3. 检查状态
pm2 list
```

### 使用 ecosystem.config.js

创建一个配置文件来管理多个应用：

```javascript
module.exports = {
  apps: [{
    name: 'my-app',
    script: 'app.js',
    instances: 2,  // 集群模式
    autorestart: true,
    watch: false,
    max_memory_restart: '1G',
    env: {
      NODE_ENV: 'development',
      PORT: 3000
    },
    env_production: {
      NODE_ENV: 'production',
      PORT: 8080
    }
  }]
};
```

使用配置启动：

```bash
# 开发环境
pm2 start ecosystem.config.js

# 生产环境
pm2 start ecosystem.config.js --env production
```

## 4. 日志管理

```bash
# 查看日志
pm2 logs <app-name>

# 清空日志
pm2 flush

# 指定日志文件路径
pm2 start app.js --log ~/logs/my-app.log
```

## 5. 生产环境部署示例

### 步骤 1: 准备项目

```bash
# 安装依赖
npm install

# 构建项目（如果是 TypeScript 或需要构建）
npm run build
```

### 步骤 2: 使用 PM2 启动

```bash
# 启动应用
pm2 start dist/app.js --name production-app

# 查看状态
pm2 list

# 监控应用
pm2 monit
```

### 步骤 3: 设置开机自启

```bash
# 生成启动脚本
pm2 startup

# 保存当前配置
pm2 save
```

### 步骤 4: 更新应用（零停机）

```bash
# 拉取最新代码
git pull

# 安装新依赖
npm install

# 重新加载应用
pm2 reload production-app

# 检查状态
pm2 list
```

## 6. 实用技巧

### 使用环境变量

```bash
# 命令行方式
pm2 start app.js --name my-app --env production PORT=3000

# 使用配置文件
# ecosystem.config.js 中
env: {
  NODE_ENV: 'development'
},
env_production: {
  NODE_ENV: 'production',
  PORT: 8080
}
```

### 使用进程集群模式

```bash
# 使用 CPU 核心数启动多个实例
pm2 start app.js -i max --name my-cluster-app
```

### 查看进程详情

```bash
# 查看 CPU、内存使用
pm2 monit

# 查看详细信息
pm2 show <app-id>

# 查看性能指标
pm2 metrics <app-id>
```

## 7. 故障排查

### 应用崩溃自动重启

PM2 默认会在应用崩溃时自动重启：

```bash
# 查看崩溃历史
pm2 logs | grep "Error"

# 查看应用日志
pm2 logs <app-name> --lines 100
```

### 检查端口占用

```bash
# 如果端口被占用，使用 --port 选项
pm2 start app.js --port 3001
```

### 内存限制

```bash
# 设置内存限制，超过会自动重启
pm2 start app.js --max-memory-restart 1G
```

## 8. PM2 高级功能

### 使用 ecosystem.config.js 的高级配置

```javascript
module.exports = {
  apps: [{
    name: 'my-app',
    script: './app.js',
    
    // 集群模式
    instances: 'max',
    
    // 自动重启
    autorestart: true,
    
    // 监控文件变化（开发环境）
    watch: true,
    ignore_watch: ['node_modules', 'logs'],
    
    // 内存限制
    max_memory_restart: '500M',
    
    // 错误日志
    error_file: './logs/err.log',
    out_file: './logs/out.log',
    
    // 环境变量
    env: {
      NODE_ENV: 'development'
    },
    env_production: {
      NODE_ENV: 'production'
    }
  }]
};
```

## 总结

PM2 是 Node.js 生产环境部署的必备工具：

✅ **简单易用** - 常用命令简单易记  
✅ **零停机部署** - reload 命令保证服务不中断  
✅ **自动重启** - 崩溃自动恢复  
✅ **进程管理** - 集群模式充分利用多核 CPU  
✅ **日志管理** - 集中管理应用日志  
✅ **开机自启** - 系统重启后自动恢复服务

开始使用 PM2，让你的 Node.js 应用更加稳定可靠！

## 参考资料

- [PM2 官方文档](https://pm2.keymetrics.io/docs/)
- [PM2 GitHub 仓库](https://github.com/Unitech/PM2)
- [ecosystem.config.js 配置示例](https://pm2.keymetrics.io/docs/usage/application-declaration/)
