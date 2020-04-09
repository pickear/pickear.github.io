title: Hexo博客：(1)安装
author: Dylan
tags:
  - hexo
categories:
  - hexo
date: 2018-08-22 23:09:00
---
## Hexo简介
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。这是官方对Hexo的描述。

## 安装
Hexo的安装( [文档](https://hexo.io/zh-cn/docs/) )依懒Git,Nodejs，首先要安装这两个工具。下面介绍在centos环境下安装Hexo过程。
* 安装Git

```shell
$ yum -y install git
```
* 安装Nodejs
```shell
$ yum -y install nodejs
```
* 安装Hexo
```shell
$ npm install -g hexo-cli
```
* 初始化Hexo
```shell
$ hexo init blog    ###blog可以随意命名
$ cd blog
$ npm install
```
当初始化完后，在blog目录下生成以下的目录结构:
```
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```
* 启动Hexo
在blog目录下执行以下命令:
```shell
$ hexo generate   ##生成静态文件，等同于hexo g
$ hexo server ##启动服务，等同于hexo s，可以在后面加-p xxx下指定端口
```
默认服务监听4000端口，访问: <http://localhost:4000/>，如果能正常访问，看到下面的界面，说明Hexo已经安装成功了。
![hexo](/images/blog/hexo_index.png)