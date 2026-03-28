---
title: 把错误处理做系统：分层、统一格式和全局收口
slug: error-handling-architecture
author: tomyi
status: published
is_featured: false
allow_comments: true
enable_jsx: false
summary: 随手 raise HTTPException 能跑，但扩大后很快失控。这篇讲如何通过错误分类、分层边界和全局 handler，让错误处理变成可维护的系统。
excerpt: 随手 raise HTTPException 能跑，但扩大后很快失控。这篇讲如何通过错误分类、分层边界和全局 handler，让错误处理变成可维护的系统。
meta_title: FastAPI 错误处理架构：分层、统一格式与全局收口 | 工程化最佳实践
meta_description: 从 HTTPException 的滥用到系统化的错误处理：如何通过错误分类、分层边界设计和全局 exception handler，让错误处理变成可维护的工程化系统。
meta_keywords: FastAPI, 错误处理, exception handler, HTTPException, 分层架构, 错误分类, 统一响应格式
tags:
  - FastAPI
  - Error Handling
  - Architecture
---

# 把错误处理做系统：分层、统一格式和全局收口

## 先说结论

很多 FastAPI 项目的错误处理都是这样长大的：

- 早期随手 `raise HTTPException`，能跑就行
- 接口变多之后，错误格式开始各说各话
- 业务层开始混入 HTTP 状态码，service 越来越难复用
- 某个异常被吞掉，排查时找不到任何日志

这不是因为某一行代码写错了，而是因为没有把错误处理当成一个需要设计的系统。

这篇要讲的就是：怎么从"到处 raise"变成"统一收口"——错误分类、分层边界、响应格式设计、全局 handler。

## 先把错误分清楚

错误处理写不清楚，一个常见原因是没有先做分类。不同类型的错误，应该在不同的层处理，对外暴露的方式也不同。

至少可以先分成这几类：

- **请求错误**：字段缺失、格式不对、参数校验失败
- **认证错误**：未登录、token 无效、token 过期
- **权限错误**：登录了，但没有权限操作资源
- **资源错误**：资源不存在
- **业务错误**：状态冲突、规则不满足、重复操作
- **基础设施错误**：数据库异常、第三方服务失败、网络超时
- **未知错误**：没有预料到的异常

粗略映射成状态码，大概会是这样：

| 错误类型 | 常见状态码 | 例子 |
| --- | --- | --- |
| 请求错误 | `422` | 邮箱格式错误、字段缺失 |
| 认证错误 | `401` | token 无效 |
| 权限错误 | `403` | 不能修改别人的文章 |
| 资源错误 | `404` | 文章不存在 |
| 业务错误 | `400` / `409` | 文章已归档、邮箱已存在 |
| 基础设施错误 | `503` / `500` | 数据库不可用、第三方超时 |
| 未知错误 | `500` | 未预期异常 |

这个分类的意义不是为了学术完整，而是为了后面更容易决定：哪些错误该在哪一层处理，哪些该暴露给客户端，哪些只该进入日志和监控。

换句话说，分类解决的是“这是什么错”，而分层解决的是“这类错应该在哪一层被处理和转换”。

## 先分清楚：什么是 HTTP 异常，什么是业务异常

在讨论分层之前，最好先把这两个概念拆开，因为后面很多决策都建立在这个区分之上。

### HTTP 异常

HTTP 异常表达的是：
**这个请求最后应该返回什么 HTTP 状态码和什么错误响应。**

它更关心的是接口边界和协议语义，比如：

- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict`

在 FastAPI 里最典型的就是：

```python
from fastapi import HTTPException


async def valid_post(post_id: int) -> Post:
    post = await posts_repo.get(post_id)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    return post
```

这里表达的重点是：

- 这是一个 HTTP 请求边界上的错误
- 对客户端来说，应该返回 `404`
- 错误信息是 `"Post not found"`

所以 HTTP 异常回答的是：**这个请求应该怎么返回。**

### 业务异常

业务异常表达的是：
**从业务规则角度看，到底发生了什么问题。**

比如：

- 邮箱已经被注册
- 文章已经归档，不能再修改
- 当前订单状态不允许取消
- 用户余额不足

这类错误的重点不在于"应该返回 400 还是 409"，而在于它们在业务上代表什么问题。

例如：

```python
class EmailAlreadyExistsError(Exception):
    pass


async def register_user(payload: UserCreate) -> User:
    exists = await users_repo.exists_by_email(payload.email)
    if exists:
        raise EmailAlreadyExistsError()

    return await users_repo.create(payload)
```

这里表达的重点不是 HTTP 协议，而是：

- 发生了一个"邮箱已存在"的业务冲突

所以业务异常回答的是：**为什么失败。**

### 两者最核心的区别

可以把它们记成一句话：

- 业务异常解释"为什么失败"
- HTTP 异常决定"怎么返回"

再看一个对比例子会更直观。

业务层这样写：

```python
class PostArchivedError(Exception):
    pass


async def update_post(post: Post, payload: PostUpdate):
    if post.status == "archived":
        raise PostArchivedError()

    ...
```

这里 service 只表达：

- 文章已归档
- 所以不能更新

它没有关心 HTTP。

如果改成这样：

```python
async def update_post(post: Post, payload: PostUpdate):
    if post.status == "archived":
        raise HTTPException(status_code=400, detail="Archived post cannot be updated")

    ...
```

那这段业务逻辑就已经开始同时关心：

- 业务规则本身
- HTTP 状态码
- 对外错误文案

这就是为什么这两类异常最好分开理解。
一个更贴近领域语义，一个更贴近接口边界。

把这个概念分清楚以后，后面“依赖层该抛什么、service 层该抛什么、handler 该接什么”这些问题就会顺很多。

## 错误应该在哪一层处理

很多团队的错误处理之所以越来越乱，不是因为异常类不够多，而是因为没有分层边界。

比较稳的分工通常是：

- **路由层**：负责 HTTP 输入输出，不承载大段业务异常逻辑
- **依赖层**：负责资源存在、权限和前置条件校验
- **service 层**：负责抛出业务异常，不直接关心 HTTP 语义
- **基础设施层**：抛出底层异常，或转换成更接近业务语义的异常
- **全局异常处理器**：把各层的异常统一翻译成稳定的外部响应

资源存在性和权限校验放在依赖层很自然：

```python
async def valid_post(post_id: int) -> Post:
    post = await posts_repo.get(post_id)
    if not post:
        raise HTTPException(status_code=404, detail="Post not found")
    return post


async def owned_post(
    post: Post = Depends(valid_post),
    user: User = Depends(current_user),
) -> Post:
    if post.author_id != user.id:
        raise HTTPException(status_code=403, detail="Forbidden")
    return post
```

偏业务语义的错误，更适合在 service 层表达：

```python
class PostArchivedError(Exception):
    pass


async def publish_post(post: Post) -> Post:
    if post.status == "archived":
        raise PostArchivedError()

    post.status = "published"
    return await posts_repo.save(post)
```

业务层更适合表达"发生了什么业务错误"，而不是表达"应该返回哪个 HTTP 状态码"。

## `HTTPException` 和业务异常怎么分工

依赖层直接抛 `HTTPException`，service 层抛业务异常——这套分工的好处很直接：

- 业务层不会绑死在 HTTP 语义上
- 这段逻辑如果要被 CLI、任务队列或消息消费者复用，也不需要携带 `HTTPException`
- 全局异常处理器统一决定它最后怎样对外暴露

### 如果业务层到处直接抛 `HTTPException`，会发生什么

最直接的问题不是"前端能不能处理"，因为 Web、小程序、App 最终都能消费 HTTP 响应。
真正的问题在于：业务逻辑会被 HTTP 协议语义绑死，导致复用范围越来越窄。

例如，下面这段逻辑放在 service 里就很别扭：

```python
async def register_user(payload: UserCreate) -> User:
    exists = await users_repo.exists_by_email(payload.email)
    if exists:
        raise HTTPException(status_code=409, detail="Email already exists")

    return await users_repo.create(payload)
```

短期只服务于一个 HTTP API 当然能跑。但一旦后面有消息消费者在收到事件后自动创建用户，或者后台任务批量导入用户，这段逻辑就开始别扭——这些场景并不想处理一个 HTTP 异常对象，它们真正关心的是失败原因是不是"邮箱已存在"、这个错误应不应该重试、是跳过当前数据还是终止整个任务。

所以这里最核心的判断标准不是"客户端是不是只有 Web 前端"，而是：

- 你是在表达业务问题
- 还是在表达 HTTP 返回方式

如果把这两者混在一起，后面常见的后果就是：

- service 层越来越依赖 FastAPI
- 非 HTTP 场景复用开始变差
- 错误处理越来越分散
- 多端虽然都能拿到 HTTP 状态码，但缺少更稳定的业务错误码和领域语义

这也是为什么更推荐让业务层抛业务异常，再由 API 层或全局异常处理器把它映射成 HTTP 响应。
这样既保留了业务逻辑的可复用性，也保留了对外接口的一致性。

## 统一错误响应格式

在谈怎么处理之前，先确定格式：统一后的错误 JSON 应该长什么样。

一个比较实用的格式通常至少包含这些字段：

```json
{
  "code": "post_archived",
  "message": "Archived post cannot be updated",
  "details": null,
  "request_id": "req_123456"
}
```

它们分别解决不同问题：

- `code`：给前端和调用方做稳定的程序判断，不依赖 `message` 文字
- `message`：给人看的错误说明，允许变化
- `details`：放字段级错误或补充上下文，比如校验失败时的具体字段信息
- `request_id`：串日志、监控和问题排查，从请求头透传

用 Pydantic 定义这套结构：

```python
class ErrorResponse(BaseModel):
    code: str
    message: str
    details: dict | list | None = None
    request_id: str | None = None
```

一旦有了统一格式，就可以在文档里也显式声明，让接口契约和实际行为保持一致：

```python
@router.post(
    "/posts/{post_id}/publish",
    responses={
        400: {"description": "文章状态不允许发布", "model": ErrorResponse},
        404: {"description": "文章不存在", "model": ErrorResponse},
    },
)
async def publish_post(post_id: int):
    ...
```

到这里为止，我们已经回答了两件事：

- 错误应该怎么分类
- 不同层应该抛什么类型的异常

下一步才是把这些异常统一收口成一种稳定的外部响应格式。

## 用全局 handler 统一收口

有了格式，下一步是让所有异常都经过同一个响应转换层，而不是到处手动转换。

前面的示例用的是普通 `Exception` 子类来表达业务语义，但到了全局 handler 这一层会遇到一个问题：handler 怎么知道该返回什么状态码和错误码？解决方案是设计一个携带这些属性的基类 `AppError`，用类变量声明每种异常的固定属性：

```python
class AppError(Exception):
    status_code: int = 500
    code: str = "internal_error"
    message: str = "Internal server error"
    details: dict | list | None = None


class EmailAlreadyExistsError(AppError):
    status_code = 409
    code = "email_exists"
    message = "Email already exists"


class PostArchivedError(AppError):
    status_code = 400
    code = "post_archived"
    message = "Archived post cannot be updated"
```

然后注册全局 handler，分别处理业务异常和 `HTTPException`，但最终都转换成同一个 `ErrorResponse`：

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse

app = FastAPI()


@app.exception_handler(AppError)
async def handle_app_error(request: Request, exc: AppError):
    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            code=exc.code,
            message=exc.message,
            details=exc.details,
            request_id=request.headers.get("X-Request-ID"),
        ).model_dump(),
    )


@app.exception_handler(HTTPException)
async def handle_http_exception(request: Request, exc: HTTPException):
    detail = exc.detail
    if isinstance(detail, dict):
        code = detail.get("code", "http_error")
        message = detail.get("message", "HTTP error")
        details = detail.get("details")
    else:
        code = "http_error"
        message = str(detail)
        details = None

    return JSONResponse(
        status_code=exc.status_code,
        content=ErrorResponse(
            code=code,
            message=message,
            details=details,
            request_id=request.headers.get("X-Request-ID"),
        ).model_dump(),
    )
```

这样无论业务代码里抛的是 `raise EmailAlreadyExistsError()` 还是 `raise HTTPException(...)`，前端拿到的都是同一种结构。

统一格式的关键不是"所有异常都继承同一个类"，而是"所有异常最终都经过同一个响应转换层"。

最后，给未知异常留一个兜底处理器：

```python
@app.exception_handler(Exception)
async def handle_unexpected_error(request: Request, exc: Exception):
    logger.exception(
        "Unexpected error",
        extra={"request_id": request.headers.get("X-Request-ID")},
    )
    return JSONResponse(
        status_code=500,
        content=ErrorResponse(
            code="internal_error",
            message="Internal server error",
            request_id=request.headers.get("X-Request-ID"),
        ).model_dump(),
    )
```

兜底的意义不是"永远捕获所有异常"，而是：客户端拿到统一格式，服务端保留完整日志，未知错误不会静悄悄消失。

这套机制对前端和监控都有直接收益：前端可以稳定地按 `code` 做分支处理，而不是到处猜 `detail` 的格式；监控平台可以按错误类型分组告警，`request_id` 也能把日志和 Sentry/OTEL 的上报串起来。

## 落地清单

- 先做错误分类，确定哪些类型对应哪些状态码。
- 依赖层处理资源存在和权限校验，直接抛 `HTTPException`。
- service 层抛业务异常，不直接关心 HTTP 语义。
- 定义统一的 `ErrorResponse` 格式，包含 `code`、`message`、`details`、`request_id`。
- 用 `AppError` 基类管理业务异常，用类变量声明固定属性。
- 注册全局 handler，把 `AppError` 和 `HTTPException` 都转换成 `ErrorResponse`。
- 给未知异常加兜底 handler，保证日志完整、格式统一。

---

下一篇：[数据库别拖后腿：迁移命名、约束约定和 SQL 优先](./07-migrations-naming-and-sql-first.md)
