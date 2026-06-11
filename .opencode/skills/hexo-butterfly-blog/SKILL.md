---
name: hexo-butterfly-blog
description: 使用 Hexo + Butterfly 主题从零搭建现代科技风个人技术博客，并部署到 GitHub Pages。适用于需要创建新博客、配置主题、设置 CI/CD 部署的场景。当用户需要搭建博客、配置 Hexo、或部署技术博客时使用。
---

# Hexo + Butterfly 技术博客搭建指南

本 Skill 指导如何使用 Hexo 框架和 [Butterfly 主题](https://butterfly.js.org/) 从零搭建一个现代科技风个人技术博客，并通过 GitHub Actions 自动部署到 GitHub Pages。

## 环境要求

- Node.js >= 20（推荐 22+）
- npm >= 10
- Git
- GitHub 账号（用于部署）

## 第一步：项目初始化

**1.1 安装 Hexo CLI**

```bash
npm install -g hexo-cli
```

**1.2 初始化项目**

```bash
hexo init <project-name>
cd <project-name>
npm install
```

**1.3 安装核心依赖**

```bash
npm install hexo-theme-butterfly
npm install hexo-renderer-pug
npm install hexo-renderer-stylus
npm install hexo-generator-search
npm install hexo-generator-sitemap
npm install hexo-generator-feed
npm install hexo-abbrlink
npm install hexo-wordcount
npm install hexo-filter-gitcalendar
```

**1.4 删除默认文件**

```bash
# 删除默认主题配置和示例文章
rm _config.landscape.yml
rm source/_posts/hello-world.md
```

---

## 第二步：配置 _config.yml

以下是站点基础配置的关键部分：

```yaml
# Site
title: <用户名>'s Blog
subtitle: 'Your subtitle here'
description: '博客描述'
keywords:
  - 关键词1
  - 关键词2
author: <用户名>
language: zh-CN
timezone: 'Asia/Shanghai'

# URL
url: https://<username>.github.io
permalink: :year/:month/:day/:title/
permalink_defaults:
pretty_urls:
  trailing_index: false
  trailing_html: false

# Writing
post_asset_folder: true  # 启用文章独立资源目录

# Category & Tag
category_map:
  # 映射中文分类到 URL-friendly 名称
  AI Infra: ai-infra
  Java: java
  LeetCode: leetcode

# Theme
theme: butterfly
```

---

## 第三步：配置蝴蝶主题 _config.butterfly.yml

### 导航菜单

```yaml
menu:
  首页: / || fas fa-home
  归档: /archives/ || fas fa-archive
  标签: /tags/ || fas fa-tags
  分类: /categories/ || fas fa-folder-open
  项目: /projects/ || fas fa-rocket
  友链: /friends/ || fas fa-link
  关于: /about/ || fas fa-heart
```

### 代码块配置

```yaml
code_blocks:
  theme: darker  # 必须使用 'darker'，其他值可能导致 CSS 编译错误
  macStyle: true
  height_limit: 450
  word_wrap: true
  copy: true
  language: true
```

### 图像配置

```yaml
# favicon 文件必须与配置路径一致（.ico 或 .png）
favicon: /img/favicon.ico

# GitHub 头像的正确 URL 格式
avatar:
  img: https://github.com/<username>.png
  effect: true
```

### 评论系统 (Giscus)

Giscus 基于 GitHub Discussions，无需后端。

1. 在 GitHub 仓库中启用 Discussions
2. 访问 https://giscus.app 生成配置
3. 将生成的 `repo_id` 和 `category_id` 填入配置

```yaml
comments:
  use: Giscus
  text: true
  lazyload: false
  count: true
  card_post_count: false

giscus:
  repo: <username>/<username>.github.io
  repo_id: <giscus-app-生成>
  category_id: <giscus-app-生成>
  light_theme: light
  dark_theme: dark
```

### 数学公式支持

```yaml
math:
  use: katex
  per_page: false
  katex:
    copy_tex: true
```

---

## 第四步：创建页面

### 必需页面

```bash
# 创建页面文件
mkdir -p source/about source/projects source/friends
mkdir -p source/tags source/categories
mkdir -p source/_posts/ai-infra source/_posts/java source/_posts/leetcode
```

**About 页面 (`source/about/index.md`)**

```markdown
---
title: 关于我
date: 2025-01-01 00:00:00
type: "about"
comments: false
---

## 你好，我是 <用户名>

欢迎来我的个人博客！这里是我分享技术、记录学习的地方。

## 技术栈

- AI Infrastructure
- Java & Distributed Systems
- Algorithms & Data Structures

## 联系我

- GitHub: [github.com/<username>](https://github.com/<username>)
- Email: your-email@example.com
```

**项目页面 (`source/projects/index.md`)**

```markdown
---
title: 项目展示
date: 2025-01-01 00:00:00
type: "projects"
comments: false
---

<div class="project-container">

## 项目列表

<div class="project-card">

### 项目名称
<div class="project-desc">项目描述</div>

<div class="project-tags">

`Python` `PyTorch` `CUDA`

</div>

[GitHub](https://github.com/<username>) | [文档](#)

</div>
</div>

<style>
.project-card {
  background: var(--anzhiyu-card-bg);
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  border: 1px solid var(--light-grey);
  transition: all 0.3s ease;
}
.project-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0,0,0,0.1);
}
.project-desc {
  color: var(--font-color);
  opacity: 0.8;
  margin: 0.5rem 0;
}
</style>
```

**友链页面 (`source/friends/index.md`)**

```markdown
---
title: 友情链接
date: 2025-01-01 00:00:00
type: "links"
comments: true
---

## 我的友链

<div class="flink-list">
<div class="flink-card">

**[<用户名>'s Blog](https://<username>.github.io)**

![avatar](https://github.com/<username>.png)

一句话介绍

</div>
</div>

<style>
.flink-card {
  display: inline-block;
  background: var(--anzhiyu-card-bg, #fff);
  border: 1px solid var(--light-grey, #eee);
  border-radius: 12px;
  padding: 1rem 1.5rem;
  margin: 0.5rem;
  text-align: center;
  transition: all 0.3s ease;
}
.flink-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0,0,0,0.1);
}
.flink-card img {
  width: 60px;
  height: 60px;
  border-radius: 50%;
}
</style>

---

## 申请友链

按以下格式提供信息：

- name: 你的博客名称
- link: 你的博客链接
- avatar: 你的头像链接
- descr: 一句话简介
```

**标签和分类页面 (`source/tags/index.md` 和 `source/categories/index.md`)**

```markdown
---
title: 标签
date: 2025-01-01 00:00:00
type: "tags"
comments: false
---
```

```markdown
---
title: 分类
date: 2025-01-01 00:00:00
type: "categories"
comments: false
---
```

### 示例文章格式

```markdown
---
title: 文章标题
tags:
  - AI Infra
  - 分布式训练
categories:
  - AI Infra
cover: 'https://images.unsplash.com/photo-xxx?w=1280'
description: 文章摘要
katex: false    # 是否启用 KaTeX 数学公式
mermaid: false  # 是否支持 Mermaid 图表
mathjax: false  # 是否支持 MathJax
top_img:        # 留空则使用默认
date: 2025-05-24 10:00:00
---

# 标题

正文内容...
```

---

## 第五步：GitHub Actions 自动部署

**5.1 在项目根目录创建 `.github/workflows/deploy.yml`**

```yaml
name: Deploy Hexo Blog

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build site
        run: npx hexo generate

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

**5.2 初始化 Git 仓库并推送**

```bash
git init
git add -A
git commit -m "Initial commit"
git remote add origin git@github.com:<username>/<username>.github.io.git
git push -u origin main
```

**5.3 在 GitHub 仓库中启用 Pages**

- Settings > Pages > Build and deployment
- Source 选择 "GitHub Actions"（不需要选 gh-pages 分支）

---

## 常见问题与解决方案

### 1. favicon 404 错误

**问题**: 浏览器控制台报 favicon 404
**原因**: 配置的 `favicon` 路径与实际文件扩展名不匹配（如配置为 `.png` 但实际是 `.ico`）
**解决**: 确保 `/img/` 目录下的文件扩展名与配置一致

```yaml
# 如果实际文件是 .ico
favicon: /img/favicon.ico
```

### 2. 头像无法显示

**问题**: 头像图片加载 404
**原因**: 使用了 `https://avatars.githubusercontent.com/u/<username>` 格式，但此 API 需要用户 ID 而非用户名
**解决**: 使用以下格式获取 GitHub 头像

```yaml
avatar:
  img: https://github.com/<username>.png
```

### 3. CSS 编译错误：$highlight-tools has no property .bg-color

**问题**: `npx hexo generate` 报错 Stylus 编译失败
**原因**: `code_blocks.theme` 配置值无效，`darker` 是唯一支持的值
**解决**: 将 `code_blocks.theme` 改为 `darker`

```yaml
code_blocks:
  theme: darker  # 必须使用 this 值
```

### 4. Footer.nav 配置格式错误

**问题**: Footer 导航链接无法正常渲染
**原因**: `footer.nav` 需要嵌套的数组结构，而不是扁平列表
**正确格式**:

```yaml
footer:
  nav:
    - content:
        - title: 链接
          item:
            - title: GitHub
              url: https://github.com/<username>
            - title: RSS
              url: /atom.xml
```

### 5. 部署后访问不到新内容

**问题**: 访问显示 "Not Found" 或旧版本
**解决**:
- 检查 GitHub Actions 部署状态（绿色勾号）
- 清除浏览器缓存（Ctrl+F5 强制刷新）
- 确认 Pages 的 Source 设置为 "GitHub Actions"

---

## 构建与验证命令

```bash
# 清理并重新生成
npx hexo clean && npx hexo generate

# 本地预览（默认 http://localhost:4000）
npx hexo server

# 部署前验证
npx hexo clean
npx hexo generate  # 确保没有错误输出
```

---

## 下一步（部署后）

1. **配置 Giscus 评论**: 启用 Discussions 后，到 https://giscus.app 获取配置并更新 `_config.butterfly.yml`
2. **添加 Google Analytics / Microsoft Clarity**: 在 `_config.butterfly.yml` 中填入 tracking ID
3. **申请 HTTPS 证书**: GitHub Pages 已自带，但需确认域名设置正确
4. **自定义域名**: 在仓库根目录放入 `CNAME` 文件，并在 DNS 服务商配置 CNAME 记录
5. **提交搜索引擎收录**: 使用 `sitemap.xml` 提交 Google Search Console 和 Bing Webmaster

---

## 快速参考

| 命令 | 说明 |
|------|------|
| `npx hexo new post "文章标题"` | 创建新文章 |
| `npx hexo clean` | 清除缓存和 public 目录 |
| `npx hexo generate` | 生成静态站点 |
| `npx hexo server` | 启动本地服务器（端口 4000） |
| `npx hexo server -p 5000` | 指定端口启动 |
| `git add -A && git commit -m "..." && git push` | 提交更改触发 CI |

---

## 配置文件模板速查

- `_config.yml`: 站点基础配置（标题、URL、时区、分类映射）
- `_config.butterfly.yml`: 主题配置（导航、代码块、评论、SEO、样式）
- `.github/workflows/deploy.yml`: GitHub Actions 自动部署流水线
- `package.json`: 依赖项（hexo, hexo-theme-butterfly, hexo-renderer-pug, hexo-renderer-stylus）
- `source/about/index.md`: 关于页面模板
- `source/projects/index.md`: 项目展示页面模板
- `source/friends/index.md`: 友链页面模板
- `source/tags/index.md`: 标签云页面
- `source/categories/index.md`: 分类页面

---

## 关键约束与注意事项

1. **代码块主题**: 只能使用 `code_blocks.theme: darker`，其他值会导致 CSS 编译失败
2. **favicon 扩展名**: 配置路径必须与 source/img/ 下的实际文件一致
3. **GitHub 头像 URL**: 必须使用 `https://github.com/<username>.png` 格式
4. **Footer.nav 格式**: 必须是嵌套数组，不能是扁平列表
5. **Pages 部署方式**: 选择 "GitHub Actions" 作为 source，而非 "gh-pages branch"
6. **Git 初始推送**: 可能需要使用 `--force-with-lease` 覆盖仓库初始内容
7. **Giscus**: 需要先启用 GitHub Discussions，生成配置后才能使用评论功能
