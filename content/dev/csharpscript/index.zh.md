---
title: 使用C#实现脚本功能
date:  2026-04-03
description: 介绍一种.Net 8+实现脚本功能的一种方式
toc: true
slug: csharpscript
categories:
    - Dev
tags:
    - csharp
    - script
---

## CS-Script介绍

CS-Script 是一个基于 CLR 的脚本系统，采用符合 ECMA 标准的 C# 作为编程语言。最成熟的 C# 脚本解决方案之一。

直接复制安装,本篇文章目前最新的CS-Script为
``` xml
<PackageReference Include="CS-Script" Version="4.14.4" />
```
[cs-script github地址](https://github.com/oleg-shilo/cs-script)

官方给了一个demo用法，如下显示:
``` csharp
public interface ICalc
{
    int Sum(int a, int b);
}

// you can, but don't have to inherit your script class from ICalc
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
## 实战前言

那么用法就很简单了，我们需要定义一个接口，然后传递一些参数就行了.

``` csharp
  public interface IScriptProcessor : ISingleton
  {
      Task Execute(ScriptContext scriptContext);
  }
```

ScriptContext 这个对象就是我们需要的一些参数,一些对象,一般来说我们在开发的时候用的对象比较多,所以我们可以把要操作的对象

放在scriptContext里面,因为是引用类型所以修改起来也很方便

## 代码模板
### 防止格式化工具格式化

需要创建一个ScriptTemplateProcessor的cs文件,这个文件来当代码模板,编译的时候读取这个文件,记得设置为`复制到输出目录-始终复制`

<div style="background: #d4edda; border-left: 4px solid #28a745; color: #155724; padding: 12px 16px; border-radius: 4px; margin: 16px 0;">
  <strong>✅ cs文件好处：</strong>不容易写错,格式化方便,有代码提示
</div>

```csharp
using Doer.Script;

//DOT_NOT_CHANGE_TEMPLATE//using System;
//DOT_NOT_CHANGE_TEMPLATE//using System.Collections.Generic;
//DOT_NOT_CHANGE_TEMPLATE//using System.Linq;
//DOT_NOT_CHANGE_TEMPLATE//using System.Text;
//DOT_NOT_CHANGE_TEMPLATE//using System.Threading.Tasks;
//DOT_NOT_CHANGE_TEMPLATE//using HSMS4Net.Extensions;

/// <summary>
/// 脚本模板请勿修改!
/// </summary>
public class ScriptTemplateProcessor : IScriptProcessor
{
    private NLog.ILogger _logger;
    private string _equipmentId;
    private ScriptContext _context;

    public async Task Execute(ScriptContext scriptContext)
    {
        //初始化变量
        InitialVar(scriptContext);

        //DO_NOT_CHANGE_OR_DEL_{51B89679-FEE6-4FDF-9222-22A290B1533A}
        await Task.CompletedTask;
    }
    

    private void InitialVar(ScriptContext scriptContext)
    {
        if (scriptContext == null)
        {
            throw new ArgumentNullException(nameof(scriptContext), "ScriptContext cannot be null");
        }
        _context = scriptContext;

      

        _logger = scriptContext.Logger;
        if (_logger == null)
        {
            throw new Exception("scriptContext.Logger can be not null");
        }

        _equipmentId = scriptContext.EquipmentId;
        if (string.IsNullOrWhiteSpace(_equipmentId))
        {
            throw new Exception("scriptContext.EquipmentId cannot be null, empty, or whitespace");
        }
    }
}
```



### 缓存模版

作用时每个相同的脚本只需要缓存(编译)一次

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
        public static string InjectedStart = "//START_{D85A735E-5D6F-4B54-B199-300C888F549D}";
        public static string InjectedEnd = "//END_{943A6A08-0022-4BB7-98A0-1D5CEA10F952}";
        public static string ReplaceCode = "//DO_NOT_CHANGE_OR_DEL_{51B89679-FEE6-4FDF-9222-22A290B1533A}";

        private static readonly ConcurrentDictionary<string, IScriptProcessor> _scriptCache =
            new ConcurrentDictionary<string, IScriptProcessor>();

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
        /// 编译脚本并缓存脚本
        /// </summary>
        /// <param name="userScriptCode"></param>
        /// <param name="context"></param>
        /// <returns></returns>
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
        /// 执行并缓存脚本
        /// </summary>
        /// <param name="userScriptCode"></param>
        /// <param name="context"></param>
        /// <returns></returns>
        public static async Task ExecuteScriptAsync(string userScriptCode, ScriptContext context)
        {
            string scriptHash = ComputeHash(userScriptCode);

            IScriptProcessor processor = _scriptCache.GetOrAdd(scriptHash, hash =>
            {
#if DEBUG
                Console.WriteLine($"{DateTime.Now:HH:mm:ss.fff} 脚本未缓存，开始编译...");
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
    }
}
```



### 替换模板
```csharp
using System.Text;

namespace Doer.Script
{
    public class ScriptBuilder
    {
        public static string InjectedStart = "//START_{D85A735E-5D6F-4B54-B199-300C888F549D}";
        public static string InjectedEnd = "//END_{943A6A08-0022-4BB7-98A0-1D5CEA10F952}";
        public static string ReplaceCode = "//DO_NOT_CHANGE_OR_DEL_{51B89679-FEE6-4FDF-9222-22A290B1533A}";

        public static string FuckCodeFormat = "//DOT_NOT_CHANGE_TEMPLATE//";

        public static DateTime _fileLastWrite = DateTime.Now.AddMinutes(-19960521);
        public static string _scriptFileTemp;

        public static string Build(string code)
        {
            var file = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, @"ScriptTemplateProcessor.cs");
            if (!File.Exists(file))
            {
                throw new FileNotFoundException(file);
            }

            //缓存策略,不用每次都读取文件模板
            var lastWriteTime = File.GetLastWriteTime(file);

            if (lastWriteTime != _fileLastWrite)
            {
                _fileLastWrite = lastWriteTime;

                var temp = File.ReadAllText(file, Encoding.UTF8);
                _scriptFileTemp = temp;
            }

            if (string.IsNullOrWhiteSpace(_scriptFileTemp))
            {
                throw new Exception($"Read _scriptTemp error,please check Doer ScriptBuilder");
            }
            var injected = $"{InjectedStart}\n{code}\n{InjectedEnd}";
            return _scriptFileTemp.Replace(ReplaceCode, injected).Replace(FuckCodeFormat, "");
        }
    }
}
```



## 代码编译

### 定位代码错误行号
在`CachedScriptExecutor`类中如下代码负责定位代码错误行号,其原理是利用CSScriptLib报错的信息进行正则提取减去代码模板开始行的位置,得到用户真正写的脚本错误的行号

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



### 最终使用

```csharp
       private async Task ExecuteScripts(string scriptCode, ScriptContext scriptContext)
       {
           if (!string.IsNullOrWhiteSpace(scriptCode))
           {
               try
               {
                   var script = ScriptBuilder.Build(scriptCode);
                   await CachedScriptExecutor.ExecuteScriptAsync(script, scriptContext);
               }
               catch (Exception ex)
               {
                   _logger.Error(ex, $"执行脚本异常");
               }
           }
       }
```



