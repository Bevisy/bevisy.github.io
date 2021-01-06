---
title: "通过Travis CI 部署静态页面到 github page"
date: 2020-01-21T00:25:33+08:00
draft: false
tags: 
    - CI/CD
categories:
---

# Travis CI 部署 Hexo 生成文件到 master 分支

文档主要实现Hexo利用 Travis CI 完成 github page 部署。

由于 username.github.io 仓库不允许切换 github page 分支，必须将 master 分支作为静态资源分支，所以新建 develop 分支作为开发分支。

详细的步骤参考[将 Hexo 部署到 GitHub Pages](https://hexo.io/zh-cn/docs/github-pages)，此处主要贴出区别的 Travis CI 配置文件：

```sh
sudo: false
language: node_js
node_js:
  - 13.6.0
cache: npm
branches:
  only:
    - develop
script:
  - hexo generate
deploy:
  provider: pages
  skip-cleanup: true
  github-token: $GH_TOKEN
  keep-history: false
  target_branch: master
  local-dir: public
  on:
    branch: develop
```

上述配置信息解释参考 [GitHub Pages Deployment](https://docs.travis-ci.com/user/deployment/pages/)。

# 参考文档

[GitHub Pages Deployment](https://docs.travis-ci.com/user/deployment/pages/)

[将 Hexo 部署到 GitHub Pages](https://hexo.io/zh-cn/docs/github-pages)