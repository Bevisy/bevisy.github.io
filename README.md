# bevisy.github.io

## 安装 Hugo

需要 extended 版本（主题用到 SCSS/SASS）。

```sh
# macOS
brew install hugo

# 或用 Go 安装指定版本
go install -tags extended github.com/gohugoio/hugo@v0.164.0
```

Hugo 版本保持与 GitHub Action（.github/workflows/gh-pages-deploy.yaml）一致，当前为 v0.164.0。

## 构建博客

```sh
# 初始化主题子模块
git submodule update --init themes/hugo-theme-stack

# 生成静态文件
hugo --minify

# 本地预览
hugo server
```

## 新建文章

每篇文章使用目录格式，目录名用 kebab-case 英文小写：

```sh
# 创建目录和 index.md
mkdir -p content/post/my-new-post
hugo new content/post/my-new-post/index.md
```

新建后需在 frontmatter 中添加 slug，值等于目录名，锁定 permalink：

```yaml
slug: "my-new-post"
```

详细约定见 AGENTS.md。

## SEO 配置

博客已启用以下搜索引擎优化能力（均在 Hugo 内置 + 主题基础上配置）：

- sitemap.xml — Hugo 内置自动生成，包含所有页面 URL 与 lastmod，正式构建后域名指向 https://bevisy.github.io
- robots.txt — enableRobotsTXT = true（config.toml），模板见 layouts/robots.txt，允许全部抓取并声明 sitemap 位置
- RSS feed — index.xml，config.toml 中 rssFullContent = true 输出全文
- OpenGraph / Twitter Card — 每篇文章分享到社交平台时生成标题/描述/配图卡片

### OpenGraph 兜底图

文章 frontmatter 有 image 字段时，分享卡片使用文章自己的头图；
没有 image 字段的老文章，回退到站点默认图 static/img/og-default.jpg（1200x630，约 134KB）。

兜底逻辑通过覆盖主题模板实现（项目 layouts/ 优先级高于主题，不修改 submodule）：

- layouts/_partials/head/opengraph/provider/base.html — og:image 兜底
- layouts/_partials/head/opengraph/provider/twitter.html — twitter:image 兜底

config.toml 相关配置：

```toml
enableRobotsTXT = true

[params.defaultImage.opengraph]
enabled = true
local = false
src = "img/og-default.jpg"
```

替换兜底图：直接覆盖 static/img/og-default.jpg 即可，无需改配置。推荐尺寸 1200x630（1.91:1），文件大小 200KB 以内。

## 响应式布局定制

hugo-theme-stack 默认在 768–1280px 区间按百分比显示左中右三栏，会把主内容区挤压到 ~40%。通过站点级 `assets/scss/custom.scss` 覆盖（不修改主题 submodule）：

- 右侧栏（首页 widgets / 文章 TOC）延迟到 xl（≥1280px）才显示
- 左栏宽度 `clamp(200px, 22%, 300px)`、右栏固定 300px，其余宽度全部给主栏
- 验证过 768–1920px 全区间：1024–1279px 主栏从 ~380px 提升到 716–775px

改侧栏宽度直接编辑 `assets/scss/custom.scss` 后重新 `hugo --minify` 即可。

## 评论（giscus）

评论后端从 Disqus 换为 giscus（GitHub Discussions，国内可访问，Disqus 在国内被墙）。零模板改动，纯配置：

- `config.toml` 的 `[params.comments]` 设 `provider = "giscus"`，子配置见 `[params.comments.giscus]`
- 后端是仓库 bevisy/bevisy.github.io 的 Discussions（分类 Announcements），评论按文章 URL（mapping=pathname）挂到 Discussion
- 评论数据存在 GitHub Discussions，可在仓库 Discussions 标签页管理
- 切换别的提供商（utterances/gitalk/...），改 `provider` 并补对应配置块即可，主题原生支持全部提供商

## 部署

推送到 main 分支后，GitHub Action 自动构建并部署到 GitHub Pages。
