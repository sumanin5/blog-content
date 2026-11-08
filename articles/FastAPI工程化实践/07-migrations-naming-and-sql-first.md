---
allow_comments: true
author: tomyi
author_id: 019c21db-4c02-7836-9d5b-36c688f8b4e0
category: fastapi-engineering-practice
category_id: 019d359a-ec80-7ce3-a755-4a2b1ca21df4
date: '2026-03-28 18:00:39'
enable_jsx: false
is_featured: false
meta_description: 讲解如何通过规范的迁移命名、数据库约束约定和 SQL 优先的思路，让 FastAPI 项目的数据库管理变得可维护且高效。深入 Alembic、SQLAlchemy
  和 PostgreSQL。
meta_keywords: FastAPI, 数据库迁移, Alembic, SQLAlchemy, PostgreSQL, 数据库设计, SQL约束, 命名规范
meta_title: FastAPI 数据库规范：迁移命名、SQL 约束与 SQL 优先策略 | 工程化指南
slug: migrations-naming-and-sql-first-cu50vq
status: published
summary: FastAPI 项目的数据库体验，很大程度上取决于迁移是否可读、命名是否一致，以及你有没有把该放进 SQL 的工作继续留在 Python 里做。
tags:
- FastAPI
- SQLAlchemy
- PostgreSQL
- Alembic
title: 07-数据库别拖后腿：迁移命名、约束约定和 SQL 优先
use_server_rendering: true
---

# 数据库别拖后腿：迁移命名、约束约定和 SQL 优先

## 先说结论

如果你想让 FastAPI 项目在半年后仍然可维护，数据库层至少要把三件事做对：

- 迁移文件可读、可回滚
- 键、索引、约束命名一致
- 能在 SQL 层完成的数据处理，不要无意义搬回 Python

很多项目的性能和维护问题，最后都不是出在 FastAPI，而是出在数据库边界混乱。

## 迁移文件别只是"能跑"

数据库层最容易积累一种很隐蔽的技术债：
代码写乱了，重构还有机会慢慢补；迁移命名一乱、约束命名一乱、数据库对象风格一乱，后面排查和协作成本会持续上升，而且越晚越难补。

Alembic 最常见的误区有三个：

- 迁移名过于随意，比如 `fix`、`update`
- 迁移逻辑依赖运行时环境，导致不同环境结果不同
- 只写 `upgrade()`，不认真写 `downgrade()`

更稳的做法很简单：

- 文件名可读
- 结构变更静态可预测
- 回滚路径明确

### 配置可读的迁移文件名

Alembic 默认生成的文件名是随机 hash，比如 `3f1a2b_fix.py`。只需要在 `alembic.ini` 里加一行配置，就能改成日期加描述的格式：

```ini
# alembic.ini
file_template = %%(year)d-%%(month).2d-%%(day).2d_%%(slug)s
```

之后生成出来的文件名就是：

```text
2026-03-25_create_user_table.py
2026-03-26_add_email_index_to_user.py
2026-03-27_create_post_table.py
```

这种命名对排查问题和代码审查都友好得多。

一眼看过去，你至少能马上知道三件事：

- 这条迁移大概是什么时候引入的
- 它改的是哪一类数据库对象
- 如果线上出问题，你应该优先怀疑哪次结构变更

而如果迁移文件名全是：

```text
3f1a2b_fix.py
7cd991_update.py
af20e1_changes.py
```

那后面做这些工作都会更痛苦：

- 排查线上 schema 变化
- 代码审查迁移文件
- 回滚某次数据库变更
- 新人理解数据库演化过程

迁移文件不是"生成完就不看"的垃圾桶，它本质上是数据库结构的历史记录。
既然是历史记录，就应该可读。

## 迁移为什么必须静态、可预测、可回滚

这三件事看起来很基础，但恰恰是很多数据库问题的根源。

### 1. 静态

迁移描述的应该是确定的结构变化，而不是依赖运行时环境或当前数据状态的动态逻辑。

"静态"说的不是"不能在迁移里用变量"，而是：**迁移执行后产生的 schema 结果，不应该因为运行时的外部状态不同而不同。** 任何人在任何环境跑同一条迁移，得到的数据库结构都应该完全一致。

真正违反静态原则的是让迁移的结果依赖这些东西：

- 环境变量（不同机器的配置可能不同）
- 当前 ORM model 的定义（model 会随代码演进改变）
- 表里现有的数据（不同环境数据不同）

**误区一：通过环境变量控制迁移行为**

```python
def upgrade():
    table_name = os.getenv("TABLE_NAME", "users")
    op.create_table(table_name, ...)
```

你在本地没设 `TABLE_NAME`，用了默认值 `users`，创建了 `users` 表。同事的环境里 `TABLE_NAME=accounts`，跑同一条迁移创建了 `accounts` 表。两人跑了"同一条迁移"，数据库结构却不一样了。

**误区二：在迁移里导入 ORM model**

这是更常见也更隐蔽的问题：

```python
# 迁移写于 2025 年，当时 User 有 first_name 和 last_name 两个字段
from app.models import User
from sqlalchemy.orm import Session

def upgrade():
    op.add_column("user", sa.Column("full_name", sa.String))

    bind = op.get_bind()
    session = Session(bind=bind)
    for user in session.query(User).all():
        user.full_name = f"{user.first_name} {user.last_name}"
    session.commit()
```

写的时候能跑。但半年后 `User` model 把 `first_name` 和 `last_name` 合并删掉了，这条历史迁移再跑就会直接崩——而你只是想在新环境初始化数据库。

正确的做法是用原始 SQL 处理数据，不依赖 ORM model 的当前状态：

```python
def upgrade():
    op.add_column("user", sa.Column("full_name", sa.String))
    op.execute("""
        UPDATE "user"
        SET full_name = first_name || ' ' || last_name
        WHERE full_name IS NULL
    """)
```

这段 SQL 描述的是一个固定的结构事实。无论 ORM model 以后怎么变，这条迁移永远能正确执行。

**误区三：根据当前数据状态决定是否变更**

```python
def upgrade():
    conn = op.get_bind()
    result = conn.execute(text("SELECT COUNT(*) FROM \"user\""))
    if result.scalar() > 0:
        op.add_column("user", sa.Column("migrated", sa.Boolean))
```

这种写法让迁移结果取决于"当时表里有没有数据"。空表跑一次，有数据跑一次，得到的 schema 是不同的。迁移就失去了作为版本记录的意义。

### 2. 可预测

团队在执行一条迁移时，应该大致能知道它会改哪些表、新增哪些列或约束、是否影响现有数据。如果每次迁移都像开盲盒，数据库层就会变成最难协作的部分。

一个最常见的反例是：在已有数据的表上加 `NOT NULL` 列，却没有给默认值：

```python
# 危险：在有数据的表上添加 NOT NULL 列，没有默认值
def upgrade():
    op.add_column(
        "user",
        sa.Column("role", sa.String(50), nullable=False),
    )
```

这条迁移在本地开发环境（表是空的）能跑通，部署到线上（表里有几万行数据）就会直接报错：

```text
ERROR: column "role" contains null values
```

正确的做法通常有两种：

```python
# 方案一：先加可空列，填充数据，再收紧约束
def upgrade():
    op.add_column("user", sa.Column("role", sa.String(50), nullable=True))
    op.execute("UPDATE \"user\" SET role = 'member' WHERE role IS NULL")
    op.alter_column("user", "role", nullable=False)

def downgrade():
    op.drop_column("user", "role")
```

```python
# 方案二：加列时直接给 server_default
def upgrade():
    op.add_column(
        "user",
        sa.Column("role", sa.String(50), nullable=False, server_default="member"),
    )

def downgrade():
    op.drop_column("user", "role")
```

两种方案的选择取决于这个默认值是否需要永久保留：如果只是过渡期用，选方案一，填完数据后可以再把 `server_default` 去掉；如果本来就该有默认值，方案二更简洁。

### 3. 可回滚

不是所有项目都会频繁做 downgrade，但"能不能回滚"和"你有没有认真想过回滚路径"是两回事。

一个常见的坏习惯是 `downgrade()` 什么都不写：

```python
def upgrade():
    op.create_table(
        "post",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("title", sa.String(200), nullable=False),
        sa.Column("author_id", sa.Integer, sa.ForeignKey("user.id")),
    )
    op.create_index("post_author_id_idx", "post", ["author_id"])

def downgrade():
    pass  # 什么都不做
```

`downgrade()` 空着不代表"不需要回滚"，只是代表"回滚会静悄悄什么都不做"，然后 Alembic 认为已经成功降级了，而实际上表还在那里。

认真写的 `downgrade()` 应该是：

```python
def upgrade():
    op.create_table(
        "post",
        sa.Column("id", sa.Integer, primary_key=True),
        sa.Column("title", sa.String(200), nullable=False),
        sa.Column("author_id", sa.Integer, sa.ForeignKey("user.id")),
    )
    op.create_index("post_author_id_idx", "post", ["author_id"])

def downgrade():
    op.drop_index("post_author_id_idx", table_name="post")  # 先删依赖
    op.drop_table("post")                                    # 再删主体
```

注意顺序：`downgrade` 里要先删依赖（索引、外键），再删主体（表），顺序反了会报依赖错误。

有些操作的 `downgrade` 比较棘手，提交前需要特别想清楚：

- **删列**：`downgrade` 理论上是加回去，但数据已经永久丢失
- **列类型变更**：回滚需要转回原来的类型，数据可能被截断
- **删表**：重建表的结构可以写出来，但数据无法恢复

遇到这类不可逆操作，`downgrade` 里至少应该留一条注释说明原因，而不是空着：

```python
def downgrade():
    # 不可逆：列删除后数据已丢失，无法自动恢复
    raise NotImplementedError("This migration cannot be automatically reversed")
```

认真写 `downgrade()` 的意义，不只是回滚功能本身，更是逼你在提交迁移前把结构变化想清楚。

## 用自定义 Base 统一通用字段

如果不做任何处理，每个 model 都要重复写一遍和业务无关的基础字段：

```python
class User(Base):
    __tablename__ = "user"

    id: Mapped[int] = mapped_column(primary_key=True)         # 和 User 无关
    created_at: Mapped[datetime] = mapped_column(...)          # 和 User 无关
    updated_at: Mapped[datetime] = mapped_column(...)          # 和 User 无关
    email: Mapped[str] = mapped_column(unique=True)            # User 自己的字段
    username: Mapped[str]                                      # User 自己的字段


class Post(Base):
    __tablename__ = "post"

    id: Mapped[int] = mapped_column(primary_key=True)         # 和 Post 无关，重复
    created_at: Mapped[datetime] = mapped_column(...)          # 和 Post 无关，重复
    updated_at: Mapped[datetime] = mapped_column(...)          # 和 Post 无关，重复
    title: Mapped[str] = mapped_column(String(200))            # Post 自己的字段
    author_id: Mapped[int] = mapped_column(ForeignKey(...))    # Post 自己的字段
```

`id`、`created_at`、`updated_at` 和具体的业务 model 本身没有关系，却要在每个 model 里抄一遍。抄多了还容易出现不一致：有的叫 `created_at`，有的叫 `create_time`，有的忘了写 `updated_at`。

在 SQLAlchemy 里，所有 model 都需要继承同一个 `Base`。既然如此，可以把这些和业务无关的公共字段直接定义在 `Base` 里，每个 model 只写自己特有的部分：

```python
# app/database.py
from datetime import datetime

from sqlalchemy import DateTime, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


class Base(DeclarativeBase):
    id: Mapped[int] = mapped_column(primary_key=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )
```

之后所有 model 继承 `Base`，不需要重复写这三个字段：

```python
# app/models/user.py
from app.database import Base


class User(Base):
    __tablename__ = "user"

    email: Mapped[str] = mapped_column(unique=True)
    username: Mapped[str]


class Post(Base):
    __tablename__ = "post"

    title: Mapped[str] = mapped_column(String(200))
    author_id: Mapped[int] = mapped_column(ForeignKey("user.id"))
```

`User` 和 `Post` 自动都有 `id`、`created_at`、`updated_at`，不会出现某张表漏写或命名不一致的情况（有的叫 `created_at`，有的叫 `create_time`）。

`server_default=func.now()` 把时间戳交给数据库处理，不依赖应用层时钟，在多实例部署或批量写入时更可靠。

如果某张表确实不需要自增主键（比如关联表用复合主键），在子类里覆盖即可：

```python
class PostTag(Base):
    __tablename__ = "post_tag"

    id: Mapped[None] = None  # 覆盖掉 Base 里的 id
    post_id: Mapped[int] = mapped_column(ForeignKey("post.id"), primary_key=True)
    tag_id: Mapped[int] = mapped_column(ForeignKey("tag.id"), primary_key=True)
```

`Base` 提供默认值，需要例外的地方在子类里覆盖。

## 数据库命名规范比你想的更重要

没有统一命名时，数据库对象会越来越混乱：

- 外键名不一致
- 索引名风格不一致
- 表名单复数混用
- 审计字段命名混乱

建议至少统一这些规则：

- 表名统一风格
- 外键统一使用 `_id`
- 布尔字段统一使用 `is_` 或 `has_`
- 时间字段统一使用 `_at`
- 索引、唯一约束、外键有稳定命名模板

### 用 naming_convention 自动生成统一约束名

靠手动规范很难持续。更可靠的方式是在上面已经定义好的 `Base` 里加上 `naming_convention`，让所有约束名自动生成：

```python
# app/database.py
from datetime import datetime

from sqlalchemy import DateTime, MetaData, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

NAMING_CONVENTION = {
    "ix": "%(column_0_label)s_idx",
    "uq": "%(table_name)s_%(column_0_name)s_key",
    "ck": "%(table_name)s_%(constraint_name)s_check",
    "fk": "%(table_name)s_%(column_0_name)s_fkey",
    "pk": "%(table_name)s_pkey",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)

    id: Mapped[int] = mapped_column(primary_key=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), server_default=func.now(), onupdate=func.now()
    )
```

通用字段和命名约定都在同一个 `Base` 里，继承一次，全部生效。

配置好之后，Alembic autogenerate 生成的迁移文件里，约束名会自动遵循这套规则，不再需要手动指定：

```python
# 自动生成的迁移里，约束名会是这样的：
op.create_index("user_email_idx", "user", ["email"])
op.create_unique_constraint("user_email_key", "user", ["email"])
op.create_foreign_key("post_author_id_fkey", "post", "user", ["author_id"], ["id"])
```

而不是：

```python
op.drop_constraint("fk_2930a", "post", type_="foreignkey")
op.create_index("idx_x8d2", "post", ["author_id"])
```

还需要在 Alembic 的 `env.py` 里引用同一个 `metadata`，autogenerate 才能识别这套规则：

换句话说，命名规范不是只写在 model 里就会自动生效，它必须沿着同一份 `metadata` 一路传到 Alembic，迁移生成和数据库结构才能真正保持一致。

```python
# alembic/env.py
from app.database import Base

target_metadata = Base.metadata
```

### 命名规范的真正价值

这件事平时看起来不起眼，但会直接影响迁移质量、错误排查和数据库可读性。

比如你在数据库报错里看到这样的信息：

```text
duplicate key value violates unique constraint "uq_7f3d1c2"
```

和看到这样的信息：

```text
duplicate key value violates unique constraint "user_email_key"
```

排查体验完全不是一个等级。

前者需要你继续追查这串名字到底对应哪个表、哪个字段；
后者几乎一眼就知道问题在 `user.email` 这个唯一约束上。

所以命名规范的真正价值不是"更整齐"，而是：

- 报错更容易理解
- 迁移更容易审查
- 数据库对象更容易定位
- 团队沟通时更少歧义

## SQL 优先，不等于把所有逻辑都塞进 SQL

前面讨论的是数据库结构如何保持长期可维护，接下来要讨论的是数据库查询如何保持长期高效。
这时候就会碰到一个很重要的原则：`SQL First`。

"SQL 优先"的真正意思是：

能让数据库一次做完的筛选、聚合、关联，不要拆成多次查询再回 Python 循环处理。

它不是说：

- 所有逻辑都应该写成原始 SQL
- ORM 没意义
- 一切都要靠一条巨型查询解决

它真正想表达的是：

**数据库负责它最擅长的集合运算，Python 负责它更擅长的业务编排。**

数据库之所以值得优先承担这些工作，是因为它本来就是为数据检索和集合运算优化了几十年的系统。
过滤、排序、分页、聚合、去重、join 关联，这些事情对数据库来说是主业；而对 Python 这样的通用动态语言来说，更多只是“拿到结果之后再处理”的能力。

所以如果你明知道数据库可以一次完成，却仍然写成：

- 先查一张表
- 再查另一张表
- 再在 Python 里循环拼接
- 再在 Python 里做过滤和聚合

那你其实是在用更慢、更重、也更容易出错的方式，去做数据库本来就能做好的事情。

这也是为什么 `SQL First` 不只是一个性能优化建议，它更像是一种数据边界原则：
**能在数据库里一次完成的数据处理，就不要无意义地搬回 Python 再手工拼。**

## N+1：SQL First 最典型的反例

如果要选一个最能说明“为什么 SQL First 很重要”的问题，N+1 几乎一定排在前面。

它最典型的形态是：

- 先查一次主数据
- 然后在循环里，再为每条记录额外查一次关联数据
- 最后总查询数变成 `1 + N`

例如，下面这种写法就非常容易形成 N+1：

```python
# N+1 查询：每个用户额外发一条 SQL
users = await repo.list_users()
result = []
for user in users:
    post_count = await repo.count_posts_by_user(user.id)
    result.append({**user.__dict__, "post_count": post_count})
```

如果这里一共有 100 个用户，那么你发出的就不是 1 条查询，而是 101 条查询。
主查询先查出用户列表，然后每个用户再额外触发一条文章计数查询。

这就是 N+1 最典型的模式：
代码表面上很自然，数据库却在背后被你反复敲了很多次。

在真实 Web 项目里，N+1 往往不是以“统计文章数”这种教学案例出现的，而是藏在更日常的接口里。

比如文章列表页就是一个高发场景：

```python
# 错误写法：列表页每篇文章再查一次作者
posts = await posts_repo.list_recent_posts()
result = []

for post in posts:
    author = await users_repo.get(post.author_id)
    result.append(
        {
            "id": post.id,
            "title": post.title,
            "author_name": author.username,
        }
    )
```

如果一页有 20 篇文章，这段代码就可能变成 21 次查询。
文章越多，延迟越明显，而且这个问题在本地开发阶段通常不容易被肉眼发现。

### N+1 为什么会特别痛

N+1 的问题不只是“多查了几次”，而是它会同时放大多种成本：

- 查询次数爆炸
- 每次查询都要有一次网络往返
- 数据量一大，延迟会迅速放大
- 本地开发数据少时不明显，线上最容易暴露

很多团队第一次遇到 N+1 时，会误以为是：

- FastAPI 不够快
- ORM 不够快
- Python 本身太慢

但更常见的真实原因是：
数据库本来可以一次完成的工作，被拆成了多次查询再回 Python 手工聚合。

也正因为如此，N+1 不只是一个性能细节，而是最典型的数据边界错误之一。

### 把聚合和关联留在数据库里

更合理的做法，是让数据库一次把聚合完成：

```python
from sqlalchemy import func, select

# 一次查询：JOIN + GROUP BY 在数据库里完成聚合
stmt = (
    select(User, func.count(Post.id).label("post_count"))
    .outerjoin(Post, Post.author_id == User.id)
    .group_by(User.id)
)
rows = await db.execute(stmt)

result = [
    {"user": user, "post_count": count}
    for user, count in rows.all()
]
```

这段代码的重点不是“写得更像 SQL”，而是：

- 关联在数据库里做
- 聚合在数据库里做
- Python 最后只做轻量的结果组装

文章列表页的作者信息，也更适合在数据库里一次取完：

```python
stmt = (
    select(Post, User.username.label("author_name"))
    .join(User, User.id == Post.author_id)
    .order_by(Post.created_at.desc())
)
rows = await db.execute(stmt)

result = [
    {
        "id": post.id,
        "title": post.title,
        "author_name": author_name,
    }
    for post, author_name in rows.all()
]
```

这些例子的共同点都不是“SQL 写得更复杂”，而是：
数据库一次把该做的关联和统计做完，Python 不再扮演手工数据拼装器的角色。

对于更复杂的统计场景，也可以把预聚合放进子查询：

```python
post_count_subq = (
    select(Post.author_id, func.count(Post.id).label("post_count"))
    .group_by(Post.author_id)
    .subquery()
)

stmt = (
    select(User, post_count_subq.c.post_count)
    .outerjoin(post_count_subq, post_count_subq.c.author_id == User.id)
)
```

两种写法的共同点都是：

- 筛选在数据库里完成
- 聚合在数据库里完成
- 关联在数据库里完成
- Python 只负责把结果集组装成需要的结构

这才是 SQL First 真正想强调的边界。

## SQL First 的边界在哪里

SQL First 不是说 ORM 没意义，也不是说以后必须全写原始 SQL。你完全可以继续使用 ORM、Query Builder 或 SQLAlchemy 的表达式 API——重点不是"写法像不像 SQL"，而是让 join、聚合、排序和分页最终由数据库完成，而不是把数据全取回 Python 再手工过滤和聚合。

SQL First 在某些场景下收益最明显，在某些场景下反而要克制。

**最值得应用的接口类型**：列表页、管理后台统计、复杂筛选接口、需要返回嵌套数据的读取接口。这类接口最容易出现 N+1、过量数据传输和 Python 层重复聚合，是 SQL First 最直接的受益场景。

**需要克制的情况**：不是所有东西都适合塞进一条 SQL。如果查询已经复杂到新人根本读不懂、改一个字段就要重写大半条语句、业务规则开始混杂在 SQL 和 Python 之间，那维护成本可能已经开始反噬。

SQL First 是一个数据边界原则，不是"一切都用 SQL"的教条。更准确的说法是：

**数据库做它最擅长的集合运算，Python 做它最擅长的业务编排。两者各司其职，不要错位。**

## 落地清单

- 在 `alembic.ini` 里配置 `file_template`，启用日期格式文件名。
- 禁止提交含糊迁移名，命名要能表达"改了什么"。
- 迁移文件只描述确定的结构变化，不依赖环境变量、ORM model 或当前数据状态。
- 数据迁移用原始 SQL，不要在迁移里导入 ORM model。
- 认真写 `downgrade()`，不可逆操作至少留注释说明。
- 在 `Base` 里统一定义 `id`、`created_at`、`updated_at`，所有 model 继承，不重复定义。
- 在 `Base` 的 `MetaData` 里配置 `naming_convention`，让约束名自动统一。
- 确保 `alembic/env.py` 引用同一份 `Base.metadata`，命名规范才能传到 autogenerate。
- 统一外键、布尔、时间字段的命名风格。
- 对高频列表接口检查是否存在 N+1 查询。
- 优先把筛选、排序、聚合下推到 SQL 层。
- 对超复杂查询保留可维护性边界，不要盲目单 SQL 化。

---

下一篇：[测试不是负担：FastAPI 项目的测试分层与代码质量](/posts/articles/testing-and-code-quality-pa5un6)