---
title: 使用 Github Actions 替换 Travis 自动发布 Hexo 博客
date: 2022-02-08 19:39:46
tags:
- Github
- Github Action
categories:
- 杂
---

# 使用 Github Action 替换 Travis 自动发布 Hexo 博客

之前通过 Hexo + Github Pages 的模式搭建了这个博客，并且通过 Travis 自动集成发布我更新的博客文章。但是从 2021 年 12 月份开始 Travis CI 不再免费为开源项目提供持续集成的托管服务了。也是因为我好久没写博客了，导致这个问题直到最近才被发现。寻找替代方案，找了一圈发现 Github 自己提供了 Github Actions 的持续集成服务。那就折腾折腾替换成 Github Actions 来发布博客。

## 准备工作
## 介绍

我的博客项目地址是：https://github.com/Win-Man/win-man.github.io 。repo 中分了两个分支，一个 master 分支和一个 dev 分支。dev 分支负责存放原始的 hexo 文件，master 分支负责存放 hexo generate 生成的 html 文件，然后通过 https://win-man.github.io/ 这个网址访问到的就是 master 分支上的 html 文件了。

之前 Travis CI 持续集成服务中是会监测 dev 分支的更新，每当我有将提交 push 到 dev 分支上的时候，会触发 Travis CI 的持续集成服务，基于 dev 分支的内容自动更新 master 分支。所以想用 Github Actions 替换 Travis CI 也是类似的思路：

1. 监测到 dev 分支有更新
2. 基于 dev 分支内容执行 hexo deploy 



## 步骤

### 生成秘钥

这一步是为了让 Github Actions 有权限将更新 push 到 repo 。

1. 创建一个新的 SSH 秘钥

```shell
ssh-keygen -f hexo-deploy-key
```

会生成 hexo-deploy-key 和 hexo-deploy-key.pub 两个文件

2. 在 win-man.github.io 这个项目添加 Secrets

添加 hexo-deploy-key 的内容作为 Secrets
![add secrets](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20220208200006.png)

3. 在 win-man.github.io 这个项目添加 Deploy key

添加 hexo-deploy-key.pub 的内容作为 Deploy key
![add deploy key](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20220208200219.png)

### 配置 Github Action

1. 切换到 dev 分支下，然后创建 workflows 目录, 并配置 main.yaml 文件

```shell
mkdir -p .github/workflows/
touch .github/workflows/main.yaml
```

2. 填写配置文件

配置文件内容如下:

```yaml
name: hexo-deploy
on:
  push:
    branches: [ dev ]
  pull_request:
    branches: [ dev ]

  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  build:
    # The type of runner that the job will run on
    runs-on: ubuntu-latest

    # Steps represent a sequence of tasks that will be executed as part of the job
    steps:
      # Checks-out your repository under $GITHUB_WORKSPACE, so your job can access it
      - name: Checkout dev branch
        uses: actions/checkout@v2
        with:
            ref: dev
      - name: Config node.js
        uses: actions/setup-node@v2
        with:
          node-version: '13'
          cache: 'npm'
      - name: Install hexo
        env:
          ACTION_DEPLOY_KEY: ${{ secrets.HEXO_DEPLOY_SECRET }}
        run: |
          mkdir -p ~/.ssh/
          echo "$ACTION_DEPLOY_KEY" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa
          ssh-keyscan github.com >> ~/.ssh/known_hosts
          git config --global user.name "Win-Man"
          git config --global user.email 825895587@qq.com
          npm install hexo-cli -g
          npm install
      - name: Generate and deploy
        run: |
          hexo clean
          hexo deploy
```

3.  提交 Github Actions 配置

4. 查看 Github Actions 并测试

每当 dev 分支有代码更新的时候，就会触发 Github Actions 的工作流自动发布博客了。

![actions](https://raw.githubusercontent.com/Win-Man/pic-storage/master/img/20220208200903.png)

## 问题
### node version 错误导致生成 html 为空
#### 问题描述

Github Actions job 正常触发也正常运行了，没有报错，但是访问 https://win-man.github.io/ 这个网址却是一篇空白。检查 master 分支上的 html 文件发现每篇文章的 html 文件是生成了，但是里面的内容是空的。

#### 问题原因

nodejs 版本太高，hexo 不支持。

#### 解决方案

将 main.yaml 中 node-version 的配置设置为 13 版本。


## 参考文档
* [Github Action docs](https://docs.github.com/cn/actions)
* [使用 Github Actions 部署 Hexo 博客](https://makefile.so/2021/11/28/use-github-actions-to-deploy-hexo-blog/)