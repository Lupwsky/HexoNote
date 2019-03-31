---
title: Java-Guava工具类之缓存的使用
date: 2019-01-26 13:25:51
categories: Java
---

之前一直有听说过谷歌的 Guava, 但是在项目中也仅是非常简单使用过其中的缓存功能 , 称着这段时间项目不是很忙, 抓紧时间系统的了解下 (主要参考 [官方的文档](https://github.com/google/guava) 和 [Guava官方教程(中文版)](https://willnewii.gitbooks.io/google-guava/content/index.html)), 这里记录下自己的学习笔记, 这篇笔记主要是关于 Guava 中缓存的使用

# 缓存的使用

## 创建缓存

创建缓存可以使用 Guava 提供的 CacheBuilder 类来创建, 使用示例如下:

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
        // 缓存大小
        .maximumSize(100)
        .build(new CacheLoader<String, String>() {
            // 对应 get() 方法
            @Override
            @ParametersAreNonnullByDefault
            public String load(String s) {
                log.info("key = {}, value = {}", s, s + "Value");
                return s + "Value";
            }

            // 对应 getAll() 方法, 默认的调用父类的 getAll(), 父类 getAll() 方法遍历调用 get() 方法
            // 如有特殊的需求可以在这里重写 getAll() 方法
            @Override
            public Map<String, String> loadAll(Iterable<? extends String> keys) throws Exception {
                return super.loadAll(keys);
            }
        });
```

<!-- more -->

或者不使用 LoadingCahce, 直接使用 Cache 对象, 使用 Callback 的方式, 使用如下:

```java
Cache<String, String> cache = CacheBuilder.newBuilder().maximumSize(1000).build();  
String resultValue = cache.get("A", () -> {  
    public String call(String key) {  
        return key + "G";
    }  
});  
```

## 获取缓存

```java
// get 方法返回已经缓存的值, 如果不存在, 则从 CacheLoader.load(s) 方法中加载方法
String cacheValue = cache.get("test");
log.info("cacheValue = {}", cacheValue);

// 再次测试, 确认读取的值是缓存
cacheValue = cache.get("test");
log.info("cacheValue = {}", cacheValue);

// 返回的是一个不可变的 Map
ImmutableMap<String, String> dataMap = cache.getAll(Lists.newArrayList("test1", "test2"));
dataMap.forEach((key, value) -> {
    log.info("[getAll] key = {}, value = {}", key, value);
});
```

## get(K, Callable) 方法获取缓存

所有类型的 Guava Cache, 不管有没有添加自动加载功能, 都支持 get(K, Callable) 方法, 返回缓存中相应的值, 或者用给定的 Callable 运算并把结果加入到缓存中, 即 get(K, Callable) 方法 是如果有缓存则返回, 否则运算, 缓存, 然后返回值, Callable 运算会覆盖自动加载功能

```java
cache.get("FF", () -> {
        log.info("get(K, Callable) 方法会覆盖自动加载功能");
        return "FFFF";
    });
```

## 显示添加和删除缓存

```java
// 显示添加缓存
cache.put("TT", "TTTT");

// 显示删除缓存
cache.invalidate("TT");
```

# 缓存回收策略

Guava Cache 提供了三种基本的缓存回收方式: 基于容量回收, 定时回收和基于引用回收

## 基于容量的回收

* 如果要规定缓存项的数目不超过固定值, 只需使用 CacheBuilder.maximumSize(long), 缓存将尝试回收最近没有使用或总体上很少使用的缓存项, 注意的是在缓存项的数目达到限定值之前, 缓存就可能进行回收操作——通常来说, 这种情况发生在缓存项的数目逼近限定值时
* 不同的缓存项有不同的 "权重" 例如, 如果你的缓存值, 占据完全不同的内存空间，
* 可以使用 CacheBuilder.weigher(Weigher) 指定一个权重函数, 并且用 CacheBuilder.maximumWeight(long) 指定最大总权重,在权重限定场景中, 除了要注意回收也是在权重逼近限定值时就进行了, 知道权重函数是在缓存创建时就开始计算的

## 定时回收

* expireAfterAccess(long, TimeUnit): 缓存项在给定时间内没有被读/写访问, 则回收, 这种缓存的回收顺序和基于大小回收策略一样
* expireAfterWrite(long, TimeUnit):  缓存项在给定时间内没有被写访问 (创建或覆盖), 则回收, 这种缓存的回收顺序和基于大小回收策略一样

## 基于引用的回收

* CacheBuilder.weakKeys(): 使用弱引用存储键, 当键没有其它 (强或软) 引用时, 缓存项可以被垃圾回收
* CacheBuilder.weakValues(): 使用弱引用存储值, 当值没有其它 (强或软) 引用时, 缓存项可以被垃圾回收
* CacheBuilder.softValues(): 使用软引用存储值, 软引用只有在响应内存需要时, 才按照全局最近最少使用的顺序回收

使用软引用的会对性能进行一定的影响, 通常建议使用基于容量回收的策略

# 缓存移出监听器

使用 CacheBuilder.removalListener(RemovalListener) 可以声明一个监听器, 以便缓存项被移除时能够感知, 并做一些额外操作

```java
LoadingCache<String, String> cacheRemoveListener = (LoadingCache<String, String>) CacheBuilder.newBuilder()
        .maximumSize(1000)
        .removalListener((RemovalListener<String, String>) removalNotification -> {
            String key = removalNotification.getKey();
            String value = removalNotification.getValue();
            log.info("[移除监听] key = {}, value = {}", key, value);
        }).build();
```

注: 默认情况下, 监听器方法是在移除缓存时同步调用的, 如果监听器里面执行方法时耗较长, 会极大的拖慢正常的缓存请求, 可以使用 RemovalListeners.asynchronous(RemovalListener, Executor) 创建一个异步的监听器来处理这种情况

# 缓存的清理机制

使用 CacheBuilder 构建的缓存不会 "自动" 执行清理和回收工作, 也不会在某个缓存项过期后马上清理, 它会在写操作时顺带做少量的维护工作, 或者写操作实在太少的话, 偶尔在读操作时清理, 这样设计的原因在于: 如果要自动地持续清理缓存, 就必须有一个线程去维护, 这个线程会和用户操作竞争共享锁,
如果你的缓存是高吞吐的, 那就无需担心缓存的维护和清理等工作, 如果缓存只会偶尔有写操作, 而你又不想清理工作阻碍了读操作, 可以创建自己的维护线程, 以固定的时间间隔调用 Cache.cleanUp()

# 缓存的刷新

LoadingCache.refresh(K) 用于刷新缓存, 这个过程可以是异步的, 在刷新操作全部完成前, 缓存仍然可以向其他线程返回旧值, 刷新缓存会调用 CacheLoader.reload(K, V), 如果没有重写 reload() 方法, 默认会调用父类的 reload() 方法, 父类的 reload() 方法则是直接调用的是 load() 方法, 如果需要定时刷新, CacheBuilder.refreshAfterWrite(long, TimeUnit) 可以为缓存增加自动定时刷新功能

# 缓存提供的统计功能

CacheBuilder.recordStats() 用来开启 Guava Cache 的统计功能统计打开后, Cache.stats() 方法会返回 CacheStats 对象以提供如下统计信息:

* hitRate(): 缓存命中率
* averageLoadPenalty(): 加载新值的平均时间, 单位为纳秒
* evictionCount(): 缓存项被回收的总数, 不包括显式清除