title: '我的shiro之旅: 三浅谈shiro的filter'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 10:59:00
---
前段时间比较懒，项目也有些紧，没有写什么东西。现在再对Shiro做一些整理。上一篇主要介绍了一个完整而又简单的shiro集成到项目的例子，主要是spring项目。这篇文章，想谈一下关于shiro的filter，这需要读者对shiro有一定的理解，至少有用过shiro。

```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<property name="securityManager" ref="securityManager" />
		<property name="loginUrl" value="/login" />
		<property name="successUrl" value="/home" />
		<property name="unauthorizedUrl" value="/unauthorized" />
		<!-- The 'filters' property is usually not necessary unless performing 
			an override, which we want to do here (make authc point to a PassthruAuthenticationFilter 
			instead of the default FormAuthenticationFilter: -->
		<!-- <property name="filters">
			<map>
				<entry key="authc">
					<bean class="org.apache.shiro.web.filter.authc.PassThruAuthenticationFilter" />
				</entry>
			</map>
		</property> -->
		<property name="filterChainDefinitions">
			<value>
				/admin = authc,roles[admin]
				/edit = authc,perms[admin:edit]
				/home = user
				/** = anon
			</value>
		</property>
	</bean>
```
从上面的配置我们可以看到，当用户没有登录的时候，会重发一个login请求，引导用户去登录。当然，这个login请求做些什么工作，引导用户去那里，完全由开发者决定。successUrl是当用户登录成功，重发home请求，引导用户到主页。unauthorizedUrl指如果请求失败，重发/unauthorized请求，引导用户到认证异常错误页面。

|Filter Name|Class|
|:---------:|:---:|
|anon|org.apache.shiro.web.filter.authc.AnonymousFilter|
|authc|org.apache.shiro.web.filter.authc.FormAuthenticationFilter|
|authcBasic|org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter|
|logout|org.apache.shiro.web.filter.authc.LogoutFilter|
|noSessionCreation|org.apache.shiro.web.filter.session.NoSessionCreationFilter|
|perms|org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter|
|port|org.apache.shiro.web.filter.authz.PortFilter|
|rest|org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter|
|roles|org.apache.shiro.web.filter.authz.RolesAuthorizationFilter|
|ssl|org.apache.shiro.web.filter.authz.SslFilter|
|user|org.apache.shiro.web.filter.authc.UserFilter|

以上是shiro的一些Filter，如我们在filterChainDefinitions里配置了/admin=authc,roles[admin]，那么/admin这个请求会由org.apache.shiro.web.filter.authc.FormAuthenticationFilter和org.apache.shiro.web.filter.authz.RolesAuthorizationFilter这两个filter来处理，其中authc,roles只是filter的别名。如要更改别名，可以通过filters来改变。如上面的配置

```xml
		<property name="filters">
			<map>
				<entry key="authc">
					<bean class="org.apache.shiro.web.filter.authc.PassThruAuthenticationFilter" />
				</entry>
			</map>
		</property>
<property name="filters">
			<map>
				<entry key="authc">
					<bean class="org.apache.shiro.web.filter.authc.PassThruAuthenticationFilter" />
				</entry>
			</map>
		</property>
```
把PassThruAuthenticationFilter添加别名为authc，这时/admin请求会交给PassThruAuthenticationFilter处理，替换了原来由FormAuthenticationFilter来处理。

由此一来，如果有些特殊的请求需要特殊处理，就可以自己写一个filter，添加一个别名，如：

```xml
		<property name="filters">
			<map>
				<entry key="new">
					<bean class="org.xx.xx.NewFilter" />
				</entry>
			</map>
		</property>
<property name="filters">
			<map>
				<entry key="new">
					<bean class="org.xx.xx.NewFilter" />
				</entry>
			</map>
		</property>
```
请求用/new = new，这样/new请求交由NewFilter来处理。