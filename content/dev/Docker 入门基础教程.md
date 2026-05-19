---
title: "Docker 入门基础教程"
date: 2026-05-19
tags: ["docker", "容器化", "devops", "入门"]
categories: ["技术教程", "DevOps"]
description: "Docker 入门基础教程，涵盖 Docker 核心概念、安装配置、常用命令、镜像和容器管理、Dockerfile 编写等内容，帮助初学者快速掌握 Docker 使用"
---

# Docker 入门基础教程

## 什么是 Docker？

Docker 是一个开源的应用容器引擎，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器完全使用沙箱机制，相互之间不会有任何接口。

## Docker 的核心概念

### 1. 镜像（Image）

镜像是一个只读的模板，可以用来创建容器。一个镜像可以创建多个容器。

**常见操作：**
```bash
# 查看本地镜像
docker images

# 拉取镜像
docker pull nginx:latest

# 删除镜像
docker rmi nginx:latest

# 搜索镜像
docker search nginx
```

### 2. 容器（Container）

容器是用镜像运行中的实例。容器可以被启动、开始、停止、删除。

**常见操作：**
```bash
# 查看运行中的容器
docker ps

# 查看所有容器（包括已停止的）
docker ps -a

# 启动容器
docker start <container_id>

# 停止容器
docker stop <container_id>

# 删除容器
docker rm <container_id>

# 运行容器
docker run -d -p 80:80 --name my-nginx nginx
```

### 3. 仓库（Repository）

仓库是存放镜像的地方，类似代码仓库。

**常用仓库地址：**
- Docker Hub: https://hub.docker.com/
- 阿里云容器镜像服务
- 腾讯云容器镜像服务

## Docker 安装

### Windows 系统安装

1. 下载 Docker Desktop: https://www.docker.com/products/docker-desktop/
2. 安装时选择 WSL2 模式（推荐）或 Hyper-V 模式
3. 安装完成后自动启动 Docker Desktop

### Linux 系统安装（以 Ubuntu 为例）

```bash
# 更新 apt 包索引
sudo apt update

# 安装必要包
sudo apt install ca-certificates curl gnupg lsb-release

# 添加 Docker 官方 GPG 密钥
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 设置仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 启动 Docker
sudo systemctl start docker

# 设置 Docker 开机启动
sudo systemctl enable docker

# 验证安装
docker --version
sudo docker run hello-world
```

### Mac 系统安装

直接从官网下载 Docker Desktop for Mac，双击安装即可。

## Docker 常用命令

### 镜像管理

```bash
# 列出所有镜像
docker images

# 查看镜像详细信息
docker inspect <image_name>

# 清理未使用的镜像
docker image prune

# 清理所有未使用的镜像
docker image prune -a
```

### 容器管理

```bash
# 查看容器日志
docker logs <container_id>

# 进入运行中的容器
docker exec -it <container_id> bash

# 重启容器
docker restart <container_id>

# 查看容器资源使用情况
docker stats

# 暂停容器
docker pause <container_id>

# 恢复容器
docker unpause <container_id>
```

### 网络管理

```bash
# 查看所有网络
docker network ls

# 创建网络
docker network create my-network

# 连接容器到网络
docker network connect my-network <container_id>

# 删除网络
docker network rm my-network
```

### 数据卷管理

```bash
# 查看所有数据卷
docker volume ls

# 创建数据卷
docker volume create my-volume

# 删除数据卷
docker volume rm my-volume
```

## Dockerfile 编写

Dockerfile 是用于构建 Docker 镜像的文本文件，包含了一系列构建指令。

### 基础结构

```dockerfile
# 基础镜像
FROM ubuntu:20.04

# 维护者信息
MAINTAINER Your Name <your.email@example.com>

# 设置环境变量
ENV NODE_ENV=production

# 复制文件
COPY package.json ./

# 安装依赖
RUN npm install

# 暴露端口
EXPOSE 3000

# 设置工作目录
WORKDIR /app

# 复制应用代码
COPY . .

# 启动命令
CMD ["npm", "start"]
```

### 实用示例

#### 1. Python 应用

```dockerfile
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["python", "app.py"]
```

#### 2. Node.js 应用

```dockerfile
FROM node:16-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

EXPOSE 3000

CMD ["node", "index.js"]
```

#### 3. 多阶段构建

```dockerfile
# 构建阶段
FROM node:16 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# 运行阶段
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## Docker Compose

Docker Compose 是用于定义和运行多容器 Docker 应用的工具。

### docker-compose.yml 示例

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"
    environment:
      - NODE_ENV=production
    volumes:
      - ./app:/app
    depends_on:
      - db

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 常用命令

```bash
# 启动所有服务
docker-compose up -d

# 停止所有服务
docker-compose down

# 查看服务状态
docker-compose ps

# 查看服务日志
docker-compose logs -f

# 重启服务
docker-compose restart web
```

## 最佳实践

### 1. 镜像优化

- 使用小基础镜像（如 Alpine）
- 减少 Dockerfile 层数
- 利用缓存机制
- 使用 .dockerignore 文件

### 2. 安全性

- 不使用 root 用户运行应用
- 定期更新基础镜像
- 避免在镜像中存储敏感信息
- 使用只读文件系统

### 3. 网络优化

- 合理使用网络模式
- 限制容器间通信
- 使用健康检查

## 常见问题解决

### 1. 容器启动失败

```bash
# 查看日志
docker logs <container_id>

# 检查端口占用
netstat -tunlp | grep 80
```

### 2. 磁盘空间不足

```bash
# 清理未使用的资源
docker system prune -a

# 查看磁盘使用情况
docker system df
```

### 3. 权限问题

```bash
# 将用户添加到 docker 组
sudo usermod -aG docker $USER

# 重新登录生效
newgrp docker
```

## 总结

Docker 是现代软件开发和部署的重要工具，通过容器化技术可以大大提高开发效率和部署可靠性。希望本教程能帮助您快速入门 Docker。

## 相关资源

- Docker 官方文档：https://docs.docker.com/
- Docker Hub: https://hub.docker.com/
- Dockerfile 最佳实践：https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- Docker Compose 文档：https://docs.docker.com/compose/

---

**参考资料：**
- Docker 官方文档
- 开源社区最佳实践
- 实际项目经验总结
