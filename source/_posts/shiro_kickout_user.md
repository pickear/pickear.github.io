title: '我的shiro之旅: 十二shiro踢出用户(同一用户只能一处登录)'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 11:57:00
---
看了一下官网，没有找到关于如何控制同一用户只能一处登录的介绍，网上也没有找到相关的文章。可能有些人会记录用户的登录信息，然后达到踢出用户的效果。这里介绍一个更简单的方法。

如果我们跟shiro的源码，我们可以看到。当用户登录成功后，shiro会把用户名放到session的attribute中，key为DefaultSubjectContext_PRINCIPALS_SESSION_KEY，这个key的定义是在shiro的org.apache.shiro.subject.support.DefaultSubjectContext中，这个类有三个public的静态属性，其他都为private。其中PRINCIPALS_SESSION_KEY这个属性记录的是用户名，而AUTHENTICATED_SESSION_KEY属性记录的是用户认证，当用户登录成功后，这个attribute的值是true。

曾经我想把AUTHENTICATED_SESSION_KEY这个attribute的值设置为false，表示用户是退出状态，这样达到退出用户的目的，不过没有成功，shiro判断用户是否是登录状态并不从这里判断。不过既然我们可以通过用户名可以找到用户对应的session，也很容易将该session删除，让用户重新建立一个新的sesison。这里给出一个帮助类，带有跳出用户的功能。

```java

package com.concom.security.infrastructure.helper;

import java.util.Collection;

import org.apache.commons.lang.StringUtils;
import org.apache.shiro.SecurityUtils;
import org.apache.shiro.session.Session;
import org.apache.shiro.session.mgt.eis.SessionDAO;
import org.apache.shiro.subject.PrincipalCollection;
import org.apache.shiro.subject.Subject;
import org.apache.shiro.subject.support.DefaultSubjectContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.concom.lang.helper.TimeHelper;
import com.concom.security.application.memcache.CurrentUserMemcacheService;
import com.concom.security.application.user.UserService;
import com.concom.security.domain.user.User;

/**
 * @author Dylan
 * @time 2013-8-12
 */
public class ShiroSecurityHelper {
	
	private final static Logger log = LoggerFactory.getLogger(ShiroSecurityHelper.class);

	private static UserService userService;

	private static CurrentUserMemcacheService currentUserMemcacheService;
	
	private static SessionDAO sessionDAO;

	/**
	 * 把user放到cache中
	 * 
	 * @param user
	 */
	public static void setUser(User user) {
		currentUserMemcacheService.save(user);
	}

	/**
	 * 清除当前用户的缓存
	 */
	public static void clearCurrentUserCache() {
		if (hasAuthenticated()) {
			currentUserMemcacheService.remove(getCurrentUsername());
		}
	}

	/**
	 * 从cache拿当前user
	 * 
	 * @return
	 */
	public static User getCurrentUser() {
		if (!hasAuthenticated()) {
			return null;
		}
		User user = currentUserMemcacheService.get(getCurrentUsername());
		try {
			if (null == user) {
				user = userService.getByUsername(getCurrentUsername());
				ShiroSecurityHelper.setUser(user);
			}
			return user;
		} catch (Exception e) {
			e.printStackTrace();
			return null;
		}
	}

	/**
	 * 获得当前用户名
	 * 
	 * @return
	 */
	public static String getCurrentUsername() {
		Subject subject = getSubject();
		PrincipalCollection collection = subject.getPrincipals();
		if (null != collection && !collection.isEmpty()) {
			return (String) collection.iterator().next();
		}
		return null;
	}

	/**
	 * 
	 * @return
	 */
	public static Session getSession() {
		return SecurityUtils.getSubject().getSession();
	}

	/**
	 * 
	 * @return
	 */
	public static String getSessionId() {
		Session session = getSession();
		if (null == session) {
			return null;
		}
		return getSession().getId().toString();
	}
	
	/**
	 * @param username
	 * @return
	 */
	public static Session getSessionByUsername(String username){
		Collection<Session> sessions = sessionDAO.getActiveSessions();
		for(Session session : sessions){
			if(null != session && StringUtils.equals(String.valueOf(session.getAttribute(DefaultSubjectContext.PRINCIPALS_SESSION_KEY)), username)){
				return session;
			}
		}
		return null;
	}
	
	/**踢除用户
	 * @param username
	 */
	public static void kickOutUser(String username){
		Session session = getSessionByUsername(username);
		if(null != session && !StringUtils.equals(String.valueOf(session.getId()), ShiroSecurityHelper.getSessionId())){
			ShiroAuthorizationHelper.clearAuthenticationInfo(session.getId());
			log.info("############## success kick out user 【{}】 ------ {} #################", username,TimeHelper.getCurrentTime());
		}
	}

	/**
	 * @param userService
	 * @param currentUserMemcacheService
	 * @param sessionDAO
	 */
	public static void initStaticField(UserService userService,CurrentUserMemcacheService currentUserMemcacheService,SessionDAO sessionDAO){
		ShiroSecurityHelper.userService = userService;
		ShiroSecurityHelper.currentUserMemcacheService = currentUserMemcacheService;
		ShiroSecurityHelper.sessionDAO = sessionDAO;
	}
	
	/**
	 * 判断当前用户是否已通过认证
	 * @return
	 */
	public static boolean hasAuthenticated() {
		return getSubject().isAuthenticated();
	}

	private static Subject getSubject() {
		return SecurityUtils.getSubject();
	}


}
```
再通过spring注入帮助类需要的静态属性。

```xml

<bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
		<property name="staticMethod" value="com.concom.security.infrastructure.helper.ShiroSecurityHelper.initStaticField" />
		<property name="arguments">
			<list>
				<ref bean="userService"/>
				<ref bean="currentUserMemcacheService"/>
				<ref bean="sessionDAO"/>
			</list>
		</property>
</bean>
```
读者可以不关注userService，currentUserMemcacheService，其中踢出功能用到了SessionDAO，定义可能看我的shiro之旅: 七 shiro session 共享，文章里的spring配置文件有介绍，这里不再作描述。从这个类名我们就可以猜想到，这个类是用户管理session的。ShiroAuthorizationHelper这个帮助类在我的shiro之旅: 九 shiro 清理缓存的权限信息这篇文章介绍到。通过用户名使用用户对应的session，然后将该session删除，这个kickOutUser方法应该在用户登录之前调用。

session删除后，当用户再请求服务器时，服务端shiro会抛出there is no session的异常，然后从新为请求建立一个新的session，就像用户很长时间没有点击浏览器，shiro的定时器定时将失效的session清除的时候也抛出这个异常一样。不过这个对用户是透明的，对用户的体验没有影响。

这是其中的一种方式，也许有更好的实现方式。如果读者有更好的实现方式，希望能与我分享。