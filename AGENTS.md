# AGENTS.md

本仓库只存放 Blog FR 的文章内容。推送到 `main` 会通过 Webhook 立即同步到线上博客，**每次推送都等同于发布**。

字段的完整说明见 `.agent/skills/blog-post-frontmatter/SKILL.md` 与 `.agent/skills/category-metadata/SKILL.md`；本文件是上传时必须遵守的规则，冲突时以本文件为准。

## 1. 目录规则

- 文章路径只能是 `articles/<专栏>/<文章>.md` 或 `ideas/<专栏>/<文章>.md`（`.mdx` 同理）。
- **不要**在仓库根目录、`articles/`、`ideas/` 第一层或专栏的子目录里放 `.md` 笔记：除说明文件外，这些位置的 `.md` 也会被同步成文章（归入 `uncategorized`）。
- 说明类文档只能使用这些文件名：`README`、`AGENTS`、`CLAUDE`、`LICENSE`、`CONTRIBUTING`、`CHANGELOG`。
- **专栏目录名必须是小写英文加连字符**，例如 `fastapi-engineering-practice`。中文目录名会被转成拼音 slug，后端写回 `index.md` 时会另建一个拼音目录，造成重复专栏目录（`ideas/人间世/` 与 `ideas/ren-jian-shi/` 就是这样产生的）。专栏的中文名写在 `index.md` 的 `title` 里。

## 2. 新建专栏

在 `<类型>/<专栏目录>/` 下创建 `index.md`：

```yaml
---
title: FastAPI 工程化实践
excerpt: 一句话介绍，显示在专栏列表和专栏页标题下方。
icon: "⚡"
cover: assets/fastapi-engineering-cover.png
sort: 1
post_sort: title_asc
hidden: false
---

# FastAPI 工程化实践

专栏介绍（正文会作为专栏页的描述，支持 Markdown）。

## 系列文章

- [01-项目结构怎么设计](./01-how-to-structure-fastapi-projects.md)
```

- 系列专栏用 `post_sort: title_asc`，配合标题里的编号前缀保证顺序。
- 正文里链接本专栏的文章时写相对路径 `./01-xxx.md`，同步时会转换成线上地址；不要手写 `/posts/...` 地址。
- `icon` 少于 10 个字符时按 emoji 处理；用图片时写相对路径，例如 `assets/icon.svg`。

## 3. 新建文章

### 解析器的硬性要求

同步时只有下面这些会报错或影响是否上线，其余都是约定：

| 字段 | 是否必需 | 不写 / 写错时 |
| --- | --- | --- |
| `author` | **必需** | 同步报错，文章不会导入；值必须是已存在的用户名（默认 `tomyi`） |
| `status` | 强烈建议 | **默认是草稿**，只在后台可见；只接受 `draft` / `published`，写其他值（如 `archived`）会报错 |
| `date` | 可选 | 不写则按同步时间发布；写未来时间即定时发布 |
| `title` | 可选 | 使用文件名（不含扩展名） |
| `slug` | 可选 | 由文件名生成并加随机后缀，回填到 frontmatter |
| `summary` | 可选 | 从正文自动截取摘要；也可写 `description` |
| `tags` | 可选 | 列表或逗号分隔字符串（`tags: FastAPI, Python`）均可 |

`date` 支持的写法：`2026-12-01`、`2026-12-01 09:00:00`、`'2026-12-01 09:00:00'`、`2026-12-01T09:00:00+08:00`。**不支持** `2026/12/01` 这类斜杠格式，会导致同步报错。

文件名没有格式要求，中文、英文均可。

### 最小可用写法（想法流常用）

```yaml
---
author: tomyi
status: published
---

正文……
```

标题取文件名，发布时间取同步时间，摘要从正文截取。

### 推荐写法（专栏系列文章）

```yaml
---
title: 09-FastAPI 后台任务该交给谁
author: tomyi
status: published
summary: 80–120 字的中文摘要，显示在列表卡片和搜索结果中。
tags:
  - FastAPI
  - Python
meta_title: FastAPI 后台任务：BackgroundTasks 与任务队列的边界 | FastAPI 工程化实践
meta_description: 一句话 SEO 描述，包含核心关键词。
meta_keywords: FastAPI, 后台任务, Celery
cover: assets/background-tasks-cover.png
---

# FastAPI 后台任务该交给谁

正文……
```

以下是**约定**，不遵守也能同步，但会影响展示效果：

- `articles/` 的系列专栏：文件名用 `NN-<英文短名>.md`，`title` 带同样的 `NN-` 前缀，配合 `index.md` 的 `post_sort: title_asc` 保证顺序。`ideas/` 不需要编号。
- 写 `summary`，自动截取的摘要常常不完整。
- 标签优先复用已有标签，大小写保持一致（`FastAPI`、`Next.js`）。
- 正文只写一个一级标题 `#`。
- 只有用到 JSX 组件时才设 `enable_jsx: true`；数学公式、Mermaid 不需要。
- 想让文章下线：改成 `status: draft`，不要删除文件。

## 4. 后端回填字段：不要手写

`slug`、`author_id`、`category`、`category_id`、`cover_media_id`、`icon_id` 由同步过程自动写回。

- 新文章**不要**写这些字段，推送后 `git pull` 拿到回填值。
- 已有文章**不要**修改这些字段，否则文章会与线上记录脱钩（例如改 `slug` 会改变文章地址，旧链接失效）。
- **移动文章到另一个专栏**：移动文件，同时删除 `category` 和 `category_id` 两行。`category_id` 优先于目录路径，不删的话文章仍属于原专栏。

## 5. 图片

- 图片放在专栏的 `assets/` 子目录，用**相对于当前 `.md` 文件的路径**引用：`cover: assets/xxx.png`、`![说明](assets/xxx.png)`。
- **不要只写裸文件名**（`cover: cover.png`）而文件实际在子目录里：本地路径找不到时，后端会在媒体库里按文件名模糊搜索，可能挂上其他专栏的同名图片。
- 文件名带上专栏或文章前缀，避免 `cover.png`、`icon.svg` 这种全库重名的名字。
- 优先使用 `.png`、`.jpg`、`.webp`；单张图片控制在 1 MB 以内。封面建议横图，宽度不小于 1600px。

## 6. 提交规则

- 修改前先 `git pull`；后端会推送 `chore: sync metadata from database [skip ci]` 提交。
- 一次提交只做一件事，提交信息写清楚涉及的专栏或文章，例如 `feat(fastapi): 新增 09 后台任务`。
- 不要 force push，不要改写已推送的历史，否则增量同步会失效。
- 不要提交 `.claude/`（本地配置，含凭据）和 `.gitops_last_sync` 之外的同步状态文件。
- **未经用户确认，不要**：删除文章（删除文件即删除线上文章）、重命名或移动专栏目录、批量修改 frontmatter、推送到远端。

## 7. 写作风格

- 面向工程实践：先给结论，再讲问题和实现，最后讲取舍与边界。
- 使用真实的代码、目录和流程；避免“最佳”“完美”“终极”等夸张表述。
- 代码块标注语言；中文与英文、数字之间留空格。
