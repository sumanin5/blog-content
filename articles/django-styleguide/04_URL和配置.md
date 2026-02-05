---
allow_comments: true
author: tomyi
author_id: 019c21db-4c02-7836-9d5b-36c688f8b4e0
category: django-styleguide
category_id: 019c2769-763b-734d-873b-c0b734d830ab
date: '2026-02-05 09:40:42'
enable_jsx: false
is_featured: false
meta_description: 学习如何科学地管理 Django 各类配置项，利用环境变量和集成配置保持项目的灵活性与安全性。
meta_keywords: Django, Settings, URLs, 环境变量, 配置管理
meta_title: Django URL 与系统配置规范 | 项目工程化
slug: django-styleguide-04-urls-settings-p95s0f
status: published
summary: 项目工程化配置：优雅地管理 URL 路由和 settings.py，支持多环境与环境变量。
tags:
- Django
- 配置管理
- 项目规范
title: 04-URL和配置(Settings)
use_server_rendering: true
---

# Django Styleguide 中文翻译 - URL 和配置

## URLs

> 🎯 **核心原则：** 1 个 URL 对应 1 个 API，即 1 个 URL 对应 1 个操作

### 基本组织原则

我们通常按照与 API 相同的方式组织 URL：

- **1 个 URL 对应 1 个 API**
- **1 个 URL 对应 1 个操作（action）**

### 按领域拆分 URL

**通用经验法则：** 将不同领域（domain）的 URL 分别放在各自的 `domain_patterns` 列表中，然后从 `urlpatterns` 进行 include。

#### 基础示例

以下是使用上述 API 的示例：

```python
from django.urls import path, include

from project.education.apis import (
    CourseCreateApi,
    CourseUpdateApi,
    CourseListApi,
    CourseDetailApi,
    CourseSpecificActionApi,
)

# 课程相关的所有 URL 模式
course_patterns = [
    path('', CourseListApi.as_view(), name='list'),
    path('<int:course_id>/', CourseDetailApi.as_view(), name='detail'),
    path('create/', CourseCreateApi.as_view(), name='create'),
    path('<int:course_id>/update/', CourseUpdateApi.as_view(), name='update'),
    path(
        '<int:course_id>/specific-action/',
        CourseSpecificActionApi.as_view(),
        name='specific-action'
    ),
]

# 主 URL 配置
urlpatterns = [
    path('courses/', include((course_patterns, 'courses'))),
]
```

#### 📝 代码详解

**course_patterns 列表：**

- `''` - 列表接口，对应 `/courses/`
- `'<int:course_id>/'` - 详情接口，对应 `/courses/123/`
- `'create/'` - 创建接口，对应 `/courses/create/`
- `'<int:course_id>/update/'` - 更新接口，对应 `/courses/123/update/`
- `'<int:course_id>/specific-action/'` - 特定操作接口

**include 的用法：**

```python
path('courses/', include((course_patterns, 'courses')))
```

- 第一个参数 `'courses/'` 是 URL 前缀
- `course_patterns` 是要包含的 URL 模式列表
- `'courses'` 是命名空间（namespace），用于反向解析

#### 💡 这种拆分的好处

**1. 灵活性更高**

```python
# 可以轻松地将不同领域的 URL 移动到单独的模块

# urls/courses.py
course_patterns = [...]

# urls/users.py
user_patterns = [...]

# urls/orders.py
order_patterns = [...]

# 主 urls.py
from .urls.courses import course_patterns
from .urls.users import user_patterns
from .urls.orders import order_patterns

urlpatterns = [
    path('courses/', include((course_patterns, 'courses'))),
    path('users/', include((user_patterns, 'users'))),
    path('orders/', include((order_patterns, 'orders'))),
]
```

**2. 减少合并冲突**

对于大型项目，如果所有 URL 都在一个 `urls.py` 文件中：

- ❌ 多人同时修改容易产生合并冲突
- ❌ 文件过长，难以维护

拆分后：

- ✅ 不同团队可以修改各自领域的 URL 文件
- ✅ 减少 `urls.py` 的合并冲突
- ✅ 代码组织更清晰

**3. 更好的代码组织**

```
project/
├── education/
│   ├── apis.py
│   └── urls.py          # 教育模块的 URL
├── users/
│   ├── apis.py
│   └── urls.py          # 用户模块的 URL
└── main_urls.py         # 主 URL 配置
```

---

### 树状结构展示

如果你更喜欢看到完整的 URL 树状结构，可以不提取单独的变量，直接在 include 中嵌套。

#### 实际项目示例

这是来自 [Django Styleguide Example](https://github.com/HackSoftware/Django-Styleguide-Example/blob/master/styleguide_example/files/urls.py) 的真实示例：

```python
from django.urls import path, include

from styleguide_example.files.apis import (
    FileDirectUploadApi,
    FilePassThruUploadStartApi,
    FilePassThruUploadFinishApi,
    FilePassThruUploadLocalApi,
)

urlpatterns = [
    path(
        "upload/",
        include(([
            path(
                "direct/",
                FileDirectUploadApi.as_view(),
                name="direct"
            ),
            path(
                "pass-thru/",
                include(([
                    path(
                        "start/",
                        FilePassThruUploadStartApi.as_view(),
                        name="start"
                    ),
                    path(
                        "finish/",
                        FilePassThruUploadFinishApi.as_view(),
                        name="finish"
                    ),
                    path(
                        "local/<str:file_id>/",
                        FilePassThruUploadLocalApi.as_view(),
                        name="local"
                    )
                ], "pass-thru"))
            )
        ], "upload"))
    )
]
```

#### 📝 URL 结构解析

这种写法会生成以下 URL：

```
/upload/direct/                    → FileDirectUploadApi
/upload/pass-thru/start/          → FilePassThruUploadStartApi
/upload/pass-thru/finish/         → FilePassThruUploadFinishApi
/upload/pass-thru/local/{file_id}/ → FilePassThruUploadLocalApi
```

#### 两种风格对比

**方式 1：提取变量（推荐用于大型项目）**

```python
# 优点：
# ✅ 易于移动到单独的模块
# ✅ 变量名提供额外的文档说明
# ✅ 更容易测试和重用

course_patterns = [...]
user_patterns = [...]
order_patterns = [...]

urlpatterns = [
    path('courses/', include((course_patterns, 'courses'))),
    path('users/', include((user_patterns, 'users'))),
    path('orders/', include((order_patterns, 'orders'))),
]
```

**方式 2：树状嵌套（适合查看整体结构）**

```python
# 优点：
# ✅ 一眼看到完整的 URL 层级结构
# ✅ 嵌套关系更直观
# ✅ 适合展示复杂的嵌套路由

urlpatterns = [
    path('api/', include(([
        path('v1/', include(([
            path('users/', ...),
            path('orders/', ...),
        ], 'v1'))),
    ], 'api'))),
]
```

#### 💡 选择建议

| 场景                 | 推荐方式           | 原因                 |
| -------------------- | ------------------ | -------------------- |
| 大型项目（多人协作） | 方式 1（提取变量） | 减少冲突，易于模块化 |
| 中小型项目           | 方式 1 或 2 均可   | 根据团队偏好         |
| 需要展示嵌套关系     | 方式 2（树状嵌套） | 层级结构更清晰       |
| URL 需要在多处复用   | 方式 1（提取变量） | 便于重用和维护       |

**这取决于你和你的团队的偏好。** 选择一种风格并在整个项目中保持一致。

---

## Settings

> 🎯 **核心原则：** 分离 Django 设置和第三方设置，所有内容都应在 `base.py` 中包含

### 文件夹结构

我们倾向于遵循 [`cookiecutter-django`](https://github.com/cookiecutter/cookiecutter-django) 的文件夹结构，但做了一些调整：

- **分离 Django 特定设置和其他设置**
- **所有内容都应该包含在 `base.py` 中**
  - `production.py` 中不应该有独占的内容
  - 只在生产环境运行的内容通过环境变量控制

#### 标准项目结构

以下是 [`Styleguide-Example`](https://github.com/HackSoftware/Styleguide-Example) 项目的文件夹结构：

```
config/
├── __init__.py
├── django/                      # Django 相关设置
│   ├── __init__.py
│   ├── base.py                 # 基础设置（核心）
│   ├── local.py                # 本地开发设置
│   ├── production.py           # 生产环境设置
│   └── test.py                 # 测试环境设置
├── settings/                    # 第三方和其他设置
│   ├── __init__.py
│   ├── celery.py               # Celery 配置
│   ├── cors.py                 # CORS 配置
│   ├── sentry.py               # Sentry 错误追踪配置
│   └── sessions.py             # Session 配置
├── urls.py                      # 主 URL 配置
├── env.py                       # 环境变量读取工具
├── wsgi.py                      # WSGI 应用
└── asgi.py                      # ASGI 应用
```

---

### 配置文件说明

#### 📂 config/django/ - Django 相关设置

##### 1. base.py（基础设置）

```python
# config/django/base.py

# 包含大部分设置
# 从 config/settings 导入所有其他配置

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    # ...
]

MIDDLEWARE = [...]

DATABASES = {...}

# ... 其他 Django 核心设置 ...

# 在文件末尾导入所有第三方配置
from config.settings.cors import *  # noqa
from config.settings.sessions import *  # noqa
from config.settings.celery import *  # noqa
from config.settings.sentry import *  # noqa
```

**作用：**

- ✅ 包含所有核心 Django 设置
- ✅ 导入所有第三方集成配置
- ✅ 作为所有环境的基础配置

##### 2. production.py（生产环境）

```python
# config/django/production.py

from config.django.base import *  # noqa

# 覆盖生产环境特定的设置
DEBUG = False

ALLOWED_HOSTS = env.list('DJANGO_ALLOWED_HOSTS')

# 生产环境的安全设置
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

**作用：**

- ✅ 从 `base.py` 导入所有设置
- ✅ 覆盖生产环境特定的设置（如安全配置）
- ❌ 不应包含任何独占的集成或功能

##### 3. test.py（测试环境）

```python
# config/django/test.py

from config.django.base import *  # noqa

# 测试环境优化
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': ':memory:',
    }
}

# 禁用密码哈希以加速测试
PASSWORD_HASHERS = [
    'django.contrib.auth.hashers.MD5PasswordHasher',
]

# 禁用 Celery 的异步任务
CELERY_TASK_ALWAYS_EAGER = True
```

**使用方式：**

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = config.django.test
```

**作用：**

- ✅ 优化测试运行速度（内存数据库、简化密码哈希）
- ✅ 禁用异步任务，使测试同步执行

##### 4. local.py（本地开发，可选）

```python
# config/django/local.py

from config.django.base import *  # noqa

# 本地开发的便利设置
DEBUG = True

# 本地调试工具
INSTALLED_APPS += [
    'debug_toolbar',
    'django_extensions',
]

MIDDLEWARE = ['debug_toolbar.middleware.DebugToolbarMiddleware'] + MIDDLEWARE

INTERNAL_IPS = ['127.0.0.1']
```

**使用方式：**

```python
# manage.py
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.django.local')
```

**作用：**

- ✅ 本地开发的调试工具（如 Django Debug Toolbar）
- ✅ 可选，如果不需要可以直接使用 `base.py`

---

#### 📂 config/settings/ - 第三方和集成设置

这个目录存放所有非 Django 核心的配置：

##### 示例：celery.py

```python
# config/settings/celery.py

from config.env import env

CELERY_BROKER_URL = env('CELERY_BROKER_URL', default='redis://localhost:6379/0')
CELERY_RESULT_BACKEND = env('CELERY_RESULT_BACKEND', default='redis://localhost:6379/0')

CELERY_TIMEZONE = 'UTC'
CELERY_TASK_TRACK_STARTED = True
CELERY_TASK_TIME_LIMIT = 30 * 60  # 30 分钟
```

##### 示例：cors.py

```python
# config/settings/cors.py

from config.env import env

CORS_ALLOW_CREDENTIALS = True
CORS_ALLOWED_ORIGINS = env.list('CORS_ALLOWED_ORIGINS', default=[])
```

**组织方式的好处：**

- ✅ **模块化：** 每个集成有自己的配置文件
- ✅ **易于查找：** 知道在哪里找到特定的配置
- ✅ **减少冲突：** 不同开发者可以修改不同的配置文件

---

#### 🔧 config/env.py - 环境变量工具

```python
# config/env.py

import environ

env = environ.Env()
```

**使用方式：**

```python
# 在任何需要读取环境变量的地方
from config.env import env

DEBUG = env.bool('DJANGO_DEBUG', default=False)
SECRET_KEY = env('DJANGO_SECRET_KEY')
DATABASE_URL = env('DATABASE_URL')
```

**为什么要单独的 `env.py`？**

- ✅ **集中管理环境变量读取**：所有从 OS 层级进入应用的变量都经过这一个入口。
- ✅ **保证单例与性能**：避免在每个拆分的配置文件中重复创建 `Env()` 实例计数，利用 Python 模块导入的单例特性。
- ✅ **行为统一控制**：便于在全球范围内统一配置环境变量的解析行为（如类型检查规则、Schema 定义等）。
- ✅ **降低耦合**：如果未来更换环境变量库（如从 `django-environ` 换成原生 `os`），只需修改这一个文件。

---

### 💡 深度解析：状态管理与整洁架构

将 `env.py` 独立出来，本质上是一种 **“状态的集中统一管理”**。在工程实践中，这种模式遵循了 **整洁架构（Clean Architecture）** 的核心思想：

#### 1. 将环境视为“外部细节”

在架构设计中，操作系统环境变量属于“底层细节”，它们往往是杂乱无章的纯字符串。`env.py` 扮演了一个 **“适配器（Adapter）”** 的角色：

- **输入**：来自 OS 的原始、类型未定义的字符串流。
- **处理**：进行类型转换（int, bool, list）、设置默认值。
- **输出**：可预测、强类型的应用状态。

#### 2. 唯一的真理来源 (Single Source of Truth)

如果不统一读取，每个 `settings/*.py` 都在自行解析环境变量，这会导致逻辑碎片化。例如，如果两个文件对同一个变量的默认值理解不一致，会引发极其隐蔽的 Bug。集中管理确保了“读取”和“解析”这两个动作在整个生命周期内只发生一次，且结果一致。

#### 3. 提升可测试性与透明度

通过集中化的 `env.py`，开发者可以非常容易地通过 `print(env.ENVIRON)` 观察到整个项目能感知到的所有外部输入，这在排查 Docker 或 CI/CD 配置问题时是极其高效的导航台。

---

### 环境变量前缀

#### 为什么使用 `DJANGO_` 前缀？

在很多示例中，你会看到环境变量通常带有 `DJANGO_` 前缀：

```bash
DJANGO_DEBUG=True
DJANGO_SECRET_KEY=xxx
DJANGO_ALLOWED_HOSTS=example.com,www.example.com
DJANGO_SETTINGS_MODULE=config.django.production
```

**使用前缀的场景：**

- ✅ 同一环境运行多个应用（Django + Node.js + Go 等）
- ✅ 区分不同应用的配置变量
- ✅ 避免命名冲突

#### HackSoft 的实践

> 💡 **HackSoft 的做法：** 只为 Django 特定的环境变量添加前缀

在 HackSoft，我们通常不会在同一环境中运行多个应用，所以：

**加前缀的变量（Django 特定）：**

```bash
DJANGO_SETTINGS_MODULE=config.django.production
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=example.com
DJANGO_CORS_ORIGIN_WHITELIST=https://example.com
```

**不加前缀的变量（通用或第三方）：**

```bash
AWS_SECRET_KEY=xxx
CELERY_BROKER_URL=redis://localhost:6379/0
EMAILS_ENABLED=True
DATABASE_URL=postgres://user:pass@localhost/dbname
SENTRY_DSN=https://xxx@sentry.io/xxx
```

#### 💡 命名建议

| 变量类型        | 是否加前缀      | 示例                                         |
| --------------- | --------------- | -------------------------------------------- |
| Django 核心设置 | ✅ 加 `DJANGO_` | `DJANGO_DEBUG`, `DJANGO_SECRET_KEY`          |
| DRF 相关        | ✅ 加 `DJANGO_` | `DJANGO_CORS_ORIGIN_WHITELIST`               |
| AWS 服务        | ❌ 不加         | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| Celery          | ❌ 不加         | `CELERY_BROKER_URL`                          |
| 数据库          | ❌ 不加         | `DATABASE_URL`                               |
| 第三方服务      | ❌ 不加         | `SENTRY_DSN`, `STRIPE_API_KEY`               |

**核心原则：保持一致性！** 无论你选择哪种方式，确保在整个项目中保持一致。

---

### 第三方集成配置

#### 设计原则

由于所有配置都应该在 `base.py` 中导入，但有时我们不想为本地开发配置某个集成，因此采用以下方法：

**标准模式：**

1. 集成特定的设置放在 `config/settings/some_integration.py`
2. 总是有一个布尔设置 `USE_SOME_INTEGRATION`，从环境变量读取，默认为 `False`
3. 如果值为 `True`，则继续读取其他设置，如果环境中缺少必需的变量则失败

#### 实际示例：Sentry 配置

```python
# config/settings/sentry.py

from config.env import env

# 第一步：读取 DSN，默认为空字符串
SENTRY_DSN = env('SENTRY_DSN', default='')

# 第二步：只有在 DSN 存在时才配置 Sentry
if SENTRY_DSN:
    import sentry_sdk
    from sentry_sdk.integrations.django import DjangoIntegration
    from sentry_sdk.integrations.celery import CeleryIntegration

    sentry_sdk.init(
        dsn=SENTRY_DSN,
        integrations=[
            DjangoIntegration(),
            CeleryIntegration(),
        ],
        environment=env('SENTRY_ENVIRONMENT', default='development'),
        traces_sample_rate=env.float('SENTRY_TRACES_SAMPLE_RATE', default=0.0),
        send_default_pii=True,
    )
```

**完整文件参考：** [Styleguide-Example/config/settings/sentry.py](https://github.com/HackSoftware/Styleguide-Example/blob/master/config/settings/sentry.py)

#### 📝 工作流程详解

**场景 1：生产环境（启用 Sentry）**

```bash
# .env (生产环境)
SENTRY_DSN=https://xxx@sentry.io/xxx
SENTRY_ENVIRONMENT=production
SENTRY_TRACES_SAMPLE_RATE=1.0
```

执行流程：

1. ✅ `SENTRY_DSN` 有值
2. ✅ 导入 `sentry_sdk` 并配置
3. ✅ Sentry 错误追踪启用

**场景 2：本地开发（不启用 Sentry）**

```bash
# .env (本地开发)
# SENTRY_DSN 未设置或为空
```

执行流程：

1. ✅ `SENTRY_DSN` 为空字符串
2. ✅ 跳过 `if` 块，不配置 Sentry
3. ✅ 不会因为缺少 Sentry 配置而报错
4. ✅ 本地开发可以正常运行

#### 更多集成示例

**示例：Celery 配置**

```python
# config/settings/celery.py

from config.env import env

USE_CELERY = env.bool('USE_CELERY', default=False)

if USE_CELERY:
    CELERY_BROKER_URL = env('CELERY_BROKER_URL')  # 必需
    CELERY_RESULT_BACKEND = env('CELERY_RESULT_BACKEND')  # 必需

    CELERY_TIMEZONE = 'UTC'
    CELERY_TASK_TRACK_STARTED = True
    CELERY_TASK_TIME_LIMIT = 30 * 60
```

**示例：AWS S3 配置**

```python
# config/settings/storage.py

from config.env import env

USE_S3_STORAGE = env.bool('USE_S3_STORAGE', default=False)

if USE_S3_STORAGE:
    AWS_ACCESS_KEY_ID = env('AWS_ACCESS_KEY_ID')  # 必需
    AWS_SECRET_ACCESS_KEY = env('AWS_SECRET_ACCESS_KEY')  # 必需
    AWS_STORAGE_BUCKET_NAME = env('AWS_STORAGE_BUCKET_NAME')  # 必需
    AWS_S3_REGION_NAME = env('AWS_S3_REGION_NAME', default='us-east-1')

    DEFAULT_FILE_STORAGE = 'storages.backends.s3boto3.S3Boto3Storage'
```

#### 💡 这种模式的好处

| 好处                | 说明                                     |
| ------------------- | ---------------------------------------- |
| ✅ **本地开发友好** | 不需要配置所有第三方服务就能运行项目     |
| ✅ **渐进式配置**   | 可以先用默认值，需要时再配置             |
| ✅ **明确的依赖**   | 清楚地知道哪些环境变量是必需的           |
| ✅ **避免意外错误** | 本地开发不会因为缺少生产环境的密钥而崩溃 |

---

### 从 .env 读取配置

#### 为什么使用 .env 文件？

拥有本地 `.env` 文件是为设置提供值的好方法：

**优点：**

- ✅ 不需要在系统环境变量中设置
- ✅ 易于管理和版本控制（通过 `.env.example`）
- ✅ 不同项目可以有不同的配置
- ✅ 便于新开发者快速上手

#### 配置方法

[`django-environ`](https://django-environ.readthedocs.io/en/latest/) 提供了读取 `.env` 文件的方法：

```python
# config/django/base.py (文件开头)

import os

from config.env import env, environ

# 构建项目内部路径，如：os.path.join(BASE_DIR, ...)
BASE_DIR = environ.Path(__file__) - 3

# 读取 .env 文件
env.read_env(os.path.join(BASE_DIR, ".env"))
```

#### 📝 路径计算详解

```python
BASE_DIR = environ.Path(__file__) - 3
```

假设文件结构：

```
/home/user/myproject/
├── config/
│   └── django/
│       └── base.py        ← __file__ 在这里
├── .env                    ← 目标文件
└── manage.py
```

计算过程：

- `__file__` = `/home/user/myproject/config/django/base.py`
- `- 1` = `/home/user/myproject/config/django/`
- `- 2` = `/home/user/myproject/config/`
- `- 3` = `/home/user/myproject/` ← `BASE_DIR`

---

#### .env 文件示例

```bash
# .env (本地开发环境)

# Django 核心设置
DJANGO_DEBUG=True
DJANGO_SECRET_KEY=your-secret-key-here
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1

# 数据库
DATABASE_URL=postgres://user:password@localhost:5432/mydb

# Redis
REDIS_URL=redis://localhost:6379/0

# Celery
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0

# AWS (可选，本地开发可能不需要)
# AWS_ACCESS_KEY_ID=xxx
# AWS_SECRET_ACCESS_KEY=xxx
# AWS_STORAGE_BUCKET_NAME=my-bucket

# Sentry (可选，本地开发可以不配置)
# SENTRY_DSN=https://xxx@sentry.io/xxx

# 邮件
EMAILS_ENABLED=False
# EMAIL_HOST=smtp.gmail.com
# EMAIL_PORT=587
# EMAIL_HOST_USER=your-email@gmail.com
# EMAIL_HOST_PASSWORD=your-password
```

---

#### .env.example 文件

> ⚠️ **重要安全提示（绝对红线）：** 不要将 `.env` 提交到源代码控制！

**正确的做法：**

1. **将 `.env` 添加到 `.gitignore`：**

```gitignore
# .gitignore

.env
*.env
.env.local
```

2. **创建 `.env.example` 作为模板：**

```bash
# .env.example (提交到 Git)

# Django 核心设置
DJANGO_DEBUG=
DJANGO_SECRET_KEY=
DJANGO_ALLOWED_HOSTS=

# 数据库
DATABASE_URL=

# Redis
REDIS_URL=

# Celery
CELERY_BROKER_URL=
CELERY_RESULT_BACKEND=

# AWS（可选）
# AWS_ACCESS_KEY_ID=
# AWS_SECRET_ACCESS_KEY=
# AWS_STORAGE_BUCKET_NAME=

# Sentry（可选）
# SENTRY_DSN=

# 邮件
EMAILS_ENABLED=
# EMAIL_HOST=
# EMAIL_PORT=
# EMAIL_HOST_USER=
# EMAIL_HOST_PASSWORD=
```

3. **在 README 中说明：**

````markdown
## 环境配置

1. 复制 `.env.example` 为 `.env`：

   ```bash
   cp .env.example .env
   ```

2. 编辑 `.env` 文件，填入实际的配置值。

3. 不要提交 `.env` 文件到 Git！
````

#### 💡 最佳实践总结

| 实践                           | 说明                           | 优先级   |
| ------------------------------ | ------------------------------ | -------- |
| ✅ 使用 `.env` 文件            | 方便本地开发配置               | 高       |
| ✅ 提供 `.env.example`         | 帮助新开发者了解需要哪些配置   | 高       |
| ✅ 将 `.env` 加入 `.gitignore` | 防止泄露敏感信息               | **必须** |
| ✅ 生产环境使用真实环境变量    | 不依赖 `.env` 文件             | 高       |
| ✅ 为可选配置添加注释          | 说明哪些是必需的，哪些是可选的 | 中       |

---

## 📚 本章总结

### URLs 最佳实践

✅ **DO（推荐）：**

- 1 个 URL 对应 1 个 API 操作
- 按领域拆分 URL 到不同的 `patterns` 列表
- 使用命名空间（namespace）组织 URL
- 选择一种风格（提取变量 vs 树状嵌套）并保持一致

❌ **DON'T（不推荐）：**

- 所有 URL 都堆在一个 `urlpatterns` 列表中
- 混合使用多种风格
- 没有命名空间导致 URL 名称冲突

---

### Settings 最佳实践

✅ **DO（推荐）：**

- 使用 `config/django/` 和 `config/settings/` 分离配置
- 所有配置都在 `base.py` 中包含
- 使用环境变量控制集成的启用/禁用
- 提供 `.env.example` 但不提交 `.env`
- 为环境变量选择一致的命名规范

❌ **DON'T（不推荐）：**

- 在 `production.py` 中有独占的集成配置
- 硬编码敏感信息（密钥、密码）
- 提交 `.env` 文件到源代码控制
- 本地开发需要配置所有生产环境的服务

---

### 环境变量最佳实践

✅ **DO（推荐）：**

- 使用 `django-environ` 读取环境变量
- 为可选配置提供合理的默认值
- 使用布尔标志控制集成的启用
- 文档中明确列出所有必需的环境变量

❌ **DON'T（不推荐）：**

- 在代码中硬编码配置值
- 没有默认值导致本地开发困难
- 缺少环境变量时静默失败

---

### 快速检查清单

在实施本章建议时，请确保：

- [ ] URL 按领域组织，使用 `include()` 和命名空间
- [ ] 配置文件分为 `config/django/` 和 `config/settings/`
- [ ] 所有配置在 `base.py` 中导入
- [ ] 使用环境变量控制集成启用
- [ ] 有 `.env.example` 文件，`.env` 在 `.gitignore` 中
- [ ] 环境变量命名保持一致（是否使用前缀）
- [ ] 文档中说明了环境配置步骤

---

## 🔗 相关资源

- [cookiecutter-django](https://github.com/cookiecutter/cookiecutter-django) - Django 项目模板
- [django-environ](https://django-environ.readthedocs.io/) - 环境变量管理
- [Django Settings Best Practices](https://djangostars.com/blog/configuring-django-settings-best-practices/)
- [12 Factor App - Config](https://12factor.net/config) - 配置管理最佳实践

---

**下一章：** [05\_错误和异常处理.md](/posts/articles/django-styleguide-05-exception-handling-c4meub) - 学习如何优雅地处理 API 错误和异常