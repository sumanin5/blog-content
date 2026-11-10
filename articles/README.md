# articles · 专栏专题

对应站点导航「专栏专题」（`/posts/articles/categories`）。这里放成体系的技术系列，每个专栏是一组有顺序的文章。

## 专栏

| 目录 | 名称 | 分类 slug | 文章数 | 状态 |
| --- | --- | --- | --- | --- |
| [FastAPI工程化实践](FastAPI工程化实践/index.md) | FastAPI 工程化实践 | `fastapi-engineering-practice` | 8 | 公开 |
| [Django风格指南](Django风格指南/index.md) | Django 风格指南 (HackSoft) | `django-styleguide` | 8 | 公开 |
| [Next.js缓存机制](Next.js缓存机制/index.md) | Next.js 缓存机制全解析 | `next-cache` | 6 | 公开 |
| [技术杂记](技术杂记/index.md) | Tech | `tech` | 1 | 隐藏（`hidden: true`） |

> 目录名可以使用中文；分类 URL 由各目录 `index.md` 的显式 `slug` 决定，目录改名不改变分类地址。每个分类只保留一个源目录。

## 本类型约定

- 每个专栏都是系列：文件名 `NN-<英文短名>.md`，`title` 以同样的 `NN-` 开头，`index.md` 设 `post_sort: title_asc`。
- `index.md` 正文写系列简介和按篇章分组的文章列表，链接用 `./NN-xxx.md` 相对路径。
- 图片放在专栏的 `assets/` 目录。

新增专栏或文章的完整格式见仓库根目录的 [AGENTS.md](../AGENTS.md)。新增专栏后请同步更新上表。
