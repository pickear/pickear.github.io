title: '我的shiro之旅: 十四shiro 自动登录'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 12:08:00
---

shiro有几种状态，其中包括guest,user,authenticated。guest就是游客，authenticated就是认证后的用户，而user是介于两者之前。user并不代表用户已经成功认证，当用户上次登录时选择rememberMe，下次用户再访问时就是user状态。登录时选择rememberMe，shiro会通过一种加密方式将principal(我们理解为用户名)加密保存到cookie中。shiro可以通过这个cookie解密得到principal。所以当状态为user时，虽然并不代表用户已经通过认证，但我们却可以通过Subject拿到用户名。先贴出代码：

```java
public void autoLogin(HttpServletRequest request, HttpServletResponse response) {

		Subject subject = SecurityUtils.getSubject();
		if(subject.isRemembered()){
			String username = ShiroSecurityHelper.getCurrentUsername();
			LOG.info("用户【{}】自动登录----{}", username,TimeHelper.getCurrentTime());
			User user = userService.getByUsername(username);
			baseLogin(user, request, response);
			ShiroAuthorizationHelper.clearAuthorizationInfo(username); // 用户是自动登录，首先清一下用户权限缓存，让重新加载
		}
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
ShiroAuthorizationHelper在文章我的shiro之旅: 九 shiro 清理缓存的权限信息有介绍。我们来看看DelegatingSubject类的isRemembered()方法的实现。

```java
public boolean isRemembered() {
        PrincipalCollection principals = getPrincipals();
        return principals != null && !principals.isEmpty() && !isAuthenticated();
    }
```
当能拿到principals和用户没认证的时候，这个方法才返回true，principals可以通过cookie解密得到。如果用户是认证状态的，这个方法将返回false。

如此看来,shiro的自动登录实现还是比较简单。

securityManager有个rememberMeManager，用来管理rememberMe的，默认情况下，这个cookie会保存一年时间，我们可以看一下RememberMeManager接口的实现类org.apache.shiro.web.mgt.CookieRememberMeManager的源码:

```java
public class CookieRememberMeManager extends AbstractRememberMeManager {

    //TODO - complete JavaDoc

    private static transient final Logger log = LoggerFactory.getLogger(CookieRememberMeManager.class);

    /**
     * The default name of the underlying rememberMe cookie which is {@code rememberMe}.
     */
    public static final String DEFAULT_REMEMBER_ME_COOKIE_NAME = "rememberMe";

    private Cookie cookie;

    /**
     * Constructs a new {@code CookieRememberMeManager} with a default {@code rememberMe} cookie template.
     */
    public CookieRememberMeManager() {
        Cookie cookie = new SimpleCookie(DEFAULT_REMEMBER_ME_COOKIE_NAME);
        cookie.setHttpOnly(true);
        //One year should be long enough - most sites won't object to requiring a user to log in if they haven't visited
        //in a year:
        cookie.setMaxAge(Cookie.ONE_YEAR);
        this.cookie = cookie;
    }

    public Cookie getCookie() {
        return cookie;
    }

 
    @SuppressWarnings({"UnusedDeclaration"})
    public void setCookie(Cookie cookie) {
        this.cookie = cookie;
    }
    protected void rememberSerializedIdentity(Subject subject, byte[] serialized) {

        if (!WebUtils.isHttp(subject)) {
            if (log.isDebugEnabled()) {
                String msg = "Subject argument is not an HTTP-aware instance.  This is required to obtain a servlet " +
                        "request and response in order to set the rememberMe cookie. Returning immediately and " +
                        "ignoring rememberMe operation.";
                log.debug(msg);
            }
            return;
        }


        HttpServletRequest request = WebUtils.getHttpRequest(subject);
        HttpServletResponse response = WebUtils.getHttpResponse(subject);

        //base 64 encode it and store as a cookie:
        String base64 = Base64.encodeToString(serialized);

        Cookie template = getCookie(); //the class attribute is really a template for the outgoing cookies
        Cookie cookie = new SimpleCookie(template);
        cookie.setValue(base64);
        cookie.saveTo(request, response);
    }

    private boolean isIdentityRemoved(WebSubjectContext subjectContext) {
        ServletRequest request = subjectContext.resolveServletRequest();
        if (request != null) {
            Boolean removed = (Boolean) request.getAttribute(ShiroHttpServletRequest.IDENTITY_REMOVED_KEY);
            return removed != null && removed;
        }
        return false;
    }
    protected byte[] getRememberedSerializedIdentity(SubjectContext subjectContext) {

        if (!WebUtils.isHttp(subjectContext)) {
            if (log.isDebugEnabled()) {
                String msg = "SubjectContext argument is not an HTTP-aware instance.  This is required to obtain a " +
                        "servlet request and response in order to retrieve the rememberMe cookie. Returning " +
                        "immediately and ignoring rememberMe operation.";
                log.debug(msg);
            }
            return null;
        }

        WebSubjectContext wsc = (WebSubjectContext) subjectContext;
        if (isIdentityRemoved(wsc)) {
            return null;
        }

        HttpServletRequest request = WebUtils.getHttpRequest(wsc);
        HttpServletResponse response = WebUtils.getHttpResponse(wsc);

        String base64 = getCookie().readValue(request, response);
        // Browsers do not always remove cookies immediately (SHIRO-183)
        // ignore cookies that are scheduled for removal
        if (Cookie.DELETED_COOKIE_VALUE.equals(base64)) return null;

        if (base64 != null) {
            base64 = ensurePadding(base64);
            if (log.isTraceEnabled()) {
                log.trace("Acquired Base64 encoded identity [" + base64 + "]");
            }
            byte[] decoded = Base64.decode(base64);
            if (log.isTraceEnabled()) {
                log.trace("Base64 decoded byte array length: " + (decoded != null ? decoded.length : 0) + " bytes.");
            }
            return decoded;
        } else {
            //no cookie set - new site visitor?
            return null;
        }
    }
    private String ensurePadding(String base64) {
        int length = base64.length();
        if (length % 4 != 0) {
            StringBuilder sb = new StringBuilder(base64);
            for (int i = 0; i < length % 4; ++i) {
                sb.append('=');
            }
            base64 = sb.toString();
        }
        return base64;
    }
    protected void forgetIdentity(Subject subject) {
        if (WebUtils.isHttp(subject)) {
            HttpServletRequest request = WebUtils.getHttpRequest(subject);
            HttpServletResponse response = WebUtils.getHttpResponse(subject);
            forgetIdentity(request, response);
        }
    }

    public void forgetIdentity(SubjectContext subjectContext) {
        if (WebUtils.isHttp(subjectContext)) {
            HttpServletRequest request = WebUtils.getHttpRequest(subjectContext);
            HttpServletResponse response = WebUtils.getHttpResponse(subjectContext);
            forgetIdentity(request, response);
        }
    }
    private void forgetIdentity(HttpServletRequest request, HttpServletResponse response) {
        getCookie().removeFrom(request, response);
    }
}
```
从构造器中看到cookie.setMaxAge(Cookie.ONE_YEAR);设置cookie的有效时间为一年。也许，这对我们来说会有点长，我们可以更改这些设置。配置如下：

```xml
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		<property name="realm" ref="shiroRealm" />
		<!-- <property name="sessionMode" value="native"/> -->
		<property name="cacheManager" ref="shiroCacheManager"/>
		<property name="sessionManager" ref="shiroSessionManager"/>
		<property name="rememberMeManager.cookie.name" value="rememberMe"/>
		<property name="rememberMeManager.cookie.domain" value=".xxx.com"/>
		<property name="rememberMeManager.cookie.path" value="/"/>
		<property name="rememberMeManager.cookie.maxAge" value="604800"/> <!-- 7天有效期 -->
		<!-- <property name="subjectDAO" ref="subjectDAO"/> -->
</bean>
```
当然，我们也可以实现自己的rememberMeManager。