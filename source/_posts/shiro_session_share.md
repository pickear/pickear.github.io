title: '我的shiro之旅: 七shiro session 共享'
author: Dylan
tags:
  - shiro
categories:
  - java
date: 2018-09-16 11:19:00
---
很久没有写过博文，一来是因为工作比较紧，二来是因为很长一段时间没有写博文的心情。今天，想继续写写shiro的一些文章，这篇文章只要想分享的是shiro 共享session的内容。

在这里先给出我的shiro配置文件:
```xml
        <bean id="shiroRealm" class="com.concom.security.infrastructure.shiro.ShiroRealm"/>

	<bean id="shiroCacheManager" class="com.concom.security.infrastructure.shiro.cache.ShiroCacheManager"/>
	
	<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
		<property name="realm" ref="shiroRealm" />
		<!-- <property name="sessionMode" value="native"/> -->
		<property name="cacheManager" ref="shiroCacheManager"/>
		<property name="sessionManager" ref="shiroSessionManager"/>
		<!-- <property name="subjectDAO" ref="subjectDAO"/> -->
	</bean>
	
	<bean id="sessionDAO" class="org.apache.shiro.session.mgt.eis.EnterpriseCacheSessionDAO">
		<!-- <property name="sessionIdGenerator" ref="sessionIdGenerator"/> -->
	</bean>
	<!-- <bean id="sessionIdGenerator" class="com.concom.security.infrastructure.shiro.ShiroSessionIdGenerator"/> -->

	<bean id="shiroSessionManager" class="org.apache.shiro.web.session.mgt.DefaultWebSessionManager">
		<property name="sessionDAO" ref="sessionDAO"/>
		<!-- <property name="sessionValidationScheduler" ref="shiroSessionValidationScheduler"/> -->
		<property name="sessionValidationInterval" value="1800000"/>  <!-- 相隔多久检查一次session的有效性 -->
		<property name="globalSessionTimeout" value="1800000"/>  <!-- session 有效时间为半小时 （毫秒单位）-->
		<property name="sessionIdCookie.domain" value="${domain}"/>
		<property name="sessionIdCookie.name" value="${sessionId_name}"/>
		<property name="sessionIdCookie.path" value="${path}"/>
		<!-- <property name="sessionListeners">
			<list>
				<bean class="com.concom.security.interfaces.listener.SessionListener"/>
			</list>
		</property> -->
	</bean>
	
	<!-- <bean id="shiroSessionValidationScheduler" class="org.apache.shiro.session.mgt.ExecutorServiceSessionValidationScheduler">
		<constructor-arg name="sessionManager" ref="sessionManager"/>
		<property name="interval" value="10000"/>  定时半小时检验一次有所session是否有效（毫秒单位）
	</bean> -->

	<!-- Post processor that automatically invokes init() and destroy() methods -->
	<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor" />
```

从shiro的DefaultWebSecurityManager源码中我们可以看到，该类继承了CachingSecurityManager类，CachingSecurityManager里有个CacheManager属性，这就是shiro的cache，我们主要实现这个cache就可以。贴出源码。

```java
public abstract class CachingSecurityManager implements SecurityManager, Destroyable, CacheManagerAware {

    /**
     * The CacheManager to use to perform caching operations to enhance performance.  Can be null.
     */
    private CacheManager cacheManager;

    /**
     * Default no-arg constructor that will automatically attempt to initialize a default cacheManager
     */
    public CachingSecurityManager() {
    }

    /**
     * Returns the CacheManager used by this SecurityManager.
     *
     * @return the cacheManager used by this SecurityManager
     */
    public CacheManager getCacheManager() {
        return cacheManager;
    }

    /**
     * Sets the CacheManager used by this {@code SecurityManager} and potentially any of its
     * children components.
     * <p/>
     * After the cacheManager attribute has been set, the template method
     * {@link #afterCacheManagerSet afterCacheManagerSet()} is executed to allow subclasses to adjust when a
     * cacheManager is available.
     *
     * @param cacheManager the CacheManager used by this {@code SecurityManager} and potentially any of its
     *                     children components.
     */
    public void setCacheManager(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
        afterCacheManagerSet();
    }

    /**
     * Template callback to notify subclasses that a
     * {@link org.apache.shiro.cache.CacheManager CacheManager} has been set and is available for use via the
     * {@link #getCacheManager getCacheManager()} method.
     */
    protected void afterCacheManagerSet() {
    }

    /**
     * Destroys the {@link #getCacheManager() cacheManager} via {@link LifecycleUtils#destroy LifecycleUtils.destroy}.
     */
    public void destroy() {
        LifecycleUtils.destroy(getCacheManager());
        this.cacheManager = null;
    }

}
```
cacheManager是一个接口，shiro给出了一个实现，就是MemoryConstrainedCacheManager，下面也贴出MemoryConstrainedCacheManager的源码。

```java
public class MemoryConstrainedCacheManager extends AbstractCacheManager {

    /**
     * Returns a new {@link MapCache MapCache} instance backed by a {@link SoftHashMap}.
     *
     * @param name the name of the cache
     * @return a new {@link MapCache MapCache} instance backed by a {@link SoftHashMap}.
     */
    @Override
    protected Cache createCache(String name) {
        return new MapCache<Object, Object>(name, new SoftHashMap<Object, Object>());
    }
}
```
从MemoryConstrainedCacheManager我们可以看到，MemoryConstrainedCacheManager继承了AbstractCacheManager，我们自己建一个类，也继承AbstractCacheManager，然后实现createCache(String name)方法，替换shiro的默认实现即可。在这里，也给出我的实现。

```java
package com.concom.security.infrastructure.shiro.cache;

import org.apache.shiro.cache.AbstractCacheManager;
import org.apache.shiro.cache.Cache;
import org.apache.shiro.cache.CacheException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.concom.security.infrastructure.shiro.cache.memcache.ShiroMemcachedCache;

/**
 * @author Dylan
 * @time 2014年1月6日
 */
@SuppressWarnings("rawtypes")
public class ShiroCacheManager extends AbstractCacheManager {

	private static final Logger log = LoggerFactory.getLogger(ShiroCacheManager.class);

	@Override
	protected Cache createCache(String cacheName) throws CacheException {

		if (log.isTraceEnabled()) {
			log.trace("create a cache name : " + cacheName);
		}

		return new ShiroMemcachedCache<Object, Object>(cacheName);
	}

}
```
createCache方法返回的是一个Cache，Cache也是一个接口，我写了一个ShiroMemcachedCache，实现了Cache。ShiroMemcachedCache主要是对缓存的一些增删改查的操作。其中ShiroMemcachedCache的实现如下：

```java
package com.concom.security.infrastructure.shiro.cache.memcache;

import java.util.ArrayList;
import java.util.Collection;
import java.util.HashSet;
import java.util.Iterator;
import java.util.List;
import java.util.Set;

import org.apache.commons.lang.StringUtils;
import org.apache.shiro.cache.Cache;
import org.apache.shiro.cache.CacheException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.concom.security.infrastructure.memcache.MemcacheRepository;
/**
 * @author Dylan
 * @time 2014年1月6日
 * @param <K>
 * @param <V>
 */
@SuppressWarnings("unchecked")
public class ShiroMemcachedCache<K,V> implements Cache<K, V>  {
	
	private static MemcacheRepository cache;
	
	private String CACHE_PREFIX;
	
	private final static Logger LOG = LoggerFactory.getLogger(ShiroMemcachedCache.class);
	
	
	public ShiroMemcachedCache(String cacheName){
		CACHE_PREFIX = cacheName+"-";
	}
	
	@Override
	public V get(K key) throws CacheException {
		V value = ((V)getCache().get(keyToString(key)));
		if(LOG.isDebugEnabled()){
			LOG.debug("get the entity json is " + key + " : " + value);
		}
		return value;
	}

	@Override
	public V put(K key, V value) throws CacheException {
		if(LOG.isDebugEnabled()){
			LOG.debug("begin save the "+ key + " : " + value+" to memcache");
		}
		getCache().save(keyToString(key), value);
		return value;
	}

	@Override
	public V remove(K key) throws CacheException {
		if(LOG.isDebugEnabled()){
			LOG.debug("begin remove the "+key+" from memcache");
		}
		V value = get(key);
		getCache().remove(keyToString(key));
		return value;
	}

	@Override
	public void clear() throws CacheException {
		for(Iterator<K> keys =keys().iterator();keys.hasNext();){
			K key = keys.next();
			remove(key);
		}
	}

	@Override
	public int size() {
		return keys().size();
	}

	@Override
	public Set<K> keys() {
		Set<String> keys = new HashSet<String>();
		for(String key : getCache().keys()){
			if(StringUtils.startsWith(key, CACHE_PREFIX)){
				keys.add(key);
			}
		}
		return (Set<K>)keys ;
	}

	@Override
	public Collection<V> values() {
		List<V> values = new ArrayList<V>();
		for(Iterator<K> keys =keys().iterator();keys.hasNext();){
			K key = keys.next();
			V value = getValue(key);
			values.add(value);
		}
		return values;
	}
	
	private V getValue(K key) throws CacheException {
		V value = ((V)getCache().get(String.valueOf(key)));
		if(LOG.isDebugEnabled()){
			LOG.debug("get the entity json is " + key + " : " + value);
		}
		return value;
	}
	
	private String keyToString(K key){
		String k = String.valueOf(key);
		if(StringUtils.startsWith(k, CACHE_PREFIX)){
			return k;
		}
		return CACHE_PREFIX+k;
	}

	public static MemcacheRepository getCache() {
		return cache;
	}

	public static void setCache(MemcacheRepository cache) {
		ShiroMemcachedCache.cache = cache;
	}

}
```

上面缓存用的是memcached。这样，shiro的缓存共享就完成了。篇幅有点长，这篇先写到这里，下一篇将继续讲shiro的cache共享。