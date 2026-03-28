---
allow_comments: true
author: tomyi
author_id: 019c21db-4c02-7836-9d5b-36c688f8b4e0
category: fastapi-engineering-practice
category_id: 019d359a-ec80-7ce3-a755-4a2b1ca21df4
date: '2026-03-28 18:00:39'
enable_jsx: false
is_featured: false
meta_description: 深度讲解在 FastAPI 中如何正确使用 Pydantic 模型，明确数据边界，分离请求、响应、配置、内部数据模型。
meta_keywords: FastAPI, Pydantic, 数据校验, 模型设计, 数据边界, 配置管理, BaseSettings
meta_title: Pydantic 数据边界：如何区分请求、响应、配置和内部模型 | FastAPI 最佳实践
slug: pydantic-models-and-settings-1p1ect
status: published
summary: Pydantic 最有价值的地方不是省几行校验代码，而是帮你把请求、响应、配置和内部数据边界理顺。
tags:
- FastAPI
- Pydantic
- Validation
title: 03-FastAPI 里的数据边界：Pydantic 模型和配置别再混着写
use_server_rendering: true
---

# FastAPI 里的数据边界：Pydantic 模型和配置别再混着写

## 先说结论

在 FastAPI 项目里，Pydantic 最重要的作用不是“自动校验”，而是帮你明确数据边界。

至少要把这几类东西分开：

- 请求模型
- 响应模型
- 内部领域对象
- 配置对象

很多项目后面变得难改，不是因为模型太多，而是因为同一个模型被拿去做了太多事。

## 先搞清楚 Pydantic 到底在帮你做什么

很多人把 Pydantic 理解成"字段校验工具"，但它实际做的事比这多得多：

**运行时类型校验**：Python 的 type hints 只是标注，运行时不会真的检查。Pydantic 会在数据进来的那一刻做真正的校验。

```python
from pydantic import BaseModel

class OrderItem(BaseModel):
    product_id: int
    quantity: int
    price: float

# type hints 不会拦住这个，但 Pydantic 会
OrderItem(product_id="abc", quantity=2, price=10.0)
# ValidationError: Input should be a valid integer
```

**自动类型转换**：合理的类型会被自动转换，而不是直接拒绝。

```python
# 字符串 "3" 会被自动转换成 int 3
item = OrderItem(product_id="123", quantity="3", price="10.5")
print(item.quantity)  # 3 (int, 不是 str)
```

**序列化控制**：你可以精确控制哪些字段对外暴露、怎么命名、怎么转换。

```python
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: int
    username: str
    email: str
    # password 不在这里 → 不会被序列化到响应里
```

这三件事加在一起，Pydantic 在 FastAPI 里的真正角色是：**请求进来时守门，响应出去时过滤，中间还帮你把类型理顺**。

理解了这一点，再来看为什么模型要拆分就会清楚很多。

## 一个常见问题：一个模型到处复用

很多项目一开始会写出这样的代码：

```python
class UserSchema(BaseModel):
    id: int | None = None
    username: str
    email: EmailStr
    password: str
    is_active: bool = True
    created_at: datetime | None = None
```

然后这个 `UserSchema` 被同时用于创建请求、更新请求、响应返回、甚至数据库存储。

这在早期确实省事，但很快就会出现问题：

- 创建时 `id` 和 `created_at` 不该由客户端传，但模型里有。
- 更新时大部分字段应该可选，但模型里 `username` 和 `email` 是必填。
- 响应时 `password` 不该暴露，但模型里有这个字段。
- 最终你会发现到处都是 `exclude`、`Optional` 和注释来"绕过"模型定义。

## 更稳的拆分方式

建议从最常见的三类模型开始：

```python
from pydantic import BaseModel, EmailStr, Field


# 创建请求：所有必填字段，包含 password，不包含 id
class UserCreate(BaseModel):
    username: str = Field(min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(min_length=8)


# 更新请求：所有字段可选，不包含 password 和 id
# 如果密码更新是独立流程，通常单独建 UserPasswordUpdate 之类的模型
class UserUpdate(BaseModel):
    username: str | None = Field(default=None, min_length=3, max_length=50)
    email: EmailStr | None = None


# 响应模型：包含 id，不包含 password
# 响应模型除了描述返回结构，也是在避免把内部字段意外暴露给客户端
class UserResponse(BaseModel):
    id: int
    username: str
    email: EmailStr
```

这看起来比“一个模型走天下”啰嗦一点，但它换来的是更清楚的接口契约。

## 多用内置验证，不要把简单规则都写成手工校验

对比一下两种写法就知道差别有多大：

```python
# 手工校验：散落在 service 层，容易遗漏
@router.post("/products")
async def create_product(data: dict):
    if not data.get("name") or len(data["name"]) > 100:
        raise HTTPException(400, "name is required and must be under 100 chars")
    if data.get("price") is not None and data["price"] < 0:
        raise HTTPException(400, "price must be non-negative")
    if data.get("category") not in ("electronics", "clothing", "food"):
        raise HTTPException(400, "invalid category")
    ...
```

```python
# Pydantic 内置校验：在模型层一次解决
from enum import Enum

class Category(str, Enum):
    electronics = "electronics"
    clothing = "clothing"
    food = "food"

class ProductCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    price: float = Field(ge=0)
    category: Category
    url: HttpUrl | None = None

@router.post("/products")
async def create_product(data: ProductCreate):
    ...  # 走到这里时数据一定是合法的
```

第二种写法的收益是：错误更早暴露，路由层更干净，而且 FastAPI 会自动返回标准的 422 响应，客户端能直接知道哪个字段不对、为什么不对。

## 可以有一个公共 BaseModel，但别过度包装

如果你想统一一些项目级行为，可以定义一个公共模型基类：

```python
from pydantic import BaseModel, ConfigDict


class AppBaseModel(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,       # 支持从 ORM 对象转换
        str_strip_whitespace=True,   # 自动去除字符串首尾空格
    )
```

然后所有业务模型继承它：

```python
class UserCreate(AppBaseModel):
    username: str = Field(min_length=3, max_length=50)
    email: EmailStr
    password: str = Field(min_length=8)
```

但这里不要走太远。一个好的公共 BaseModel 应该是薄的，只放确定全项目通用的配置。如果你发现自己在往里塞业务逻辑或者复杂的 validator，那就是过度包装。

## 配置也应该按边界拆，不要只有一个巨大的 Settings

另一个常见问题是把所有环境变量都堆到一个 `Settings` 类里。

短期看很集中，长期看会有这些问题：

- 文件会不断膨胀。
- 不同模块的配置耦合在一起。
- 很难看出哪些配置属于哪个领域。

更实用的做法是：

- 全局配置放在 `config.py`
- 模块特定配置放在对应模块目录里

比如：

```text
src/
├── auth/
│   └── config.py
├── database/
│   └── config.py
└── config.py
```

用代码来看这个分层：

```python
# src/config.py — 全局配置，所有模块都可能用到
from pydantic_settings import BaseSettings, SettingsConfigDict


class AppSettings(BaseSettings):
    env: str = "dev"
    debug: bool = False
    app_name: str = "my-app"

    model_config = SettingsConfigDict(env_prefix="APP_")
```

```python
# src/database/config.py — 只有数据库模块关心
from pydantic_settings import BaseSettings


class DatabaseSettings(BaseSettings):
    url: str = "postgresql+asyncpg://localhost/mydb"
    pool_size: int = 5
    echo: bool = False

    model_config = SettingsConfigDict(env_prefix="DB_")
```

```python
# src/auth/config.py — 只有认证模块核心
from pydantic_settings import BaseSettings


class AuthSettings(BaseSettings):
    secret_key: str
    token_expire_minutes: int = 30
    algorithm: str = "HS256"

    model_config = SettingsConfigDict(env_prefix="AUTH_")
```

在依赖注入中使用时，每个模块只拿自己需要的配置：

```python
from functools import lru_cache
from src.auth.config import AuthSettings


@lru_cache
def get_auth_settings() -> AuthSettings:
    return AuthSettings()


@router.post("/login")
async def login(settings: AuthSettings = Depends(get_auth_settings)):
    ...
```

这样每个模块的配置是独立的，改 `AUTH_SECRET_KEY` 不会影响数据库模块，改 `DB_POOL_SIZE` 也不会碰到认证逻辑。

## 但也别一上来就拆得过细

这里有个很重要的边界：

如果项目还小，配置项只有十几个，没必要为了“架构整洁”把它拆成很多文件。

配置拆分的前提是：

- 配置已经有明显的领域边界
- 模块负责人不同
- 或者配置确实已经长到难维护

否则就是过早抽象。

## 数据库模型和 API 模型不是同一件事

SQLAlchemy 的 ORM 模型和 Pydantic 的 API 模型在职责上完全不同：

- ORM 模型描述的是数据库结构：表、列、关系、约束。
- API 模型描述的是接口契约：客户端能传什么、能看到什么。

把两者混用，短期省了几行代码，长期会出现两类问题：

第一，暴露内部字段。ORM 对象通常包含 `hashed_password`、`internal_status`、关联外键等不该出现在 API 响应里的字段。用 ORM 对象直接返回，很容易意外泄露这些内容。

第二，两套变更周期互相干扰。数据库加一列，如果 API 模型和 ORM 模型是同一个，这列就自动出现在响应里。数据库字段的重命名也会直接破坏 API 契约。

正确的做法是让两者独立演化：

```python
# ORM 模型：描述数据库结构
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    username: Mapped[str]
    email: Mapped[str]
    hashed_password: Mapped[str]  # 不该出现在 API 响应里

# API 响应模型：只暴露客户端应该看到的字段
class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    username: str
    email: str
    # hashed_password 不在这里
```

`from_attributes=True` 让 Pydantic 能直接从 ORM 对象读取属性，不需要手动转换，两者仍然保持独立。

## 落地清单

- 至少拆分 `Create`、`Update`、`Response` 三类模型。
- 优先使用 Pydantic 内置字段约束。
- 公共 BaseModel 只保留少量稳定能力。
- 配置先按"全局 / 模块"两层拆分，不要一步到位拆太碎。
- ORM 模型和 API 模型独立定义，用 `from_attributes=True` 桥接，不要直接返回 ORM 对象。

---

下一篇：[依赖注入与 RESTful 路由设计](./04-dependencies-and-rest-routes.md)