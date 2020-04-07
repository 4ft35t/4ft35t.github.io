---
title: "使用 Github Actions 自动部署 hugo"
date: 2020-04-07T14:03:04+08:00
tags: ["hugo", "github"]
categories: ["Github"]
toc: true
---

Hugo 是一款开源的使用 go 语言写的静态网站生成器，生成的静态页面可以轻松部署到 github pages。

GitHub Actions 是 GitHub 的持续集成服务，持续集成由很多操作组成，比如抓取代码、运行测试、登录远程服务器，发布到第三方服务等等。GitHub 把这些操作就称为 actions。如果你需要某个 action，不必自己写复杂的脚本，直接引用他人写好的 action 即可，整个持续集成过程，就变成了一个 actions 的组合。[Github actions 市场](https://github.com/marketplace?type=actions)，可以搜索别人提交的 actions。

## 自动部署要点
- username.github.io 只能使用 master 分支
- Github actions 的配置 yaml 文件只能放在默认分支

## 编写脚本
自动部署可以从源码仓库部署到发布仓库，也可以在单一仓库的分支之间部署。
本文的 hugo markdown 文件位于 username.github.io 的 `source` 分支，生成的 html 文件在同仓库的 `master` 分支发布。

#### 切换默认分支
__Repository Settings__ - __Branches__ 切换 source 为默认分支。

#### 添加 Actions
__Actions__ - __New workflows__，选择 __Simple workflow__
然后填入以下内容

```yml
name: Auto deploy hugo
on:
  push:
    branches: # 触发分支
      - source

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@master
        with:
          submodules: true # 检查 Hugo themes 更新

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          publish_dir: ./public
          publish_branch: master # 部署目的分支
```

引用的两个 actions 及作用
- [peaceiris/actions-hugo](https://github.com/peaceiris/actions-hugo) 获取最新的 hugo
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) 发布静态页面

## 设置部署权限
可以使用 Github Token 或者私钥给 workflow 仓库写权限。
本文使用私钥，设置步骤如下
1. `ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""` 生成公私钥对
2. __Repository Settings__ - __Deploy Keys__，粘贴 `gh-pages.pub` 内容并勾选 ` Allow write access`
3. __Repository Settings__ - __Secrets__，粘贴 `gh-pages` 内容, name 必须为 `ACTIONS_DEPLOY_KEY`
![](https://github.com/peaceiris/actions-gh-pages/blob/master/images/secrets-1.jpg?raw=true =800x)

设置完成后，向 source 分支提交代码，会自动生成静态页面发布到 master 分支。
效果类似
![](https://github.com/peaceiris/actions-gh-pages/blob/master/images/log_overview.jpg?raw=true =800x)
