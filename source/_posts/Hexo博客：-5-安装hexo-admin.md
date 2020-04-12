title: Hexo博客：(5)安装hexo-admin
author: Dylan
tags:
  - hexo
  - hexo-admin
categories:
  - 技术杂粹
date: 2018-08-25 13:43:00
---
### 前言
搭起了hexo博客，第一件事当然是要写几篇博文。hexo可以通过命令创建博文，然后找到对应的markdown文件，写入博文的内容然后发布。这样子多少都有点显得不方便。一来，需要登到服务器;二来，直接在服务器编辑markdown文件不够直观。有没有更方便的方式，比如有个后台，可以登录到后台直接编辑博文然后发布？hexo-admin插件就是这样的一个角色。

### 安装
1. 安装hexo-admin
```shell 
$ cd blog   ##blog目录为Hexo的要目录
$ npm install --save hexo-admin
$ hexo s
```
2. 获取password_hash

此时，访问http://127.0.0.1:4000/admin，不需要用户密码就可以登录。(注意：此处要用127.0.0.1本地方法才不需要用户密码登录，如果是通过域名的方式不行)。
![login](/images/blog/hexo_admin_login.png)

进入后面后，选择"setting"菜单，按图勾选两个复选框，点击"Setup authentification here"
![setup](/images/blog/hexo_admin_setup.png)

填写username,password,seret等信息，获得password_hash。
![setup2](/images/blog/hexo_admin_setup2.png)

3. 同步到github脚本
在写完博文发布(deploy)的时候，同时希望推送到github。我们可以写一个脚本，让deploy之后执行脚本，推送到github:
```shell
$ cd blog
$ mkdir admin_script
$ touch admin_script/hexo-generate.sh
```
编辑hexo-generate.sh脚本，写上以下内容:
```
#!/usr/bin/env sh
hexo clean && hexo g && hexo d
```
4. 配置hexo
编辑hexo配置文件(_config.yml)，将步骤2所获得的信息添加到配置文件中。内容如下:

```
admin:
   username: dylan
   password_hash: weourwrj9888083482348238432480
   secret: hello dylan
   deployCommand: './admin_script/hexo-generate.sh'
```
5. 写博文
访问http://127.0.0.1:4000/admin，用户名密码登录。然后在Posts菜单点击"New Posts"，创建一篇博文。
![new_posts](/images/blog/hexo_admin_new_posts.png)
默认情况下，博文是以草稿(_drafts)的形式编写，如果编写完后要发布正文，将_drafts改为_posts即可。右上角设置按钮可以为博文添加分类和标签。如图:
![new_posts](/images/blog/hexo_admin_edit_posts.png)

6. 发布博文
按照步骤5将_drafts改为_posts后，点右上角的勾。然后，选中"Deploy"菜单，直接点击"deploy"。
![deploy](/images/blog/hexo_admin_deploy_posts.png)
在此，一篇博文写好并发布了。同时，这篇博文也会同步到github上。
