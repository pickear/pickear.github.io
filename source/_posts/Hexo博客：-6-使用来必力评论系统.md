title: Hexo博客：(6)使用来必力评论系统
author: Dylan
tags:
  - livere
  - 来必力
categories:
  - 技术杂粹
date: 2018-08-28 09:21:00
---
### 前方
NexT支持的评论系统有很多，例如:Hypercomments,畅言，友言，来必力，网易云跟帖,valine,gitment，gitalk等，网上也都有很多资料。但有些评论系统现在已经关闭了。这里，介绍来必力评论系统如何集成在NexT上。

1. 注册帐号
首先，要到 [来必力官网](https://livere.com) 注册个帐号。

2. 获取data-uid
注册完登录后，选择"安装",选择"City"免费版。
![livere_install](/images/blog/livere_install.png)

"安装"完之后，来到"代码管理",获取data-uid。
![livere_install](/images/blog/livere_code.png)

3. 配置NexT
修改NexT配置文件(themes/next/_config.yml)，找到livere_uid,将获得的data-uid填写到livere_uid后面:

```
livere_uid: MTAyMC8zEWERzNi8xNTY2Mw==
```
4. 重启查看
```shell
$ hexo clean & hexo g
```
![livere_install](/images/blog/livere_comment.png)