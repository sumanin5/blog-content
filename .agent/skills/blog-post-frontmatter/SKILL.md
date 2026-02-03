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
