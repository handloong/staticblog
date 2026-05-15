---
title: 常用docker部署
date: 2026-05-15
description: 记录一些常用软件的docker部署命令
slug: common-docker-deployment
toc: true
categories:
    - IT
tags:
    - docker
---

### cloudbeaver

``` shell
mkdir -p cb-data/workspace cb-data/log

docker run -d \  
--name cloudbeaver \  
--restart unless-stopped \  
-p 8978:8978 \  
-v $(pwd)/cb-data/workspace:/opt/cloudbeaver/workspace \  
-v $(pwd)/cb-data/log:/opt/cloudbeaver/log \  
dbeaver/cloudbeaver:ea
```
### rocketmq
``` shell
docker run -d \
    --name rmqnamesrv \
    --network host \
    apache/rocketmq:5.3.4 sh mqnamesrv

docker run -d \
    --name rmqbroker \
    --network host \
    -e "NAMESRV_ADDR=localhost:9876" \
    apache/rocketmq:5.3.4 \
    sh mqbroker --enable-proxy
```

### nacos
``` shell
docker run --name nacos-standalone-derby \
    -e MODE=standalone \
    --network host \
    --restart always \
    -v /usr/local/nacos/data:/home/nacos/data \
    -v /usr/local/nacos/logs:/home/nacos/logs \
    -v /usr/local/nacos/backup:/home/nacos/backup \
    -d nacos/nacos-server:v2.5.2
```