title: Hexo博客：(7)Travis CI自动化编译发布
author: Dylan
tags:
  - hexo
  - Travis CI
categories:
  - 技术杂粹
date: 2020-04-19 07:44:00
---

前面有介绍过通过[hexo d](https://pickear.github.io/2018/08/23/Hexo%E5%8D%9A%E5%AE%A2%EF%BC%9A-3-%E4%B8%8A%E4%BC%A0%E5%88%B0github/) 命令来发布博文到github，到目前为主，前面的文章的前提是要自己有服务器。这篇文章将介绍，在没有服务器的前提下，怎么通过提交原码到github，用traci-ci持续集成工具自动编译发布博文。这些都将是全自动的，只要将博文提交到github就会自动发布。
#### (1)创建${username}.github.io仓库
前面的文章 [Hexo博客：(3)上传到github](https://pickear.github.io/2018/08/23/Hexo%E5%8D%9A%E5%AE%A2%EF%BC%9A-3-%E4%B8%8A%E4%BC%A0%E5%88%B0github/) 已经介绍过如何注册github和创建username.github.io仓库(其中username为github的帐号名，比如你的github帐号为pickear，那仓库为pickear.github.io，这是固定的)
#### (2)创建hexo分支
将username.github.io这个仓库checkout到本地。然后用打开命令提示符，进入仓库的要目录，用git命令创建并切换到hexo分支。
```shell
git checkout -b hexo
```
#### (3)添加hexo源码
首先初始化hexo，具体请看博文 [Hexo博客：(1)安装](https://pickear.github.io/2018/08/22/Hexo%E5%8D%9A%E5%AE%A2%EF%BC%9A-1-%E5%AE%89%E8%A3%85/) 。初始化完后，将hexo博客的源码拷贝到pickear.github.io仓库下，目录如下:
```
pickear.github.io
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```
拷贝完后，用git add命令将所有源码文件添加到git，并用git commit命令提交到本地仓库
```shell
git add .
git commit -m "初始化博客"
```

#### (4)github创建token
travis-ci要获取github的源码编译并将编译后的博文提交到pickear.github.io的master分支上，就需要github的相关权限。这里创建token就是给travis-ci使用的。在github的右上角帐号处Settings/Developer settings/Personal access tokens处通过"Generate new token"创建一个token。并记录下创建的token，因为以后是无法再显示了，只能重新创建。
![图片](/images/blog/github_token1.png)

![图片](/images/blog/github_token2.png)

#### (5)登录travis-ci
访问[travis-ci 官网](https://www.travis-ci.org/) 并通过github授权登录travis-ci。

![图片](/images/blog/travis-ci_login.png)

#### (6)同步仓库到travis-ci
在travis-ci的右上角帐号处，通过Settings可以看到所有github的仓库。如果看不到，可以通过左边的Sync acount来进行同步。然后在想要自动化编译的仓库处打开右边的开关。
![图片](/images/blog/travis-ci_settings.png)
之后在My Repositories就可以看到需要编译的仓库。
![图片](/images/blog/travis-ci_repositeries.png)

#### (7)设置travis-ci仓库
在My Repositories 选择pickear/pickear.github.io这个仓库。然后点右边的More options/Settings对仓库进行设置。创建GH_TOKEN变量，并将前面创建github的token作为变量值填进去。这个GH_TOKEN将在.travis.yml(travis-ci的编译配置文件，后面讲到)。
![图片](/images/blog/travis-ci_repositories_settings.png)

#### (8)创建.travis.yml
在本地仓库的要目录下，创建.travis.yml文件。文件里有个GH_TOKEN变量，就是上一步骤在travis-ci的仓库settings添加的GH_TOKEN变量。这里不直接将token写在.travis.yml是为了防止泄露，因为这个文件将上传到github上，所有人都可以看到。
```shell
language: node_js
node_js: stable

# 指定缓存模块，可选。缓存可加快编译速度。
cache:
  apt: true
  yarn: true
  directories:
    - node_modules
  
before_install:
    #此处的user.name的值请改为自己的github用户名
  - git config user.name "pickear"
    #此处的user.email的值请改为自己的github注册邮箱
  - git config user.email "xxxxxx@gmail.com"
  - curl -o- -L https://yarnpkg.com/install.sh | bash
  - export PATH=$HOME/.yarn/bin:$PATH
  - npm install -g hexo-cli
install:
  - yarn
script:
  - hexo clean
  - hexo generate

after_success:
  - cd ./public
  - git init
  - git add --all .
  - git commit -m "Travis CI Auto Builder"
    #此处的github.com/pickear/pickear.github.io.git请改为自己的仓库地址。GH_TOKEN变量为travis-ci后台配置的变量，值是github的token
  - git push --force --quiet "https://${GH_TOKEN}@github.com/pickear/pickear.github.io.git" master:master
# E: Build LifeCycle

branches:
  only:
    #此处就是github的hexo存放源码的分支名
    - hexo
```

#### (9)发布博客
添加.travis.yml添加到仓库，并将本地库存提交到远程github仓库中。
```shell
git add .
git commit -m "travis-ci持续集成"
git push origin hexo
```
如果提交正常，过一会，访问https://pickear.github.io/(这里指的是你的github.io地址)，就可以正常访问博客了。并且，在github的远程仓库master分支中，存在了travis-ci编译出来的静态文件。

#### (10)写博文
hexo的博文源文件是在source\_posts下的，我们可以在此目录下添加一个md文件，例如"Hexo博客：-8-Travis CI自动化编译发布.md"，然后在文件里写博文。博文的语法是markdown，博文前可以添加分类，标签，作者和日期等信息:
```markdown
title: Hexo博客：(7)Travis CI自动化编译发布
author: Dylan
tags:
  - hexo
  - Travis CI
categories:
  - 技术杂粹
date: 2020-04-19 07:44:00
---

前面有介绍过通过[hexo d](https://pickear.github.io/2018/08/23/Hexo%E5%8D%9A%E5%AE%A2%EF%BC%9A-3-%E4%B8%8A%E4%BC%A0%E5%88%B0github/) 命令来发布博文到github，到目前为主，前面的文章的前提是要自己有服务器。这篇文章将介绍，在没有服务器的前提下，怎么通过提交原码到github，用traci-ci持续集成工具自动编译发布博文。这些都将是全自动的，只要将博文提交到github就会自动发布。
#### (1)创建${username}.github.io仓库
前面的文章 [Hexo博客：(3)上传到github](https://pickear.github.io/2018/08/23/Hexo%E5%8D%9A%E5%AE%A2%EF%BC%9A-3-%E4%B8%8A%E4%BC%A0%E5%88%B0github/) 已经介绍过如何注册github和创建username.github.io仓库(其中username为github的帐号名，比如你的github帐号为pickear，那仓库为pickear.github.io，这是固定的)
#### (2)创建hexo分支
将username.github.io这个仓库checkout到本地。然后用打开命令提示符，进入仓库的要目录，用git命令创建并切换到hexo分支。
```
添加完后，再按照前面的"发布博客"的步骤进行发布就可以。至于怎么样装修博客，使用NexT主题，添加评论模块，添加阅读数，可能参照前面的hexo相关博文。
![图片](/images/blog/pickear.github.io.png)