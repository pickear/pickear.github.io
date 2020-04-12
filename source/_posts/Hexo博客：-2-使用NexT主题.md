title: Hexo博客：(2)使用NexT主题
author: Dylan
tags:
  - hexo
  - NexT
categories:
  - hexo
date: 2018-08-23 10:13:00
---
## 安装NexT
Hexo 安装主题的方式非常简单，只需要将主题文件拷贝至站点目录的 themes 目录下， 然后修改下配置文件即可。具体到 NexT 来说，安装步骤如下。[NexT文档](http://theme-next.iissnan.com/getting-started.html)  [NexT的Git](https://github.com/iissnan/hexo-theme-next)

#### 下载主题
使用Git来克隆NexT主题到本地，如果要更新，通过git pull快速更新方式。
```shell
$ cd blog  ##blog为hexo博客目录，看[Hexo博客：(1)安装]。
$ git clone https://github.com/iissnan/hexo-theme-next themes/next
```
#### 启用主题
与所有 Hexo 主题启用的模式一样。 当 克隆/下载 完成后，打开Hexo文件(_config.yml)， 找到 theme 字段，并将其值更改为 next。
```
theme: next
```
配置完后，清一下Hexo缓存，再访问<http://localhost:4000>验证。
```shell
$ hexo clean & hexo g & hexo s
```
![NexT](/images/blog/NexT_index.png)
#### 主题设定
目前NexT 支持三种 Scheme，他们是：
* Muse - 默认 Scheme，这是 NexT 最初的版本，黑白主调，大量留白
* Mist - Muse 的紧凑版本，整洁有序的单栏外观
* Pisces - 双栏 Scheme，小家碧玉似的清新
编辑NexT配置文件(themes/next/_config.yml)，索引Scheme，你会看到有三行 scheme 的配置，将你需用启用的 scheme 前面注释 # 去除即可。
```
#scheme: Muse
#scheme: Mist
scheme: Pisces
```
然后再清缓存重新生成静态文件，验证Scheme:
```shell
$ hexo clean & hexo g & hexo s
```
#### 装饰你的主题
##### 设置字段
编辑Hexo配置文件(_config.yml)，找到 language ,将 language 设置成你所需要的语言。例如中文用zh-Hans:
```
language: zh-Hans
```
NexT支持的语言如下:

|语言|代码|设定示例|
|:---:|:--:|:-----:|
|English|en|language: en|
|简体中文|zh-Hans|language: zh-Hans|
|Français|fr-FR|language: fr-FR|
|Português|pt|language: pt or language: pt-BR|
|繁體中文|zh-hk 或者 zh-tw|language: zh-hk|
|Русский язык|ru|language: ru|
|Deutsch|de|language: de|
|日本語|ja|language: ja|
|Indonesian|id|language: id|
|Korean|ko|language: ko|

##### 设置菜单
修改NexT配置文件(themes/next/_config.yml)，找到menu，修改以下内容:
```
menu:
  home: /
  archives: /archives
  #about: /about
  #categories: /categories
  tags: /tags
  #commonweal: /404.html
```
NexT 默认的菜单项有（标注!的项表示需要手动创建这个页面）：


|键值|设定值|显示文本（简体中文）|
|:---:|:--:|:-------------------:|
|home|home: /|主页|
|archives|archives: /archives|归档页|
|categories|categories: /categories|分类页！|
|tags|tags: /tags|标签页 ！|
|about|about: /about|关于页面 ！|
|commonweal|commonweal: /404.htm|公益 404！|

菜单显示的文字，将根据NexT的languages目录(themes/next/languages/)下言语的定义文件{languages}.yml

categories，tags，about，commonweal都需要自己手动创建的。通过hexo new page来创建，例如创建categories:
```shell
$ hexo new page categories
```
然后编辑source/categories/index.md。注意,title默认为categories，可以删除，否则在该页面会显示"categories"字样。comments表示是否允许评论。:
```
---
title:
date: 2018-08-22 13:45:58
type: "categories"
comments: false
---
```
##### 设置头像
修改Hexo配置文件(_config.yml)，找到avatar，将值设置成头像地址。其中，头像的链接地址可以是：

|地址|值|
|:-:|:-:|
|完整的互联网 URI|http://example.com/avatar.png|
|站点内的地址|将头像放置主题目录下的 source/uploads/（新建 uploads 目录若不存在）,配置为：avatar: /uploads/avatar.png。或者 放置在 source/images/ 目录下 ,配置为：avatar: /images/avatar.png|

##### 设置昵称
修改Hexo配置文件(_config.yml)，找到author，将值设置为你的昵称。

##### 站点描述
修改Hexo配置文件(_config.yml)，找到description ，将值设置为你的站点描述。站点描述可以是你喜欢的一句签名。

##### 首页只显示预览
默认情况下，NexT首页会显示所有博文的全文。每一篇博文在首页都显示全文，使首页显示的篇幅帮长。要想拿首页只显示每一篇博文的预览，只需要修改NexT配置文件(themes/next/_config.yml)，找到auto_excerpt，将enable改为true：
```
auto_excerpt:
  enable: false
  length: 150
```
##### 站点建立时间
修改NexT配置文件(themes/next/_config.yml)，找到since，设置值如下:
```
since: 2018
```
##### 版权
修改NexT配置文件(themes/next/_config.yml)，找到copyright，设置值如下:
```
copyright: Dylan 版权所有
```
##### 取消power by
在网页底部有"Hexo 强力驱动"等字样，如果想去掉，修改NexT配置文件(themes/next/_config.yml)，找到powered,设置值如下:
````
powered: false

theme:
  # Theme & scheme info link (Theme - NexT.scheme).
  enable: false
 # Version info of NexT after scheme info (vX.X.X).
   version: false
````
##### 添加访客数
修改NexT配置文件(themes/next/_config.yml)，找到busuanzi_count，将enable值改为true:
```
busuanzi_count:
  # count values only if the other configs are false
  enable: true
  # custom uv span for the whole site
  site_uv: true
  site_uv_header: <i class="fa fa-user"></i>
  site_uv_footer:
  # custom pv span for the whole site
  site_pv: true
  site_pv_header: <i class="fa fa-eye"></i>
  site_pv_footer:
  # custom pv span for one page only
  page_pv: false
  page_pv_header: <i class="fa fa-file-o"></i>
  page_pv_footer:
```
这里将page_pv设置为false，即不为每篇博文统计访问量。博文的访问量通过LeanCloud来实现，请看 [Hexo博客：(4)添加服务](http://blog.gogl.top/2018/08/24/Hexo%E5%8D%9A%E5%AE%A2%EF%BC%9A-4-%E6%B7%BB%E5%8A%A0%E6%9C%8D%E5%8A%A1/)
##### 添加GitHub,Email
在NexT主题的配置文件_config.yaml打开social，将GitHub,E-Mail前面的#号去掉，如:
```
social:
GitHub: https://github.com/pickear || github
E-Mail: mailto:pickear@gmail.com || envelope
```