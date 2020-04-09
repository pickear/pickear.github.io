title: '我的shiro之旅:一 shiro简介'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 10:44:00
---
前段时间，因为项目需要，用shiro搭建了一个权限系统。现在项目已完成，希望通过以文章的形式，对shiro进行一些总结。

也希望在总结过程中，对shiro有更深刻的认识。首先，对shiro进行一个简单介绍。

##### 一 什么是shiro
Shiro是一个强大易用的Java安全框架，提供了认证、授权、加密和会话管理功能，可为任何应用提供安全保障.

##### 二 shiro的几个特性
易于使用 - 易用性是这个项目的最终目标。应用安全有可能会非常让人糊涂，令人沮丧，并被认为是“必要之恶”。若是能让它简化到新手都能很快上手，那它将不再是一种痛苦了。
广泛性 - 没有其他安全框架可以达到Apache Shiro宣称的广度，它可以为你的安全需求提供“一站式”服务。
灵活性 - Apache Shiro可以工作在任何应用环境中。虽然它工作在Web、EJB和IoC环境中，但它并不依赖这些环境。Shiro既不强加任何规范，也无需过多依赖。
Web能力 - Apache Shiro对Web应用的支持很神奇，允许你基于应用URL和Web协议（如REST）创建灵活的安全策略，同时还提供了一套控制页面输出的JSP标签库。
可插拔 - Shiro干净的API和设计模式使它可以方便地与许多的其他框架和应用进行集成。你将看到Shiro可以与诸如Spring、Grails、Wicket、Tapestry、Mule、Apache Camel、Vaadin这类第三方框架无缝集成。
支持 - Apache Shiro是Apache软件基金会成员，这是一个公认为了社区利益最大化而行动的组织。项目开发和用户组都有随时愿意提供帮助的友善成员。像Katasoft这类商业公司，还可以给你提供需要的专业支持和服务。
##### 三 shiro几个重要概念

###### Subject 
在考虑应用安全时，你最常问的问题可能是"当前用户是谁"，或"当前用户允许做x吗"。当我们写代码或设计用户界面时，问自己这些问题很平常：应用通常都是基于用户有事构建的，并且你希望功能描述是基于每个用户的。所以，对于我们而言，考虑应用安全的最自然方式就是基于当前用户。shiro用Subject概念从根本上体现了这种思考方式。

在shiro中，要得到Subject非常容易，shiro提供了一个工具类，让应用很容易得到当前用户，代码:

```java
Subject currentUser = SecurityUtils.getSubject();
```
###### SecurityManager 
Subject的“幕后”推手是SecurityManager.Subject代表了当前用户的安全操作，SecurityManager则管理所有用户的安全操作。它是Shiro框架的核心，充当“保护伞”，引用了多个内部嵌套安全组件，它们形成了对象图。但是，一旦SecurityManager及其内部对象图配置好，它就会退居幕后，应用开发人员几乎把他们的所有时间都花在Subject API调用上。

###### Realms 
Shiro的第三个也是最后一个概念是Realm。Realm充当了Shiro与应用安全数据间的“桥梁”或者“连接器"。也就是说，当切实与像用户帐户这类安全相关数据进行交互，执行认证（登录）和授权（访问控制）时，Shiro会从应用配置的Realm中查找很多内容。