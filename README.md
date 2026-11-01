# Blog Content

[Blog FR](https://github.com/sumanin5/blog_fr) 的文章内容仓库。推送到 `main` 后，GitHub Webhook 通知博客后端拉取本仓库，并把变更的 Markdown 同步为线上文章；后台修改的元数据会以 `chore: sync metadata from database` 提交写回这里。

**推送即发布**：每次 push 都会直接改变线上内容。

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
- 二级目录是专栏，目录名就是专栏 slug。
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

> 除上表的说明文件名外，任何 `.md` 都会被当成文章，包括放在仓库根目录或子目录里的笔记。说明类文档请使用上面的文件名。

## 写作与上传流程

1. `git pull`，拿到后端写回的元数据提交。
2. 在 `articles/<专栏>/` 或 `ideas/<专栏>/` 下新建或修改文章，格式见 [AGENTS.md](AGENTS.md)。
3. 提交并推送到 `main`。
4. 几秒内文章出现在后台；公开页面的缓存最多 5 分钟后刷新。
5. 再 `git pull` 一次，拿到后端回填的 `slug`、`category_id` 等字段。

## 相关文档

- [AGENTS.md](AGENTS.md)：文章、专栏、图片的上传格式规则
- [.agent/skills/blog-post-frontmatter](.agent/skills/blog-post-frontmatter/SKILL.md)：文章 frontmatter 字段详解
- [.agent/skills/category-metadata](.agent/skills/category-metadata/SKILL.md)：专栏 `index.md` 字段详解
