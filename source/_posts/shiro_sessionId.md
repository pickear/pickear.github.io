title: '我的shiro之旅: 六自定义shiro的sessionId'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 11:16:00
---
shiro有自己的sesison概念，shiro的session并不是java ee的session。通常，我们看到shiro的sessionId格式类似c6395bbc-425d-43b3-a444-04fee5a92e95，是因为shiro产生sesisonId是通过UUID生成的。我们可以看到shiro-core-xx.jar的org.apache.shiro.session.mgt.eis包下有个JavaUuidSessionIdGenerator，shiro的sessionId默认是通过该类生成的。可以看看源码。
```java
public class JavaUuidSessionIdGenerator implements SessionIdGenerator {

    /**
     * Ignores the method argument and simply returns
     * {@code UUID}.{@link java.util.UUID#randomUUID() randomUUID()}.{@code toString()}.
     *
     * @param session the {@link Session} instance to which the ID will be applied.
     * @return the String value of the JDK's next {@link UUID#randomUUID() randomUUID()}.
     */
    public Serializable generateId(Session session) {
        return UUID.randomUUID().toString();
    }
}
```
是通过调用java的UUID来生成一个唯一的ID。

我们可以自定义类，重写generateId方法，然后把它注入到shiro中。我们可以拿java ee的sessionId作为shiro的sessionId返回。不过我们可以看到generateId方法的参数是Session，这个Session是shiro自定义的，通过它无法拿到java ee的sessionId。我的做法是，写一个拦截器，当讲求过来时，拿到sessionId放到当前线程里，需要的时候从当前线程里拿。类也很简单，代码如下：

```java
/**
 * @author Dylan
 */
public final class HttpSessionIdHolder {

	private HttpSessionIdHolder(){}
	
	private final static ThreadLocal<String> SESSIONID = new ThreadLocal<String>();
	
	/**
	 * @return
	 */
	public static String getSessionId(){
		return SESSIONID.get();
	}
	
	/**
	 * @param request
	 */
	public static void setSessionId(String SessionId){
		SESSIONID.set(SessionId);
	}
	
	/**
	 * 
	 */
	public static void removeSessionId(){
		SESSIONID.remove();
	}

}
```
我们在拦截器里从request拿到sessionId，然后调用setSessionId方法把sessionId保存到当前线程中，需要的时候调用getSessionId就可以得到。

当然，也可以用其他方法。

那么，怎么注入给shiro替换默认的sessionId产生方式呢。在securityManager有个sessionManager属性，在sessionManager有个sessionDAO属性，在sessionDAO有个sessionIdGenerator只要替换就行。如

```xml
	<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		<property name="sessionManager" ref="sessionManager"/>
	</bean>
	<bean id="sessionDAO" class="com.neting.security.infrastructure.shiro.cache.CustomShiroSessionDao">
		<property name="sessionIdGenerator" ref="sessionIdGenerator"/>
	</bean>
	<bean id="sessionIdGenerator" class="com.neting.security.infrastructure.shiro.ShiroSessionIdGenerator"/>
	<bean id="sessionManager" class="org.apache.shiro.web.session.mgt.DefaultWebSessionManager">
		<property name="sessionDAO" ref="sessionDAO"/>
	</bean>
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		<property name="sessionManager" ref="sessionManager"/>
	</bean>
	<bean id="sessionDAO" class="com.neting.security.infrastructure.shiro.cache.CustomShiroSessionDao">
		<property name="sessionIdGenerator" ref="sessionIdGenerator"/>
	</bean>
	<bean id="sessionIdGenerator" class="com.neting.security.infrastructure.shiro.ShiroSessionIdGenerator"/>
	<bean id="sessionManager" class="org.apache.shiro.web.session.mgt.DefaultWebSessionManager">
		<property name="sessionDAO" ref="sessionDAO"/>
	</bean>
```

更简单的方式，在我的shiro之旅: 十 自定义shiro的SessionIdCookie这篇文章有介绍。