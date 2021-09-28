---
title: "我如何架設這個部落格的？"
date: 2021-02-24T16:27:23+08:00
draft: false
categories:
- Blogging
tags:
- Hugo
- Golang
description: |-
  用 Hugo + Github Pages + Github Actions 快速架設部落格網站。
---

用 Hugo + Github Pages + Github Actions 快速架設部落格網站。

![](/static/images/vUVbWN9.png)
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
hugo new posts/first-post.md
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

## 部署到 Github Pages
在 `.github/workflows/gh-pages.yaml` 新增打包與部署 hugo 的 Github Actions:
```yaml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch name to trigger deployment
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.83.1'

      - name: Build
        run: hugo --minify  # Package the blog into ./public/

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public # Copy contents in ./public/ to branch gh-pages
          cname: minghsu.io
```
接著新增一個新的分支 `gh-pages`，Github Actions 會將部落格打包並部署到這個分支：
```bash
git checkout -b gh-pages
git push origin gh-pages
```
把 code 推到 Github 上之後，Github Actions 會開始部署部落格。不過這次的部署會失敗，因為我們還沒設定 Github Pages 的 branch。因此接著我們要到 `Settings -> Pages` 將 Github Pages 的分支設定為 `gh-pages` 並設定 project folder 為 root。現在 Github Pages 會重新開始部署，成功之後瀏覽 `https://minghsu.io` 就能看到部落格！
## Reference
- https://themes.gohugo.io/hyde/
- https://gohugo.io/templates/lookup-order/