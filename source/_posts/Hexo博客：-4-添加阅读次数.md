title: Hexo博客：(4)添加阅读次数
author: Dylan
date: 2018-08-24 13:30:31
tags:
  - hexo
  - leancloud
categories:
  - 技术杂粹
---
### 前言
NexT支持通过LeanCloud添加阅读次数统计。

1. 注册帐号
注册 [LeanCloud](https://leancloud.cn) 帐号并登录。

2. 创建应用
登录后，使用开发版的服务，创建一个应用。
![create_app](/images/blog/leancloud_create_app.png)

3. 获得app_key
应用创建成功后，app_key就会自动生成，在"设置"处可以获得app_key。
![app_setting](/images/blog/leancloud_setting_app.png)
![app_key](/images/blog/leancloud_app_key.png)

4. 创建Counter
class是用来存储数据的，LeanCloud提供了很多下划线开关的class，在这里我们需要创建一个名字为Counter的class。注意，名字必须为Counter，大小写一致，因为NexT里用的是这个名字。如果用其他名字，需要改NexT。
![counter](/images/blog/leancloud_counter.png)

5. 修改leancloud_visitors配置
编辑NexT的配置文件(themes/next/_config.yml)，找到leancloud_visitors，enable改为true,将获得的app_id，app_key配置到文件中，：
```
leancloud_visitors:
  enable: true
  app_id: xxxxxxxx
  app_key: xxxxxxxxxxxx
```

6. 重新生成静态文件
```shell
$ hexo clean & hexo g
```

7. 刷新页面
![read_count](/images/blog/leancloud_read_counter.png)

8. web安全
最后，因为我们的app_id,app_key都是直接暴露在外的。所以设置一下域名白名单提高安全性。
![web_safe](/images/blog/leancloud_web_safe.png)