# Blog Content

[Blog FR](https://github.com/sumanin5/blog_fr) 的文章内容仓库。推送到 `main` 后，GitHub Webhook 通知博客后端拉取本仓库，并把变更的 Markdown 同步为线上文章。网站只单向导入，不回写或推送。

合并或推送到配置的内容分支后才会触发导入；导入成功且 `status: published` 的文章才会公开。新文章建议先用 `draft`。本文对应显式 slug / 中文目录版本，须先部署支持该协议的后端，再发布本分支的内容变更。

## 目录结构

```text
blog-content/
├── articles/                     # 「专栏专题」
│   └── <专栏目录>/
│       ├── index.md              # 专栏元数据与介绍
│       ├── 01-<文章>.md          # 文章
│       └── assets/               # 图片、图标（也可以叫 media/）
├── ideas/                        # 「想法流」
│   └── <专栏目录>/ ...
├── .agent/skills/                # 写作辅助技能（frontmatter、专栏元数据）
├── AGENTS.md / CLAUDE.md         # AI 助手的写作与上传规则
└── README.md
```

- 一级目录决定内容类型，只能是 `articles` 或 `ideas`。
- 二级目录是专栏，可以使用中文名；分类地址取自该目录 `index.md` 的显式 slug，与目录名解耦。
- 文章只放在专栏目录的第一层；子目录（`assets/`、`media/`）只放图片。

各类型下的专栏清单见 [articles/README.md](articles/README.md) 和 [ideas/README.md](ideas/README.md)。

## 哪些文件会被同步

| 文件 | 处理方式 |
| --- | --- |
| `<类型>/<专栏>/*.md`、`*.mdx` | 同步为文章 |
| `<类型>/<专栏>/index.md` | 同步为专栏元数据与介绍 |
| 文章或 `index.md` 引用的本地图片 | 自动上传到媒体库 |
| `README`、`AGENTS`、`CLAUDE`、`LICENSE`、`CONTRIBUTING`、`CHANGELOG`（任意层级，不区分大小写） | **跳过** |
| 以 `.` 开头的文件或目录（`.agent/`、`.claude/`、`.gitops_last_sync`） | **跳过** |

> 本版本会拒绝根目录或过深目录中的文章 Markdown；说明类文档请使用上面的文件名。README、AGENTS 的提交仍会触发 GitHub push webhook，但后端会跳过这些文件；它们不会被导入成文章。

## 写作与上传流程

### 首次准备

在当前内容仓库根目录执行（需要 Python 和 Git）：

```powershell
python -m pip install -r tools/requirements.txt
python tools/install-hooks.py
python tools/content_contract.py
```

每次新克隆都要安装本地钩子。重复文章 slug、同板块重复分类 slug、缺少 slug 或索引、路由冲突都会阻止提交。钩子可以绕过，因此远端还应将 CI 的 `Content contract` 配置为必需合并检查。仅有工作流文件不等于已经配置分支保护。

### 新建文章

例如新建 `ideas/比特漫笔/我的第一篇文章.md`，图片放在同一分类的 `assets/` 中：

```text
ideas/比特漫笔/
├── index.md
├── 我的第一篇文章.md
└── assets/我的第一篇文章-封面.png
```

文章内容示例：

```markdown
---
title: 我的第一篇文章
slug: my-first-article
author: tomyi
status: draft
summary: 一句话介绍文章。
cover: assets/我的第一篇文章-封面.png
tags:
  - 随笔
---

# 我的第一篇文章

这里写正文。
```

`author` 必须是网站上已有的用户名。`slug` 使用小写英文字母、数字和连字符，文章在整个仓库内唯一；文件名和目录名可以用中文。现有文章改名或移动时保留原 slug，避免改变链接。

检查并提交（示例只添加本篇文章和封面）：

```powershell
python tools/content_contract.py
git add -- "ideas/比特漫笔/我的第一篇文章.md" "ideas/比特漫笔/assets/我的第一篇文章-封面.png"
python tools/content_contract.py --staged
git commit -m "feat(content): 新增我的第一篇文章"
```

本地提交不会上传，也不会改变网站。确认要发布时，将 `status` 改为 `published`，提交后再推送分支、通过 PR 合并到配置的内容分支。检查后台同步结果；同步失败不代表已经发布成功。本文仅描述操作步骤，不自动执行 push。

### 单篇文章封面与分类封面

| 写在哪里 | 作用 |
| --- | --- |
| 分类 `index.md` 的 `cover` | 分类卡片 / 分类页封面 |
| 文章 `.md` 的 `cover` | 文章列表卡片、文章详情页、分享预览图 |
| 正文 `![说明](assets/图片.png)` | 正文内图片，不会自动成为封面 |

文章没有填写 `cover` 时不显示文章封面，也不会自动继承分类封面。路径相对于当前 Markdown 文件，必须指向实际存在的文件；新文章无需手填 `cover_media_id`。

导入时，后端读取图片并按内容哈希复用或上传至媒体库，将关联的 `cover_media_id` 保存到文章行；列表读取 medium 缩略图，详情和分享读取 xlarge，SVG 使用原图。封面不需要再插入正文，避免显示两次。分享卡片还取决于分享平台抓取与缓存，前端部署须设置 `NEXT_PUBLIC_SITE_URL=https://ty1547.com`，用于生成完整图片地址。

若历史文章同时有有效 `cover_media_id`，它优先于 `cover`；要替换该文章封面，应移除这个旧 ID，再填写新图片路径。不要批量清理其他历史身份字段。

替换图片建议使用新文件名，并修改文章的 `cover` 路径一起提交。仅修改图片字节、没有修改任何 Markdown 时，普通增量导入不会自动重导引用它的全部文章，需要手动重导或全量对账。

### 修改、下线与新增分类

- 修改文章：编辑原文件并保留 slug，检查后提交。
- 下线文章：改为 `status: draft`；删除源文件可能删除数据库中的文章，请谨慎使用。
- 新建分类：在中文分类目录中创建 `index.md`，至少填写 `title` 和显式 `slug`，完整模板见 [AGENTS.md](AGENTS.md)。分类 slug 在同一板块内唯一。
- 图片与 Markdown 一起提交；后台编辑只修改数据库，后续源文件导入可能覆盖它。
- 日常使用增量导入；初始化、手动修复或失败后的对账会使用全量同步。两者都只从仓库进入数据库，不回写源文件。

## 相关文档

- [AGENTS.md](AGENTS.md)：文章、专栏、图片的上传格式规则
- [.agent/skills/blog-post-frontmatter](.agent/skills/blog-post-frontmatter/SKILL.md)：文章 frontmatter 字段详解
- [.agent/skills/category-metadata](.agent/skills/category-metadata/SKILL.md)：专栏 `index.md` 字段详解

## 快速检查

`python -m pip install -r tools/requirements.txt`

`python tools/install-hooks.py`

`python tools/content_contract.py`

提交钩子检查暂存区，CI 检查提交快照，网站导入前再次检查；不连接数据库或上传图片。
