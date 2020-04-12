title: Hexo博客：(7)给NexT主题添加最新文章模块
author: Dylan
tags:
  - hexo
  - NexT
categories:
  - hexo
date: 2020-04-12 13:44:00
---
#### 插入代码
首先我们找到侧边栏模块 next/layout/_macro/sidebar.swig ,这个负责渲染侧边栏。在sidebar.swig的if theme.links的end if后面添加以下代码:
```shell
{% if theme.recent_posts %}
<div class="links-of-blogroll motion-element {{ "links-of-blogroll-" + theme.recent_posts_layout  }}">
  <div class="links-of-blogroll-title">
	<!-- modify icon to fire by szw -->
	<i class="fa fa-history fa-{{ theme.recent_posts_icon | lower }}" aria-hidden="true"></i>
	{{ theme.recent_posts_title }}
  </div>
  <ul class="links-of-blogroll-list">
	{% set posts = site.posts.sort('-date') %}
	{% for post in posts.slice('0', '5') %}
	  <li>
		<a href="{{ url_for(post.path) }}" title="{{ post.title }}" target="_blank">{{ post.title }}</a>
	  </li>
	{% endfor %}
  </ul>
</div>
{% endif %}
```
#### 添加配置
在NexT主题目录下的_config.yaml配置文件，添加下面配置:
```
recent_posts_title: 最新文章
recent_posts_layout: block
recent_posts: true
```
配置完后，清一下Hexo缓存，再访问<http://localhost:4000>验证。
```shell
$ hexo clean & hexo g & hexo s
```

![图片](/images/blog/recent_posts.png)