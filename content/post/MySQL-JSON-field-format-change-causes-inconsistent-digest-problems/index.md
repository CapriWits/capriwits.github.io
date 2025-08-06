---
title: "MySQL JSON 字段格式变更导致摘要不一致问题及解决方案"
slug: "MySQL-JSON-field-format-change-causes-inconsistent-digest-problems"
date: 2025-08-06T23:28:25+08:00
author: CapriWits
image: mysql-cover.png
tags:
  - MySQL
categories:
  - Database
hidden: false
math: true
comments: true
draft: false
---

# MySQL JSON 字段格式变更导致摘要不一致问题及解决方案

![](https://img.shields.io/badge/MySQL-8.0.39-blue)

![](https://img.shields.io/badge/Golang-1.24.5-pink)

在实际开发中，为了增强数据结构的灵活性，MySQL 的 `JSON` 类型被广泛使用。然而，在使用过程中发现，**MySQL 会对插入的 JSON 数据在存储或查询时进行自动格式化处理**，例如：

- **调整键值对顺序**（基于键名排序）；
- **在 key 与 value 之间添加空格**；
- **去除或添加某些结构性字符（如换行）**。

这种格式调整虽然不影响 JSON 的语义，但在某些业务场景下却可能造成严重问题，例如**基于 JSON 数据计算的摘要（如 SHA1 或 MD5）发生不一致**。

---

# 现象举例

1. 在业务逻辑中使用 `{"key":"value"}` 插入 MySQL。
2. 插入前计算摘要为：

    ```json
    SHA1("{"key":"value"}") = 228458095a9502070fc113d99504226a6ff90a9a
    ```

3. 从 MySQL 查询结果为：

    ```json
    {"key": "value"}
    ```

   （注意 key 和 value 之间多了一个空格）

4. 查询结果的摘要为：

    ```json
    SHA1("{"key": "value"}") = e735dd8e68d8938109f02c383cc9d056c942a514
    ```


> 结果：两次摘要不同，无法用于数据一致性校验或签名验证。

---

# 根因分析

- **MySQL JSON 类型底层以二进制 JSON 存储**，查询时为了展示友好，可能会自动格式化。
- 不同语言或客户端（如 Java 的 Jackson、Go 的 encoding/json）对 JSON 的序列化顺序、空格控制也可能不同。
- 即使语义一致，**JSON 文本结构上的任何变化都会导致摘要计算结果不同**。

---

# Golang 示例

创建 MySQL 测试表

```sql
CREATE TABLE IF NOT EXISTS json_test
(
    id      INT AUTO_INCREMENT PRIMARY KEY,
    content JSON,
    digest  CHAR(40) GENERATED ALWAYS AS (SHA1(content)) STORED
)
```

测试代码

```go
package main

import (
	"crypto/sha1"
	"database/sql"
	"encoding/hex"
	"fmt"
	"log"

	_ "github.com/go-sql-driver/mysql"
)

type JSONTest struct {
	ID      int
	Content string // 原始JSON字符串
	Digest  string // MySQL计算的摘要
}

func main() {
	// 数据库配置
	db, err := sql.Open("mysql", "mysql8:233233@tcp(127.1:3306)/db1?charset=utf8mb4&parseTime=True&loc=Local")
	if err != nil {
		log.Fatal("数据库连接失败:", err)
	}
	defer db.Close()

	// 测试用例
	testCases := []struct {
		name  string
		input string
	}{
		{
			name:  "紧凑JSON",
			input: `{"name":"Alice","age":30,"active":true}`,
		},
		{
			name:  "带空格JSON",
			input: `{ "name": "Bob", "age": 25, "active": false }`,
		},
	}

	for _, tc := range testCases {
		fmt.Printf("\n===== 测试用例: %s =====\n", tc.name)
		fmt.Printf("原始JSON: %s\n", tc.input)

		// 在Go中计算预期摘要
		goDigest := computeSHA1(tc.input)
		fmt.Printf("Go计算的SHA1: %s\n", goDigest)

		// 插入数据库
		id, err := insertJSON(db, tc.input)
		if err != nil {
			log.Printf("插入失败: %v", err)
			continue
		}

		// 查询数据库
		result, err := queryJSON(db, id)
		if err != nil {
			log.Printf("查询失败: %v", err)
			continue
		}

		// 打印并验证结果
		fmt.Println("\n=== 数据库查询结果 ===")
		fmt.Printf("返回的JSON内容: %s\n", result.Content)
		fmt.Printf("MySQL计算的摘要: %s\n", result.Digest)

		// 摘要比对
		if result.Digest == goDigest {
			fmt.Println("✅ 摘要一致: MySQL与Go计算结果相同")
		} else {
			fmt.Println("❌ 摘要不一致: MySQL与Go计算结果不同")
		}

		// JSON内容比对
		if result.Content == tc.input {
			fmt.Println("✅ JSON内容一致: 与原始输入完全相同")
		} else {
			fmt.Println("⚠️ JSON内容变化: 与原始输入存在差异")
			fmt.Printf("原始长度: %d | 返回长度: %d\n", len(tc.input), len(result.Content))
		}
	}
}

func computeSHA1(data string) string {
	hasher := sha1.New()
	hasher.Write([]byte(data))
	return hex.EncodeToString(hasher.Sum(nil))
}

func insertJSON(db *sql.DB, jsonStr string) (int64, error) {
	res, err := db.Exec(`INSERT INTO json_test (content) VALUES (?)`, jsonStr)
	if err != nil {
		return 0, err
	}
	return res.LastInsertId()
}

func queryJSON(db *sql.DB, id int64) (*JSONTest, error) {
	var result JSONTest
	err := db.QueryRow(
		`SELECT id, content, digest FROM json_test WHERE id = ?`, id,
	).Scan(&result.ID, &result.Content, &result.Digest)
	return &result, err
}

```

> ===== 测试用例: 紧凑JSON =====  
原始JSON: {"name":"Alice","age":30,"active":true}  
Go计算的SHA1: 96179b68d41495dfd8f1c9bc7a093d92b56db6fe
>
>
> === 数据库查询结果 ===  
> 返回的JSON内容: {"age": 30, "name": "Alice", "active": true}  
> MySQL计算的摘要: 2882e0534aec7725267c9295e89f61a47a3326ad  
> ❌ 摘要不一致: MySQL与Go计算结果不同  
> ⚠️ JSON内容变化: 与原始输入存在差异  
> 原始长度: 39 | 返回长度: 44  
>
> ===== 测试用例: 带空格JSON =====  
> 原始JSON: { "name": "Bob", "age": 25, "active": false }  
> Go计算的SHA1: 704bf774b19ca144d1e554968a72a6bf136088e0  
>
> === 数据库查询结果 ===  
> 返回的JSON内容: {"age": 25, "name": "Bob", "active": false}  
> MySQL计算的摘要: ce10e44c98b4332e900469a6b2ad740bbfe59343  
> ❌ 摘要不一致: MySQL与Go计算结果不同  
> ⚠️ JSON内容变化: 与原始输入存在差异  
> 原始长度: 45 | 返回长度: 43  
>

---

# 解决方案

## ✅ 方案一：业务侧生成摘要并写入

- 在写入数据库前，由业务逻辑使用紧凑 JSON（无空格、固定顺序）生成摘要值。
- 将该摘要作为独立字段插入数据库，避免依赖 MySQL 返回的 JSON 字符串。

```sql
INSERT INTO data_table (content, sha1) VALUES ('{"key":"value"}', '228458095a9502070fc113d99504226a6ff90a9a');
```

## ✅ 方案二：存储 JSON 字符串，而非 JSON 类型

- 将 `json` 字段声明为 `TEXT` 或 `VARCHAR` 类型。
- 避免 MySQL 对 JSON 内容进行解析和格式化，存什么、取什么。

```sql
CREATE TABLE data_table
(
    id      INT PRIMARY KEY,
    content TEXT,
    sha1    CHAR(40)
);
```

> ⚠️ 缺点是无法使用 JSON 类型的查询语法（如 JSON_EXTRACT）。

---

# 总结

MySQL 的 JSON 类型在插入与查询过程中可能会自动格式化，从而导致基于 JSON 字符串的摘要不一致。
为避免此类问题，应在业务层确保摘要计算的一致性，或避免使用 MySQL 的 JSON 类型进行签名和哈希计算的核心数据字段。