title: '我的shiro之旅: 十三shiro 用户的登录与退出'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 12:05:00
---

shiro的登录与退出非常简单，在这里简单给出代码，更详细可以参考官网。

1. 用户的登录

```java
public void login(User user, HttpServletRequest request, HttpServletResponse response) {
	user.encodePassword();
	baseLogin(user, request, response);
}

public void baseLogin(User user, HttpServletRequest request, HttpServletResponse response) {
		
	try {
		Subject subject= SecurityUtils.getSubject();
		if (subject.isAuthenticated()) {
			return;
		}
		//如果用户已登录，先踢出
		ShiroSecurityHelper.kickOutUser(user.getUsername());
			
		boolean rememberMe = ServletRequestUtils.getBooleanParameter(request, "rememberMe", false);
		UsernamePasswordToken token = new UsernamePasswordToken(user.getUsername(), user.getPassword(), rememberMe);
		subject.login(token); // 登录
	} catch (Exception e) {
		//做一些异常处理
	}finally{
		ShiroAuthorizationHelper.clearAuthorizationInfo(sessionUser.getUsername());
	}
}
```
这里需要注意的是在调用subject的login方法,其中传给 UsernamePasswordToken的用户密码应该是加密的，因为通常我们数据库保存的密码是加密后的，否则，将会登录不成功。当调用subject的login方法进行用户认证明，shiro将会调用我们息定义的realm相关方法，前面的文章也有介绍。上面的rememberMe会在下篇文章提到。当然，在登录之前，我们还应该做一些验证，比如用户输入的用户名密码是否为空，用户的有效期之类的，在这里不给出。

2. 用户退出

```java
public void logout() {
	Subject subject = SecurityUtils.getSubject();
	if (subject.isAuthenticated()) {
		subject.logout(); // session 会销毁，在SessionListener监听session销毁，清理权限缓存
		if (LOG.isDebugEnabled()) {
			LOG.debug("用户" + username + "退出登录");
		}
	}
}
```
只要简单调用subject的logout方法就可以了。