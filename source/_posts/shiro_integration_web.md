title: '我的shiro之旅:五shiro与普通web项目集成'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 11:12:00
---
在第二篇文章讲了shiro与web项目集成，其中只要是与spring项目的集成。公司有些旧项目，是用servlet写的，并不适合用第二篇文章去搭建shiro。这里想讲讲shiro与普通的项目集成。与普通的web项目集成，shiro只需要两个依赖包，在项目的pom.xml下加入以下依赖：
```xml
		<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-core</artifactId>
			<version>1.2.2</version>
		</dependency>
		<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-web</artifactId>
			<version>1.2.2</version>
		</dependency>
<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-core</artifactId>
			<version>1.2.2</version>
		</dependency>
		<dependency>
			<groupId>org.apache.shiro</groupId>
			<artifactId>shiro-web</artifactId>
			<version>1.2.2</version>
		</dependency>
```
在项目的web.xml加入shiro的listener

```xml
	<context-param>
		<param-name>shiroConfigLocations</param-name>
		<param-value>classpath:META-INF/shiro/shiro.ini</param-value>
	</context-param>

	<listener>
		<listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
	</listener>
	
	<filter>
		<filter-name>ShiroFilter</filter-name>
		<filter-class>org.apache.shiro.web.servlet.ShiroFilter</filter-class>
	</filter>

	<filter-mapping>
		<filter-name>ShiroFilter</filter-name>
		<url-pattern>*.html</url-pattern>
		<dispatcher>REQUEST</dispatcher>
		<dispatcher>FORWARD</dispatcher>
		<dispatcher>INCLUDE</dispatcher>
		<dispatcher>ERROR</dispatcher>
	</filter-mapping>
<context-param>
		<param-name>shiroConfigLocations</param-name>
		<param-value>classpath:META-INF/shiro/shiro.ini</param-value>
	</context-param>

	<listener>
		<listener-class>org.apache.shiro.web.env.EnvironmentLoaderListener</listener-class>
	</listener>
	
	<filter>
		<filter-name>ShiroFilter</filter-name>
		<filter-class>org.apache.shiro.web.servlet.ShiroFilter</filter-class>
	</filter>

	<filter-mapping>
		<filter-name>ShiroFilter</filter-name>
		<url-pattern>*.html</url-pattern>
		<dispatcher>REQUEST</dispatcher>
		<dispatcher>FORWARD</dispatcher>
		<dispatcher>INCLUDE</dispatcher>
		<dispatcher>ERROR</dispatcher>
	</filter-mapping>
```
上面配置了shiro.ini文件在classpath的META-INF/shiro目录下。shiro.ini内容如下:

```
# =======================
# Shiro INI configuration
# =======================
[main]
myRealm = com.neting.security.infrastructure.shiro.ShiroRealm

securityManager.realm = $myRealm

authc.loginUrl = /security/login.html
authc.successUrl = /home.html
roles.unauthorizedUrl = /security/unauthorized.html
perms.unauthorizedUrl = /security/unauthorized.html

#myFilter
anyRoles = com.neting.security.interfaces.filter.AnyRolesAuthorizationFilter
anyPerms = com.neting.security.interfaces.filter.AnyPermissionsAuthorizationFilter

[users]

[roles]

[urls]
/admin=anyRoles[A,B],anyPerms["admin:a","admin:b"]
/** = anon
```
其中myRealm是自己写的realm，下面会给出代码，anyRoles，anyPerms为自定义的filter，上一篇文章已经讲过。

realm的代码如下，跟与spring项目集成的一样：

```java
/**
 * @author Dylan
 */
public class ShiroRealm extends AuthorizingRealm{

	private final static Logger LOG = LoggerFactory.getLogger(ShiroRealm.class);
	
	private UserService userService = UserServiceImpl.getInstance();
	
	public ShiroRealm() {
		setName("ShiroCasRealm"); // This name must match the name in the User
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
```
到这里,shiro与普通的web项目集成已经完成。可以看到，shiro的非常易用的，无论是定制自己的realm，还是定义自己的filter，都可以体现shiro的易用性和灵活性。shiro对所有的功能都有一套默认的实现，比如说realm。用户可以完全不用自定义自己的reaml，shiro本身就提供了很多种realm的实现。在shiro-core-xx.jar的org.apache.shiro.realm包下，我们可以找到像JdbcRealm，JndiLdapRealm，IniRealm等等。shiro不仅易用和灵活，而且也可插拔。在很多地方，我们都可以用自己的实现在替换shiro的默认实现，当我们取消自己的实现时，shiro又会用默认的实现替上。在后面讲到shiro的集群和shiro共享时，我们就能深深感受到shiro的这种特性。