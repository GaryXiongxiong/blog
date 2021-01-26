---
title: 快速搭建易于打理的个人博客 - 基于Github Pages与Actions的博客自动部署
tags:
  - Github
  - Github Pages
  - 自动构建
categories:
  - blog
date: 2021-01-25 18:06:18
---

Hexo是一款快速，简洁且高效的博客框架。可以快速地将Markdown格式文章转化为静态博客页面。Hexo本身附带一键部署功能，可将生成的静态页面部署至GitHub pages，但本文讨论的是另一种部署方式。通过Github Actions在提交博客源码时自动生成静态页面并部署至Github Pages。

<!--more-->

## Github Pages

> Github Pages 被设计用以发布存储于Github仓库的个人、组织、项目相关的静态页面。

Github Pages 的配置非常方便，我们只需将静态页面上传至一个Github仓库，并在仓库配置中开启GithubPages相关选项

![image-20210126104911097](/images/image-20210126104911097.png)

其中在**Source**部分中配置静态页面所在的分支与目录。设置完成后即可通过{UserName}.github.io访问。这里也可以使用自定义域名，详细配置见[Github官方指南](https://docs.github.com/articles/using-a-custom-domain-with-github-pages/)。

## Github Actions

> Github Actions是Github推出的自动构建、测试、部署的工作流。类似于Gitlab的CI/CD或Jenkins。可以通过Github的pull、merge、issue等动作触发。

有了Github Pages，我们只需要将Hexo生成的静态页面推送至我们配置了Github Pages对应的仓库即可。如果我们采用手动操作，有如下几个步骤：

1. 本地安装Hexo并初始化我们的博客（仅需一次）
2. 写点什么
3. 通过NPM+Hexo生成静态页面
4. 将静态页面所在文件夹init为一个git文件夹，并关联到我们创建好的Github仓库，并推送

而采用Github Actions，我们仅需将这几个动作写成脚本，自动化即可。

#### 初始化本地Hexo博客

这部分不详细说明Hexo的使用，可见[Hexo官方指南](https://hexo.io/zh-cn/docs/)，非常详细且有中文。

```shell
$ npm install -g hexo-cli
$ hexo init <目录名>
$ cd <目录名>
$ npm install
```

之后我们可以添加几篇初始内容。

```shell
$ hexo new "文章标题"
$ hexo new page --path about/me "页面标题"
```

之后，我们本地运行Hexo查看下效果。

```shell
$ hexo server
```

至此，我们已经在本地完成了Hexo的初始化。之后将其推送至一个新的Github仓库，作为源码仓库。

#### 配置Actions自动部署

重点来了，之后我们要配置Github Actions，让源码仓库在有新的push内容时，自动生成静态页面，并将静态页面推送至Github Pages仓库。

首先，为了让Actions拥有向Pages仓库push内容的权限，我们需要在我们的Github账号中新建一个Personal Access Token。前往个人Settings - Developer settings - Personal access token。并创建一个新的Token。

![image-20210126111006651](/images/image-20210126111006651.png)

![image-20210126111142998](/images/image-20210126111142998.png)

之后，我们需要将这个新创建的token添加至博客源码仓库。前往源码仓库的设置页面：Settings-Secret, 添加一个新的 repository secret。

![image-20210126111338255](/images/image-20210126111338255.png) 

完成之后我们就可以开始编写Actions脚本了。添加Action的方法为，在源码仓库中，新建 `.github/workflows`目录。并在其中创建一个`yml`文件，文件名可自由定义。这个`yml`文件就是自动构建的脚本，具体的编写可见[Github官方指南](https://docs.github.com/en/actions/learn-github-actions)。这里放出本博客的构建脚本并进行说明：

```yml
name: Garys blog deployment #自动构建工作流名称
on: 
  push:
    branches:
      - "master" 
      #触发器，详见https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#onpushpull_requestpaths
jobs: # 构建任务列表，一个工作流中可有多个构建任务
  generate_public: # 构建任务1
    name: generate & deploy
    runs-on: ubuntu-latest # 定义运行环境，这里选择ubuntu
    steps: # 任务步骤列表，本任务中有4项任务
    
      # 1. 将本仓库代码拉至当前工作目录，这里使用了Github提供的预定义Action，可在 https://github.com/actions/checkout 查看其源码，本质上是通过git拉取当前仓库指定分支的代码。
      - name: Checkout
        uses: actions/checkout@v2.3.1
        with:
          persist-credentials: false
          
      # 2. 通过npm安装项目依赖，这里直接通过run运行npm命令
      - name: Setup Hexo
        run: |
          npm install
          
      # 3. 通过npm生成hexo静态页面
      - name: Generate
        run: |
          npm run-script build
          
      # 4. 将生成的静态页面目录推送至Github pages仓库。这里使用了Github pages部署action，源码与文档见 https://github.com/JamesIves/github-pages-deploy-action ，其通过Git将指定目录推送至指定仓库。
      - name: Deploy
        uses: JamesIves/github-pages-deploy-action@3.7.1
        with:
          GITHUB_TOKEN: ${{ secrets.BLOG_DEPLOYMENT }} # 刚才配置的Repository secret，secrets后的名称与刚才配置的名称一致。
          BRANCH: master # 目标分支
          FOLDER: public # 静态页面所在当前残酷的目录
          REPOSITORY_NAME: GaryXiongxiong/Myblog # 目标仓库
          GIT_CONFIG_NAME: GaryXiongxiong
          GIT_CONFIG_EMAIL: i@jiangyixiong.top
          CLEAN: true
```

配置完成后，只需把我们更新后的源码仓库推送至Github，之后便可在Github仓库页面的Actions标签中查看构建日志：

![image-20210126113749255](/images/image-20210126113749255.png)

构建成功后，我们便可通过我们Github pages的链接查看更新后的博客啦！