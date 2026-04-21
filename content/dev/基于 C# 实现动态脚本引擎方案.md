---
title: 基于 C# 实现动态脚本引擎方案
date: 2026-04-03
description: 深入解析 .NET 8+ 环境下利用 CS-Script 实现动态脚本加载与执行的最佳实践
toc: true
slug: csharpscript-dynamic-execution
categories:
    - Dev
    - 技术分享
tags:
    - csharp
    - script
    - dynamic-execution
---

## 1. CS-Script 技术概览

CS-Script 是目前 .NET 生态中最成熟、基于公共语言运行时 (CLR) 的动态脚本解决方案。它允许开发者使用符合 ECMA 标准的 C# 语言编写脚本，并通过程序集加载机制在运行时动态编译和执行代码。

**依赖安装**
本项目当前采用的 CS-Script 版本为 `4.14.4`：
``` xml
<PackageReference Include="CS-Script" Version="4.14.4" />
```
更多技术细节可查阅 [CS-Script GitHub 仓库](https://github.com/oleg-shilo/cs-script)。

**核心用法示例**
官方文档提供了典型的使用场景：通过接口泛型定义加载动态代码：
``` csharp
public interface ICalc
{
    int Sum(int a, int b);
}

// 脚本类继承自接口 (非强制，但推荐)
ICalc calc = CSScript.Evaluator
                     .LoadCode<ICalc>(@"using System;
                                        public class Script
                                        {
                                            public int Sum(int a, int b)
                                            {
                                                return a+b;
                                            }
                                        }");
int result = calc.Sum(1, 2);
```

## 2. 架构设计思路

实现逻辑遵循 **“接口定义 -> 上下文注入 -> 动态编译”** 的模式。

首先，我们需要定义一个标准化的接口，用于约束脚本的执行行为：
``` csharp
  public interface IScriptProcessor
  {
      Task Execute(ScriptContext scriptContext);
  }
```

**关于 `ScriptContext` 上下文对象**
`ScriptContext` 是承载脚本执行所需参数与依赖的核心载体。考虑到脚本通常需要操作丰富的业务对象，我们将所有需注入的依赖对象封装于该上下文中。由于 `ScriptContext` 为引用类型，脚本代码可直接修改其内部属性，从而实现灵活的状态共享与数据更新。

## 3. 核心实现模块

### 3.1 脚本模板构建 (Script Template)

为了防止代码格式化器（Formatter）破坏脚本结构或影响编译，我们采用外部模板文件策略。需创建一个 `ScriptTemplateProcessor.cs` 文件作为编译时的代码骨架，并在项目配置中将其设置为 `复制到输出目录 - 始终复制`。

> **✅ 采用 .cs 文件作为模板的优势**：
> - 避免手动拼接字符串导致语法错误。
> - 享受 IDE 的代码提示、语法高亮与智能重构功能。
> - 便于维护与版本控制。

```csharp
using Doer.Script;

//DOT_NOT_CHANGE_TEMPLATE//using System;
//DOT_NOT_CHANGE_TEMPLATE//using System.Collections.Generic;
//DOT_NOT_CHANGE_TEMPLATE//using System.Linq;
//DOT_NOT_CHANGE_TEMPLATE//using System.Text;
//DOT_NOT_CHANGE_TEMPLATE//using System.Threading.Tasks;
//DOT_NOT_CHANGE_TEMPLATE//using HSMS4Net.Extensions;

/// <summary>
/// 脚本模板类，请勿修改核心逻辑结构！
/// </summary>
public class ScriptTemplateProcessor : IScriptProcessor
{
    private NLog.ILogger _logger;
    private string _equipmentId;
    private ScriptContext _context;

    public async Task Execute(ScriptContext scriptContext)
    {
        // 1. 初始化上下文变量
        InitialVar(scriptContext);

        // DO_NOT_CHANGE_OR_DEL_{51B89679-FEE6-4FDF-9222-22A290B1533A}
        // 此处为用户脚本插入点
        await Task.CompletedTask;
    }
  

    private void InitialVar(ScriptContext scriptContext)
    {
        if (scriptContext == null)
        {
            throw new ArgumentNullException(nameof(scriptContext), "ScriptContext cannot be null");
        }
        _context = scriptContext;

        // 依赖注入：Logger
        _logger = scriptContext.Logger;
        if (_logger == null)
        {
            throw new Exception("scriptContext.Logger cannot be null");
        }

        // 依赖注入：设备ID
        _equipmentId = scriptContext.EquipmentId;
        if (string.IsNullOrWhiteSpace(_equipmentId))
        {
            throw new Exception("scriptContext.EquipmentId cannot be null, empty, or whitespace");
        }
    }
}
```

### 3.2 脚本编译与缓存 (CachedScriptExecutor)

为了提升系统性能，避免重复编译相同内容的脚本，我们引入了基于哈希值的编译缓存机制。

```csharp
using CSScriptLib;
using System.Collections.Concurrent;
using System.Security.Cryptography;
using System.Text;
using System.Text.RegularExpressions;

namespace Doer.Script
{
    public class CachedScriptExecutor
    {
        // 脚本注入标识符
        public static string InjectedStart = "//START_{D85A735E-5D6F-4B54-B199-300C888F549D}";
        public static string InjectedEnd = "//END_{943A6A08-0022-4BB7-98A0-1D5CEA10F952}";
        public static string ReplaceCode = "//DO_NOT_CHANGE_OR_DEL_{51B89679-FEE6-4FDF-9222-22A290B1533A}";

        // 线程安全缓存字典
        private static readonly ConcurrentDictionary<string, IScriptProcessor> _scriptCache =
            new ConcurrentDictionary<string, IScriptProcessor>();

        /// <summary>
        /// 计算脚本内容的 SHA256 哈希值，用于缓存键
        /// </summary>
        private static string ComputeHash(string input)
        {
            using (var sha256 = SHA256.Create())
            {
                byte[] hashedBytes = sha256.ComputeHash(Encoding.UTF8.GetBytes(input));
                StringBuilder builder = new StringBuilder();
                for (int i = 0; i < hashedBytes.Length; i++)
                {
                    builder.Append(hashedBytes[i].ToString("x2"));
                }
                return builder.ToString();
            }
        }

        /// <summary>
        /// 异步编译脚本并尝试从缓存获取
        /// </summary>
        /// <param name="userScriptCode">用户提供的原始脚本代码</param>
        /// <returns>编译结果 (是否成功, 异常信息)</returns>
        public static async Task<(bool, Exception?)> CompileScriptAsync(string userScriptCode)
        {
            string scriptHash = ComputeHash(userScriptCode);

            try
            {
                IScriptProcessor processor = _scriptCache.GetOrAdd(scriptHash, hash =>
                {
                    var startTime = DateTime.Now;
                    try
                    {
                        var compiledProcessor = CSScript.Evaluator.LoadCode<IScriptProcessor>(userScriptCode);
                        return compiledProcessor;
                    }
                    catch (Exception)
                    {
                        // 编译失败时移除缓存，防止脏数据
                        _scriptCache.TryRemove(hash, out _);
                        throw;
                    }
                });

                await Task.CompletedTask;
                return (true, null);
            }
            catch (Exception ex)
            {
                var mapped = MapCompilerErrorToUserCode(ex, userScriptCode);
                return (false, mapped ?? ex);
            }
        }

        /// <summary>
        /// 执行脚本：优先从缓存加载，无缓存则编译后执行
        /// </summary>
        public static async Task ExecuteScriptAsync(string userScriptCode, ScriptContext context)
        {
            string scriptHash = ComputeHash(userScriptCode);

            IScriptProcessor processor = _scriptCache.GetOrAdd(scriptHash, hash =>
            {
#if DEBUG
                Console.WriteLine($"{DateTime.Now:HH:mm:ss.fff} 脚本未命中缓存，开始编译...");
#endif
                var startTime = DateTime.Now;
                try
                {
                    var compiledProcessor = CSScript.Evaluator.LoadCode<IScriptProcessor>(userScriptCode);

#if DEBUG
                    var endTime = DateTime.Now;
                    Console.WriteLine($"{DateTime.Now:HH:mm:ss.fff} 脚本编译完成，耗时: {(endTime - startTime).TotalMilliseconds:F0} ms");
#endif
                    return compiledProcessor;
                }
                catch (Exception ex)
                {
                    _scriptCache.TryRemove(hash, out _);
                    var mapped = MapCompilerErrorToUserCode(ex, userScriptCode);
                    Console.WriteLine($"{DateTime.Now:HH:mm:ss.fff} 脚本编译失败: {(mapped ?? ex).Message}");
                    throw mapped ?? ex;
                }
            });

            await processor.Execute(context);
        }

        /// <summary>
        /// 错误诊断：将编译器抛出的内部行号映射回用户脚本的实际行号
        /// </summary>
        private static Exception? MapCompilerErrorToUserCode(Exception ex, string script)
        {
            try
            {
                var startIdx = script.IndexOf(InjectedStart);
                if (startIdx < 0) return null;
              
                // 计算用户代码起始行
                var pre = script.Substring(0, startIdx);
                var userStartLine = CountLines(pre) + 1;

                // 正则提取编译器报错坐标 (行，列)
                var m = Regex.Match(ex.Message, @"\((\d+),(\d+)\)");
                if (!m.Success) return null;
              
                var line = int.Parse(m.Groups[1].Value);
                var col = int.Parse(m.Groups[2].Value);

                if (line >= userStartLine)
                {
                    // 修正行号
                    var userLine = line - userStartLine;
                    // 替换错误信息中的虚拟脚本路径为真实路径
                    var msg = Regex.Replace(ex.Message, @"<script>\(\d+,\d+\)", $"UserScript({userLine},{col})");
                    return new Exception(msg, ex);
                }
                return null;
            }
            catch { return null; }
        }

        private static int CountLines(string text)
        {
            if (string.IsNullOrEmpty(text)) return 0;
            int count = 0;
            int idx = 0;
            while ((idx = text.IndexOf('\n', idx)) != -1)
            {
                count++;
                idx++;
            }
            return count;
        }
    }
}
```

### 3.3 动态代码构建 (ScriptBuilder)

该模块负责将用户提供的纯脚本代码与模板文件进行组装。通过计算文件修改时间实现本地缓存，减少文件 IO 开销。

```csharp
using System.Text;

namespace Doer.Script
{
    public class ScriptBuilder
    {
        public static string InjectedStart = "//START_{D85A735E-5D6F-4B54-B199-300C888F549D}";
        public static string InjectedEnd = "//END_{943A6A08-0022-4BB7-98A0-1D5CEA10F952}";
        public static string ReplaceCode = "//DO_NOT_CHANGE_OR_DEL_{51B89679-FEE6-4FDF-9222-22A290B1533A}";
      
        // 格式化保护标记
        public static string FuckCodeFormat = "//DOT_NOT_CHANGE_TEMPLATE//";

        // 文件缓存相关字段
        public static DateTime _fileLastWrite = DateTime.Now.AddMinutes(-19960521);
        public static string _scriptFileTemp;

        /// <summary>
        /// 组装完整脚本：读取模板 + 插入用户代码 + 移除格式保护符
        /// </summary>
        public static string Build(string code)
        {
            var file = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, @"ScriptTemplateProcessor.cs");
            if (!File.Exists(file))
            {
                throw new FileNotFoundException($"Template file not found: {file}");
            }

            // 智能缓存：仅在模板文件变更时重新读取
            var lastWriteTime = File.GetLastWriteTime(file);

            if (lastWriteTime != _fileLastWrite)
            {
                _fileLastWrite = lastWriteTime;
                _scriptFileTemp = File.ReadAllText(file, Encoding.UTF8);
            }

            if (string.IsNullOrWhiteSpace(_scriptFileTemp))
            {
                throw new Exception($"Failed to load script template. Please check Doer ScriptBuilder configuration.");
            }

            // 组装逻辑：模板 - 占位符 + 用户代码包裹层
            var injected = $"{InjectedStart}\n{code}\n{InjectedEnd}";
            return _scriptFileTemp.Replace(ReplaceCode, injected).Replace(FuckCodeFormat, "");
        }
    }
}
```

## 4. 关键技术详解

### 4.1 错误行号定位机制 (Error Line Mapping)

在动态编译中，编译器报错的行号通常基于**完整文件**（模板 + 用户代码）。为了让开发者能准确定位错误，我们需要在 `CachedScriptExecutor` 中实现行号映射算法：

1.  **定位起始点**：查找用户代码插入标记 (`InjectedStart`) 的位置。
2.  **计算偏移量**：统计插入标记前的行数，得到用户代码的起始行号。
3.  **正则解析**：提取编译器报错信息中的 `(行, 列)` 坐标。
4.  **修正与替换**：若报错行号大于起始行号，则减去偏移量得到用户行号，并替换错误消息中的源文件引用。

核心实现代码如下：
```csharp
 private static Exception? MapCompilerErrorToUserCode(Exception ex, string script)
 {
     try
     {
         var startIdx = script.IndexOf(InjectedStart);
         if (startIdx < 0) return null;
         var pre = script.Substring(0, startIdx);
         var userStartLine = CountLines(pre) + 1;

         var m = Regex.Match(ex.Message, @"\((\d+),(\d+)\)");
         if (!m.Success) return null;
         var line = int.Parse(m.Groups[1].Value);
         var col = int.Parse(m.Groups[2].Value);

         if (line >= userStartLine)
         {
             var userLine = line - userStartLine;
             var msg = Regex.Replace(ex.Message, @"<script>\(\d+,\d+\)", $"UserScript({userLine},{col})");
             return new Exception(msg, ex);
         }
         return null;
     }
     catch { return null; }
 }

 private static int CountLines(string text)
 {
     if (string.IsNullOrEmpty(text)) return 0;
     int count = 0;
     int idx = 0;
     while ((idx = text.IndexOf('\n', idx)) != -1)
     {
         count++;
         idx++;
     }
     return count;
 }
```

### 4.2 最终调用流程

整合 `ScriptBuilder` 和 `CachedScriptExecutor` 的完整调用示例：

```csharp
private async Task ExecuteScripts(string scriptCode, ScriptContext scriptContext)
{
    if (!string.IsNullOrWhiteSpace(scriptCode))
    {
        try
        {
            // 1. 构建完整脚本文件内容
            var script = ScriptBuilder.Build(scriptCode);
            
            // 2. 执行编译并运行（含缓存优化）
            await CachedScriptExecutor.ExecuteScriptAsync(script, scriptContext);
        }
        catch (Exception ex)
        {
            _logger.Error(ex, $"执行脚本异常，脚本内容：{scriptCode}");
        }
    }
}
```