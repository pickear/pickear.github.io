title: '我的shiro之旅: 十五shiro登录成功后,跳转到登录前的页面'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 12:11:00
---

很多时候，我们需要做到，当用户登录成功后，跳转回登录前的页面。如果用户是点击"登录"链接去到登录页面进行登录的，我们很容易跟踪用户的登录前的页面。比如，在"登录"链接后加一个url参数，如:http://www.xxx.com/login.html?url=http://www.xxx.com/xx.html，这个url就是当前页面。用户浏览不同页面，"登录"链接后面的url跟着改变。这样，跳转到登录页面时都会带有上一个页面的url作为参数，登录后也很容易拿到这个参数进行重定向到登录前的页面。

但当我们用配置/xxx.html=authc这种方式，限制用户访问/xxx.html连接时必须是认证过的用户，否则shiro的filter将会重定向到登录页面，上面的方法应当好处理了。不过shiro在跳转前有记录跳转前的页面。前没有认证的用户请求需要认证的链接时，shiro在跳转前会把跳转过来的页面链接保存到session的attribute中，key的值叫shiroSavedRequest，我们可以能过WebUtils类拿到。

当用户登录成功后，可能通过String url = WebUtils.getSavedRequest(request).getRequestUrl();，拿到跳转到登录页面前的url，然后redirect到这个url。其实我们可以看看这个方法的源码：

```java
public static SavedRequest getSavedRequest(ServletRequest request) {
        SavedRequest savedRequest = null;
        Subject subject = SecurityUtils.getSubject();
        Session session = subject.getSession(false);
        if (session != null) {
            savedRequest = (SavedRequest) session.getAttribute(SAVED_REQUEST_KEY);
        }
        return savedRequest;
    }
```
从session中拿到SaveRequest。不过值得注意的是，这个SaveRequest是在用户通过上面方式跳转登录时shiro才会保存，并且不会改变，除非下一次跳转再次发生。并不是每一个请求，shiro都会把上一个请求保存到session中。所以，不能通过WebUtils.getSavedRequest(request)在任何地方调用来拿到上一个页面的请求。这个方法的调用，更应该是在用户登录成功后，重定向到页面时使用。
