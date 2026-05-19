---
title: Linux 入门教程 - 从零基础到精通
date: 2026-05-19
description: 全面详解 Linux 操作系统基础知识，包括安装、常用命令、文件管理、权限控制、软件包管理、Shell 脚本编程等，适合 Linux 初学者
tags: [Linux, 教程，入门，操作系统]
categories: [Linux, 教程]
author: BU22 技术团队
---

# Linux 入门教程

Linux 是一款免费、开源、功能强大的操作系统，被广泛应用于服务器、嵌入式设备、超级计算机等领域。本教程将带您从零开始，逐步掌握 Linux 的使用。

## 目录

1. [Linux 简介](#linux-简介)
2. [安装 Linux](#安装-linux)
3. [Linux 桌面环境](#linux-桌面环境)
4. [常用命令](#常用命令)
5. [文件管理](#文件管理)
6. [用户与权限](#用户与权限)
7. [软件包管理](#软件包管理)
8. [Shell 脚本编程](#shell-脚本编程)
9. [网络配置](#网络配置)
10. [常见问题解决](#常见问题解决)

---

## 一、Linux 简介

### 什么是 Linux？

Linux 是一种类 Unix 操作系统内核，由林纳斯·托瓦兹（Linus Torvalds）于 1991 年首次发布。Linux 遵循 GNU 通用公共许可证（GPL），是完全免费和开源的。

### Linux 的特点

- **开源免费**：可以免费下载、使用和修改
- **安全性高**：权限控制严格，病毒较少
- **稳定性强**：可长时间运行不重启
- **多用户多任务**：支持多个用户同时使用
- **丰富的软件生态**：拥有海量开源软件

### Linux 发行版

| 发行版 | 特点 | 适用场景 |
|--------|------|----------|
| Ubuntu | 用户友好，社区活跃 | 桌面、服务器 |
| CentOS | 稳定，企业级 | 服务器 |
| Debian | 稳定，软件丰富 | 服务器、桌面 |
| Fedora | 最新特性，社区驱动 | 开发者、桌面 |
| Arch Linux | 滚动更新，高度可定制 | 高级用户 |

---

## 二、安装 Linux

### Windows 安装双系统

#### 步骤一：准备工具

```bash
# 下载工具
- Rufus: https://rufus.ie/
- Ventoy: https://www.ventoy.net/
```

#### 步骤二：制作启动盘

1. 插入 U 盘（至少 4GB）
2. 打开 Rufus
3. 选择下载的 ISO 镜像文件
4. 点击"开始"制作启动盘

#### 步骤三：BIOS 设置

1. 重启电脑，进入 BIOS（通常按 F2、F12 或 Del）
2. 设置 U 盘为第一启动项
3. 保存并重启

#### 步骤四：分区安装

1. 进入安装界面，选择"手动分区"
2. 创建分区：
   - `/`（根分区）：至少 20GB
   - `swap`（交换分区）：内存的 1-2 倍
   - `/home`（用户分区）：剩余空间
3. 选择引导安装位置
4. 开始安装

### 虚拟机安装（推荐初学者）

#### 使用 VirtualBox

```bash
# 下载 VirtualBox
# https://www.virtualbox.org/

# 创建虚拟机
1. 新建虚拟机
2. 分配内存（建议 2GB 以上）
3. 创建虚拟硬盘（建议 20GB 以上）
4. 挂载 ISO 镜像
5. 启动安装
```

### WSL（Windows Subsystem for Linux）

```bash
# Windows 10/11 直接启用
# PowerShell（管理员）运行：
wsl --install

# 或手动安装 Ubuntu
wsl --install Ubuntu
```

---

## 三、Linux 桌面环境

### 常用桌面环境

| 桌面环境 | 资源占用 | 特点 |
|----------|----------|------|
| GNOME | 高 | 现代化，功能完整 |
| KDE Plasma | 中 | 高度可定制 |
| XFCE | 低 | 轻量级，稳定 |
| LXDE | 很低 | 适合老机器 |

### 安装桌面环境（Ubuntu/Debian）

```bash
# 安装 GNOME
sudo apt update
sudo apt install ubuntu-desktop

# 安装 KDE
sudo apt install kubuntu-desktop

# 安装 XFCE
sudo apt install xfce4
```

---

## 四、常用命令

### 文件操作命令

```bash
# 查看当前目录
pwd

# 列出文件
ls -la

# 切换目录
cd /path/to/directory

# 创建目录
mkdir new_directory

# 创建文件
touch newfile.txt

# 复制文件
cp source.txt destination.txt

# 复制目录
cp -r source_dir destination_dir

# 移动/重命名文件
mv oldname.txt newname.txt

# 删除文件
rm file.txt

# 删除目录
rm -rf directory/

# 查看文件内容
cat file.txt
less file.txt
head -n 10 file.txt
tail -f file.txt
```

### 系统信息命令

```bash
# 查看系统信息
uname -a

# 查看内核版本
uname -r

# 查看 CPU 信息
lscpu

# 查看内存信息
free -h

# 查看磁盘空间
df -h

# 查看磁盘使用
du -sh /path

# 查看进程
ps aux
top
htop

# 查看网络状态
netstat -tuln
ss -tuln

# 查看系统日志
dmesg
journalctl -xe
```

### 文本处理命令

```bash
# 查找文本
grep "pattern" file.txt

# 排序
sort file.txt

# 去重
sort file.txt | uniq

# 统计行数
wc -l file.txt

# 替换文本
sed 's/old/new/g' file.txt

# 文本流处理
awk '{print $1}' file.txt
```

---

## 五、文件管理

### 文件权限

```bash
# 查看权限
ls -l file.txt

# 修改权限
chmod 755 file.txt
chmod u+x file.txt
chmod -R 755 directory/

# 修改所有者
chown user:group file.txt
chown -R user:group directory/
```

### 符号链接

```bash
# 创建符号链接
ln -s target linkname

# 创建硬链接
ln target linkname
```

### 压缩与解压

```bash
# tar 压缩
tar -czvf archive.tar.gz directory/

# tar 解压
tar -xzvf archive.tar.gz

# zip 压缩
zip -r archive.zip directory/

# zip 解压
unzip archive.zip
```

---

## 六、用户与权限

### 用户管理

```bash
# 创建用户
sudo adduser newuser

# 删除用户
sudo deluser olduser

# 添加用户到组
sudo usermod -aG sudo newuser

# 查看用户信息
id username

# 修改用户密码
passwd username
```

### 权限类型

```bash
# 权限位
r = 4 (读)
w = 2 (写)
x = 1 (执行)

# 权限组合
7 = rwx (完全权限)
6 = rw- (读写)
5 = r-x (读执行)
4 = r-- (只读)
0 = --- (无权限)
```

### sudo 权限

```bash
# 配置 sudoers
sudo visudo

# 示例配置
username ALL=(ALL:ALL) ALL
%nobody ALL=(ALL:ALL) ALL
```

---

## 七、软件包管理

### Ubuntu/Debian (APT)

```bash
# 更新软件源
sudo apt update

# 升级软件
sudo apt upgrade

# 安装包
sudo apt install package_name

# 卸载包
sudo apt remove package_name

# 删除配置文件
sudo apt purge package_name

# 查找包
apt search keyword

# 查看包信息
apt show package_name

# 清理缓存
sudo apt clean
```

### CentOS/RHEL (DNF/YUM)

```bash
# 更新
sudo dnf update

# 安装包
sudo dnf install package_name

# 卸载包
sudo dnf remove package_name

# 查找包
dnf search keyword
```

### Arch Linux (Pacman)

```bash
# 同步和更新
sudo pacman -Syu

# 安装包
sudo pacman -S package_name

# 卸载包
sudo pacman -R package_name

# 查找包
pacman -Ss keyword
```

---

## 八、Shell 脚本编程

### 编写第一个脚本

```bash
#!/bin/bash
# 脚本名称：hello.sh

echo "Hello, Linux!"
echo "当前日期：$(date)"
echo "当前用户：$(whoami)"
echo "当前目录：$(pwd)"
```

### 变量

```bash
#!/bin/bash

# 定义变量
name="John"
age=25

# 使用变量
echo "Name: $name"
echo "Age: $age"

# 系统变量
echo "当前日期：$DATE"
echo "主机名：$HOSTNAME"
```

### 条件判断

```bash
#!/bin/bash

# if 语句
if [ $age -gt 18 ]; then
    echo "成年人"
else
    echo "未成年人"
fi

# if-elif-else
if [ $score -ge 90 ]; then
    echo "优秀"
elif [ $score -ge 60 ]; then
    echo "及格"
else
    echo "不及格"
fi

# 测试条件
[ -f file.txt ]    # 文件存在
[ -d directory ]   # 目录存在
[ -z "$var" ]      # 变量为空
[ "$a" -eq "$b" ]  # 数字相等
```

### 循环

```bash
#!/bin/bash

# for 循环
for i in {1..5}; do
    echo "数字：$i"
done

# while 循环
count=1
while [ $count -le 5 ]; do
    echo "计数：$count"
    ((count++))
done

# until 循环
count=5
until [ $count -eq 0 ]; do
    echo "倒计时：$count"
    ((count--))
done
```

### 函数

```bash
#!/bin/bash

# 定义函数
greet() {
    local name=$1
    echo "你好，$name!"
}

# 调用函数
greet "张三"

# 带返回值函数
add() {
    echo $(($1 + $2))
}

result=$(add 5 3)
echo "结果：$result"
```

### 实例：备份脚本

```bash
#!/bin/bash
# backup.sh - 自动备份脚本

BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)
SOURCE_DIR="/data"
BACKUP_FILE="backup_$DATE.tar.gz"

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份文件
tar -czvf $BACKUP_DIR/$BACKUP_FILE $SOURCE_DIR

# 检查备份是否成功
if [ $? -eq 0 ]; then
    echo "备份成功：$BACKUP_FILE"
    
    # 保留最近 7 天的备份
    find $BACKUP_DIR -name "backup_*.tar.gz" -mtime +7 -delete
else
    echo "备份失败!"
    exit 1
fi
```

---

## 九、网络配置

### IP 地址配置

```bash
# 查看网络接口
ip addr show

# 查看路由表
ip route show

# 配置静态 IP (Ubuntu)
sudo nano /etc/netplan/01-netcfg.yaml

# 示例配置
network:
  ethernets:
    eth0:
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### 网络连接测试

```bash
# ping 测试
ping www.baidu.com

# 追踪路由
traceroute www.baidu.com

# 查看端口监听
netstat -tuln

# 下载文件
wget http://example.com/file.zip
curl -O http://example.com/file.zip

# 上传文件
scp file.txt user@host:/path/
```

### SSH 远程连接

```bash
# 连接远程服务器
ssh user@hostname

# 指定端口连接
ssh -p 2222 user@hostname

# 复制文件到远程
scp file.txt user@hostname:/path/

# 配置 SSH 免密登录
ssh-keygen -t rsa
ssh-copy-id user@hostname
```

---

## 十、常见问题解决

### 问题 1: 无法执行命令 "Permission denied"

```bash
# 添加执行权限
chmod +x script.sh

# 使用 sudo
sudo command
```

### 问题 2: 内存不足

```bash
# 查看内存使用
free -h

# 创建 Swap 分区
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 永久生效
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### 问题 3: 磁盘空间不足

```bash
# 查看磁盘使用
df -h

# 清理缓存
sudo apt clean

# 查找大文件
find / -type f -size +100M

# 删除旧日志
sudo journalctl --vacuum-time=7d
```

### 问题 4: 网络连接问题

```bash
# 检查网络状态
ip link show

# 重启网络服务
sudo systemctl restart NetworkManager

# 修改 DNS
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```

### 问题 5: 软件安装失败

```bash
# 更新软件源
sudo apt update

# 修复损坏的包
sudo apt --fix-broken install

# 清理并重装
sudo apt purge package_name
sudo apt autoremove
sudo apt install package_name
```

---

## 附录

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| Ctrl+C | 终止当前命令 |
| Ctrl+Z | 暂停当前命令 |
| Ctrl+A | 移动光标到行首 |
| Ctrl+E | 移动光标到行尾 |
| Ctrl+L | 清屏 |
| Ctrl+R | 搜索历史命令 |
| Ctrl+D | 退出终端 |
| Tab | 自动补全 |

### 推荐学习资源

- **官方文档**: https://www.kernel.org/doc/html/latest/
- **Linux 命令行手册**: https://man7.org/linux/man-pages/
- **在线练习**: https://overthewire.org/wargames/
- **视频教程**: Linux 命令速成（B 站）
- **社区论坛**: https://ubuntuforums.org/

### 实践建议

1. **多使用终端**: 减少 GUI 依赖，提高效率和理解
2. **阅读文档**: `man command` 或 `command --help`
3. **记录命令**: 建立自己的命令笔记
4. **多练习**: 在虚拟机或 WSL 中反复实践
5. **参与开源**: 学习他人的代码和脚本

---

## 总结

Linux 是一个功能强大的操作系统，掌握 Linux 技能对于开发者、运维人员来说至关重要。本教程涵盖了 Linux 基础知识的各个方面，包括：

- ✅ Linux 安装方法
- ✅ 常用命令使用
- ✅ 文件权限管理
- ✅ 软件包管理
- ✅ Shell 脚本编程
- ✅ 网络配置
- ✅ 问题排查

建议初学者从虚拟机或 WSL 开始练习，逐步熟悉命令和系统管理。坚持实践，你会越来越熟练！

**持续学习，不断进步！**
