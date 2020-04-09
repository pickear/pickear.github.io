title: '我的shiro之旅: 十自定义shiro的SessionIdCookie'
author: Dylan
date: 2018-09-16 11:33:38
tags:
---
在使用shiro的时候，曾经有段时间很苦恼，因为我cookie的sessionId经常无故被改，然后抛There is no session with id [xxxx]的异常。我们知道，当请求过来，shiro创建session后，会把sessionId写回到客户端的cookie中。每二个请求过来时，shiro会拿到这个cookie里的sessionId去查找，如果查找不到，就有可能抛There is no session with id的异常。通过抛这个异常的会有两种情况，一种就是我刚才说的，cookie被改写了，因为shiro默认的cookie名叫JSESSIONID，当在web.xml里配置的请求拦截配得不合理时，就有可能被改写。在我的项目中，因为有些静态资源配置了不拦截，就会被容器改写了这个cookie。后来，干脆把cookie的名字改了，不再叫JSESSIONID。还有一种就是浏览器打开很久都没有操作，然后shiro定时清理了不活动的session，这时浏览器再发请求过来，因为session已被清理，也会抛There is no session with id。更改cookie的名字方法如下：

```xml
<bean id="shiroSessionManager" class="org.apache.shiro.web.session.mgt.DefaultWebSessionManager">
		<property name="sessionDAO" ref="sessionDAO"/>
		<!-- <property name="sessionValidationScheduler" ref="shiroSessionValidationScheduler"/> -->
		<property name="sessionValidationInterval" value="1800000"/>  <!-- 相隔多久检查一次session的有效性 -->
		<property name="globalSessionTimeout" value="1800000"/>  <!-- session 有效时间为半小时 （毫秒单位）-->
		<property name="sessionIdCookie.domain" value=".xxx.com"/>
		<property name="sessionIdCookie.name" value="jsid"/>
		<property name="sessionIdCookie.path" value="/"/>
		<!-- <property name="sessionListeners">
			<list>
				<bean class="com.concom.security.interfaces.listener.SessionListener"/>
			</list>
		</property> -->
	</bean>
```
我们打开shiro的DefaultWebSessionManager类源码就可以知道，里面有个private Cookie sessionIdCookie;属性，这个就是sessionId的cookie。在DefaultWebSessionManager的构造器里，初始化的名字取的是ShiroHttpSession.DEFAULT_SESSION_ID_NAME，也就是"JSESSIONID"，在这里我们不去改源码，只要通过spring改sessionIdCookie的属性即可。

```java
public DefaultWebSessionManager() {
        Cookie cookie = new SimpleCookie(ShiroHttpSession.DEFAULT_SESSION_ID_NAME);
        cookie.setHttpOnly(true); //more secure, protects against XSS attacks
        this.sessionIdCookie = cookie;
        this.sessionIdCookieEnabled = true;
    }
```
在上面的xml配置中，写了sessionIdCookie.domain的值为.xxx.com，这个根据自己的项目而定。这里配置为的是某域名及其二级域名都能共享这个cookie。还有sessionIdCookie.name，这个就是cookie的名字，值为jsid。这样我们就可以看到，cookie会多了一个名字jsid，值为当前session的id的键值。在上面还配置了两个关于session有效期的配置，一个是sessionValidationInterval，表示shiro的定时器多久去检查一次session的有效性，一个是globalSessionTimeout，表示session的有效时长。当然，我们也完全可以息定义定时器，把定时器注入到DefaultWebSessionManager的sessionValidationScheduler属性中，只是个人觉得没这个必要。