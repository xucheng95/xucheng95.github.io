---
title: Hexo 使用指南
date: 2026-08-19 20:37:00
tags:
- Hexo
categories: Hexo
mathjax: false
---

> 本文基于当前博客项目的实际配置编写，作为自己的操作手册。环境：Hexo 8.1.2 + Butterfly 5.7.0。

## 1. 项目结构

本仓库是**双分支**结构：

| 分支 | 内容 | 用途 |
|---|---|---|
| `hexo` | 全部源码：`_config.yml`、`_config.butterfly.yml`、`source/_posts/`、`themes/butterfly/` | **日常写作都在这个分支** |
| `main` | 构建产物（`public/` 的内容） | GitHub Pages 部署目标，一般不用手动碰 |

```
项目根目录（hexo 分支）
├── _config.yml               # Hexo 主配置（站点信息、URL、部署）
├── _config.butterfly.yml     # Butterfly 主题配置（菜单、配色、评论等）
├── package.json              # 依赖清单（hexo 8.1.2、butterfly 5.7.0）
├── source/
│   ├── _posts/               # ★ 文章都放这里（.md 文件）
│   ├── categories/index.md   # 分类页
│   ├── tags/index.md         # 标签页
│   └── img/                  # 站点图片（头像、首页图等）
├── scaffolds/                # 新文章模板
├── themes/butterfly/         # 主题源码（5.7.0，一般不动）
└── public/                   # 构建产物（gitignore 忽略，由 hexo generate 生成）
```

## 2. 环境准备

```bash
# 安装依赖（已装过可跳过；第一次或换机器时执行）
npm ci          # 按 package-lock.json 精确安装
```

> ⚠️ **本机 npm 缓存问题**：`~/.npm` 目录存在 root 所有权的遗留文件，npm 默认缓存写入会报 `EPERM`。解决方案：
> 1. 一劳永逸：`sudo chown -R 501:20 ~/.npm`
> 2. 或临时绕开：所有 npm 命令加 `--cache "$PWD/.npm-cache"`（本仓库已把 `.npm-cache/` 加入 gitignore）

> ⚠️ **git 身份**：本仓库已配置为 `Xucheng95 <xuchengsiyou@163.com>`（仓库级 `.git/config`），提交自动使用，无需每次设置。

> ⚠️ **SSH 推送**：本机默认 SSH key 关联的是 `feima95` 账号，推送本仓库会无权限。已在 `~/.ssh/config` 配置别名 `github-xucheng95`（使用专用 key），**本仓库所有 git 操作走这个别名**，无需手动处理。

## 3. 日常写作流程

### 3.1 新建文章

```bash
# 方式一：脚手架（推荐）
npx hexo new "文章标题"

# 方式二：手动创建
# 在 source/_posts/ 下新建 标题.md，参考下面的模板
```

文章的 front matter 模板：

```markdown
---
title: 文章标题
date: 2026-08-19 20:37:00
tags:
- 标签1
- 标签2
categories: 分类名
mathjax: true   # 含数学公式时设为 true
---
```

### 3.2 本地预览

```bash
npx hexo server
# 浏览器打开 http://localhost:4000
# 保存文章后自动刷新，实时看效果
```

### 3.3 构建与部署

```bash
npx hexo clean      # 清理缓存（db.json、public/）
npx hexo generate   # 构建，生成 public/
npx hexo deploy     # 部署：把 public/ 推送到 main 分支 → GitHub Pages 自动更新
```

**完整流程**：写完文章 → 本地预览确认 → `clean + generate` → `deploy`。

## 4. 常用命令速查

| 命令 | 作用 |
|---|---|
| `npx hexo new "标题"` | 新建文章 |
| `npx hexo server` | 本地预览（默认 http://localhost:4000） |
| `npx hexo clean` | 清理缓存 |
| `npx hexo generate` | 构建到 `public/` |
| `npx hexo deploy` | 部署到 GitHub Pages（main 分支） |
| `git push origin hexo` | 推送源码改动（文章/配置） |

## 5. 写作注意事项

1. **数学公式**：用 KaTeX 语法，`$$...$$` 块级、`$...$` 行内；front matter 需 `mathjax: true`。
2. **文章配图**：`_config.yml` 开启了 `post_asset_folder: true`，图片放在 `source/_posts/文章名/` 目录下，正文用相对路径引用。
3. **标签与分类**：从文章 front matter 自动生成，无需手动维护 categories/tags 页面。
4. **首页空问题**：没有任何文章时首页不会生成（访问 `/` 404），写一篇即恢复，属正常。
5. **部署走别名**：`_config.yml` 的 deploy 地址是 `git@github-xucheng95:...`，不要改回 `git@github.com`，否则推送会因权限被拒。

## 6. 常见问题

| 问题 | 解决 |
|---|---|
| 首页 404 | 没有文章，写一篇即可 |
| npm 报 EPERM | `sudo chown -R 501:20 ~/.npm` 或加 `--cache` 参数 |
| deploy 推不上去 | 确认 `github-xucheng95` 别名和 `~/.ssh/id_ed25519_xucheng95` 存在 |
| 公式不显示 | front matter 加 `mathjax: true`，确认用 `$$`/`$` 语法 |
