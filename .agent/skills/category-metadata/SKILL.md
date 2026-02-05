---
name: category-metadata
description: 专门用于创建、校验和维护博客分类目录下的 index.md 元数据文件。当需要新增分类、调整分类图标/封面或更新分类描述时使用。
---

# Category Metadata (分类元数据管理)

此技能用于指导如何正确编写分类目录下的 `index.md` 文件。这些文件会被系统自动扫描并同步到数据库。

## When to use this skill (何时使用)

- 在 `content/articles/` 或 `ideas/` 下创建新分类目录时。
- 需要修改分类的显示名称（title）、图标（icon）、排序（sort）或封面（cover）时。
- 需要通过 `hidden: true` 隐藏某个分类时。
- 编辑分类的详细 Markdown 介绍（description）时。

## How to use it (如何执行)

### 1. 文件定位与自动推导

- **文件路径**: 必须位于分类目录的正下方，文件名为 `index.md`。
- **Post Type 自动推导**:
  - 路径包含 `articles/` → `post_type: ARTICLES`
  - 路径包含 `ideas/` → `post_type: IDEAS`
- **Slug 自动推导**: 使用目录名。例如 `content/articles/tech-share/` 推导出 `slug: tech-share`。

### 2. Frontmatter 编写规范

```yaml
---
# ========== 必填字段 ==========
title: "显示名称"

# ========== 可选字段 ==========
hidden: false # 是否隐藏（默认 false）。映射至 is_active (hidden: true => is_active: false)
cover_media_id: "UUID" # 优先使用的封面 UUID
cover: "image.png" # 降级使用的封面文件名或路径（自动匹配媒体库）
icon: "🚀" # 图标：支持 emoji (< 10字符) 或 文件路径/UUID (≥ 10字符)
sort: 1 # 排序顺序（默认 0，越小越靠前）
post_sort: "published_at_desc" # 文章列表默认排序（可选）
---
# 正文标题 (可选)
正文内容将作为该分类的详细描述存储到数据库中，支持 Markdown。
```

### `post_sort` / `post_sort_order` - 文章列表排序

- **类型**: 字符串 (枚举值)
- **必填**: 否
- **默认值**: `published_at_desc`
- **说明**: 定义该分类下文章列表的默认排序列
- **可选值**:
  - `published_at_desc`: 按发布时间**倒序**（最新在前）
  - `published_at_asc`: 按发布时间**正序**（最旧在前）
  - `title_asc`: 按标题 **A -> Z** 正序
  - `title_desc`: 按标题 **Z -> A** 倒序
- **示例**: `post_sort: "title_asc"`

### 3. 字段特殊逻辑校验

- **图标 (icon)**:
  - 如果字符串长度 **< 10**：视为 **Emoji**，映射至 `icon_preset`。
  - 如果字符串长度 **≥ 10**：视为 **路径或 UUID**，映射至 `icon_id`。
- **排序 (sort/order)**: 系统同时兼容 `sort` 和 `order` 字段，最终映射为 `sort_order`。

## Reference Templates (参考模板)

### 模板 A：标准配置 (推荐)

```yaml
---
title: "技术分享"
icon: "💻"
sort: 1
---
这里是关于前端、后端及 DevOps 的技术沉淀。
```

### 模板 B：带封面的配置

```yaml
---
title: "摄影集"
cover: "landscape.jpg"
icon: "📷"
---
生活不只是代码，还有远方的风景。
```

### 模板 C：使用 SVG 图标

```yaml
---
title: "设计资源"
icon: "assets/icons/design.svg"
---
各种开发与设计相关的资产。
```

## Best Practices (最佳实践)

1. **正文即描述**: 不要只写 Frontmatter，正文描述对于分类详情页的 SEO 和用户体验非常重要。
2. **Slug 唯一性**: 虽然 Slug 是推导的，但请确保在同一 Post Type 下目录名不要冲突。
3. **Emoji 简化**: 推荐优先使用 Emoji 作为 `icon`，因为它渲染速度快且无需加载额外资源。
4. **隐藏分类**: 对于正在筹备或不想公开的分类，使用 `hidden: true` 而非删除目录。
