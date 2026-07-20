# Hugo + GitHub Pages 部署完整配置流程

本指南记录从 Hugo 站点搭建到通过 GitHub Actions 自动部署到 GitHub Pages 的完整流程，以及排查 "Actions 一直 Queued" 问题的过程。

---

## 一、环境准备

### 1.1 下载 Hugo Extended

本示例使用 Hugo 扩展版 `0.164.0`：

```bash
hugo version
# 期望输出包含：hugo v0.164.0 ... extended ...
```

> GitHub Actions 中也需要使用相同版本，见下文 workflow 配置。

### 1.2 初始化 Hugo 站点

```bash
hugo new site myblog
cd myblog
```

---

## 二、安装主题

本示例使用 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题。

### 方式 A：使用 Git Submodule（推荐，便于更新主题）

```bash
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 方式 B：直接复制主题文件（Vendoring）

如果你不想用 submodule，也可以把主题文件直接复制到 `themes/PaperMod/` 下并提交到仓库。

本示例最终采用方式 B，因此 GitHub Actions 中不需要 `submodules: true`。

---

## 三、站点基础配置

编辑 `hugo.toml`：

```toml
baseURL = 'https://zhuye-s11.github.io/myblog/'
languageCode = 'zh-CN'
title = 'My New Hugo Project'
theme = 'PaperMod'
```

> `baseURL` 必须是 GitHub Pages 的实际访问地址，格式为 `https://<用户名>.github.io/<仓库名>/`。

---

## 四、创建第一篇内容

```bash
hugo new content posts/hello.md
```

编辑 `content/posts/hello.md`，把 `draft: true` 改为 `draft: false`，否则 Hugo 不会渲染草稿。

---

## 五、Git 仓库初始化与提交

```bash
git init
git add .
git commit -m "init hugo site"
```

### 5.1 创建 .gitignore

```gitignore
# Hugo build output
public/
resources/_gen/
.hugo_build.lock

# OS files
.DS_Store
Thumbs.db
```

> `public/` 是 Hugo 构建产物，由 GitHub Actions 生成，不需要提交到仓库。

### 5.2 推送到 GitHub

```bash
git remote add origin https://github.com/zhuye-s11/myblog.git
git branch -M main
git push -u origin main
```

---

## 六、GitHub Pages 设置

打开仓库页面：

```
https://github.com/zhuye-s11/myblog/settings/pages
```

在 **Build and deployment** 中：

- **Source**：选择 **GitHub Actions**

保存即可。

> 不要选择 "Deploy from a branch"，否则 Actions 部署会失败或不被触发。

---

## 七、GitHub Actions Workflow 配置

创建文件 `.github/workflows/hugo.yml`：

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

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.164.0'
          extended: true

      - name: Build
        run: hugo --minify --baseURL "https://zhuye-s11.github.io/myblog/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-22.04
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 7.1 关键说明

- `on.push.branches: ["main"]`：只有推送到 `main` 分支时才触发部署。
- `workflow_dispatch`：支持在 GitHub 页面上手动触发。
- `permissions.pages: write` 和 `id-token: write`：GitHub Pages 部署必需。
- `actions/upload-pages-artifact@v3` + `actions/deploy-pages@v4`：官方推荐的 Pages 部署组合。

---

## 八、常见问题：Actions 一直卡在 Queued

### 8.1 现象

在 GitHub Actions 页面看到 workflow run 状态为 `Queued`，长时间（数小时）不开始执行。

### 8.2 已排查的非原因

- Hugo 代码和构建命令正确 ✅
- Workflow YAML 格式正确 ✅
- `main` 分支推送正常 ✅
- GitHub Actions 仓库权限已开启 ✅
- GitHub Actions 免费额度充足 ✅
- GitHub Pages Source 已设为 GitHub Actions ✅

### 8.3 根因

Workflow 中配置了 `concurrency` 并发组：

```yaml
concurrency:
  group: "pages"
  cancel-in-progress: false
```

当之前的某个 run 异常卡住且未正确释放锁时，后续所有 run 都会在这个 `pages` group 上排队，表现为全部 `Queued`。

### 8.4 解决方案

**移除 `concurrency` 配置**：

```yaml
permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    ...
```

然后取消所有卡住的旧 run，重新 push 或手动触发新 run。

---

## 九、验证部署

### 9.1 查看 Actions 运行状态

打开：

```
https://github.com/zhuye-s11/myblog/actions
```

确认 `build` 和 `deploy` 两个 job 都是 ✅ 绿色。

### 9.2 访问站点

部署成功后，访问：

```
https://zhuye-s11.github.io/myblog/
```

也可以在 **Settings > Pages** 中看到绿色提示：

> Your site is live at `https://zhuye-s11.github.io/myblog/`

---

## 十、后续更新站点流程

每次本地新增或修改内容后：

```bash
# 1. 创建/编辑内容
hugo new content posts/xxx.md

# 2. 本地预览（可选）
hugo server -D

# 3. 构建验证
hugo --minify --baseURL "https://zhuye-s11.github.io/myblog/"

# 4. 提交并推送
git add .
git commit -m "add new post"
git push origin main
```

push 到 `main` 后，GitHub Actions 会自动构建并部署。

---

## 十一、参考

- [Hugo 官方文档](https://gohugo.io/documentation/)
- [GitHub Pages 文档](https://docs.github.com/en/pages)
- [peaceiris/actions-hugo](https://github.com/peaceiris/actions-hugo)
- [actions/deploy-pages](https://github.com/actions/deploy-pages)
