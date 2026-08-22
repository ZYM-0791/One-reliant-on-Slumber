---
title: PHP代码审计速查
published: 2026-08-22
description: 做了几道CTF Web题后感觉要专门学一下php，找了个课程，本文是整理的一些PHP语法速查
tags: [CTF, Web安全, PHP，源码审计]
category: 技巧整理
draft: false
---

>现在只有一小部分，后面这个分类会更新

## 一、弱类型

### 1.1 弱比较 vs 强比较

PHP 是弱类型语言（大概就是不用声明类型），`==` 会自动转换类型再比较，这是最常被利用的特性

```php
"0" == 0            // true
"0e12345" == "0e54321"  // true！科学计数法都等于0
"1admin" == 1       // true！字符串开头数字部分被提取
"admin" == 0        // true
null == false       // true
"" == false         // true
```

>看到 `==` 要警惕。像 `if ($input == 0)` 这种，传 `"abc"` 也能过——因为 `"abc"` 被转成 `0`

## 二、超全局变量（用户可控的入口）

审计第一步永远是找输入(因为输入的就是我们可控制或可利用的，sql注入和文件上传等都是),PHP里用户可控的输入都从超全局变量来：

| 变量 | 来源 | 可控程度 |
|------|------|---------|
| `$_GET` | URL 参数 | **完全可控** |
| `$_POST` | POST body | **完全可控** |
| `$_REQUEST` | GET+POST+COOKIE | **完全可控** |
| `$_COOKIE` | Cookie | **完全可控** |
| `$_SERVER['HTTP_*']` | HTTP 请求头 | **完全可控** |

> `$_SERVER` 里所有以 `HTTP_` 开头的键都来自客户端请求头，**完全可控**。比如 `HTTP_HOST`、`HTTP_USER_AGENT`、`HTTP_X_FORWARDED_FOR`

## 三、文件包含与伪协议

### 3.1 文件包含函数

```php
include($file);       // 失败只警告，继续执行
require($file);       // 失败报致命错误，停止执行
include_once($file);  // 同include，但只包含一次
require_once($file);  // 同require，但只包含一次
```

`include` 不只是"读文件"，而是**把文件内容当 PHP 代码执行**。这是文件包含漏洞的核心

### 3.2 PHP 伪协议

```php
// 读源码
?file=php://filter/read=convert.base64-encode/resource=flag.php

// 执行POST body中的代码（需 allow_url_include=On）
?file=php://input
// POST body: <?php system('id');?>

// 数据流封装（需 allow_url_include=On）
?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCdpZCcpOz8+

> `php://filter` 不受 `allow_url_include` 限制，是最常用的伪协议，其他可能需要额外开启配置

## 四、危险函数速查

### 4.1 代码执行

```php
eval($_GET['code']);                        // 执行PHP代码
assert("eval($_POST[0])");                  // PHP7前可执行代码
call_user_func($_GET['f'], $_GET['a']);     // 动态调用函数
$func = $_GET['f']; $func();               // 可变函数调用
```

### 4.2 命令执行

```php
system('whoami');       // 执行并直接输出
exec('whoami');         // 执行，返回最后一行
shell_exec('whoami');   // 执行，返回完整输出
passthru('whoami');     // 执行并原始输出
```

### 4.3 变量覆盖

```php
// $$ 可变变量
foreach ($_GET as $key => $value) {
    $$key = $value;  // ?flag=x → $flag='x'
}

// extract
extract($_GET);  // 直接把GET参数导入变量
```

## 五、过滤绕过

### 6.1 SQL 注入绕过

| 过滤内容 | 绕过方式 |
|---------|---------|
| `select` | `SeLeCt`、`selselectect`、`/*!select*/` |
| 空格 | `/**/`、`%09`、`%0a`、括号 `(select(1))` |
| `database()` | `schema()` |
| `substr()` | `mid()`、`substring()` |
| `=` | `like`、`regexp`、`in`、`<>` |
| 逗号 | `join`、`substr(x from 1 for 1)` |

## 六、审计技巧

总结 PHP 代码审计的流程：

**找输入**（`$_GET`/`$_POST`/`$_SERVER`）→ **追数据流**（变量赋值、函数传参）→ **看终点**（SQL/命令/文件/代码执行）→ **查过滤**（有没有过滤、能不能绕过）→ **构造 payload**

> 这部分会持续更新，后面学到新的知识点再补，目前是做完了几道题后借鉴了别人总结的一些点整理的，后续做题遇到新的或者跟语法课程学的时候想到可以作为洞的会更新

---

Orion notes · 学习·实践·分享
