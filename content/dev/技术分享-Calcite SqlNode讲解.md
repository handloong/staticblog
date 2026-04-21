---
title: Calcite SqlNode讲解
date: 2026-04-21
description: 本文章由何进进投稿
toc: true
slug: calcite-sqlnode
categories:
    - Dev
    - 技术分享
tags:
    - Calcite
    - 技术分享
---

Apache Calcite 是一个**开源的框架**，用于**构建数据管理系统**。它的核心定位是：**为各种数据源提供统一的 SQL 接口、优化和执行引擎，而不直接存储数据** 。现在讨论框架中 `SqlNode` 类的实现。`SqlNode` 是 Calcite 的核心类之一，表示 SQL 抽象语法树（AST）中的节点。SQL 的每个组成部分（如 SELECT、WHERE 子句、函数调用等）都可以表示为一个 `SqlNode` 或其子类的实例。以下是代码中的关键点和功能解释：

------

### 1. **类的定义与作用**

- **`SqlNode` 是一个抽象类**，它定义了 SQL 语法树节点的基本行为。
- 子类包括：
  - `SqlLiteral`（表示常量值）
  - `SqlIdentifier`（表示表名、列名等标识符）
  - `SqlCall`（表示函数调用或表达式）
  - `SqlNodeList`（表示节点列表，如 SELECT 的字段列表）

**作用**：
`SqlNode` 是 SQL 语句的核心组成部分，提供统一接口处理不同类型的 SQL 组件。

------

### 2. **主要成员变量**

- `protected final SqlParserPos pos`
  - 表示节点在 SQL 源代码中的位置，包含行号和列号，用于错误定位。
  - **构造函数**：所有 `SqlNode` 子类都需要提供位置信息，`requireNonNull(pos, "pos")` 确保位置不为 null。

------

### 3. **关键方法**

以下方法是 `SqlNode` 的核心功能：

#### (1) **SQL 转字符串**

```java
public SqlString toSqlString(UnaryOperator<SqlWriterConfig> transform)
```

- 作用：将 `SqlNode` 转换为标准 SQL 字符串。

- 实现：调用 `unparse` 方法（子类重写此方法）实现 SQL 拼接。

- 示例用途：

  ```java
  SqlNode sqlNode = ...; // SQL 抽象树节点
  SqlString sql = sqlNode.toSqlString(dialect -> dialect.withAlwaysUseParentheses(true));
  ```

#### (2) **`unparse` 方法**

```java
public abstract void unparse(SqlWriter writer, int leftPrec, int rightPrec);
```

- 作用：将 `SqlNode` 写入 `SqlWriter`，生成 SQL。
- 参数：
  - `leftPrec` 和 `rightPrec`：节点的左右操作符优先级，用于决定是否添加括号。
- 实现：由具体子类完成（如 SELECT、INSERT 的生成）。

#### (3) **克隆功能**

```java
public abstract SqlNode clone(SqlParserPos pos);
```

- 作用：创建当前节点的副本，可以改变位置信息。

- 示例：

  ```java
  SqlNode newNode = oldNode.clone(new SqlParserPos(1, 10));
  ```

#### (4) **验证 SQL 合法性**

```java
public abstract void validate(SqlValidator validator, SqlValidatorScope scope);
```

- 作用：验证 SQL 是否符合语法和语义规则。
- 示例用途：在 SQL 执行前检查表、列是否存在，或检查操作是否合法。

#### (5) **节点比较**

```java
public abstract boolean equalsDeep(@Nullable SqlNode node, Litmus litmus);
```

- 作用：深度比较两个节点是否相等。
- 用途：在优化器中比较两个 SQL 表达式是否语义相等。

------

### 4. **静态工具方法**

`SqlNode` 提供了一些静态工具方法，方便操作节点列表：

#### (1) **深度比较节点**

```java
public static boolean equalDeep(@Nullable SqlNode node1, @Nullable SqlNode node2, Litmus litmus)
```

- 作用：递归比较两个节点及其子节点是否相等。
- 示例用途：在 SQL 优化过程中判断两个表达式是否等价。

#### (2) **创建节点列表**

```java
public static <T extends @Nullable SqlNode> Collector<T, ArrayList<@Nullable SqlNode>, SqlNodeList> toList()
```

- 作用：将多个节点收集为一个 `SqlNodeList`，表示节点集合。
- 用途：如 SELECT 的字段列表或 GROUP BY 子句。

------

### 5. **子类实现示例**

`SqlNode` 是一个抽象类，子类必须实现：

- `unparse`：如何将节点转为 SQL。
- `validate`：如何验证节点的合法性。
- `clone`：如何复制节点。

**例如：**

- `SqlLiteral` 实现 `unparse` 将常量值转换为字符串。
- `SqlCall` 实现 `unparse` 生成函数调用。

------

### 6. **实际用途**

在 Calcite 框架中，`SqlNode` 被广泛用于：

- **SQL 解析阶段**：从用户输入的 SQL 生成 AST（抽象语法树）。
- **SQL 验证阶段**：检查语法和语义。
- **SQL 转换阶段**：将 AST 转换为其他 SQL 方言（如 MySQL、PostgreSQL）。
- **SQL 优化阶段**：修改或重写 AST 以提高查询性能。

------

### 7. **总结**

`SqlNode` 是 Calcite 中 SQL 抽象语法树的基类，封装了 SQL 节点的基础行为，具有以下特点：

- 抽象设计，支持各种 SQL 类型的扩展。
- 提供丰富的工具方法（如比较、克隆、转字符串）。
- 用于 SQL 的解析、验证、优化和生成。

你可以通过继承 `SqlNode` 来实现自定义 SQL 结构，例如支持特定数据库的方言功能或自定义的 SQL 语法。