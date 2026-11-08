---
name: blog-post-frontmatter
description: 生成、校验和更新博客文章的 YAML Frontmatter 元数据（以同步代码的实际解析规则为准）。当创建新文章或优化 SEO 信息时使用。
---

# Blog Post Frontmatter 规范

此技能定义了本工作区内博客文章（Markdown）必须遵守的 Frontmatter 元数据标准。

## 适用场景

- 创建新的博客文章 (`.md`) 时。
- 需要补充或优化文章的 SEO 信息（meta_title, description）时。
- 检查文章格式是否合规时。

## 核心规则（以同步代码的实际行为为准）

解析逻辑见主仓库 `backend/app/git_ops/components/metadata.py` 与 `processors/`。与仓库根目录 `AGENTS.md` 保持一致。

1. **author 和 slug 必需**：作者为已有用户名；slug 显式稳定、全仓库文章唯一，移动目录时保留。
2. **`status` 不写默认是草稿**：只接受 `draft` / `published`（不区分大小写），其他值同步报错。要上线必须写 `status: published`。
3. **其余字段都可选**，不写时的行为见下表。
4. **单向导入**：后端不回填、不写文件、不推送。新文章不要添加数据库 ID；分类身份取自同目录 index.md 的显式 slug。
5. **内容类型和专栏由目录决定**：`articles/<专栏>/` 或 `ideas/<专栏>/`，frontmatter 里的 `type` 不起作用，不用写。

## 最小模板（想法流常用）

```yaml
---
author: tomyi
slug: my-stable-article
status: published
---
```

标题取文件名，发布时间取同步时间，摘要从正文自动截取。

## 推荐模板（专栏系列文章）

```yaml
---
title: "09-文章主标题"
author: tomyi
slug: my-stable-article
status: published
summary: "80–120 字的中文摘要，用于列表卡片与搜索结果。"
tags:
  - FastAPI
  - Python
meta_title: "SEO 标题 | 专栏名"
meta_description: "SEO 描述，包含核心关键词。"
meta_keywords: "关键词1, 关键词2"
cover: assets/article-cover.png
is_featured: false
---
```

## 字段详解

| 字段 | 必需 | 不写时 / 说明 |
| :--- | :--- | :--- |
| `author` | **是** | 同步报错。用户名或用户 UUID |
| `status` | 建议 | 默认 `draft`（仅后台可见）。只接受 `draft` / `published` |
| `title` | 否 | 使用文件名（不含扩展名）。专栏系列建议带 `NN-` 编号前缀 |
| `date` | 否 | 按同步时间发布；未来时间即定时发布。支持 `2026-12-01`、`2026-12-01 09:00:00`、`2026-12-01T09:00:00+08:00`，**不支持** `2026/12/01` |
| `summary` / `excerpt` | 否 | 二者等价，写一个即可；不写则从正文自动截取。也可用 `description` |
| `tags` | 否 | YAML 列表或逗号分隔字符串。尽量复用已有标签 |
| `cover` | 否 | 相对于当前文件的图片路径（如 `assets/x.png`），会自动上传；也可写媒体库 UUID。**不要写不存在的裸文件名**，不存在的源图片不会通过同名模糊匹配代替 |
| `meta_title` / `meta_description` / `meta_keywords` | 否 | 可选的 SEO 字段 |
| `is_featured` | 否 | 默认 `false`，是否首页推荐 |
| `allow_comments` | 否 | 默认 `true` |
| `use_server_rendering` | 否 | 默认 `true` |
| `enable_jsx` | 否 | 默认 `false`；只有使用 JSX 组件时才设为 `true`，数学公式与 Mermaid 不需要 |
| `slug` | **是** | 显式稳定 URL 标识，全仓库文章唯一，不回填 |
| `author_id` / `category` / `category_id` / `cover_media_id` | 历史兼容 | 新文章使用作者名、分类索引 slug 与相对媒体路径；后端不回填 |

`hidden`、`post_sort`、`sort`、`icon` 是专栏 `index.md` 的字段，写在文章里不起作用，见 `category-metadata` 技能。

## 分类索引（Category Index）的特殊处理

### 什么是分类索引？

每个分类目录下的 `index.md` 文件是分类索引，用来展示该分类的说明、覆盖图和内容组织。

**示例路径**：
- `articles/FastAPI工程化实践/index.md`
- `articles/Django风格指南/index.md`

### 分类索引中的链接处理

#### 链接自动转换 ✅

分类索引中的**相对链接会被自动转换为绝对 URL**：

```markdown
# ❌ 不要这样写（虽然也能工作，但依赖自动转换）
- [FastAPI 01](./01-how-to-structure-fastapi-projects.md)

# ✅ 系统会自动将上面的链接转换为：
- [FastAPI 01](/posts/articles/how-to-structure-fastapi-projects-ee32vp)
```

#### 工作原理

当 git 内容同步时：
1. 系统扫描 `index.md` 并识别为分类索引
2. `handle_category_sync()` 处理该文件
3. **ContentProcessor** 自动转换其中的相对链接为绝对 URL
4. 转换后的内容保存到数据库的分类描述字段

### 最佳实践

#### ✅ 推荐方式：使用相对链接

在分类索引中使用**相对路径**，让系统自动转换：

```markdown
## 系列文章

### 基础篇
- [第一篇介绍](./01-introduction.md)
- [第二篇高级](./02-advanced.md)

### 进阶篇
- [优化技巧](./03-optimization.md)
```

**优点**：
- 链接在文件系统中有效，开发时易于验证
- 系统转换相对文章链接；作者必须显式维护稳定 slug
- 即使重新组织目录结构，相对路径仍有效

#### ❌ 不推荐方式：硬编码绝对 URL

```markdown
# 不要这样做（slug 可能变化）
- [文章](./posts/articles/some-random-slug-abc123)
```

### 故障排查

**问题**：分类索引中的链接显示为相对路径而不是绝对 URL

**可能原因**：
1. 同步脚本还未运行
2. 链接格式不符合 Markdown 标准（如缺少 `]`）
3. 目标文件不在数据库中

**解决方案**：
1. 确保所有文章都已同步到数据库
2. 检查链接格式是否为 `[text](./file.md)`
3. 在主仓库 `blog_fr` 中运行 `scripts/sync-posts.sh` 或在后台触发全量同步
