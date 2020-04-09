title: '我的shiro之旅: 九shiro清理缓存的权限信息'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 11:31:00
---
在文章八讲到了shiro缓存权限信息然后达到共享目的，不过存在一个问题，当用户的权限发生改变的时候，需要用户重新登录，从新缓存用户权限信息。这篇文章将介绍在改变用户的权限时，如何清理用户的权限。我这里写了一个帮助类，先贴也代码:
```java
package com.concom.security.infrastructure.helper;

import java.io.Serializable;

import org.apache.shiro.SecurityUtils;
import org.apache.shiro.cache.Cache;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.concom.security.infrastructure.shiro.ShiroRealm;
import com.concom.security.infrastructure.shiro.cache.ShiroCacheManager;

/**
 * @author Dylan
 * @time 2014年1月8日
 */
public class ShiroAuthorizationHelper {

	/**
	 * 
	 */
	private static ShiroCacheManager cacheManager;
	
	private static Logger log = LoggerFactory.getLogger(ShiroAuthorizationHelper.class);
	
	
	/**
	 * 清除用户的授权信息
	 * @param username
	 */
	public static void clearAuthorizationInfo(String username){
		if(log.isDebugEnabled()){
			log.debug("clear the " + username + " authorizationInfo");
		}
		//ShiroCasRealm.authorizationCache 为shiro自义cache名(shiroCasRealm为我们定义的reaml类的类名)
		Cache<Object, Object> cache = cacheManager.getCache(ShiroRealm.REALM_NAME+".authorizationCache");
		cache.remove(username);
	}
	
	/**
	 * 清除当前用户的授权信息
	 */
	public static void clearAuthorizationInfo(){
		if(SecurityUtils.getSubject().isAuthenticated()){
			clearAuthorizationInfo(ShiroSecurityHelper.getCurrentUsername());
		}
	}
	
	/**清除session(认证信息)
	 * @param JSESSIONID
	 */
	public static void clearAuthenticationInfo(Serializable JSESSIONID){
		if(log.isDebugEnabled()){
			log.debug("clear the session " + JSESSIONID);
		}
		//shiro-activeSessionCache 为shiro自义cache名，该名在org.apache.shiro.session.mgt.eis.CachingSessionDAO抽象类中定义
		//如果要改变此名，可以将名称注入到sessionDao中，例如注入到CachingSessionDAO的子类EnterpriseCacheSessionDAO类中
		Cache<Object, Object> cache = cacheManager.getCache("shiro-activeSessionCache");
		cache.remove(JSESSIONID);
	}

	public static ShiroCacheManager getCacheManager() {
		return cacheManager;
	}

	public static void setCacheManager(ShiroCacheManager cacheManager) {
		ShiroAuthorizationHelper.cacheManager = cacheManager;
	}
	
	
}
```

上面的ShiroAuthorizationHelper通过spring把ShiroCacheManager注入了进来，并提供了两个静态方法。分别是clearAuthorizationInfo(String username)方法，根据用户名清空用户权限信息，clearAuthenticationInfo(String JSESSIONID)，根据sessionId来清除session。上篇文章有提到过ShiroCasRealm.authorizationCache和shiro-activeSessionCache这两个Cache的名称，在这里不再重复。如果有不清楚的，请看回上一篇文章。

因为ShiroAuthorizationHelper是个帮助类，方法是静态方法，可以利用spring来把对象注入到帮助类中。添加如下配置。

```xml
        <bean class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
		<property name="staticMethod" value="com.concom.security.infrastructure.helper.ShiroAuthorizationHelper.setCacheManager" />
		<property name="arguments" ref="shiroCacheManager"/>
	</bean>
```
将shiroCacheManager注入到ShiroAuthorizationHelper的静态方法中。这样，通过这个工具类，就可以直接调用它的静态方法去清理缓存。当权限缓存被清理后，shiro需要授权时，查找缓存没有权限信息，将会再次调用realm的doGetAuthorizationInfo去load用户的权限，再次放入缓存中。realm的实现在文章五中有介绍，这里不再重复，不清楚的可以看看文章五和文章二。这篇文章也讲到这里。谢谢。