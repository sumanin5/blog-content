---
allow_comments: true
author: tomyi
author_id: 019c21db-4c02-7836-9d5b-36c688f8b4e0
category: fastapi-engineering-practice
category_id: 019d359a-ec80-7ce3-a755-4a2b1ca21df4
date: '2026-03-28 18:00:39'
enable_jsx: false
is_featured: false
meta_description: 系统讲解如何充分利用 FastAPI 的依赖注入系统，结合 REST 约定，让路由处理器更专注业务逻辑，减少重复代码。
meta_keywords: FastAPI, 依赖注入, Depends, REST 设计, 路由设计, 权限检查, 资源校验
meta_title: FastAPI 依赖注入与 REST 约定：如何让路由更简洁 | 工程化实践
slug: dependencies-and-rest-routes-7xzmro
status: published
summary: FastAPI 的依赖项不只是注入数据库会话。把资源校验、权限检查和 REST 路径命名统一起来，路由会轻很多。
tags:
- FastAPI
- Dependency Injection
- REST
title: 把依赖注入和 REST 约定结合起来，路由会干净很多
use_server_rendering: true
---

# 把依赖注入和 REST 约定结合起来，路由会干净很多

## 先说结论

FastAPI 的依赖项最好不要只用来注入数据库连接。

它更有价值的用法是把这些前置逻辑从路由里拿出去：

- 资源是否存在
- 当前用户是否有权限
- 当前资源是否满足业务前置条件

再配合稳定的 REST 路径命名，路由会变得非常干净。

## 先搞清楚 `Depends()` 到底在做什么

在看具体用法之前，先理解 FastAPI 的 `Depends()` 的执行机制：

```python
from fastapi import Depends

async def get_db():
    # 依赖项可以负责资源的初始化和清理
    db = AsyncSessionLocal()
    try:
        yield db
    finally:
        await db.close()

async def current_user(db: AsyncSession = Depends(get_db)):
    # 依赖项也可以继续依赖其他依赖项
    user = await auth_service.get_current_user(db)
    if not user:
        raise HTTPException(status_code=401, detail="Unauthorized")
    return user

@router.get("/posts")
async def list_posts(
    db: AsyncSession = Depends(get_db),
    user: User = Depends(current_user),
):
    return await posts_service.list(db, user.id)
```

这段代码的重点不是"少写了几行"，而是你可以明确看到一棵依赖树：
路由依赖 `current_user`，`current_user` 又依赖 `get_db`，而框架会负责把这棵树解析并执行。

这里有几件事是框架自动帮你做的：

- **执行时机**：所有 `Depends()` 在路由函数调用之前执行。如果任何一个依赖抛出异常，路由函数根本不会运行。
- **自动传参**：依赖的返回值会自动注入到路由参数里。`get_db` 返回的 session 就是 `db` 参数的值。
- **依赖树解析**：`current_user` 依赖了 `get_db`，路由也依赖了 `get_db`。FastAPI 会自动解析这棵依赖树，而且**同一个请求里同一个依赖默认只执行一次**（缓存行为）。
- **生命周期管理**：用 `yield` 的依赖，`yield` 之前是"初始化"，之后是"清理"。框架会确保请求结束后执行清理逻辑。

理解了这个机制，后面的用法就不会觉得"魔法"了。

## 路由为什么容易变脏

我们写接口真正想表达的，通常只是业务动作本身——创建一篇文章、更新一个用户、删除一条评论。但数据库怎么连接、用户身份怎么解析、某个资源是否存在，这些细节不是不重要，而是不应该和业务逻辑混在一起写。

很多项目的路由函数最后会变成这样：

```python
@router.patch("/posts/{post_id}")
async def update_post(post_id: int, payload: PostUpdate):
    # 资源获取
    post = await posts_service.get(post_id)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")

    # 当前用户获取
    user = await auth_service.current_user()
    # 权限校验
    if post.author_id != user.id:
        raise HTTPException(status_code=403, detail="Forbidden")

    # 业务前置条件校验
    if post.status == "archived":
        raise HTTPException(status_code=400, detail="Archived post cannot be updated")

    return await posts_service.update(post_id, payload)
```

它不是错，但这里已经混进了四类不同职责：资源获取、当前用户获取、权限校验、业务更新动作。只要类似的接口一多，这些逻辑就会在多个路由里不断重复。

从设计原则上看，这类路由首先违背的是单一职责原则。路由本来更适合负责 HTTP 输入输出和依赖组合，但这里它同时承担了资源查询、鉴权、权限判断、业务前置条件校验和最终更新动作。职责一旦混在一起，后面几乎一定会越改越难看。

而当规则继续增长时，它又会进一步伤到开闭原则。比如以后你可能继续加这些逻辑：

- 管理员可以修改归档文章
- 普通作者只能修改草稿
- 某些文章在审核中禁止编辑
- 不同租户之间不能互相修改资源

如果这些规则都继续往同一个路由里塞，结果通常就是：

- 多一层 `if`
- 多一段权限判断
- 多一段状态判断
- 多一段异常分支

这就是很多屎山代码形成的根本原因之一：
不是一开始就写得特别糟，而是一个本来承担过多职责的函数，在每次需求变化时都只能继续被修改、继续堆逻辑，最后变成没人愿意碰的大函数。

## 用依赖项把前置逻辑分层

把这些检查拆成依赖项，让每一层只做一件事：

```python
from fastapi import Depends, HTTPException


async def valid_post(post_id: int) -> Post:
    # 第一层：资源存在性校验
    post = await posts_service.get(post_id)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    return post


async def owned_post(
    post: Post = Depends(valid_post),
    user: User = Depends(current_user),
) -> Post:
    # 第二层：所有权校验
    if post.author_id != user.id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return post


async def editable_post(post: Post = Depends(owned_post)) -> Post:
    # 第三层：业务前置条件校验
    if post.status == "archived":
        raise HTTPException(status_code=400, detail="Archived post cannot be updated")
    return post
```

路由就会收敛成这样：

```python
@router.patch("/posts/{post_id}")
async def update_post(payload: PostUpdate, post: Post = Depends(editable_post)):
    return await posts_service.update(post.id, payload)
```

这组代码真正体现的是"前置条件逐层收敛"：

- `valid_post` 负责资源存在
- `owned_post` 负责权限
- `editable_post` 负责业务前置状态

这种写法背后的设计倾向和"组合优于继承"很接近——不鼓励把能力硬塞进一个越来越大的基类里，而是把能力拆开，在真正需要的地方组合起来。一个路由之所以能工作，是因为它声明并组合了自己需要的依赖，而不是继承了什么。

依赖注入最实际的价值，是帮你把几类职责边界理清：

- 路由负责 HTTP 输入输出
- 依赖项负责资源准备和前置校验
- service 负责业务动作

它不是额外增加复杂度，而是在复杂度开始增长之前，先把职责边界理顺。

## 路径参数命名统一，会直接影响依赖复用

这一点很容易被忽略。

很多逻辑确实适合提成依赖，但这些依赖能不能自然复用，不只取决于你怎么写依赖本身，也取决于你的接口设计和参数命名是否稳定。

比如你有两个接口：

- `/profiles/{profile_id}`
- `/creators/{creator_id}`

如果 `creator` 本质上也是一种 `profile`，那你把参数名写成两个不同名字，依赖复用就会变差。

更推荐这样做：

- `/profiles/{profile_id}`
- `/creators/{profile_id}`

然后让"创作者校验"建立在"资料存在校验"之上：

```python
# 基础依赖：校验 profile 是否存在
async def valid_profile(profile_id: int) -> Profile:
    profile = await profile_service.get(profile_id)
    if not profile:
        raise HTTPException(status_code=404, detail="Profile not found")
    return profile

# 扩展依赖：在 profile 存在的基础上，校验是否为创作者
async def valid_creator(profile: Profile = Depends(valid_profile)) -> Profile:
    if not profile.is_creator:
        raise HTTPException(status_code=403, detail="Not a creator")
    return profile

# 两个路由复用同一个 profile_id 参数名，依赖链自然串联
@router.get("/profiles/{profile_id}")
async def get_profile(profile: Profile = Depends(valid_profile)):
    return profile

@router.get("/creators/{profile_id}/posts")
async def get_creator_posts(creator: Profile = Depends(valid_creator)):
    return await posts_service.list_by_author(creator.id)
```

如果 `/creators/` 路径的参数名写成了 `creator_id`，`valid_profile` 就没法直接复用了，因为它期望的参数名是 `profile_id`。这不是为了形式一致，而是为了让依赖链更自然。

所以这一段想强调的不是"参数名要统一得好看"，而是：
如果你希望资源校验、权限校验和扩展依赖能够顺着同一条链自然组合，那 REST 路径和参数命名本身也要为这种复用服务。

## REST 约定能降低阅读成本

除了依赖项，路径本身也应该尽量稳定。

这里说的"稳定"，不是指所有接口永远不变，而是指你的接口命名方式应该有一套一致、可预期的规则。
只要团队看到路径，就能大致猜到它表示什么资源、支持什么操作、参数应该长什么样。

REST 风格之所以长期有价值，不是因为它"更标准"或者"更高级"，而是因为它降低了接口的理解成本。

推荐先遵循几个简单规则：

- 路径用名词，不用动词。
- 用 HTTP 方法表达动作。
- 查询条件放 query，不要塞进动词路径里。
- 对嵌套资源只表达必要层级。

先看一组更容易维护的写法：

```text
GET    /posts
POST   /posts
GET    /posts/{post_id}
PATCH  /posts/{post_id}
DELETE /posts/{post_id}
GET    /posts/{post_id}/comments
POST   /posts/{post_id}/comments
```

路径负责表达"资源是什么"，HTTP 方法负责表达"对资源做什么"。`/posts/{post_id}/comments` 表示文章下面的评论资源，这层嵌套关系一眼就清楚，不需要额外解释。

再对比一组更容易失控的写法：

```text
GET  /getPosts
POST /createPost
POST /deletePost
GET  /searchPosts
GET  /getPostComments
```

这类写法的问题不是"看不懂"，而是团队长期维护时会越来越难统一：

- 动作被重复写进了路径和 HTTP 方法里
- 同一种资源的接口很难聚合在一起看
- 搜索、过滤、删除、列表这些语义容易各写各的
- 后续做参数复用和依赖复用时，路径也更容易发散

比如"搜索文章"更自然的写法通常不是 `/searchPosts`，而是：

```text
GET /posts?q=fastapi
GET /posts?status=published
GET /posts?author_id=123
```

因为这些操作本质上仍然是在"获取 posts"，只是多了过滤条件。

## 不要把依赖链做得过深

依赖项也可能被滥用。

如果一个路由要经过五六层依赖才能看懂，阅读体验会下降得很快：

```python
# 依赖链太深：读路由时需要跳 6 层才能理解完整的前置条件
@router.post("/posts/{post_id}/comments")
async def create_comment(
    payload: CommentCreate,
    comment_context=Depends(valid_comment_context),
    # 要理解这个路由，你得追：
    # valid_comment_context → commentable_post → editable_post
    # → owned_post → valid_post → current_user → get_db
):
    ...
```

一般来说，2-3 层依赖是比较舒适的深度。超过这个深度，说明可能有些逻辑应该放进 service 而不是依赖项。

比较稳的做法是：

- 让依赖项负责"前置检查"（资源存在、权限校验）
- 让 service 负责"业务动作"（状态变更、业务规则）
- 不要把过多副作用塞进依赖项（发邮件、写日志等应该在 service 里）

很多 FastAPI 项目不是缺少依赖注入，而是只把它用在数据库 session 注入这种最浅的层面，却没有让接口命名、参数设计和依赖复用形成同一套体系。

## 落地清单

- 把"资源存在校验"从路由里抽出来。
- 把"权限校验"建立在"资源校验"之上。
- 统一资源路径参数命名。
- 路径尽量使用名词和标准 HTTP 方法。
- 控制依赖链深度，不要让调试路径过长。

---

下一篇：[让 FastAPI 接口更专业：文档契约、response_model 和模型组织](./05-docs-and-response-models.md)