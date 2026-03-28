---
title: FastAPI 项目结构怎么设计，才能在功能变多后不失控
slug: how-to-structure-fastapi-projects
author: tomyi
status: published
is_featured: false
allow_comments: true
enable_jsx: false
summary: 按业务领域组织 FastAPI 代码，比按 routers/models/schemas 这种文件类型拆分更适合中大型项目。
excerpt: 按业务领域组织 FastAPI 代码，比按 routers/models/schemas 这种文件类型拆分更适合中大型项目。
meta_title: FastAPI 项目结构设计：领域驱动而非文件类型拆分 | FastAPI 工程化实践
meta_description: 深度讲解如何按业务领域组织 FastAPI 代码，避免类型驱动的拆分陷阱，构建可扩展的项目架构。
meta_keywords: FastAPI, 项目结构, 架构设计, 代码组织, 领域驱动, 工程化
tags:
  - FastAPI
  - Python
  - Architecture
---

# FastAPI 项目结构怎么设计，才能在功能变多后不失控

## 先说结论

如果你的 FastAPI 项目会持续长大，优先考虑按业务领域组织代码，而不是按 `routers`、`models`、`schemas`、`crud` 这类文件类型拆分。

小项目按类型拆分没有问题，但模块一多，按领域组织的收益会迅速变大。

## 为什么很多项目会越写越乱

很多教程会从这样的结构开始：

```text
app/
├── routers/
├── models/
├── schemas/
└── crud/
```

它的优点是简单，缺点也很明显：

- 改一个功能要在多个目录之间来回跳。
- 很难一眼看出系统到底有哪些业务模块。
- 同一个领域的代码被打散后，认知成本会持续变高。

对只有几个接口的小项目，这个问题不严重。对一个会持续迭代的服务，它几乎一定会成为维护成本。

## 更稳的做法：按领域组织

更适合中大型项目的结构通常长这样：

```text
src/
├── auth/
│   ├── router.py
│   ├── schemas.py
│   ├── models.py
│   ├── service.py
│   ├── dependencies.py
│   └── exceptions.py
├── posts/
│   ├── router.py
│   ├── schemas.py
│   ├── models.py
│   ├── service.py
│   ├── dependencies.py
│   └── exceptions.py
├── config.py
├── database.py
└── main.py
```

这个结构的核心不是“文件名固定”，而是一个原则：

同一个业务领域里的路由、模型、校验、业务逻辑和依赖项，尽量放在一起。

## 这种结构对日常开发的好处

它最大的价值不是“看起来更高级”，而是更顺手。

- 新人更快看懂项目边界。
- 改功能时文件跳转更少。
- 做重构时更容易判断影响范围。
- 模块独立性更强，未来拆分服务也更自然。

很多结构讨论都会落入“优雅不优雅”的层面，但真正影响开发体验的是认知切换次数。按领域组织，本质上是在降低这种切换。

## 它和领域驱动设计是什么关系

很多人第一次看到这种结构时，会问一句：这是不是 DDD。

更准确的回答是：它和领域驱动设计方向一致，但它本身还不等于 DDD。

两者最接近的地方在于，它们都强调一件事:

代码应该优先围绕业务领域组织，而不是优先围绕技术类型组织。

所以当你把项目拆成：

- `auth/`
- `posts/`
- `users/`

而不是拆成：

- `routers/`
- `models/`
- `schemas/`
- `crud/`

你其实已经在吸收 DDD 最有价值的一部分了，也就是让代码边界尽量贴近业务边界。

但 DDD 关心的内容远不止目录结构。它还会继续追问这些问题：

- 你的领域模型是不是围绕真实业务概念建立的
- 业务规则是不是封装在领域对象内部
- 实体、值对象、聚合、领域服务的边界是否清楚
- 不同子域之间的依赖方向是否合理
- 代码语言是不是和业务语言一致

而这篇文章讨论的重点没有那么重。这里更关注的是工程组织层面的问题：

- 文件怎么放更容易理解
- 模块怎么拆更容易维护
- HTTP 层、数据模型和业务逻辑怎么分开
- 团队协作时怎么减少跨目录跳转和混乱

所以更合适的理解方式是：

- 按文件类型拆分，是技术导向组织
- 按领域拆分，是业务导向组织
- DDD，则是在业务导向组织之上继续做更深入的业务建模

也就是说，这种项目结构更像是通往 DDD 的一个实用过渡层。

它不会强迫团队一开始就引入完整的 DDD 术语和模式，但会先把最有收益的一步做好：让代码先围绕业务模块站稳。

对大多数 FastAPI 项目来说，这通常是更现实的路径。因为很多团队还没复杂到需要完整 DDD，但已经复杂到不能继续把代码塞进 `routers/`、`models/`、`crud/` 这种全局目录里了。

## 一个实用的职责拆分

在单个模块里，我建议保留一套相对稳定的职责边界：

- `router.py`：HTTP 层，处理路径、状态码、依赖和响应模型。
- `schemas.py`：请求和响应的 Pydantic 模型。
- `models.py`：数据库模型。
- `service.py`：业务逻辑。
- `dependencies.py`：资源获取、权限校验、业务前置验证。
- `exceptions.py`：模块特定异常。

只要团队对这些边界理解一致，文件名本身不是重点。

## 跨模块调用时，不要偷懒

当一个模块依赖另一个模块时，尽量使用显式导入，而不是模糊导入。

```python
from src.auth import constants as auth_constants
from src.posts.service import publish_post
```

这样做的好处很直接：

- 依赖关系更清楚。
- 代码审查更容易。
- 命名冲突更少。

## 我更偏向模块级显式导入

在按领域组织的项目里，我通常不太喜欢把类型和函数散着直接导入到当前文件作用域里，比如：

```python
from src.posts.schemas import PostCreate, PostResponse
from src.posts.service import create_post
from src.posts.dependencies import valid_post
```

这种写法当然能用，但当文件开始变大以后，名字来源会越来越不明显。

我更偏向这种方式：

```python
from src.posts import dependencies, schemas, service
```

使用时显式写出来源：

```python
@router.post("/", response_model=schemas.PostResponse)
async def create_post(payload: schemas.PostCreate):
    return await service.create_post(payload)
```

这样做有几个很实际的好处：

- 一眼就能看出某个名字属于哪一层。
- `schemas.PostCreate` 和 `service.create_post` 这种写法边界感更强。
- 不同模块里重复出现的 `Create`、`Response`、`ErrorCode` 之类的名字不容易混淆。
- 当你做代码审查或重构时，依赖来源更清楚。

这种风格和“按领域组织”的目标其实是统一的。目录层面按领域划分，导入层面也尽量保留来源信息，代码的结构感会更稳定。

当然，这也不是必须机械执行的规则。如果一个文件特别小，或者只导入一两个对象，直接导入单个名字也完全可以接受。

关键不在于某一种导入语法，而在于：

- 来源是否清楚
- 风格是否一致
- 是否帮助你看出模块边界

如果一种导入方式让你读代码时更容易判断“这个名字来自 schema 层、service 层还是依赖层”，那它通常就是更适合团队的写法。

## 什么时候不用这么拆

这套结构不是银弹。

如果你的项目只有几个接口、生命周期很短，或者就是一个内部小工具，那么按类型拆分甚至单文件组织都可能更划算。

判断标准很简单：

- 业务模块会不会继续增加。
- 团队会不会变成多人协作。
- 你是否已经开始频繁跨目录找同一个功能的代码。

如果这三个问题里有两个答案是“会”，就可以考虑切到按领域组织。

## 一个可以直接拿去改的模板

```text
src/
├── users/
│   ├── router.py
│   ├── schemas.py
│   ├── models.py
│   ├── service.py
│   └── dependencies.py
├── billing/
│   ├── router.py
│   ├── schemas.py
│   ├── service.py
│   ├── client.py
│   └── exceptions.py
├── config.py
├── database.py
└── main.py
tests/
├── users/
└── billing/
```

测试目录尽量镜像业务目录，这样定位成本也会更低。

## 落地清单

- 先列出当前项目的业务领域，而不是先画目录树。
- 每个领域先放 `router.py`、`schemas.py`、`service.py` 三个核心文件。
- 把数据库模型和 HTTP 层逻辑分开。
- 把可复用校验移到 `dependencies.py`。
- 不要一开始就把 `utils.py` 塞满，能命名清楚的职责尽量单独放。

## 结构只是开始

项目结构值得最先讨论，不是因为它最基础，而是因为它往往决定了一个项目在功能持续增加之后还能不能继续高效演进。

但结构只是第一层。目录拆得再清楚，如果你在 `async def` 里继续写阻塞调用，或者没有明确区分异步 I/O、同步 I/O 和 CPU 密集任务，代码依然会很快变得难维护。

这也是为什么下一篇要继续讨论 FastAPI 里另一个更容易被误用的问题：[FastAPI 里什么时候该用 async，什么时候该用线程池](./02-async-boundaries-in-fastapi.md)。很多性能问题其实是边界问题，而不是框架问题。
