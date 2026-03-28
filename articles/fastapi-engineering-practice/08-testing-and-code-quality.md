---
title: 测试不是负担：FastAPI 项目的测试分层与代码质量
slug: testing-and-code-quality
author: tomyi
status: published
is_featured: false
allow_comments: true
enable_jsx: false
summary: 测试写不好，不是因为不够勤快，而是因为没有策略。这篇讲 FastAPI 项目里如何分层测试、搭建可靠的异步测试基础设施，以及 Ruff 怎么帮你在写代码时就发现问题。
excerpt: 测试写不好，不是因为不够勤快，而是因为没有策略。这篇讲 FastAPI 项目里如何分层测试、搭建可靠的异步测试基础设施，以及 Ruff 怎么帮你在写代码时就发现问题。
meta_title: FastAPI 测试策略：分层测试、异步测试与 Ruff 代码质量 | 工程化必读
meta_description: 从单元测试、集成测试到端到端测试的分层策略，深入异步测试的陷阱与最佳实践，以及如何用 Ruff 在编码阶段就发现问题，提升代码质量。
meta_keywords: FastAPI, 测试, pytest, 单元测试, 集成测试, 异步测试, Ruff, 代码质量, 测试分层
tags:
  - FastAPI
  - Testing
  - Ruff
  - DX
---

# 测试不是负担：FastAPI 项目的测试分层与代码质量

## 先说结论

很多 FastAPI 项目的测试写着写着就变成了负担——要么覆盖率低、要么脆，要么跑得慢。这通常不是因为不够努力，而是因为缺少策略：

- 没有想清楚哪层该测什么
- 所有测试都打到数据库，跑得慢还互相干扰
- 依赖没有隔离，fixture 一复杂就乱

这篇要讲的是：如何给 FastAPI 项目建立一套能长期维护的测试体系，包括测试分层、异步测试基础设施、依赖覆盖，以及用 Ruff 把静态问题消灭在写代码时。

## 先想清楚：测试分几层，各层测什么

测试写不好，最根本的原因往往是没有想清楚分层。

一个比较实用的分法是三层：

| 层级 | 对应 FastAPI 的哪层 | 测什么 | 依赖真实外部系统？ |
| --- | --- | --- | --- |
| 单元测试 | service 层、工具函数 | 业务规则、纯逻辑 | 否，mock 或不需要 |
| 集成测试 | 路由层、依赖层 | 接口行为、请求/响应契约 | 是，真实数据库 |
| E2E 测试 | 完整系统 | 关键业务流程 | 是，完整环境 |

这三层的核心分工：

- **单元测试**负责验证"业务规则本身是否正确"，跑得极快，不依赖任何外部系统
- **集成测试**负责验证"接口行为是否符合预期"，需要真实数据库，但不需要完整部署环境
- **E2E 测试**验证完整流程，成本最高，数量应该最少

## 集成测试还是单元测试：FastAPI 项目里的真实取舍

这是测试策略里最容易引发争论的问题，也是最值得认真想清楚的一个。

### 以 CRUD 为主的业务：集成测试就够了

很多 FastAPI 项目的实际工作量大部分是 CRUD——创建、读取、更新、删除，加上一些权限校验。对于这类业务，集成测试通常已经完全足够。

这时候如果强行写 mock 单元测试，测的是什么？

```python
# mock 掉数据库之后，这个测试验证的只是：mock 返回了什么
async def test_get_user_mock():
    mock_repo = MagicMock()
    mock_repo.get.return_value = User(id=1, email="test@example.com")

    service = UserService(repo=mock_repo)
    user = await service.get_user(1)

    assert user.email == "test@example.com"
    mock_repo.get.assert_called_once_with(1)
```

这个测试的实际价值几乎为零：它测的是 mock 的行为，不是真实的数据读写。你换一个 ORM、改一个查询方式，这个测试照样绿，但系统可能已经坏了。

纯 CRUD 业务用集成测试反而更实在——真实数据库、真实 SQL、真实结果：

```python
# 集成测试：验证真实的数据库读写行为
@pytest.mark.asyncio
async def test_create_and_get_user(client: AsyncClient):
    create_resp = await client.post("/users", json={
        "email": "test@example.com",
        "username": "testuser",
    })
    assert create_resp.status_code == 201
    user_id = create_resp.json()["id"]

    get_resp = await client.get(f"/users/{user_id}")
    assert get_resp.status_code == 200
    assert get_resp.json()["email"] == "test@example.com"
```

这个测试验证的是完整链路：路由、依赖、service、数据库写入、数据库读取。任何一环出问题都会被发现。

### 单元测试真正有价值的地方

单元测试的价值区域很明确：**纯业务逻辑、纯计算——不依赖数据库、不依赖外部服务的那部分。**

一个订单折扣计算、一个状态机流转、一个数据转换函数——这类逻辑输入输出确定，不需要任何外部系统，测起来快且精准：

```python
def calculate_discount(price: float, user_tier: str) -> float:
    if user_tier == "premium":
        return price * 0.8
    if user_tier == "member":
        return price * 0.9
    return price


def test_premium_discount():
    assert calculate_discount(100.0, "premium") == 80.0

def test_member_discount():
    assert calculate_discount(100.0, "member") == 90.0

def test_no_discount():
    assert calculate_discount(100.0, "guest") == 100.0
```

这类测试没有任何理由不写——零外部依赖、跑得极快、失败时立刻定位到具体逻辑。

### 业务逻辑和数据流耦合时怎么办

很多项目的 service 层长这样——业务判断和数据库操作混在一起：

```python
async def publish_post(post_id: int, user_id: int, db: AsyncSession) -> Post:
    post = await db.get(Post, post_id)
    if not post:
        raise HTTPException(status_code=404)
    if post.author_id != user_id:
        raise HTTPException(status_code=403)
    if post.status == "archived":
        raise PostArchivedError()
    if post.word_count < 100:
        raise PostTooShortError()

    post.status = "published"
    post.published_at = datetime.now()
    await db.commit()
    return post
```

这个函数里有三类东西混在一起：数据库读写、权限校验、业务规则（归档判断、字数判断）。

如果你想对"归档文章不能发布"和"字数不足不能发布"这两条业务规则写单元测试，只能 mock 掉数据库——这就变成了成本高但价值低的 mock 测试。

**更合理的做法是把纯业务规则提取出来：**

```python
def validate_post_publishable(post: Post) -> None:
    """纯函数：只做业务规则校验，不接触数据库"""
    if post.status == "archived":
        raise PostArchivedError()
    if post.word_count < 100:
        raise PostTooShortError()


async def publish_post(post_id: int, user_id: int, db: AsyncSession) -> Post:
    post = await db.get(Post, post_id)
    if not post:
        raise HTTPException(status_code=404)
    if post.author_id != user_id:
        raise HTTPException(status_code=403)

    validate_post_publishable(post)  # 纯业务规则独立调用

    post.status = "published"
    post.published_at = datetime.now()
    await db.commit()
    return post
```

现在 `validate_post_publishable` 是一个纯函数，只需要一个 `Post` 对象，不依赖任何外部系统：

```python
def test_archived_post_cannot_be_published():
    post = Post(status="archived", word_count=500)
    with pytest.raises(PostArchivedError):
        validate_post_publishable(post)


def test_short_post_cannot_be_published():
    post = Post(status="draft", word_count=50)
    with pytest.raises(PostTooShortError):
        validate_post_publishable(post)


def test_valid_post_can_be_published():
    post = Post(status="draft", word_count=200)
    validate_post_publishable(post)  # 不抛异常即通过
```

而 `publish_post` 整体的流程（数据库读写、权限校验、发布结果）继续用集成测试覆盖。

这才是单元测试真正有意义的位置：**把纯计算的业务逻辑从数据流中提取出来，单独封装，单独测试。** 不是为了凑覆盖率而 mock 一切，而是让有价值的逻辑自然变得可测。

### mock 什么，不 mock 什么

这是测试里最容易被忽略、也最容易走偏的一个判断。很多团队要么 mock 过度（什么都 mock，测试全绿但系统出问题），要么完全不 mock（测试依赖真实外部服务，脆且慢）。

判断 mock 是否值得，有一个核心标准：

**mock 的对象应该是你系统边界之外的东西，不是边界之内的东西。**

边界之内（数据库、service 层、业务逻辑）用真实执行，边界之外（第三方服务、外部 API、不可控依赖）用 mock 替代。

#### 真正值得 mock 的三种情况

**情况一：有真实副作用、不能在测试里真正触发**

发送邮件、发短信、调用支付接口、推送通知——这些在测试里触发了就是真的触发了。mock 是唯一合理的选择，测试要验证的是"发送行为有没有被正确调用"，而不是"邮件真的发出去没有"：

```python
async def test_register_sends_welcome_email(client: AsyncClient):
    sent_emails = []

    async def mock_send_email(to: str, subject: str, body: str):
        sent_emails.append({"to": to, "subject": subject})

    with patch("app.services.email.send", side_effect=mock_send_email):
        await client.post("/users/register", json={
            "email": "new@example.com",
            "password": "pass123",
        })

    assert len(sent_emails) == 1
    assert sent_emails[0]["to"] == "new@example.com"
```

**情况二：非确定性的值，测试里需要被控制**

`datetime.now()`、`uuid.uuid4()`、随机数——这类依赖每次调用结果不同，测试无法写出确定的断言。mock 是为了让测试结果可预测：

```python
async def test_post_published_at_is_set_correctly():
    fixed_time = datetime(2026, 1, 1, 12, 0, 0)

    with patch("app.services.posts.datetime") as mock_dt:
        mock_dt.now.return_value = fixed_time
        post = await posts_service.publish(post_id=1)

    assert post.published_at == fixed_time
```

**情况三：测试错误处理逻辑，但真实依赖很难主动制造错误**

你想测试"第三方支付 API 超时时，系统应该返回 503 并记录日志"——真实 API 不会配合你超时，只能 mock 制造这个场景：

```python
async def test_payment_timeout_returns_503(client: AsyncClient):
    with patch("app.services.payment_client.charge") as mock_charge:
        mock_charge.side_effect = TimeoutError()

        response = await client.post("/orders/pay", json={"order_id": 1})

    assert response.status_code == 503
    assert response.json()["code"] == "payment_timeout"
```

注意：这里 mock 的目的是触发一个真实环境里极难复现的场景，测的是你自己的错误处理逻辑，而不是支付服务本身。

#### 不值得 mock 的情况

**数据库**：数据库是你自己的，schema 是你定义的。mock 数据库容易掩盖真实问题——SQL 写错了、约束加错了、迁移有问题，mock 测试照样绿。上一篇提到的"mock 测试通过但真实迁移失败"就是这个陷阱的典型案例。

**service 层**（在路由集成测试里）：mock 掉 service 之后，路由测试验证的只是 mock 的返回值，不是真实业务行为。路由层的集成测试应该让整条链路真实执行。

**你自己写的业务逻辑**：如果某段逻辑需要大量 mock 才能测，通常说明的不是"应该 mock"，而是这段逻辑和外部依赖耦合过深，需要先把纯业务逻辑提取出来再测，而不是用 mock 绕过耦合。

#### 一个容易混淆的场景

有时候你会看到这样的测试：

```python
# mock 掉 repo，测试 service 层逻辑
async def test_create_user_checks_email_duplicate():
    mock_repo = AsyncMock()
    mock_repo.exists_by_email.return_value = True

    service = UserService(repo=mock_repo)

    with pytest.raises(EmailAlreadyExistsError):
        await service.register(UserCreate(email="exists@example.com", password="pass"))
```

这个 mock 有没有价值？取决于 `register` 里有没有真正值得测的业务逻辑。如果 `register` 只是查一下有没有、没有就存、有就抛异常——这是纯 CRUD，集成测试更合适。

但如果 `register` 里有更复杂的逻辑——邮箱域名白名单校验、注册频率限制、邀请码验证——那这些规则值得被单独提取成纯函数再测，而不是通过 mock repo 来间接覆盖。

**结论是一样的：不是"能不能 mock"，而是"这段逻辑本身值不值得单独测"。**

## 搭好异步测试基础设施

策略想清楚之后，下一步是把基础设施搭稳，让后续每个测试都能轻松写。

### 为什么要用异步测试

FastAPI 是异步框架。如果测试层一直用同步 `TestClient`，后面会碰到：

- 不能自然测试异步依赖
- 测试写法和生产执行模型不一致
- 依赖覆盖不够灵活

更建议尽早用 `pytest` + `pytest-asyncio` + `httpx.AsyncClient` 这套组合。

### 完整的 conftest.py

这是最值得把它做对的一个文件。一个稳定的 `conftest.py` 应该解决三件事：测试数据库连接、每个测试的数据隔离、可复用的 HTTP 客户端。

#### 隔离方案的选择：为什么需要嵌套事务

最直觉的隔离方式是"测试结束后回滚"：

```python
# 看起来合理，但有隐患
@pytest_asyncio.fixture
async def db_session():
    async with engine.begin() as conn:
        async with AsyncSession(conn) as session:
            yield session
            await conn.rollback()  # ← 问题在这里
```

这个写法有一个隐患：FastAPI 的 service 层通常会主动调用 `await session.commit()`，而一旦真正执行了 `COMMIT`，后面的 `conn.rollback()` 就回滚不了这些已提交的数据。测试之间的隔离就此失效——前一个测试写入的数据会污染后一个测试。

解决方案是使用嵌套事务（Savepoint）。SQLAlchemy 2.0 通过 `join_transaction_mode="create_savepoint"` 支持这个模式：

```
外层 BEGIN
  ↓
SAVEPOINT sp1                      ← db_session fixture 开始
  ↓
业务代码 session.commit()           → 实际执行 RELEASE SAVEPOINT，不是真正 COMMIT
  ↓
测试结束 session.rollback()         → ROLLBACK TO SAVEPOINT，所有改动撤销
  ↓
conn.rollback()                    → 回滚外层事务，数据库干净如初
```

业务代码以为自己 commit 了，但实际上只是释放了一个 savepoint，外层事务从未真正提交。

#### 使用嵌套事务的完整 conftest.py

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from httpx import ASGITransport, AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

from app.database import Base, get_db
from app.main import app

TEST_DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/test_db"

engine = create_async_engine(TEST_DATABASE_URL)


@pytest_asyncio.fixture(scope="session", autouse=True)
async def create_tables():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)


@pytest_asyncio.fixture
async def db_session():
    async with engine.connect() as conn:
        await conn.begin()  # 外层事务
        async with AsyncSession(
            conn,
            join_transaction_mode="create_savepoint",  # 关键配置
        ) as session:
            yield session
            await session.rollback()  # 回滚到 savepoint
        await conn.rollback()  # 回滚外层事务


@pytest_asyncio.fixture
async def client(db_session: AsyncSession):
    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac

    app.dependency_overrides.clear()
```

`join_transaction_mode="create_savepoint"` 是核心：它告诉 SQLAlchemy，当这个 session 调用 `commit()` 时，不要真正提交，而是创建并释放一个 savepoint。外层事务始终保持未提交状态，测试结束时可以干净回滚。

这个方案只在 PostgreSQL 上可靠，SQLite 的 savepoint 支持有限制。如果你用 SQLite 跑测试，改用每次截断表更稳：

```python
# SQLite 或不想用嵌套事务时的替代方案
@pytest_asyncio.fixture(autouse=True)
async def clean_tables():
    yield
    async with engine.begin() as conn:
        for table in reversed(Base.metadata.sorted_tables):
            await conn.execute(table.delete())
```

截断表的逻辑更直白，但每个测试多一次 DDL 操作，测试多了会慢一些。

### 使用 fixture 写测试

有了上面的 fixture，测试本身就变得很干净：

```python
# tests/test_users.py
import pytest
from httpx import AsyncClient


@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    response = await client.post("/users", json={
        "email": "test@example.com",
        "username": "testuser",
        "password": "secret123",
    })

    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
    assert "password" not in data


@pytest.mark.asyncio
async def test_create_user_duplicate_email(client: AsyncClient):
    payload = {"email": "dup@example.com", "username": "user1", "password": "pass"}

    await client.post("/users", json=payload)
    response = await client.post("/users", json=payload)

    assert response.status_code == 409
    assert response.json()["code"] == "email_exists"
```

### pytest 配置

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

`asyncio_mode = "auto"` 之后，`async def test_*` 不再需要每次加 `@pytest.mark.asyncio`。

## 依赖覆盖：在测试里替换外部依赖

FastAPI 的依赖注入系统天然支持测试时替换依赖，这是它的一个显著优势。

对于需要 mock 的外部依赖（比如发送邮件、调用第三方 API），用 `dependency_overrides` 替换：

```python
@pytest_asyncio.fixture
async def client_with_mock_email(db_session: AsyncSession):
    sent_emails = []

    async def mock_email_service():
        class MockEmailService:
            async def send(self, to: str, subject: str, body: str):
                sent_emails.append({"to": to, "subject": subject})
        return MockEmailService()

    async def override_get_db():
        yield db_session

    app.dependency_overrides[get_db] = override_get_db
    app.dependency_overrides[get_email_service] = mock_email_service

    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as ac:
        yield ac, sent_emails

    app.dependency_overrides.clear()


@pytest.mark.asyncio
async def test_register_sends_welcome_email(client_with_mock_email):
    client, sent_emails = client_with_mock_email

    await client.post("/users/register", json={
        "email": "new@example.com",
        "password": "pass123",
    })

    assert len(sent_emails) == 1
    assert sent_emails[0]["to"] == "new@example.com"
```

依赖覆盖的好处是：你替换的只是特定的依赖，其余的依赖链（数据库、认证、业务逻辑）仍然走真实路径。

## Ruff：把静态问题消灭在写代码时

测试保证行为正确，Ruff 保证静态质量。两者解决的是不同维度的问题。

Ruff 的核心优势是：

- 快（比 flake8 + isort + pylint 组合快几十倍）
- 一站式：lint、格式化、导入排序、常见 bug 检查都覆盖
- 足够快，开发者会愿意频繁运行它

### 一套适合 FastAPI 项目的配置

```toml
# pyproject.toml
[tool.ruff]
target-version = "py311"
line-length = 88

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes（未使用导入、未定义变量）
    "I",    # isort（导入排序）
    "B",    # flake8-bugbear（潜在 bug）
    "UP",   # pyupgrade（自动升级旧写法）
    "SIM",  # flake8-simplify（可简化的写法）
    "RUF",  # Ruff 自有规则
]
ignore = [
    "B008",  # 允许在函数参数默认值里调用函数（FastAPI 的 Depends() 需要这个）
]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]  # 测试文件里允许 assert
```

这套配置能捕捉的常见问题：

```python
# F401：未使用的导入 → 自动删除
import os  # 没有用到

# B006：可变默认参数 → 提醒修复
def get_items(ids: list = []):  # 危险

# UP006：旧类型注解写法 → 自动升级
from typing import List, Dict
def func(items: List[Dict]):  # 应该用 list[dict]

# SIM108：可以简化的三元表达式
x = True if condition else False  # 可以简化成 x = condition
```

### 接到本地和 CI

本地保存时自动修复：

```json
// .vscode/settings.json
{
    "[python]": {
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "charliermarsh.ruff"
    }
}
```

pre-commit 配置：

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.4.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

CI 里只检查不修复：

```yaml
# .github/workflows/ci.yml
- name: Lint
  run: ruff check . && ruff format --check .
```

## 落地清单

- 确定测试分层策略：纯函数业务逻辑写单元测试，路由层写集成测试，CRUD-heavy 的 service 层不需要强行 mock。
- 在 `conftest.py` 里用事务回滚实现测试隔离，不依赖清空数据库。
- 用 `dependency_overrides` 替换外部依赖（邮件、第三方 API），数据库保持真实。
- 给项目加上 Ruff 配置，接到本地保存和 CI。
- `asyncio_mode = "auto"` 省去每个测试手动加 `@pytest.mark.asyncio`。


