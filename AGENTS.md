# bevisy.github.io

Hugo 个人博客，主题 hugo-theme-stack（git submodule）。

## 文章目录结构

content/post/ 下每篇文章统一使用目录格式，不使用单独 .md 文件：

- 目录名：kebab-case 英文小写，不用中文、不用大写字母
- 文章文件：目录内统一为 index.md，不用与目录同名的 .md
- 每篇 index.md 必须有 slug frontmatter，值等于目录名，锁定 permalink（/p/:slug/）
- 文章 title 保持中文或英文均可，不受目录名影响
- 文章图片放在同目录下，frontmatter image 用相对文件名引用

## 构建命令

```
git submodule update --init themes/hugo-theme-stack
hugo --buildDrafts --gc --minify
```
