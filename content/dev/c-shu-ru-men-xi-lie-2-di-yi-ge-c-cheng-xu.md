---
title: "C# 入门系列 (2) - 第一个 C# 程序：你好世界"
slug: "c-shu-ru-men-xi-lie-2-di-yi-ge-c-cheng-xu"
date: 2026-05-29
draft: false
description: "C# 入门系列第二篇，带你创建第一个 C# Console 应用程序，理解代码结构"
tags: ["csharp", "入门", "系列课程", "hello world"]
categories: ["C# 入门系列"]
---

# C# 入门系列 (2) - 第一个 C# 程序：你好世界

准备好了吗？让我们开始编写你的第一个 C# 程序！在这个教程中，我们将创建简单的 Console 应用程序，并深入理解 C# 程序的基本结构。

## 创建你的第一个项目

### 方法一：使用命令行（推荐）

打开终端或命令提示符，执行以下命令：

```bash
# 创建名为 HelloWorld 的控制台应用
dotnet new console -n HelloWorld

# 进入项目目录
cd HelloWorld

# 查看项目结构
ls  # 或 dir (Windows)
```

### 方法二：使用 Visual Studio

1. 打开 Visual Studio
2. 选择 "创建新项目"
3. 选择 "控制台应用 (.NET)"
4. 输入项目名称 "HelloWorld"
5. 选择保存位置
6. 点击 "创建"

## 项目结构解析

运行 `ls` 或 `dir` 命令后，你会看到类似这样的文件结构：

```
HelloWorld/
├── HelloWorld.csproj    # 项目文件
└── Program.cs           # 程序入口文件
```

### 项目文件 (.csproj)

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

</Project>
```

**关键字段解释：**
- `<OutputType>Exe</OutputType>`：输出类型为可执行程序
- `<TargetFramework>net8.0</TargetFramework>`：目标 .NET 版本为 8.0
- `<ImplicitUsings>enable</ImplicitUsings>`：隐式使用命名空间
- `<Nullable>enable</Nullable>`：启用可空引用类型

### 程序文件 (Program.cs)

这是程序的主要代码文件，默认内容如下：

```csharp
// 问候信息
Console.WriteLine("Hello, World!");
```

## 深入理解代码

### Console.WriteLine() 方法

让我们逐步分析这段代码：

```csharp
Console.WriteLine("Hello, World!");
```

- **Console**：是一个类，代表控制台（命令行界面）
- **WriteLine**：是 Console 类的一个方法
- **""**：双引号包裹的文本是字符串参数
- **;**：分号表示语句结束

### 运行程序

在项目目录下运行：

```bash
dotnet run
```

你将看到输出：
```
Hello, World!
```

## 修改并运行程序

### 尝试不同的输出

打开 `Program.cs`，将代码修改为：

```csharp
// 多行输出
Console.WriteLine("Hello, C#!");
Console.WriteLine("这是我的第一个 C# 程序");
Console.WriteLine("我在学习编程!");

// 在单独一行输出
Console.WriteLine("");

// 输出数字
Console.WriteLine(123);
```

运行后你会看到：
```
Hello, C#!
这是我的第一个 C# 程序
我在学习编程!

123
```

### 使用 C# 插值字符串

C# 提供了更优雅的字符串处理方式：

```csharp
// 定义变量
string name = "小明";
int age = 18;

// 使用插值字符串
Console.WriteLine($"我的名字叫 {name}");
Console.WriteLine($"我今年 {age} 岁");

// 在插值字符串中执行表达式
Console.WriteLine($"明年我就 {age + 1} 岁了");
```

输出：
```
我的名字叫 小明
我今年 18 岁
明年我就 19 岁了
```

## 理解 Main 方法

在较新的 C# 版本中，你可能注意到程序没有显式的 `Main` 方法。这是因为 C# 10+ 引入了**顶级语句**。

### 顶级语句 vs 传统 Main 方法

**顶级语句（现代方式）：**
```csharp
Console.WriteLine("Hello, World!");
```

**传统 Main 方法（完整方式）：**
```csharp
using System;

class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello, World!");
    }
}
```

**两者等价，但顶级语句更简洁。**

## 接收用户输入

### 使用 Console.ReadLine()

```csharp
// 提示用户输入
Console.Write("请输入你的名字：");

// 读取输入并存储到变量
string name = Console.ReadLine();

// 输出欢迎信息
Console.WriteLine($"你好，{name}！欢迎学习 C#!");
```

运行后：
```
请输入你的名字：张三
你好，张三！欢迎学习 C#!
```

### 读取数字输入

```csharp
// 读取年龄
Console.Write("请输入你的年龄：");
string ageString = Console.ReadLine();

// 将字符串转换为整数
int age = int.Parse(ageString);

// 计算并输出
Console.WriteLine($"明年你就 {age + 1} 岁了");
```

**更好的方式：使用 TryParse**

```csharp
Console.Write("请输入你的年龄：");
string ageString = Console.ReadLine();

// 安全转换，避免程序崩溃
if (int.TryParse(ageString, out int age))
{
    Console.WriteLine($"明年你就 {age + 1} 岁了");
}
else
{
    Console.WriteLine("输入无效，请输入数字！");
}
```

## 常见错误和调试

### 错误 1：忘记分号

```csharp
Console.WriteLine("Hello")  // ❌ 缺少分号
```

**错误信息：** `CS1002: ; expected`

### 错误 2：字符串未闭合

```csharp
Console.WriteLine("Hello");  // ❌ 缺少结束引号
```

**错误信息：** `CS1002: ; expected`

### 错误 3：拼写错误

```csharp
console.WriteLine("Hello");  // ❌ Console 首字母大写
```

**错误信息：** `CS0103: The name 'console' does not exist`

### 使用 Visual Studio 的调试功能

1. 在代码左侧点击设置断点（红点）
2. 按 F5 启动调试
3. 程序会在断点处暂停
4. 使用 F10/F11 逐行执行
5. 查看变量值

## 实践练习

### 练习 1：个人信息卡

编写一个程序，输出你的个人信息：

```
姓名：张三
年龄：20
爱好：编程
城市：北京
```

### 练习 2：简单计算器

编写一个程序，计算两个数字的和：

```
请输入第一个数字：10
请输入第二个数字：20
10 + 20 = 30
```

### 练习 3：问候程序

编写一个程序，询问用户姓名并输出个性化问候：

```
你叫什么名字？李四
你好，李四！很高兴见到你！
```

## 知识点总结

| 知识点 | 说明 |
|--------|------|
| `Console.WriteLine()` | 输出内容到控制台并换行 |
| `Console.Write()` | 输出内容到控制台不换行 |
| `Console.ReadLine()` | 读取用户输入的字符串 |
| 插值字符串 `$""` | 在字符串中嵌入变量和表达式 |
| 顶级语句 | 无需 Main 方法的简化写法 |
| `int.Parse()` | 将字符串转换为整数 |
| `int.TryParse()` | 安全地转换字符串为整数 |

## 下一步

在下一篇博文中，我们将学习 C# 的核心概念：**变量与数据类型**。

### 学习建议

- ✅ 动手尝试所有代码示例
- ✅ 完成至少 2 个实践练习
- ✅ 理解每一行代码的作用
- 💡 尝试修改代码，观察结果变化
- 📝 记录不懂的问题

---

**相关文章：**
- [C# 入门系列 (1) - C# 初探](../c-shu-ru-men-xi-lie-1-c-yuan-wei-shen-xue-xi-c)
- [C# 入门系列 (3) - 变量与数据类型](../c-shu-ru-men-xi-lie-3-bian-liang-yu-shu-ju-lei-xing)

**系列导航：**
- [C# 入门系列目录](../c-shu-ru-men-xi-lie-mu-lu)
