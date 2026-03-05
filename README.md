# 我的 Hugo 博客（基于 hugo-theme-stack）

这是一个使用 [Hugo](https://gohugo.io/) 搭建的个人技术博客，主题采用 `hugo-theme-stack`，并通过 Git Submodule 管理主题源码，方便后续升级和深入定制。

## 功能简介

- **技术博客写作**：使用 Markdown 写作，支持 Page Bundle（每篇文章一个目录）、文章封面图、分类和标签。
- **暗色模式**：内置浅色 / 深色模式切换，并支持跟随系统外观。
- **多语言支持**：通过 Hugo 多语言机制，目前支持中文和英文内容。
- **现代主题外观**：基于 `hugo-theme-stack`，拥有简洁的阅读体验和良好的移动端适配。

## 主要技术栈

- **静态站点生成器**：Hugo
- **主题**：hugo-theme-stack（通过 Git Submodule 引入）
- **配置格式**：统一使用 YAML（`config/_default/*.yaml`）
- **部署方式**：GitHub Actions + GitHub Pages 自动构建和发布

## 本地开发

```bash
# 安装依赖（拉取主题子模块）
git submodule update --init --recursive

# 启动本地开发服务器
hugo server
```

启动后访问浏览器中的 `http://localhost:1313` 即可预览博客。

## 目录结构概览

- `content/`：博客文章与页面内容（如 `post/` 下为文章，`page/` 下为独立页面）
- `config/_default/`：站点与主题的拆分配置（`hugo.yaml`、`params.yaml`、`menu.yaml`、`languages.yaml` 等）
- `themes/hugo-theme-stack/`：Stack 主题源码（Git Submodule）
- `assets/`：自定义样式和图片（如头像、SCSS 覆盖等）

## 写作建议

- 新文章推荐使用 Page Bundle 形式，例如：
  - `content/post/my-first-post/index.md`
  - 同目录下可放置文章相关图片（如 `cover.jpg`），在 Front Matter 中通过 `image: "cover.jpg"` 引用。

## 部署说明（简要）

仓库内已通过 GitHub Actions 配置自动部署流程：

- 将代码推送到远程仓库的主分支（如 `main`）后，GitHub Actions 会自动：
  1. 拉取主题 Submodule
  2. 使用 Hugo 构建静态文件
  3. 将 `public/` 内容发布到 GitHub Pages

具体域名取决于仓库名称和 GitHub Pages 设置，可在 GitHub 仓库的 `Settings → Pages` 中查看。
