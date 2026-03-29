---
name: blog-post-frontmatter
description: 生成、校验和更新博客文章的 YAML Frontmatter 元数据。当创建新文章或优化 SEO 信息时使用。
---

# Blog Post Frontmatter 规范

此技能定义了本工作区内博客文章（Markdown）必须遵守的 Frontmatter 元数据标准。

## 适用场景

- 创建新的博客文章 (`.md`) 时。
- 需要补充或优化文章的 SEO 信息（meta_title, description）时。
- 检查文章格式是否合规时。

## 核心规则与注意事项

1.  **自动字段**：`author_id`, `category_id`, `date` 等字段通常由后台数据库自动回填或管理，**在手动创建时通常只需留空或移除**，除非你需要明确覆盖。
2.  **必需字段**：`title`, `author`, `tags`, `status` 是必须明确填写的。
3.  **语言要求**：除非另有说明，摘要（summary/excerpt）和 SEO 描述应使用**中文**。

## 标准 Frontmatter 模板

在创建新文章时，请使用以下模板：

```yaml
---
# ========== 核心元数据 (必填) ==========
title: "文章主标题"
author: "tomyi" # 默认作者
status: "published" # 状态：published (发布) 或 draft (草稿)
is_featured: false # 是否推荐：true/false

# ========== 内容概览 (必填) ==========
# Summary 是用于列表页展示的精简摘要，建议 80-120 字
summary: "这里写文章的简要概述，吸引读者点击。"
# Excerpt 是备用字段，通常与 summary 保持一致或更短
excerpt: "同 summary"

# ========== SEO 优化 (强烈推荐) ==========
slug: "custom-url-slug" # URL 友好的英文标识，如 "deep-learning-intro"
meta_title: "SEO 标题 | 博客名称" # 通常包含关键词，比主标题更具搜索吸引力
meta_description: "SEO 描述，用于搜索引擎结果页（SERP），应包含核心关键词。"
meta_keywords: "关键词1, 关键词2, 强化学习, AI"

# ========== 分类与标签 ==========
# 以 YAML 列表格式书写
tags:
  - 技术分享
  - 架构设计
  - 随笔

# ========== 系统/高级设置 (按需) ==========
allow_comments: true # 是否允许评论
type: "ARTICLE" # 内容类型
cover: "image.png" # 封面图路径 (可选)

# 注意：以下字段通常由系统生成，如有变动请谨慎修改
# date: "2024-xx-xx"
# author_id: "..."
# category_id: "..."
---
```

## 字段详解

| 字段     | 必填 | 说明                                                                                                        |
| :------- | :--- | :---------------------------------------------------------------------------------------------------------- |
| `title`  | 是   | 文章显示的标题。                                                                                            |
| `slug`   | 否   | 文章的 URL 尾缀，建议使用小写英文连字符格式。如果不填，系统可能会自动根据标题生成（中文标题建议手动指定）。 |
| `status` | 是   | 默认为 `published`。如果不希望被看到请设为 `draft`。                                                        |
| `tags`   | 是   | 列表格式。请尽量复用已有的标签体系。                                                                        |
| `cover`  | 否   | 封面图片路径。可以是相对路径或 `uploads/` 路径。                                                            |
| `allow_comments` | 否 | 是否允许评论。默认 `true`。                                                                                |
| `type`   | 否   | 内容类型。通常为 `ARTICLE`。                                                                                |
| `hidden` | 否   | 是否隐藏。系统字段，通常由后台管理。                                                                        |
| `post_sort` | 否 | 内容排序方式。可选值：`title_asc`（标题升序）、`published_at_desc`（发布时间降序）等。系统字段。            |
| `category_id` | 否 | 分类 ID。系统自动生成，用于关联文章到分类。                                                                |
| `cover_media_id` | 否 | 封面媒体 ID。系统自动生成，用于关联封面图片。                                                              |

## 分类索引（Category Index）的特殊处理

### 什么是分类索引？

每个分类目录下的 `index.md` 文件是分类索引，用来展示该分类的说明、覆盖图和内容组织。

**示例路径**：
- `articles/fastapi-engineering-practice/index.md`
- `articles/django-styleguide/index.md`

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
- 系统会自动转换，无需手动维护 slug
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
3. 运行 `scripts/sync-posts.sh` 重新同步
