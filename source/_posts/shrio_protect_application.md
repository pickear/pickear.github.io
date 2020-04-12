title: '我的shiro之旅:二 让Shiro保护你的应用'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 10:54:00
---
上一篇文章只是对Shiro作了一个简单的介绍，接下来的内容，将会介绍如何将shiro集成到应用中。这篇文章只要是介绍shiro跟spring项目的集成，以后也会写一些关于Shiro集成到普通web项目的一些文章。这些用到的是目前shiro的最新版本1.2.2，spring也是目前最前版本3.2.3.RELEASE，项目的依赖用的是mavne管理。spring的依赖这里就不作介绍了，相信有兴趣看这篇文件的童鞋都搭建过spring的项目，这里给出3个Shiro的依赖。在pom.xml加上以下的依赖:
```xml
<dependency>
	<groupId>org.apache.shiro</groupId>
	<artifactId>shiro-core</artifactId>
	<version>1.2.2</version>
</dependency>
<dependency>
	<groupId>org.apache.shiro</groupId>
	<artifactId>shiro-spring</artifactId>
	<version>1.2.2</version>
</dependency>
<dependency>
	<groupId>org.apache.shiro</groupId>
	<artifactId>shiro-web</artifactId>
	<version>1.2.2</version>
</dependency>
```

将shiro的依赖包引进到项目后，将shiro的filter配置到项目中，让Shiro参与项目请求的过滤。这个比较简单，只需要在web.xml添加以下配置即可:
```xml
	<filter>
		<filter-name>shiroFilter</filter-name>
		<filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
	</filter>
	
	<filter-mapping>
		<filter-name>shiroFilter</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
```
然后在spring的applicationContext.xml文件中，还要对shiro作以下配置:
```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
	<property name="securityManager" ref="securityManager" />
	<property name="loginUrl" value="/login" />
	<property name="successUrl" value="/home" />
	<property name="unauthorizedUrl" value="/unauthorized" />
	<property name="filterChainDefinitions">
		<value>
			/admin = authc,roles[admin]
			/edit = authc,perms[admin:edit]
			/home = user
			/** = anon
		</value>
	</property>
</bean>
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		<property name="realm" ref="sampleRealm" />
</bean>
<bean id="sampleRealm" class="com.spring.shiro.SampleRealm" />
<!-- Post processor that automatically invokes init() and destroy() methods -->
<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor" />
```

到这里，对shiro的配置就基本完成了，这也体现了Shiro的简单易用。其中，这里的shirFilter要跟web.xml配置的shirFilter同名。sampleRealm是自己写的一个类，下面会给出代码。而shiroFilter里的filterChainDefinitions是对请求拦截的配置。shiro有几个概念，最常用的有authc,roles,perms,user,anon，authc表示需要认证，roles表示角色，perms表示权限，anon表示游客，user表示用户，但跟authc有所不同。当应用开启了rememberMe时，用户下次访问时可以是一个user,但不是authc，authc是需要从新谁的，user不需要。下面对filterChainDefinitions作一些解释。/admin = authc,roles[admin]表示用户必需已经通过认证，并拥有admin的角色的用户才可以正常发起/admin请求。/edit = authc,perms[admin:edit]表示用户必需已经通过认证，并拥有admin:edit权限才可以正常发起/edit请求。/home = user表示用户不一定需要已经通过认证，只需要曾经被shiro记住过登录状态的用户就可以正常发起/home请求，就是前面提到的rememberMe。前用户登录的时候开启了rememberMe，关闭浏览器，下次再访问的时候就是一个user，/home请求是可以正常发的。/** = anon表示除了以上请示，其他所有请求是游客就可以正常发起。

shiro本身就提供了几个Realm，在shiro-code-1.2.2.jar这个核心包中我们可以看到，shiro提供了ActiveDirectoryRealm,JdbcRealm,JndiLdapRealm,IniRealm,PropertiesRealm,TextConfigurationRealm,AuthorizingReaml,AuthorizingRealm,CachingRealm,SimpleAccountRealm。但很多时候，shiro自身提供的Reaml并不适合我们自身的项目，我们需要实现自己的Realm。下面贴出sampleRealm的简单实现：

```java
package com.concom.security.infrastructure.shiro;

import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.shiro.authc.AccountException;
import org.apache.shiro.authc.AuthenticationException;
import org.apache.shiro.authc.AuthenticationInfo;
import org.apache.shiro.authc.AuthenticationToken;
import org.apache.shiro.authc.DisabledAccountException;
import org.apache.shiro.authc.SimpleAuthenticationInfo;
import org.apache.shiro.authc.UnknownAccountException;
import org.apache.shiro.authc.UsernamePasswordToken;
import org.apache.shiro.authz.AuthorizationInfo;
import org.apache.shiro.authz.SimpleAuthorizationInfo;
import org.apache.shiro.realm.AuthorizingRealm;
import org.apache.shiro.subject.PrincipalCollection;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;

import com.concom.security.application.user.UserService;
import com.concom.security.domain.user.Role;
import com.concom.security.domain.user.User;

/**
 * @author Dylan
 * @time 2013-8-2
 */
public class ShiroRealm extends AuthorizingRealm{

	private final static Logger LOG = LoggerFactory.getLogger(ShiroRealm.class);
	
	public final static String REALM_NAME = "ShiroCasRealm";
	
	@Autowired
	private UserService userService;
	
	public ShiroRealm() {
		setName(REALM_NAME); // This name must match the name in the User
								// class's getPrincipals() method
	//	setCredentialsMatcher(new Sha256CredentialsMatcher());
	}
	
	/**
	 * 认证
	 */
	@Override
	protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authcToken) throws AuthenticationException {
		
		UsernamePasswordToken token = (UsernamePasswordToken) authcToken;
		String username = token.getUsername();
		if(LOG.isTraceEnabled()){
			LOG.trace("开始认证 "+ username);
		}
		try {
			if(StringUtils.isBlank(username)){
				throw new AccountException("can not handle this login");
			}
			User user = userService.getByUsername(username);
			checkUser(user, username);
			return new SimpleAuthenticationInfo(user.getUsername(), user.getPassword(), getName());
		} catch (Exception e) {
			throw translateAuthenticationException(e);
		}
	}
	
	/**
	 * 授权
	 */
	@Override
	protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
		
		String username = (String)getAvailablePrincipal(principals);
		
		if(LOG.isTraceEnabled()){
			LOG.trace("开始授权 "+ username);
		}
		
		User user = userService.getByUsername(username);
		
		SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
		Set<String> rolesAsString = user.getRolesAsString();
		info.addRoles(rolesAsString);
		if(user.hasAuths()){
			info.addStringPermissions(user.getAuthAsString());
		}
		for(Role role : user.getRoles()){
			info.addStringPermissions(role.getAuthsAsString());
		}
		return info;
	}

	/**
	 * 异常转换
	 * @param e
	 * @return
	 */
	private AuthenticationException translateAuthenticationException(Exception e) {
		if (e instanceof AuthenticationException) {
			return (AuthenticationException) e;
		}
		if(e instanceof DisabledAccountException){
			return (DisabledAccountException)e;
		}
		if(e instanceof UnknownAccountException){
			return (UnknownAccountException)e;
		}
		return new AuthenticationException(e);
	}
	/**
	 * 检查用户
	 * @param user
	 * @param username
	 */
	private void checkUser(User user,String username){
		if(null == user){
			throw new UnknownAccountException(username + " can not find "+username+" from system");
		}
		if(user.isLocked()){
			throw new DisabledAccountException("the account is locked now");
		}
	}
}
```

  SampleRealm 继承自shiro的AuthorizingRealm，然后重写doGetAuthenticationInfo和doGetAuthorizationInfo。以上是SampleRealm的完整代码。下面是用户登录的几句重要代码：
  
```java
Subject currentUser = SecurityUtils.getSubject();
boolean rememberMe = ServletRequestUtils.getBooleanParameter(request, "rememberMe", false);
String passwordAsMD5 = PasswordEncoder.encode(user.getPassword());
UsernamePasswordToken token = new UsernamePasswordToken(user.getUsername(), passwordAsMD5, rememberMe);
currentUser.login(token); // 登录
```
其中user是用户在登录页面输入的用户信息，这时候传过来的密码是没有加密的。创建UsernamePasswordTaken时，构造函数里的passwordAsMD5是要对没加密的密码加密后的结果。因为shiro会从数据库查询用户的信息，匹配用户的密码。通常密码在保存到数据库的时候都经过加密的。也就是说，这里的passwordAsMD5必需与数据库里的密码是一致的。

到这里，一个简单，但又完整的shiro应用就完成了。关于User的设计，可以看一下我的shiro之旅: 十一 shiro的权限设计这篇文章。下面，将会对shiro作一个更深入的介绍。
