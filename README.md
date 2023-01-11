# bevisy.github.io

## 安装 hugo@v0.88.1

```sh
# 需要开启 tags:extended
# 解决报错：Error: Error building site: TOCSS: failed to transform "scss/style.scss" (text/x-scss). Check your Hugo installation; you need the extended version to build SCSS/SASS with transpiler set to 'libsass'.
go install -tags extended github.com/gohugoio/hugo@v0.88.1
```

## 构建博客服务

```sh
# 生成
git submodule init
git submodule update
hugo --minify

# 启动服务
hugo server
```

## 其它操作

```
# 新建文章
hugo new post/test-page.md
```
