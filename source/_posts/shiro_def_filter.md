title: '我的shiro之旅: 四自定义filter'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 11:07:00
---
上一篇文章对shiro的filter作了一些简单的介绍，接一下写写息自定义shiro的filter。使用shiro的时候，比较常用的filter有anon,authc,roles和perms。当我们想定义某个链接是拥有某些权限的用户才可以访问的时候，我们可以这样定义。/xx = roles[A,B]。在shiro中，表示当前用户同时拥有A,B两种角色才可以访问/xx这个链接，是一种&&（与）的关系，我们可以看看源码。在shiro-web-xx.jar的org.apache.shiro.web.filter.authz包下有RolesAuthorizationFilter这样一个类，这个类就是定义roles的filter。
```java
public class RolesAuthorizationFilter extends AuthorizationFilter {

    //TODO - complete JavaDoc
    @SuppressWarnings({"unchecked"})
    public boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws IOException {
        Subject subject = getSubject(request, response);
        String[] rolesArray = (String[]) mappedValue;
        if (rolesArray == null || rolesArray.length == 0) {
            //no roles specified, so nothing to check - allow access.
            return true;
        }
        Set<String> roles = CollectionUtils.asSet(rolesArray);
        return subject.hasAllRoles(roles);
    }
}
```
上面定义了subject.hasAllRoles(roles);就是当前用户必须拥有定义的所有角色才会返回true。但有时候，我们需要当前用户拥有定义的其他一个角色就可以访问，那就需要写自己的filter。也很简单，代码以下：

```java
public class AnyRolesAuthorizationFilter extends AuthorizationFilter {

	
	@Override
	protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {

		Subject subject = getSubject(request, response);
		String[] rolesArray = (String[]) mappedValue;

		if (rolesArray == null || rolesArray.length == 0) {
			// no roles specified, so nothing to check - allow access.
			return true;
		}

		Set<String> roles = CollectionUtils.asSet(rolesArray);
		for (String role : roles) {
			if (subject.hasRole(role)) {
				return true;
			}
		}
		return false;
	}

}
```
从上面的代码可以看到，当遍历，发现当前用户拥有定义的其中一个角色就立刻返回true，否则返回false。

定义好filter，只需要代码默认的roles即可。

```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<property name="securityManager" ref="securityManager" />
		<property name="loginUrl" value="/security/login.html" />
		<property name="successUrl" value="/home.html" />
		<property name="unauthorizedUrl" value="/security/unauthorized.html" />
		<property name="filters">
			<map>
				<entry key="anyRoles" value-ref="anyRolesAuthorizationFilter" />
			</map>
		</property>
		<property name="filterChainDefinitions">
			<value>
                                /admin = anyRoles[admin1,admin2]
				/** = anon
			</value>
		</property>
	</bean>
```
perms的filter也同理。看看源码：

```java
public class PermissionsAuthorizationFilter extends AuthorizationFilter {

    //TODO - complete JavaDoc

    public boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws IOException {

        Subject subject = getSubject(request, response);
        String[] perms = (String[]) mappedValue;

        boolean isPermitted = true;
        if (perms != null && perms.length > 0) {
            if (perms.length == 1) {
                if (!subject.isPermitted(perms[0])) {
                    isPermitted = false;
                }
            } else {
                if (!subject.isPermittedAll(perms)) {
                    isPermitted = false;
                }
            }
        }

        return isPermitted;
    }
}
```
自定义的filter：

```java
public class AnyPermissionsAuthorizationFilter extends AuthorizationFilter {

	@Override
	protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
		Subject subject = getSubject(request, response);
		String[] perms = (String[]) mappedValue;

		for (String perm : perms) {
			if (subject.isPermitted(perm)) {
				return true;
			}
		}
		
		return false;
	}

}
```
配置使用自定义filter

```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<property name="securityManager" ref="securityManager" />
		<property name="loginUrl" value="/security/login.html" />
		<property name="successUrl" value="/home.html" />
		<property name="unauthorizedUrl" value="/security/unauthorized.html" />
		<property name="filters">
			<map>
				<entry key="anyPerms" value-ref="anyPermissionsAuthorizationFilter" />
			</map>
		</property>
		
			<value>
				/admin/add = anyPerms["admin:delete","admin:add"]
				/** = anon
			</value>
		</property>
	</bean>
```
当用户请求/admin/add时，就会调用自定义的AnyPermissionsAuthorizationFilter来执行。

shiro的filter大概讲到这里，相信读者对shiro的filter有更深的认识。