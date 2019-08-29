[TOC]
# 三种过期策略
1. expireAfterWrite
2. expireAfterAccess
3. refreshAfterWrite:多少秒之后刷新缓存值（只有请求到来时有效，而不是新起定时任务线程）

# 选择
时效性来看，肯定是expireAfterWrite最好，当缓存值过期后，会加载新值到缓存中；但是在并发请求时，虽然只有一个请求回源load数据，但是其他请求会被阻塞等待(加锁-获取值-释放锁)，性能和吞吐量不高。而对于refreshAfterWrite，其他请求会返回旧值，这样可以大大提升吞吐量。但是，它不能保证在到达指定时间后，所有的查询都能获取到新值，而是依赖于查询请求。比如在很长一段时间内没有请求查询，那么新来的并发查询请求可能会得到一个旧值。
结合上述两种，可以控制缓存2s进行refresh，如果超过5s没有访问，则让缓存失效。（原理是在获取值时，首先判断是否过期，如果未过期，则判断是否进行refresh；否则直接回源加载；说明如果在2-5s内访问则key未过期，则进行refresh的逻辑；如果在5s外访问，则key过期，则expire的逻辑）
[expire和refresh区别](https://blog.csdn.net/abc86319253/article/details/53020432)

# 源码解析


# 代码
```java
package com.wy.guava;

import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import com.google.common.util.concurrent.ListenableFuture;
import com.google.common.util.concurrent.ListeningExecutorService;
import com.google.common.util.concurrent.MoreExecutors;

import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

public class GuavaCache {

    private static ListeningExecutorService listeningExecutorService = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(1));

    /**
     * 第一个请求回源load数据，其他线程同步阻塞等待数据返回
     */
    public static LoadingCache<String, Object> cache1 = CacheBuilder
            .newBuilder()
            .expireAfterWrite(2, TimeUnit.SECONDS)
            .build(new CacheLoader<String, Object>() {
                @Override
                public Object load(String key) throws InterruptedException {
                    System.out.println(Thread.currentThread().getName() + " load=============");
                    TimeUnit.SECONDS.sleep(5);
                    return "345";
                }
            });

    public static LoadingCache<String, Object> cache2 = CacheBuilder
            .newBuilder()
            .expireAfterAccess(2, TimeUnit.SECONDS)
            .build(new CacheLoader<String, Object>() {
                @Override
                public Object load(String key) throws InterruptedException {
                    System.out.println(Thread.currentThread().getName() + " load=============");
                    TimeUnit.SECONDS.sleep(5);
                    return "123";
                }
            });

    /**
     * 特点：
     * 其他线程返回旧值
     * 只有一个线程回源load数据（前提：需要初始化值放在缓存中，如果未初始化值，则其他请求也会阻塞等待第一个请求返回值）
     * 优点：其他线程不会被阻塞，吞吐量高
     * 缺点：同步加载数据，第一个请求依然很慢
     */
    public static LoadingCache<String, Object> cache3 = CacheBuilder
            .newBuilder()
            .refreshAfterWrite(2, TimeUnit.SECONDS)
            .build(new CacheLoader<String, Object>() {
                @Override
                public Object load(String key) throws InterruptedException {
                    System.out.println(Thread.currentThread().getName() + " load=============");
                    TimeUnit.SECONDS.sleep(10);
                    System.out.println("========end load==============");
                    return "123";
                }
            });

    /**
     * 如果未初始化值，则并发请求时，其他请求会阻塞等待结果(加锁-获取值-释放锁)；
     * 如果在2-5s内并发请求时，其他请求返回旧值，第一个请求回源load数据；
     * 如果5s外并发请求时，其他请求会阻塞等待结果；
     * （源码：先判断是否过期，如果未过期则判断是否进行refresh；否则直接load数据）
     * 避免了refresh因为长时间未访问，再突然访问时返回旧值
     */
    public static LoadingCache<String, Object> cache4 = CacheBuilder.newBuilder()
            .maximumSize(10)
            .expireAfterWrite(5, TimeUnit.SECONDS)
            .refreshAfterWrite(2, TimeUnit.SECONDS)
            .build(new CacheLoader<String, Object>() {
                @Override
                public Object load(String key) throws Exception {
                    System.out.println(Thread.currentThread().getName() + " load=============");
                    TimeUnit.SECONDS.sleep(6);
                    System.out.println("========end load==============");
                    return "123";
                }
            });

    /**
     * 重写reload方法（仅对refresh生效）
     * 异步刷新数据，这样所有请求都会优先返回旧值
     */
    public static LoadingCache<String, Object> cache5 = CacheBuilder.newBuilder()
            .maximumSize(10)
            .refreshAfterWrite(3, TimeUnit.SECONDS)
            .build(new CacheLoader<String, Object>() {
                @Override
                public Object load(String key) throws Exception {
                    System.out.println(Thread.currentThread().getName() + " load=============");
                    TimeUnit.SECONDS.sleep(6);
                    System.out.println("========end load==============");
                    return "123";
                }

                @Override
                public ListenableFuture<Object> reload(String key, Object oldValue) throws Exception {
                    System.out.println("===异步刷新=====");
                    return listeningExecutorService.submit(() -> {
                        System.out.println(Thread.currentThread().getName() + " load=============");
                        TimeUnit.SECONDS.sleep(6);
                        System.out.println("========end load==============");
                        return "345";
                    });
                }
            });


}

