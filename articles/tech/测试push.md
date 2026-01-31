---
allow_comments: true
author: tomyi
author_id: 019c0e79-300b-7322-afd4-8b36627a2530
category: tech
category_id: 019c0f8d-771b-73df-8ca7-3745697eea9c
date: '2026-01-31 02:25:04'
enable_jsx: false
is_featured: false
meta_description: ''
meta_keywords: ''
meta_title: ''
slug: test-push-yv44os
status: published
summary: 测试 Push 这篇文章的主要目的是为了测试 Git 的 Push 流程以及 CI/CD 部署是否正常。 1. 验证元数据 检查 author 字段是否正确显示为
  tomyi。 2. 验证内容渲染 代码块 数学公式 验证 LaTeX 渲染： 流程图 验证 Mermaid 渲染：
tags:
- test
- git
title: 测试 Push
use_server_rendering: true
---

# 测试 Push

这篇文章的主要目的是为了测试 Git 的 Push 流程以及 CI/CD 部署是否正常。

## 1. 验证元数据

检查 `author` 字段是否正确显示为 **tomyi**。

## 2. 验证内容渲染

### 代码块

```javascript
console.log("Hello, Blog!");
const isDeployed = true;
if (isDeployed) {
  console.log("Deployment successful!");
}
```

### 数学公式

验证 LaTeX 渲染：

$$
f(x) = \int_{0}^{x} t^2 dt = \frac{x^3}{3}
$$

### 流程图

验证 Mermaid 渲染：

```mermaid
gitGraph
   commit
   commit
   branch develop
   checkout develop
   commit
   commit
   checkout main
   merge develop
   commit
   commit
```