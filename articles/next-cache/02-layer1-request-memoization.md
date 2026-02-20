---
allow_comments: true
author: tomyi
enable_jsx: false
is_featured: false
meta_description: 深入理解 Next.js 请求记忆（Request Memoization）机制，了解它如何在单次请求内自动去重，让组件级数据获取成为可能。
meta_keywords: Next.js, 请求记忆, Request Memoization, React cache, 缓存, App Router, RSC
meta_title: Next.js 缓存第 1 层：请求记忆 | 为什么多个组件 fetch 同一数据只发一次请求
slug: nextjs-cache-02-request-memoization
status: published
summary: 请求记忆是 Next.js 四层缓存的第一层，它在单次请求内自动去重相同的 fetch 调用，是 App Router 组件级数据获取能成立的底层保障。
tags:
  - Next.js
  - 缓存
  - 前端架构
  - React
title: 02-第 1 层：请求记忆 (Request Memoization)
use_server_rendering: true
---

# 第 1 层：请求记忆 (Request Memoization)

## 它要解决什么问题？

在 App Router 中，每个服务端组件都可以独立获取数据——这是我们在背景篇中提到的"局部性和内聚性"优势。但这个优势带来了一个直接的副作用：**同一个页面中，多个组件可能请求完全相同的数据**。

比如一个典型的博客页面，Layout 需要用户信息来显示导航栏，Sidebar 需要用户信息来显示头像，Page 需要用户信息来判断权限，UserProfile 需要用户信息来渲染个人资料。四个组件各自独立地调用 `await getCurrentUser()`——如果每次调用都真正发一次网络请求，那就是 4 次完全重复的数据库查询。

请求记忆（Request Memoization）就是为了解决这个问题：**在单次请求的生命周期内，相同的 `fetch` 调用只真正执行一次，后续调用直接返回缓存结果**。它本质上就是函数级别的 memoize（记忆化），和算法中的 memoization 是同一个概念——相同输入，直接返回之前的结果，不重复计算。

这一层是 App Router "每个组件独立获取数据"这个设计能成立的底层保障。没有它，组件级数据获取就会变成性能灾难。

## 基本信息

| 属性         | 值                                                                 |
| ------------ | ------------------------------------------------------------------ |
| **位置**     | 服务端（内存）                                                     |
| **生命周期** | 单次请求——从服务端开始处理请求到响应返回，请求结束后缓存立即销毁   |
| **缓存内容** | `fetch` 调用的返回值（相同 URL + 相同参数作为缓存键）              |
| **失效时机** | 请求处理完成时自动失效，无需手动干预                               |
| **适用范围** | `fetch` 自动生效；`axios`、`prisma` 等需要用 React `cache()` 包裹  |
| **跨请求共享** | ❌ 不共享——每个用户的每次请求都是独立的缓存空间                   |

---

## 直观理解

想象一下，你的页面结构是这样的：

```mermaid
graph TB
    Layout[Layout 布局]
    Sidebar[Sidebar 侧边栏]
    Page[Page 页面]
    Profile[UserProfile 用户信息]

    Layout --> Sidebar
    Layout --> Page
    Page --> Profile

    Layout -.需要用户信息.-> API[getCurrentUser]
    Sidebar -.需要用户信息.-> API
    Page -.需要用户信息.-> API
    Profile -.需要用户信息.-> API

    style API fill:#f99,stroke:#333,stroke-width:2px
```

这 4 个组件都需要获取"当前用户信息"，你可能会写 4 次 `await getCurrentUser()`。

---

## 问题：会发 4 次请求吗？

### 传统做法：Prop Drilling 或客户端状态管理

在没有请求记忆的情况下，要避免重复请求，通常有两种做法：

**做法 1：Prop Drilling（属性钻取）**

```typescript
// ❌ 传统做法：顶层获取数据，层层传递
function Layout() {
  const user = await getCurrentUser(); // 第 1 次请求

  return (
    <>
      <Sidebar user={user} /> {/* 通过 props 传递 */}
      <Page user={user} /> {/* 通过 props 传递 */}
    </>
  );
}

function Sidebar({ user }) {
  return <div>{user.name}</div>;
}

function Page({ user }) {
  return <UserProfile user={user} />;
  {
    /* 继续传递 */
  }
}
```

**问题**：

- ❌ 代码冗长，需要一层层传递
- ❌ 中间组件被迫接收不需要的 props
- ❌ 难以维护——顶层函数要了解所有子组件的数据需求

**做法 2：客户端状态管理（如 TanStack Query）**

在传统 React 或 Next.js 客户端组件中，可以用 TanStack Query 这类库来实现类似的去重效果：

```typescript
"use client";

import { useQuery } from "@tanstack/react-query";

function Sidebar() {
  // TanStack Query 会根据 queryKey 自动去重
  const { data: user } = useQuery({
    queryKey: ["currentUser"],
    queryFn: getCurrentUser,
  });
  return <div>{user?.name}</div>;
}

function Header() {
  // 相同的 queryKey，不会重复请求
  const { data: user } = useQuery({
    queryKey: ["currentUser"],
    queryFn: getCurrentUser,
  });
  return <nav>{user?.name}</nav>;
}
```

TanStack Query 确实能解决去重问题，但它运行在客户端——数据获取发生在浏览器，需要额外的 JavaScript 发送到客户端，首屏会出现 loading 状态。而 Next.js 的请求记忆运行在服务端，数据在 HTML 返回之前就已经准备好了，用户看到的是完整渲染的页面，不需要等待客户端再发一轮请求。

---

### Next.js 的做法（请求记忆）

```typescript
// ✅ Next.js 做法：直接在每个组件请求
async function Layout() {
  const user = await getCurrentUser(); // 第 1 次：真正请求
  return (
    <>
      <Sidebar />
      <Page />
    </>
  );
}

async function Sidebar() {
  const user = await getCurrentUser(); // 第 2 次：复用第 1 次的结果
  return <div>{user.name}</div>;
}

async function Page() {
  const user = await getCurrentUser(); // 第 3 次：复用第 1 次的结果
  return <UserProfile />;
}

async function UserProfile() {
  const user = await getCurrentUser(); // 第 4 次：复用第 1 次的结果
  return <div>{user.email}</div>;
}
```

**优势**：

- ✅ 代码简洁，每个组件独立获取数据
- ✅ 不需要 Prop Drilling
- ✅ 只发送 1 次真正的请求

---

## 工作原理

```mermaid
sequenceDiagram
    participant Layout
    participant Sidebar
    participant Page
    participant Profile
    participant Memo as 请求记忆
    participant API as 数据库/API

    Layout->>Memo: getCurrentUser()
    Memo->>API: 第1次：真正请求
    API-->>Memo: 返回用户数据
    Memo-->>Layout: 返回数据

    Sidebar->>Memo: getCurrentUser()
    Note over Memo: 已有缓存，直接返回
    Memo-->>Sidebar: 返回数据（复用）

    Page->>Memo: getCurrentUser()
    Note over Memo: 已有缓存，直接返回
    Memo-->>Page: 返回数据（复用）

    Profile->>Memo: getCurrentUser()
    Note over Memo: 已有缓存，直接返回
    Memo-->>Profile: 返回数据（复用）

    Note over Memo: 请求结束，缓存销毁
```

---

## 代码示例

### 示例 1：多个组件请求相同数据

```typescript
// lib/api.ts
export async function getCurrentUser() {
  console.log("🔍 真正发送请求到数据库");

  const res = await fetch("https://api.example.com/user");

  return res.json();
}

// app/layout.tsx
export default async function Layout({ children }) {
  const user = await getCurrentUser(); // 🔍 第 1 次：真正请求
  console.log("Layout:", user.name);

  return (
    <html>
      <body>
        <Sidebar />
        {children}
      </body>
    </html>
  );
}

// components/sidebar.tsx
export default async function Sidebar() {
  const user = await getCurrentUser(); // ✅ 第 2 次：复用
  console.log("Sidebar:", user.name);

  return <div>{user.name}</div>;
}

// app/page.tsx
export default async function Page() {
  const user = await getCurrentUser(); // ✅ 第 3 次：复用
  console.log("Page:", user.name);

  return <div>{user.email}</div>;
}
```

**控制台输出**：

```
🔍 真正发送请求到数据库  ← 只有这一次真正请求
Layout: John Doe
Sidebar: John Doe
Page: John Doe
```

---

### 示例 2：不同参数的请求

```typescript
// 相同函数，相同参数 → 复用
await getPost(1); // 第 1 次：真正请求
await getPost(1); // 第 2 次：复用
await getPost(1); // 第 3 次：复用

// 相同函数，不同参数 → 不复用
await getPost(1); // 真正请求
await getPost(2); // 真正请求（参数不同）
await getPost(3); // 真正请求（参数不同）
```

---

## 适用范围

### ✅ 自动生效的场景

```typescript
// 1. 使用 fetch（自动记忆）
await fetch("https://api.example.com/data");

// 2. 使用 React cache（手动记忆）
import { cache } from "react";

const getUser = cache(async (id: number) => {
  return await db.user.findUnique({ where: { id } });
});
```

### ❌ 不生效的场景

```typescript
// 1. 使用 axios（不会自动记忆）
await axios.get("https://api.example.com/data");

// 2. 直接使用数据库客户端（不会自动记忆）
await prisma.user.findMany();

// 3. 使用其他 HTTP 库
await got("https://api.example.com/data");
```

---

## 如何让非 fetch 请求也支持记忆？

### 使用 React cache

```typescript
import { cache } from "react";
import { prisma } from "@/lib/prisma";

// ✅ 包裹数据库查询
export const getUser = cache(async (id: number) => {
  console.log("🔍 真正查询数据库");
  return await prisma.user.findUnique({
    where: { id },
  });
});

// 使用
async function Component1() {
  const user = await getUser(1); // 🔍 第 1 次：真正查询
  return <div>{user.name}</div>;
}

async function Component2() {
  const user = await getUser(1); // ✅ 第 2 次：复用
  return <div>{user.email}</div>;
}
```

---

## 生命周期

```mermaid
graph LR
    A[请求开始] --> B[第1次调用<br/>真正请求]
    B --> C[缓存结果]
    C --> D[第2次调用<br/>返回缓存]
    C --> E[第3次调用<br/>返回缓存]
    C --> F[第N次调用<br/>返回缓存]
    F --> G[请求结束]
    G --> H[缓存销毁]

    style B fill:#f99,stroke:#333,stroke-width:2px
    style D fill:#9f9,stroke:#333,stroke-width:2px
    style E fill:#9f9,stroke:#333,stroke-width:2px
    style F fill:#9f9,stroke:#333,stroke-width:2px
    style H fill:#999,stroke:#333,stroke-width:2px
```

**关键点**：

- ✅ 缓存只在**单次请求**内有效
- ✅ 请求结束后，缓存自动销毁
- ✅ 下一个用户的请求会重新开始

---

## 常见问题

### Q1: 会不会导致数据不一致？

**不会**。因为缓存只在单次请求内有效。

```typescript
// 用户 A 的请求
await getUser(1); // 查询数据库，得到 { name: 'Alice' }
await getUser(1); // 复用，返回 { name: 'Alice' }
// 请求结束，缓存销毁

// 用户 B 的请求（此时数据库中的数据已更新）
await getUser(1); // 重新查询数据库，得到 { name: 'Alice Updated' }
await getUser(1); // 复用，返回 { name: 'Alice Updated' }
// 请求结束，缓存销毁
```

---

### Q2: 如何禁用请求记忆？

**通常不需要禁用**，因为它只在单次请求内有效，不会导致数据过期。

如果确实需要禁用：

```typescript
// 方法 1：使用 AbortController（每次创建新的信号）
await fetch(url, { signal: AbortSignal.timeout(5000) });

// 方法 2：添加随机参数
await fetch(`${url}?_=${Date.now()}`);
```

> 注意：`cache: 'no-store'` 影响的是第 2 层（数据缓存），不是请求记忆。请求记忆的去重是基于 URL + 参数的引用相等性判断，要绕过它需要让每次调用的参数不同。

---

### Q3: 请求记忆和数据缓存有什么区别？

| 特性         | 请求记忆（第 1 层） | 数据缓存（第 2 层） |
| ------------ | ------------------- | ------------------- |
| **生命周期** | 单次请求            | 持久化（跨请求）    |
| **存储位置** | 内存                | 文件系统            |
| **作用范围** | 当前请求的所有组件  | 所有用户的所有请求  |
| **失效时机** | 请求结束            | 手动失效或过期      |

```mermaid
graph TB
    subgraph 请求记忆
        R1[用户A请求] --> M1[内存缓存]
        M1 --> R1E[请求结束，缓存销毁]

        R2[用户B请求] --> M2[内存缓存]
        M2 --> R2E[请求结束，缓存销毁]
    end

    subgraph 数据缓存
        R3[用户A请求] --> D[文件缓存]
        R4[用户B请求] --> D
        R5[用户C请求] --> D
        D --> P[持久化，跨请求共享]
    end

    style M1 fill:#e1f5ff,stroke:#333,stroke-width:2px
    style M2 fill:#e1f5ff,stroke:#333,stroke-width:2px
    style D fill:#fff4e1,stroke:#333,stroke-width:2px
```

---

## 总结

**请求记忆（Request Memoization）** 是 Next.js 缓存的第一层：

```mermaid
graph LR
    A[多个组件请求相同数据] --> B[请求记忆]
    B --> C[只发送1次真正请求]
    C --> D[其他组件复用结果]
    D --> E[请求结束，缓存销毁]

    style B fill:#e1f5ff,stroke:#333,stroke-width:2px
    style C fill:#9f9,stroke:#333,stroke-width:2px
```

**核心要点**：

- ✅ 自动去重，避免重复请求，减少数据库压力
- ✅ 只在单次请求内有效，请求结束后自动销毁，不会导致数据过期
- ✅ 让每个组件可以独立获取数据，不需要 Prop Drilling

**注意事项**：

- ⚠️ 只对 `fetch` 自动生效，`axios`、`prisma` 等需要手动用 React `cache()` 包裹
- ⚠️ 参数必须相同才会复用，不同参数会触发新请求
- ⚠️ 仅限服务端组件，客户端组件不支持

**下一步**：了解第 2 层 - 数据缓存（Data Cache），它会跨请求持久化数据。
