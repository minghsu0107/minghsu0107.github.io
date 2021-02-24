---
title: "我如何架設這部落格的？"
date: 2021-02-24T16:27:23+08:00
draft: false
categories:
- Blogging
tags:
- Hugo
- Go
---

用 Github Page + Hugo 快速架設部落格網站。

![](https://i.imgur.com/vUVbWN9.png)
<!--more-->
Hugo 是一個用 Go 撰寫的開源網站架設引擎。它幫助我們得以快速得架設一個靜態網站。
## 安裝 Hugo
- https://gohugo.io/getting-started/installing/
- https://github.com/gohugoio/hugo/releases/

啟動新 Hugo 專案:
```bash
hugo new site myblog -f yaml
```
## 基本設定
初始化 Repo:
```bash
git init
```

下載 Hugo 的 theme (使用 git submodule)：
```bash
git submodule add https://github.com/spf13/hyde themes/hyde
```

修改 `config.yaml`:
```yaml
baseURL: https://minghsu0107.github.io/
languageCode: en-us
title: Ming's Site

# 記得設定你的 theme
theme : 'hyde'
```
新增新的 post:
```bash
hugo new posts/first-post.md -f yaml
```
修改 `/content/posts/first-post.md`:
```yaml
---
title: "First Post"
date: 2021-02-24T16:27:23+08:00
draft: false
---
My first post!!!
```
啟動 testing server:
```bash
hugo server -w
```

成功的話會看到你的第一篇 post：

![first-post](/static/images/first-post.png)

## 部署到 Github Page
在 Github 新增 repo `minghsu0107.github.io`。接著在本機新增 branch `main` 與 subtree branch `gh-pages`:
```bash
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/minghsu0107/minghsu0107.github.io.git
git push -u origin main
git subtree push --prefix public origin gh-pages
```

接著到 Github repo 上， 從 `minghsu0107.github.io -> settings` 設定 Github page branch 為 `gh-pages`。

現在打開瀏覽器瀏覽 `https://minghsu0107.github.io` 就會看到部署完成的部落格！

## 一鍵部署
在你的 `.bashrc` 新增:
```bash
alias pub="
    cd /Users/xuhaoming/Desktop/blog \ 
    git checkout main && hugo --minify \
    git add . && git commit -m 'update' \
    git push origin main \
    git subtree push --prefix public origin gh-pages"
```
從今以後，只要執行 `pub` 即可發布最新的部落格變更！
## Reference
- https://themes.gohugo.io/hyde/
- https://gohugo.io/templates/lookup-order/