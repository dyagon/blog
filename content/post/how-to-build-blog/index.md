---
title: 从零手动搭建 Hugo Stack 主题博客：Submodule + YAML + GitHub Pages 全流程
slug: how-to-build-blog
date: 2026-03-05
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

这篇文章记录的是我和一个 AI 工具围绕 Hugo + `hugo-theme-stack` 折腾的一整段对话，经过整理之后，变成了一篇**从零手动搭建与掌控博客的技术笔记**。目标是：

- **不用官方 Starter Template**
- **用 Git Submodule 管理主题源码**
- **统一使用 YAML 配置**
- **支持暗色模式 + 多语言**
- **通过 GitHub Actions 部署到 GitHub Pages**
- **顺便避开几个 Stack 主题的典型坑**

如果你也想“handle 所有过程”，而不是直接用别人打包好的模板，这篇就是给你的。

---

## 一、Hugo Theme 的两条路：Submodule vs Hugo Modules

Hugo 主题主流有两种引入方式：

- **Git Submodule（传统方式）**
  - **特点**：主题完整源码就在 `themes/<theme-name>` 目录里，是一个独立 Git 仓库。
  - **优点**：能看 commit 历史、方便学习源码、适合重度定制。
  - **缺点**：需要自己管理 `themes/` 目录和 Submodule 更新。

- **Hugo Modules（基于 Go Modules 的方式）**
  - 在 `config/_default/module.toml`（或 yaml）里写：
    ```toml
    [[imports]]
        path = "github.com/CaiJimmy/hugo-theme-stack/v4"
    ```
  - **特点**：主题像依赖库一样从远程拉取，源码默认放在 `_vendor` 目录。
  - **优点**：项目根目录更干净，跟 Go 模块生态更统一。
  - **缺点**：主题不在 `themes/` 下，不直观；Git 历史也在 vendor 中被“打包”。

**重要结论：这两种方式是互斥的**：

- 用 **Submodule** 时：在 `config` 中用 `theme: "hugo-theme-stack"`，**不要再写 `[[imports]]`**。
- 用 **Hugo Modules** 时：用 `[[imports]]`，**就不要再在 `themes/` 里放主题，也不要写 `theme:`**。

本文选择的是：**Git Submodule + 可选 Hugo Modules vendoring（只用来本地化依赖）**。

---

## 二、初始化站点与引入 Stack 主题（Submodule 方式）

在终端里从零开始：

```bash
# 1. 创建站点目录
hugo new site my-stack-blog
cd my-stack-blog

# 2. 初始化 Git
git init

# 3. 以 Submodule 方式引入 hugo-theme-stack
git submodule add https://github.com/CaiJimmy/hugo-theme-stack.git themes/hugo-theme-stack

# 4. 删除默认生成的单文件配置（我们要用拆分配置）
rm hugo.toml
```

此时：

- `themes/hugo-theme-stack` 下是主题源码（不要直接改这里）。
- 站点还不能正常跑起来，因为缺少 Stack 期望的那套 `config/` 配置矩阵。

---

## 三、用 `config/_default` 拆分配置：hugo / params / menu / languages

先创建目录：

```bash
mkdir -p config/_default
```

### 1. `hugo.yaml`：Hugo 引擎基础配置

```yaml
# config/_default/hugo.yaml
baseURL: "https://yourusername.github.io/your-repo-name/"
title: "我的极客博客"
languageCode: "zh-cn"
DefaultContentLanguage: "zh-cn"
hasCJKLanguage: true

# 告诉 Hugo 去 themes/ 下找哪个主题
theme: "hugo-theme-stack"

paginate: 5
```

- **baseURL**：部署到 GitHub Pages 时必须改成实际访问地址。
- **theme**：这是 Submodule 路线的关键一行。

### 2. `params.yaml`：Stack 主题专属配置

以一个简化版为例（你可以继续扩展）：

```yaml
# config/_default/params.yaml

colorScheme:
  toggle: true      # 左侧底部显示暗色模式切换按钮
  default: auto     # 跟随系统；也可以写 dark / light

sidebar:
  emoji: "🍥"
  subtitle: "掌控代码，掌控生活"
  avatar:
    enabled: true
    local: true
    src: "img/avatar.png"  # 稍后会创建对应图片

article:
  math: false
  toc: true
  readingTime: true

footer:
  since: 2024
  customText: "完全手动配置版本"
```

后续你还可以把 widgets、comments 等配置一起搬进来。

### 3. `menu.yaml`：左侧导航菜单

```yaml
# config/_default/menu.yaml

main:
  - identifier: home
    name: 首页
    url: /
    weight: -100
    params:
      icon: home

  - identifier: archives
    name: 归档
    url: /archives/
    weight: -90
    params:
      icon: archive
```

### 4. `languages.yaml`：多语言（可选，但推荐提前设计）

```yaml
# config/_default/languages.yaml

zh-cn:
  languageName: "中文"
  weight: 1
  title: "我的极客博客"

en:
  languageName: "English"
  weight: 2
  title: "My Geek Blog"
```

只配中文也可以；但一旦有多语言需求，建议直接这样拆出来。

---

## 四、补齐必要目录与静态资源

在项目根目录：

```bash
# 1. 头像等图片（配合 params.yaml 里的 img/avatar.png）
mkdir -p assets/img

# 放一张图片命名为 avatar.png，放到 assets/img 里

# 2. 自定义样式入口（极其重要）
mkdir -p assets/scss
touch assets/scss/custom.scss

# 3. 内容目录（文章和独立页面）
mkdir -p content/post
mkdir -p content/page
```

- **头像路径**：`params.yaml` 里写的是 `img/avatar.png`，Stack 会从 `assets/img/avatar.png` 里找。
- **`assets/scss/custom.scss`**：所有样式微调都可以放到这里，Stack 会自动编译。

---

## 五、写第一篇文章：Page Bundle & 图片引用

Stack 推荐用 **Page Bundle**：一篇文章一个文件夹，结构清晰，图片好管理。

### 1. 创建文章目录

```bash
hugo new content/post/hello-world/index.md
```

打开生成的 `content/post/hello-world/index.md`，你可以写成这样（用 YAML front matter）：

```yaml
---
title: "Hello World"
description: "我的第一篇 Hugo Stack 测试文章"
date: 2026-03-05T10:00:00+08:00
image: ""           # 暂时不要写图片名，避免报错
categories:
  - 折腾记录
tags:
  - Hugo
  - Stack
draft: false        # 发布到线上时一定要设为 false
---
 
# Hello World

这是我用 Git Submodule + 手动 YAML 配置搭建起来的第一篇文章。
```

### 2. 关于那个经典报错：`nil pointer evaluating resource.Resource.RelPermalink`

如果你在 front matter 里写了：

```yaml
image: "cover.jpg"
```

但 `hello-world` 目录下没有 `cover.jpg`，Stack 在生成 OpenGraph 图像链接时会访问一个不存在的资源，就会报出类似：

> `nil pointer evaluating resource.Resource.RelPermalink`

**解决办法：**

- **不想配图时**：删除 `image` 那一行，或写成 `image: ""`。
- **要配图时**：确保结构如下：

```text
content/post/hello-world/
├── index.md
├── index.en.md   # 如果有多语言
└── cover.jpg     # 文件名和后缀必须与 image 字段一致
```

---

## 六、开启暗色模式与基础 UI 定制

暗色模式前面已经在 `params.yaml` 中配置过：

```yaml
colorScheme:
  toggle: true
  default: auto
```

此外，常用定制点还有：

- **侧边栏**
  - **修改头像**：替换 `assets/img/avatar.png`。
  - **修改副标题**：改 `sidebar.subtitle`。
- **文章 License / 阅读时间 / TOC**：在 `article` 段落中按需开启/关闭。
- **底部 Footer**：改 `footer.since` 和 `footer.customText`。

更复杂的修改（改 HTML 结构）建议用 **覆盖机制**：

- **原文件**（例）：`themes/hugo-theme-stack/layouts/partials/footer/custom.html`
- **你的版本**：在项目根目录创建同路径 `layouts/partials/footer/custom.html`，Stack 会优先使用你的版本。

---

## 七、配置多语言：中文 + 英文双语文章

多语言分两部分：**配置语言** + **写多语言内容**。

### 1. 配置语言（前面已经建了 `languages.yaml`）

确保在 `hugo.yaml` 里也指定了默认语言：

```yaml
DefaultContentLanguage: "zh-cn"
```

`languages.yaml` 例子回顾：

```yaml
zh-cn:
  languageName: "中文"
  weight: 1
  title: "我的极客博客"

en:
  languageName: "English"
  weight: 2
  title: "My Geek Blog"
```

### 2. 为同一篇文章添加英文版本

在 `content/post/hello-world/` 里再建一个 `index.en.md`：

```yaml
---
title: "Hello World"
description: "My first post on Hugo Stack"
date: 2026-03-05T10:00:00+08:00
image: ""       # 或 "cover.jpg" 且保证图片存在
categories:
  - Essay
tags:
  - Hugo
draft: false
---
 
## Finally did it!

This is the English version of my first post.
```

目录结构：

```text
content/post/hello-world/
├── index.md        # 中文
└── index.en.md     # 英文
```

运行 `hugo server` 后，你会看到：

- 左侧多出一个语言切换按钮（通常是地球图标）。
- 切到 English 时，标题与文章列表会切换到英文版本。

---

## 八、本地化依赖：Hugo Modules Vendoring（可选进阶）

虽然主题用的是 Git Submodule，但 Stack 以及其依赖仍然可能通过 **Hugo Modules** 从网络拉包。为了彻底本地化（离线构建、安全性、可控性），可以用 vendoring：

### 1. 初始化 Hugo 模块

在项目根目录执行：

```bash
hugo mod init my-stack-blog
```

会生成一个 `go.mod`。

### 2. Vendor 所有依赖到 `_vendor/`

```bash
hugo mod vendor
```

执行完你会看到 `_vendor/` 目录，里面包含了所有通过 Hugo Modules 引入的依赖。只要存在 `_vendor/`，Hugo 构建优先使用其中的代码，即使离线也能正常工作。

### 3. 提交到 Git

```bash
git add _vendor go.mod go.sum
git commit -m "chore: vendor hugo modules"
```

**注意**：这一步与 “是否用 Submodule 管理主题源码” 并不冲突，它只是把通过 Hugo Modules 引用到的其他依赖一并本地化。

---

## 九、使用 GitHub Actions + GitHub Pages 自动部署

### 1. 推送到 GitHub

```bash
git add .
git commit -m "init: first commit"

git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git
git push -u origin main
```

记得在 `hugo.yaml` 里把 `baseURL` 改成：

- **个人主页仓库**：`https://yourusername.github.io/`
- **普通项目仓库**：`https://yourusername.github.io/your-repo-name/`

### 2. 配置 GitHub Actions 工作流（重点是 submodules: recursive）

在本地创建 `.github/workflows/deploy.yaml`：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive   # ★ 必须：拉取 Submodule 中的主题源码
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true          # Stack 使用 SCSS，需要 extended 版

      - name: Build with Hugo
        env:
          HUGO_ENVIRONMENT: production
        run: |
          hugo --gc --minify

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

推送后，在 GitHub 仓库的 **Settings → Pages** 中把 Source 改成 **GitHub Actions** 即可。

---

## 十、两个常见坑：重复菜单项 & TOML ↔ YAML 转换

### 1. 左侧菜单出现两个“首页 / home”

- **可能原因一：重复定义菜单**
  - 既在 `config/_default/menu.yaml` 里配置了首页，又在 `hugo.yaml` 里写了 `menu:` 段落 → 建议只保留前者。
- **可能原因二：某篇内容的 Front Matter 自动注入菜单**
  - 某个页面头部有：
    ```yaml
    menu:
      main:
        name: "home"
    ```
  - 删除这段，或改名。

对于多语言，还可以用 `menus.zh-cn.yaml` / `menus.en.yaml` 分别配置中英文菜单，避免混乱。

### 2. TOML 配置如何优雅地改成 YAML？

核心规则：

- **等号 → 冒号+空格**：`key = "value"` → `key: "value"`
- **表（`[a.b]`）→ 缩进**：
  - TOML：
    ```toml
    [article.license]
      enabled = true
      default = "xxx"
    ```
  - YAML：
    ```yaml
    article:
      license:
        enabled: true
        default: "xxx"
    ```
- **列表 → `-`**：
  - TOML：`mainSections = ["post"]`
  - YAML：
    ```yaml
    mainSections:
      - post
    ```

Hugo 对 `.toml`、`.yaml`、`.json` **一视同仁**，最终都解析成同一套内部结构。为了可读性和与 Stack 官方文档一致，**推荐统一用 YAML**。

