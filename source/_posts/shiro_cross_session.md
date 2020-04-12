title: '我的shiro之旅: 十七跨域session共享的一种解决方法'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 12:18:00
---

在文章七里面有介绍session共享，不过只是在一个域名及其他子域间共享。有时候，我们需要在多个一级域名共享登录的session。当然，用shiro-cas是一个很不错的解决方法，应该也是大部分人使用的方法。不过因为种种原因，并没有选择shiro-cas的方式，就使用了其他方式代替，思路也来自shiro-cas。比如有www.a.com，www.b.com两个域名需要共享session，并且www.b.com域名下的session是主，登录在www.b.com域名下，www.a.com域名下的session需要取www.b.com域名下的session。大概画一个图：
![cross](/images/blog/shiro_cross.png)

在www.a.com中加一个filter，用于重定向到www.b.com，取得b域名下的sessionId。代码贴出来：

```java
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;

import org.apache.commons.lang.StringUtils;
import org.apache.shiro.web.servlet.AdviceFilter;
import org.apache.shiro.web.util.WebUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.concom.security.infrastructure.helper.HttpHelper;
import com.concom.security.infrastructure.helper.ShiroSecurityHelper;


/**用于跨域共享session,那个应用需要与登录应用共享session，就开放此filter，并且应该是拦截所有请求，至少是拦截所有需要保护的资源
 * @author Dylan
 * @mail pickear@gmail.com
 * @time 2014年4月14日
 */
public class CasFilter extends AdviceFilter {
	
	private final static Logger log = LoggerFactory.getLogger(CasFilter.class);
	
	private String casServerURL;  //重定向的目标地址，该地址用于获取sessionId,如www.b.com/token
	private String domain;   //filter应用的域名，如www.a.com

	@Override
	protected boolean preHandle(ServletRequest request, ServletResponse response)throws Exception {
		
		boolean hasSyn = (null == ShiroSecurityHelper.getSession().getAttribute("hasSyn") ? false : (Boolean) ShiroSecurityHelper.getSession().getAttribute("hasSyn"));
		if(ShiroSecurityHelper.hasAuthenticated() || hasSyn){//当用户已经登录或者从session中取得的hasSyn为true，说明已经同步session，不需要再重定向
			return true;
		}
		String jsid = WebUtils.getCleanParam(request, "jsid");
		HttpServletRequest httpRequest = WebUtils.toHttp(request);
		String url = httpRequest.getRequestURL().toString();
		url = StringUtils.remove(url, httpRequest.getContextPath());
		if(StringUtils.isNotBlank(jsid)){//如果jsid不为空，说明是通过www.b.com重定向回来的，将从b域名拿到的sessionId写回到自己域名下。
                       //以下两句作用是将jsid,rememberMe写到domain域名下的cookie中，读者可以自己实现。
			HttpHelper.setCookie(WebUtils.toHttp(httpRequest),WebUtils.toHttp(response), "jsid", jsid,domain,"/");
			HttpHelper.setCookie(WebUtils.toHttp(httpRequest),WebUtils.toHttp(response), "rememberMe", WebUtils.getCleanParam(request, "rememberMe"),domain,"/");
			WebUtils.issueRedirect(request, response, url);
			log.info("redirect : " + url);
			return false;
		}
		String uri = casServerURL + "?service=" + url;   //重写向到www.b.com/token下
		WebUtils.issueRedirect(request, response, uri);
		log.info("redirect : " + uri);
		return false;
	}

	@Override
	protected void postHandle(ServletRequest request, ServletResponse response)throws Exception {
		super.postHandle(request, response);
	}

	public void setCasServerURL(String casServerURL) {
		this.casServerURL = casServerURL;
	}

	public void setDomain(String domain) {
		this.domain = domain;
	}
	
}
```

www.b.com/token请求的代码应该是如下的:

```java

/**用于跨域共享session，该方法应该在登录的应用开放。如登录页面在xxx.aa.com域名下，要和***.bb.com共享session
	 * 那么，在***.bb.com应该有个filter，用来重写向到xxx.aa.com域名下的token请求，并带上回调地址，在token方法中
	 * 会拿到xxx.aa.com下的sesison，以参数的形式再重定向回***.bb.com。在bb域名的filter中拿到该jsid，应该保存
	 * 到自己域名下的cookie里，以确保bb的cookie里jsid值和xxx.aa.com相同。filter的实现为casFilter。
	 * @param request
	 * @param response
	 * @return
	 */
        @RequestMapping(value="/token",method = RequestMethod.GET)
	public void token(HttpServletRequest request, HttpServletResponse response) {
		String service = WebUtils.getCleanParam(request, "service");
		String rememberMe = HttpHelper.getCookieValue(request, "rememberMe");
		Session session = ShiroSecurityHelper.getSession();
		session.setAttribute("hasSyn", true);   //将hasSyn为true设置到session中，CasFilter会从Session中取该值
		try {
			WebUtils.issueRedirect(request, response, service+"?jsid="+session.getId()+"&rememberMe="+rememberMe);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
```

然后，将CasFilter交由spring 管理

```xml
<bean id="casFilter" class="com.xxx.security.interfaces.filter.CasFilter">
	    <property name="casServerURL" value="http://www.b.com/token"/>
	    <property name="domain" value="www.a.com"/>
</bean>
```
在shiro的配置文件中，配置CasFilter拦截那些请求，大概如下：

```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
		<property name="securityManager" ref="securityManager" />
		<property name="loginUrl" value="/security/login.html" />
		<property name="successUrl" value="/home.html" />
		<property name="unauthorizedUrl" value="/security/unauthorized.html" />
		<property name="filters">
			<map>
				<entry key="cas" value-ref="casFilter" />
			</map>
		</property>
		<property name="filterChainDefinitions">
			<value>
				/** = cas,anon
			</value>
		</property>
</bean>
```
整个过程，目的只有一个，就是让www.a.com，www.b.com两个域名的cookie共享。当然，cookie跨域共享方式很多，网上也有很多资料，包括通过iframe,js等方式。本文的这种方法，只是借仿shiro-cas的重定向方式，不过也有所不同。本文分享到这里，希望读者有更好的方式分享。
