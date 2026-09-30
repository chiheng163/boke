---
title: "Hugo + GitHub Pages 自动化发布博客的完整流程"
date: 2026-09-30
tags: ["Hugo", "GitHub Pages", "GitHub Actions", "CI/CD", "静态博客"]
author: chiheng163
---

你已经把稿子写好了，本地 `hugo server -D` 跑起来排版漂亮、图片正常。可等你推到 GitHub、等部署跑完、打开那个 `<用户名>.github.io` 地址，看到的却是 404、白屏，或者 CSS 和图片全丢。为什么本地一切正常，一上 CI 就崩？

问题通常不在文章，而在「从本地 Markdown 到公网页面」这条流水线缺了几个环节。这篇文章以 `github.com/chiheng163/boke` 这个真实仓库为样本，把链路从头串到尾——照着做完，就能让一篇文章在 merge 之后自动出现在公网。

## 为什么自建静态博客：Hugo 的定位与代价

选 Hugo 而不是托管平台（Medium、掘金、公众号），本质是拿一点运维成本换三样东西：内容的所有权、Markdown 源码入库、完全自由的排版。和同为静态生成器的 Jekyll、Hexo 相比，Hugo 的优势在「单二进制、零运行时依赖」——装一个可执行文件就能跑，不用先配 Ruby 或 Node 环境。

【为什么重要】单二进制意味着 CI 里装 Hugo 就是解压一个 tar 包，不依赖语言运行时，构建环境几乎不会因依赖漂移而失败。

代价是 Go template 的学习曲线，但博客用到的能力（列表、标签、单页）集中在少数模板里，看懂一次可长期复用。

GitHub Pages 免费，但有明确边界。官方限额是：源仓库与已发布站点都不超过 1 GB、软带宽上限 100 GB/月、单次部署超过 10 分钟超时、软限制 10 次构建/小时——不过最后这条明确写着「在使用自定义 GitHub Actions 工作流构建时不适用」，这正是本文选 workflow 模式的理由之一。public 仓库用 GitHub Free 即可，private 仓库需 Pro 及以上。

适用场景：个人技术博客、文档站、项目主页，以「写 + 发」为主的内容。不划算的是需要评论、全文搜索、在线后台编辑的站点。

## 三层结构：写作层、Git 层、CI 层

这条链路拆成三层，各自只负责一件事：

- 写作层：`content/posts/` 下的 Markdown 文件，加上站点配置 `hugo.toml`，这是内容的唯一真相源。
- Git 层：`main` 分支即发布状态，你只往分支提交，merge 进 main 就代表「要发布」。
- CI 层：GitHub Actions 负责构建 `public/` 并部署，你永远不手工碰它。

【为什么重要】把「发布」和「构建」解耦后，发布退化成一次 merge。你不需要在本地装 Hugo 也能发稿——只要 Markdown 写对了，剩下的交给 CI。同时这也锁死一条纪律：`public/` 是构建产物，会被 Hugo 重建，官方文档明确警告「不要把 publishDir 提交进仓库」，也绝不手工推 `gh-pages` 分支，否则仓库里会出现两套页面。

## 最短可用路径：从空仓库到本地看到第一篇文章

先把 Hugo 装上。当前稳定版是 v0.167.0（2026-09-28 发布），装完先校验：

```bash
hugo version
```

【为什么重要】版本号要在这里确认。后面 CI 工作流里的 `HUGO_VERSION` 要和你本地一致，否则可能出现「本地能构建、CI 语法报错」的偏差。

这里有个坑：老教程的 `hugo new site` 已经不在官方文档里了（该命令页返回 404），现在建站用 `hugo new project`：

```bash
hugo new project boke --format yaml
cd boke
```

另一个过时说法是「必须装 Hugo Extended 才能用 SCSS」。现在不成立：官方 GitHub Pages 工作流下载的就是非 extended 版本，extended 版独有的 LibSass 支持自 v0.153.0 起已被弃用，官方建议改用与任何 edition 兼容的 Dart Sass。

引入主题用 git submodule（这也是官方快速上手的方式）：

```bash
git submodule add https://github.com/gohugo-ananke/ananke themes/ananke
echo "theme = 'ananke'" >> hugo.toml
```

写第一篇文章，一个文件一篇：

```bash
hugo new content content/posts/first-post.md
hugo server -D
```

`-D` 是 `--buildDrafts` 的简写，草稿默认不发布，本地预览时需要它。

【为什么重要】本地预览跑通 ≠ CI 能构建。CI 里要多做三件事：拉取主题 submodule、装 Dart Sass 和 Node 依赖、用 CI 的 baseURL 覆盖本地配置。预览只是第一关，真正能保证「可发布」的是下面这份工作流。

## GitHub Pages 的两种构建模式与官方 Actions 工作流

GitHub Pages 有两种发布源：branch/legacy 是「推分支后由 GitHub 构建」，workflow 是「由你的自定义 Actions 构建」。本文用 workflow（API 里的 `build_type=workflow`）：既有「10 次/小时软限制不适用」的好处，也能完全控制构建步骤。

启用方式在仓库 Settings > Pages，把 Source 改成 GitHub Actions，官方说明「改动立即生效，无需点保存」。

工作流文件放在 `.github/workflows/hugo.yaml`，以下是完整可运行的一份，版本号取自 Hugo 官方文档：

```yaml
# .github/workflows/hugo.yaml
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
    env:
      HUGO_VERSION: 0.167.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O "${{ runner.temp }}/hugo.tar.gz" \
            "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz"
          tar -xzf "${{ runner.temp }}/hugo.tar.gz" -C "${{ runner.temp }}"
          echo "${{ runner.temp }}" >> "$GITHUB_PATH"
          hugo version
      - name: Install Dart Sass
        run: sudo snap install dart-sass
      - name: Checkout
        uses: actions/checkout@v7
        with:
          submodules: recursive
          fetch-depth: 0
          lfs: false
      - name: Initialize Git submodules
        run: git submodule update --init --recursive
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6
      - name: Build with Hugo
        run: |
          hugo build \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/.cache/hugo"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./public
          include-hidden-files: false

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

逐项解释，这些字段不是装饰：

- `permissions` 里 `pages: write` 授权 GITHUB_TOKEN 调 Pages API 建部署，`id-token: write` 用来申请 OIDC JWT、校验部署来源。缺了 `id-token`，部署 job 直接失败。
- `concurrency: group: "pages" / cancel-in-progress: false` 保证同一时刻只有一个部署在跑，且不取消正在进行的那次，避免生产部署被打断。
- 四个 action 版本号固定：`actions/checkout@v7`、`actions/configure-pages@v6`、`actions/upload-pages-artifact@v5`、`actions/deploy-pages@v5`。
- `--baseURL "${{ steps.pages.outputs.base_url }}/"` 的尾斜杠必须：`configure-pages` 输出的 `base_url` 不带尾斜杠，而 Hugo 的 `baseURL` 定义要求结尾斜杠。
- `upload-pages-artifact` 的 `path: ./public` 把构建产物打成名为 `github-pages` 的 gzip 包；artifact 要求单个 tar、不含符号链接，默认排除 `.git` 和 `.github`。
- `deploy` job 的 `needs: build` 和 `environment: github-pages` 都是 deploy-pages 的硬性要求，缺了 `needs` 会「独立部署、持续找不到 artifact」。

【为什么重要】这段 YAML 是整条链路的枢纽。版本号一旦写错（比如照抄 GitHub 自家已过时的 Hugo starter workflow，它写的还是 hugo 0.128.0、checkout@v4、upload@v3），构建就会失败或产物错误。所以版本号以 Hugo 官方文档为准，而不是 GitHub 的 starter 模板。

## 文章格式约定：frontmatter、slug 与资源组织

本仓库的 frontmatter 固定四个字段：

```markdown
---
title: "Hugo + GitHub Pages 自动化发布博客的完整流程"
date: 2026-09-30
tags: ["Hugo", "GitHub Pages", "GitHub Actions", "CI/CD", "静态博客"]
author: chiheng163
---
```

几点要说明清楚：

- `tags` 是分类法（taxonomy），只有在配置里定义了 `[taxonomies] tag = 'tags'` 才会生成标签页，否则只是元数据。
- `author` 不是 Hugo 保留字段，官方示例放在 `params:` 下；自定义字段应放 `params` 键下。
- `date` 写 `2026-09-30` 这种不完整格式时，Hugo 默认按 `Etc/UTC` 解析，可能造成时区偏差，要避免就在配置里显式设 `timeZone`。

URL 稳定性是最关键的一点。Hugo 渲染出的 URL 默认等于 `content` 下的文件路径（`content/posts/post-1.md` → `/posts/post-1/`），`slug` 只覆盖最后一段路径，`url` 覆盖整条路径且不被清洗。所以：

【为什么重要】slug 一旦发布就不该改。改了 slug 等于改 URL，旧链接、索引、收藏全失效。要在发布前定下 slug，并且用 ASCII——非 ASCII 文件名会被百分号编码（官方示例 `Hugö → hug%C3%B6`），中文文件名会变成一长串难读的链接。

图片有两种组织方式：放 `static/` 下，会原样拷贝进 `public/`，用绝对路径引用；或者用 page bundle（含 `index.md` 的目录），把图片和文章放一起作为 page resource 引用。前者适合全站共享资源，后者适合单篇配图。

## 发布流程：分支、PR、合并触发部署

不直接推 main 的原因有三：PR 能跑一次完整构建校验、合并可回滚、每一步都有留痕。完整命令序列如下：

```bash
git checkout -b post/hugo-github-pages-auto-deploy
# 写稿：把文章放进 content/posts/hugo-github-pages-auto-deploy.md
git add content/posts/hugo-github-pages-auto-deploy.md
git commit -m "post: Hugo + GitHub Pages 自动化发布博客的完整流程"
git push -u origin post/hugo-github-pages-auto-deploy
```

push 认证：Personal Access Token（PAT）可以直接当 HTTPS 密码用（`git clone` 提示 Password 时填 token），但只能用于 HTTPS，SSH remote 要先换成 HTTPS。免交互用 `http.extraHeader` 把认证头写进 git 配置，变量名大小写不敏感。

开 PR 用 API：

```bash
curl -X POST \
  -H "Authorization: Bearer $GITHUB_PERSONAL_ACCESS_TOKEN" \
  -H "X-GitHub-Api-Version: 2026-03-10" \
  https://api.github.com/repos/chiheng163/boke/pulls \
  -d '{"title":"post: Hugo + GitHub Pages 自动化发布博客的完整流程","head":"post/hugo-github-pages-auto-deploy","base":"main"}'
```

【为什么重要】token 只从环境变量 `GITHUB_PERSONAL_ACCESS_TOKEN` 读，绝不写进任何文件。写进仓库的历史记录里等于泄露凭据。

合并 PR 之后，别急着宣告成功，回读验证三件事：

1. Actions run 结论：`GET /repos/{owner}/{repo}/actions/runs`
2. Pages 状态：`GET /repos/{owner}/{repo}/pages`，响应里的 `status` 应为 `built`
3. 访问真实 URL 确认可打开

本仓库是项目站点（仓库名 `boke`），默认地址带仓库名路径：`https://chiheng163.github.io/boke/`，文章预期落在 `https://chiheng163.github.io/boke/posts/hugo-github-pages-auto-deploy/`。

## 常见坑与排错清单

- **baseURL 不匹配 → CSS/图片 404**。现象：页面结构在但样式全丢。处理：工作流里显式拼 `--baseURL "${{ steps.pages.outputs.base_url }}/"`，因为 `configure-pages` 输出的 `base_url` 不带尾斜杠。
- **主题 submodule 未拉取 → 构建失败**。处理：checkout 用 `submodules: recursive` 并跑 `git submodule update --init --recursive`。Pages 只能访问 public 仓库的 submodule，且必须用 `https://` 只读 URL。
- **权限字段缺失 → 部署失败**。处理：对照工作流的 `permissions` 和 `deploy` 段补齐 `pages: write`、`id-token: write`、`needs` 和 `github-pages` 环境。
- **首次部署 404**。依据官方排查清单：Status 故障、DNS、浏览器缓存、`index.html` 必须在发布源顶层（大小写敏感）。处理：确认 artifact 顶层有 `index.html`，再清缓存、查 Status。
- **「改了看不到」**。浏览器缓存是已知原因，同时 `public/` 每次部署都是新 artifact。处理：硬刷新（Ctrl+F5），或等部署跑完。
- **构建超时**。Pages 部署超过 10 分钟即超时，`deploy-pages` 默认 timeout 也是 600000 ms。处理：控制站点体积、图片缓存指向 `cacheDir`。
- **时区导致日期偏差**。`2026-09-30` 默认按 UTC 解析。处理：配置里设 `timeZone`。
- **中文 slug 与链接编码**。非 ASCII 会被百分号编码。处理：frontmatter 显式写 ASCII 的 `slug`。

## 再往前一步：把发布也自动化

`hugo new content` 生成的文件内容来自 archetype 模板。默认模板会带 `draft: true` 和 date、title 字段。你可以自定义 `archetypes/posts.md`，让每次新建文章直接生成合法的四字段 frontmatter，省掉手写。

再进一步，把「建分支 → 写稿 → commit → 开 PR」封装成一条脚本，减少手工步骤。但上定时任务前要看清官方约束：`schedule` 事件用 POSIX cron 语法、默认按 UTC 计时、只在默认分支最新提交上运行、最短间隔 5 分钟；高负载（尤其整点）时任务会延迟甚至被丢弃。所以定时发布别设在整点，也要想清楚自动提交是走 PR 还是直推——直推会跳过审查，失败时又没人看着。
