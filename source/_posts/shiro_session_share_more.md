title: '我的shiro之旅: 八shiro session共享的进一步'
author: Dylan
tags:
  - shiro
  - java
categories:
  - 编程语言
date: 2018-09-16 11:22:00
---
在第七篇文章说了一下如何实现shiro的共享。那个共享并不只是session的共享，也包括了Authorization的共享。如果读者还没看过文章七，建议先看看。

在文章七的ShiroCacheManager类继承了AbstractCacheManager，我们来看看AbstractCacheManager类的源码:
```java
public abstract class AbstractCacheManager implements CacheManager, Destroyable {

    /**
     * Retains all Cache objects maintained by this cache manager.
     */
    private final ConcurrentMap<String, Cache> caches;

    /**
     * Default no-arg constructor that instantiates an internal name-to-cache {@code ConcurrentMap}.
     */
    public AbstractCacheManager() {
        this.caches = new ConcurrentHashMap<String, Cache>();
    }

    /**
     * Returns the cache with the specified {@code name}.  If the cache instance does not yet exist, it will be lazily
     * created, retained for further access, and then returned.
     *
     * @param name the name of the cache to acquire.
     * @return the cache with the specified {@code name}.
     * @throws IllegalArgumentException if the {@code name} argument is {@code null} or does not contain text.
     * @throws CacheException           if there is a problem lazily creating a {@code Cache} instance.
     */
    public <K, V> Cache<K, V> getCache(String name) throws IllegalArgumentException, CacheException {
        if (!StringUtils.hasText(name)) {
            throw new IllegalArgumentException("Cache name cannot be null or empty.");
        }

        Cache cache;

        cache = caches.get(name);
        if (cache == null) {
            cache = createCache(name);
            Cache existing = caches.putIfAbsent(name, cache);
            if (existing != null) {
                cache = existing;
            }
        }

        //noinspection unchecked
        return cache;
    }

    /**
     * Creates a new {@code Cache} instance associated with the specified {@code name}.
     *
     * @param name the name of the cache to create
     * @return a new {@code Cache} instance associated with the specified {@code name}.
     * @throws CacheException if the {@code Cache} instance cannot be created.
     */
    protected abstract Cache createCache(String name) throws CacheException;

    /**
     * Cleanup method that first {@link LifecycleUtils#destroy destroys} all of it's managed caches and then
     * {@link java.util.Map#clear clears} out the internally referenced cache map.
     *
     * @throws Exception if any of the managed caches can't destroy properly.
     */
    public void destroy() throws Exception {
        while (!caches.isEmpty()) {
            for (Cache cache : caches.values()) {
                LifecycleUtils.destroy(cache);
            }
            caches.clear();
        }
    }

    public String toString() {
        Collection<Cache> values = caches.values();
        StringBuilder sb = new StringBuilder(getClass().getSimpleName())
                .append(" with ")
                .append(caches.size())
                .append(" cache(s)): [");
        int i = 0;
        for (Cache cache : values) {
            if (i > 0) {
                sb.append(", ");
            }
            sb.append(cache.toString());
            i++;
        }
        sb.append("]");
        return sb.toString();
    }
}
```

其中有个很重要的方法就是getCache方法，有个很重要的属性就是private final ConcurrentMap<String, Cache> caches;caches是一个线程安全的ConcurrentMap类型，用是存放Cache的，在这里，至少存放两个Cache，一个是用来存放session的，一个是用来存放权限(Authorization)的。从getCache方法我们可以知道，shiro先从caches里拿，如果拿不到相应的Cache，就调用createCache创建一个，createCache是抽象方法，由子类实现。创建完之后，放到Map中。

如果我们在getCache方法debug就可以知道，shiro创建两个Cache放到ConcurrentMap中，一个name中ShiroCasRealm.authorizationCache(这个名字的命名规则是自定义的realm的名字加".authorizationCache",realm的定义可以参考我的shiro之旅: 二 让Shiro保护你的应用)，这个是shiro启动的时候创建的，用来保存认证信息，一个叫shiro-activeSessionCache，这个是第一次创建session的时候创建的，用来保存session。当用户登录的时候，将会把用户的权限信息保存到name为ShiroCasRealm.authorizationCache的Cache中，以后需要再使用权限信息，直接从Cache中拿而不需要再从数据库中查询。这样也有一个问题，就是当用户权限改变时，就需要用户重新登录把权限信息从新load到Cache中。不过既然我们知道Cache的名字，我们就可以拿到这个Cache，然后把用户的权限信息删除，让shiro在授权时找不到权限而shiro自己从新去load，甚至我们可以删除了，然后再从新load到Cache中。这些将在以后的文章有介绍。

session我们是放到了缓存中(memcached)，那自然我们就可以在多个应用去使用它。只要从个应用的ShiroMemcachedCache连的都是一个memcached，就可以把session保存到一个memcached中，就可以从同一个memcached去得到session。