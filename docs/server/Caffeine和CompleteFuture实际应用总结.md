[TOC]



### 一.Caffeine 原理

#### 1.1 常见缓存淘汰算法

- FIFO：先进先出，在这种淘汰算法中，先进入缓存的会先被淘汰，会导致命中率很低。
- LRU：最近最少使用算法，每次访问数据都会将其放在我们的队尾，如果需要淘汰数据，就只需要淘汰队首即可。
- LFU：最近最少频率使用，利用额外的空间记录每个数据的使用频率，然后选出频率最低进行淘汰。这样就避免了 LRU 不能处理时间段的问题。

#### 1.2 LRU和LFU缺点：

- LRU 实现简单，在一般情况下能够表现出很好的命中率，是一个“性价比”很高的算法，平时也很常用。虽然 LRU 对突发性的稀疏流量（sparse bursts）表现很好，但同时也会产生缓存污染，举例来说，如果偶然性的要对全量数据进行遍历，那么“历史访问记录”就会被刷走，造成污染。

- 如果数据的分布在一段时间内是固定的话，那么 LFU 可以达到最高的命中率。但是 LFU 有两个缺点，第一，它需要给每个记录项维护频率信息，每次访问都需要更新，这是个巨大的开销；第二，对突发性的稀疏流量无力，因为前期经常访问的记录已经占用了缓存，偶然的流量不太可能会被保留下来，而且过去的一些大量被访问的记录在将来也不一定会使。

#### 1.3 W-TinyLFU 算法：

TinyLFU 算法：

- 解决第一个问题是采用了 Count–Min Sketch 算法。

- 为了解决 LFU 不便于处理随时间变化的热度变化问题，TinyLFU 采用了基于 “滑动时间窗” 的热度衰减算法，简单理解就是每隔一段时间，便会把计数器的数值减半，以此解决 “旧热点” 数据难以清除的问题。



#### 内部结构

- Cache    的内部包含着一个ConcurrentHashMap，这也是存放我们所有缓存数据的地方，众所周知，ConcurrentHashMap是一个并发安全的容器，这点很重要，可以说Caffeine其实就是一个被强化过的ConcurrentHashMap。
- Scheduler 定期清空数据的一个机制，可以不设置，如果不设置则不会主动的清空过期数据。
- Executor 指定运行异步任务时要使用的线程池。可以不设置，如果不设置则会使用默认的线程池，也就是ForkJoinPool.commonPool()

#### Caffeine提供的能力

1. 自动把数据加载到本地缓存中，并且可以配置异步；
2. 基于数量剔除策略；
3. 基于失效时间剔除策略，这个时间是从最后一次操作算起【访问或者写入】；
4. 异步刷新；
5. Key会被包装成Weak引用；Value会被包装成Weak或者Soft引用，从而能被GC掉，而不至于内存泄漏；
6. 数据剔除提醒；
7. 写入广播机制；
8. 缓存访问可以统计；


#### 1.3.1  常用配置参数

```
expireAfterWrite：写入间隔多久淘汰；
expireAfterAccess：最后访问后间隔多久淘汰；
refreshAfterWrite：写入后间隔多久刷新，实现异步加载数据。
maximumSize：缓存 key 的最大个数；
softValues：value设置为软引用，在内存溢出前可以直接淘汰；
executor：选择自定义的线程池，默认的线程池实现是 ForkJoinPool.commonPool()；
recordStats：缓存的统计数据，比如命中率等；
removalListener：缓存淘汰监听器；
```

#### 1.3.2  手动加载数据

**实战代码：**

```
public class CaffeineManualTest {

    public static void main(String[] args)  {
        // 初始化缓存，设置了1分钟的写过期，100的缓存最大个数
        Cache<Integer, Integer> cache = Caffeine.newBuilder()
                .expireAfterWrite(1, TimeUnit.MINUTES)
                .maximumSize(100)
                .build();
        int key = 1;
        // 使用getIfPresent方法从缓存中获取值。如果缓存中不存指定的值，则方法将返回 null：
        System.out.println(cache.getIfPresent(key));

        // 也可以使用 get 方法获取值，该方法将一个参数为 key 的 Function 作为参数传入。如果缓存中不存在该 key
        // 则该函数将用于提供默认值，该值在计算后插入缓存中：
        System.out.println(cache.get(key, new Function<Integer, Integer>() {
            @Override
            public Integer apply(Integer integer) {
                return 2;
            }
        }));

        // 校验key1对应的value是否插入缓存中
        System.out.println(cache.getIfPresent(key));

        // 手动put数据填充缓存中
        int value1 = 2;
        cache.put(key, value1);

        // 使用getIfPresent方法从缓存中获取值。如果缓存中不存指定的值，则方法将返回 null：
        System.out.println(cache.getIfPresent(key));

        // 移除数据，让数据失效
        cache.invalidate(key);
        System.out.println(cache.getIfPresent(key));
    }
}
```

上面提到了两个get数据的方式，一个是getIfPercent，没数据会返回Null，而get数据的话则需要提供一个Function对象，当缓存中不存在查询的key则将该函数用于提供默认值，并且会插入缓存中。当多个线程同时get的时候，由于Caffeine内部最主要的数据结构就是一个ConcurrentHashMap，而get的过程最终执行的便是ConcurrentHashMap.compute，这里仅会被执行一次。



**输出结果：**

```
null
2
2
2
null
```

#### 1.3.3  同步加载数据

```
public class CaffeineManualTest {

    LoadingCache<Integer, Integer> cache = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .maximumSize(100)
            .build(new CacheLoader<Integer, Integer>() {
                @Nullable
                @Override
                public Integer load(@NonNull Integer key) throws InterruptedException {
                    System.out.println(Thread.currentThread().getName());
                    TimeUnit.SECONDS.sleep(10);
                    // 可以从数据库加载值
                    return 1232;
                }
            });

    public void getCache(){
        // 初始化缓存，设置了1分钟的写过期，100的缓存最大个数
        int key1 = 1;
        // get数据，取不到则从数据库中读取相关数据，该值也会插入缓存中：
        Integer value1 = cache.get(key1);
        System.out.println(value1);
    }
    public static void main(String[] args)  {
        new CaffeineManualTest().getCache();
    }
}
```

同步加载数据指的是，在get不到数据时最终会调用build构造时提供的CacheLoader对象中的load函数，如果返回值则将其插入缓存中，并且返回，这是一种同步的操作。



输出结果：

```
main //主线程
1232
```



#### 1.3.4 异步加载数据

**第一种方式：**

目前异步加载数据在配置refreshAfterWrite后可以实现异步去加载，不过第一次加载会被阻塞。因为同步去获取数据，后续结果就会采用异步去进行刷新。

```
public class CaffeineManualTest {

    LoadingCache<Integer, Integer> cache = Caffeine.newBuilder()
            .expireAfterWrite(3, TimeUnit.SECONDS)
            .refreshAfterWrite(1, TimeUnit.SECONDS)
            .maximumSize(100)
            .build(new CacheLoader<Integer, Integer>() {
                @Nullable
                @Override
                public Integer load(@NonNull Integer key) throws InterruptedException {
                    System.out.println(Thread.currentThread().getName());
                    TimeUnit.SECONDS.sleep(10);
                    // 可以从数据库加载值
                    return 1232;
                }
            });

    public void getCache() throws InterruptedException {
        while (true) {
            // 初始化缓存，设置了1分钟的写过期，100的缓存最大个数
            int key1 = 1;
            // get数据，取不到则从数据库中读取相关数据，该值也会插入缓存中：
            Integer value1 = cache.get(key1);

            System.out.println(value1);
            TimeUnit.SECONDS.sleep(2);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new CaffeineManualTest().getCache();
    }
}
```

加入refreshAfterWrite后，当第一次写过写入到缓存以后，后续读取值将会采用异步刷新的方式去读取数据，复制代码亲自测试能更好理解

输出结果：

```
main 
1232 --会被阻塞十秒才能获取到值
ForkJoinPool.commonPool-worker-19
1232--不会被阻塞，获取缓存值，load异步去加载数据，下一次读取新的数据，如果重写了reload方法，则会执行reload去加载数据
1232
1232
1232
1232
1232
1232
ForkJoinPool.commonPool-worker-5
```

 重写reload：

```
public class CaffeineManualTest {

    LoadingCache<Integer, Integer> cache = Caffeine.newBuilder()
            .expireAfterWrite(3, TimeUnit.SECONDS)
            .refreshAfterWrite(1, TimeUnit.SECONDS)
            .maximumSize(100)
            .build(new CacheLoader<Integer, Integer>() {
                @Nullable
                @Override
                public Integer load(@NonNull Integer key) throws InterruptedException {
                    System.out.println(Thread.currentThread().getName());
                    TimeUnit.SECONDS.sleep(10);
                    // 可以从数据库加载值
                    return 1232;
                }

                @Nullable
                @Override
                public Integer reload(@NonNull Integer key, @NonNull Integer oldValue) throws Exception {
                    System.out.println(Thread.currentThread().getName());
                    TimeUnit.SECONDS.sleep(10);
                    return 33333;
                }
            });

    public void getCache() throws InterruptedException {
        while (true) {
            // 初始化缓存，设置了1分钟的写过期，100的缓存最大个数
            int key1 = 1;
            // get数据，取不到则从数据库中读取相关数据，该值也会插入缓存中：
            Integer value1 = cache.get(key1);

            System.out.println(value1);
            TimeUnit.SECONDS.sleep(2);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new CaffeineManualTest().getCache();
    }
}
```

输出结果：

```
main 主线程加载数据，阻塞
1232 获取到主线程的数据
ForkJoinPool.commonPool-worker-19 异步去加载数据，被sleep阻塞
1232 读取缓存值
1232 读取缓存值
1232 读取缓存值
1232 读取缓存值
1232 读取缓存值
33333 读到reload加载的数据
33333 读取缓存值
ForkJoinPool.commonPool-worker-5 
33333
33333
33333
33333
33333
33333
ForkJoinPool.commonPool-worker-19
33333
```



future 获取数据的时候，不是主动去进行加载数据的，都是等到调用的时候才去主动加载，因此，对于缓存数据来说，当前拿到的值不是最新的，会去异步加载数据，下次调用获取最新值。



**第二种方式：**

```
public class CaffeineManualTest {

    // 使用executor设置线程池
    AsyncCache<Integer, Integer> asyncCache = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .maximumSize(100).executor(Executors.newSingleThreadExecutor()).buildAsync();

    public CompletableFuture<Integer>  getCache(Integer key)  {
      return  asyncCache.get(key, new Function<Integer, Integer>() {
            @SneakyThrows
            @Override
            public Integer apply(Integer key) {
                // 执行所在的线程不在是main，而是ForkJoinPool线程池提供的线程
                System.out.println("当前所在线程：" + Thread.currentThread().getName());
                try {
                    TimeUnit.SECONDS.sleep(10);
                }catch (Exception ex){}
                int value =12121212;
                return value;
            }
        });
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException, TimeoutException {
        CompletableFuture<Integer> integerCompletableFuture = new CaffeineManualTest().getCache(1);

        System.out.println("先做点别的事情");

        TimeUnit.SECONDS.sleep(5);

        System.out.println(integerCompletableFuture.get(5,TimeUnit.SECONDS));//最多等待五秒，加上之前等待的5秒，因此异步加载数据不会超时
    }
}
```

输出结果：

```
当前所在线程：pool-1-thread-1
先做点别的事情
12121212
```



超时演示：

```
public class CaffeineManualTest {

    // 使用executor设置线程池
    AsyncCache<Integer, Integer> asyncCache = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .maximumSize(100).executor(Executors.newSingleThreadExecutor()).buildAsync();

    public CompletableFuture<Integer>  getCache(Integer key)  {
      return  asyncCache.get(key, new Function<Integer, Integer>() {
            @SneakyThrows
            @Override
            public Integer apply(Integer key) {
                // 执行所在的线程不在是main，而是ForkJoinPool线程池提供的线程
                System.out.println("当前所在线程：" + Thread.currentThread().getName());
                try {
                    TimeUnit.SECONDS.sleep(10);
                }catch (Exception ex){}
                int value =12121212;
                return value;
            }
        });
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException, TimeoutException {
        CompletableFuture<Integer> integerCompletableFuture = new CaffeineManualTest().getCache(1);

        System.out.println("先做点别的事情");

        TimeUnit.SECONDS.sleep(5);

        System.out.println(integerCompletableFuture.get(2,TimeUnit.SECONDS));
    }
}
```

输出结果：

```
当前所在线程：pool-1-thread-1
先做点别的事情
Exception in thread "main" java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1886)
	at java.base/java.util.concurrent.CompletableFuture.get(CompletableFuture.java:2021)
	at com.idslilang.go.CaffeineManualTest.main(CaffeineManualTest.java:45)

```



https://segmentfault.com/a/1190000038665523

**监听过期数据**

```
  this.cache = Caffeine.newBuilder().initialCapacity(3000).maximumSize(10000)
                .refreshAfterWrite(1,TimeUnit.SECONDS)
                .expireAfterWrite(3, TimeUnit.SECONDS)
                .executor(ThreadPoolUtils.getThreadPoolExecutor())
                .removalListener(new RemovalListener<Object, Object>() {
                    @Override
                    public void onRemoval(@Nullable Object key, @Nullable Object value, @NonNull RemovalCause removalCause) {
                        System.out.println(key);
                        System.out.println(value);
                    }
                })
                .recordStats()
                .build(initLoader());
```



主动请求的时候才会监听到，不请求的时候无法监听到。



当要cache自动清除数据，预先加载时候可以如下配置：

但是这种存在一种风险，当大量key同一时间失效的时候，会造成瞬间大量线程访问数据库，可以考虑重新申明一个队列，监听队列处理。

```
    public void init() {

        this.cache = Caffeine.newBuilder().initialCapacity(3000).maximumSize(10000)
                .expireAfterWrite(1, TimeUnit.SECONDS)
                .executor(ThreadPoolUtils.getThreadPoolExecutor())
                .removalListener(new RemovalListener<Integer, String>() {
                    @Override
                    public void onRemoval(@Nullable Integer key, @Nullable String value, @NonNull RemovalCause removalCause) {
                        System.out.println("expire la ");
                        get(key);
                    }
                })
                .scheduler(Scheduler.forScheduledExecutorService( Executors.newSingleThreadScheduledExecutor()))
                .buildAsync(initLoader());

    }
```



#### 1.3.3 异步加载数据

```
public class AdsPreCheckConfigListCache {
    private static Logger LOGGER = LoggerFactory.getLogger(AdsPreCheckConfigListCache.class);

    private AsyncLoadingCache<Integer, String> cache = null;

    /**
     * @description 初始化caffine
     * @author idslilang 
     * @updateTime 2021/6/2 14:48
     * @Return
     */
    public void init() {

        this.cache = Caffeine.newBuilder().initialCapacity(3000).maximumSize(10000)
                .expireAfterWrite(1, TimeUnit.SECONDS)
                .executor(ThreadPoolUtils.getThreadPoolExecutor())
                .removalListener(new RemovalListener<Integer, String>() {
                    @Override
                    public void onRemoval(@Nullable Integer key, @Nullable String value, @NonNull RemovalCause removalCause) {
                        System.out.println("expire la ");
                    }
                })
                .buildAsync(initLoader());

    }

    public AsyncLoadingCache<Integer, String> getCache(){
        return cache;
    }

    private AsyncCacheLoader<Integer, String> initLoader() {
        return new AsyncCacheLoader<Integer, String>() {

            @Override
            public @NonNull CompletableFuture<String> asyncLoad(@NonNull Integer key, @NonNull Executor executor) {
                System.out.println("i come load ");
                return    CompletableFuture.supplyAsync(()->{
                    return String.valueOf("load---->"+key);
                },executor);
            }

            @Override
            public @NonNull CompletableFuture<String> asyncReload(@NonNull Integer key, @NonNull String oldValue, @NonNull Executor executor) {
                System.out.println("i come reload ");
                return    CompletableFuture.supplyAsync(()->{
                    return String.valueOf("reload---->"+key);
                },executor);

            }

        };
    }

    public Object get(Integer bizId){
        try {
            return cache.get(bizId).get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
        return null;
    }

    public static void main(String[] args) throws InterruptedException, TimeoutException, ExecutionException {
        AdsPreCheckConfigListCache cache = new AdsPreCheckConfigListCache();
        cache.init();
        while (true){
            System.out.println(cache.get(1));
            TimeUnit.SECONDS.sleep(6);
        }

    }
}
```

异步加载数据时候，可以对future设置超时时间，实现更加灵活的控制。



### 二：异步编程CompleteFuture实战

#### 2.1 Future获取任务结果

使用`Future`获得异步执行结果时，要么调用阻塞方法`get()`，要么轮询看`isDone()`是否为`true`，这两种方法主线程也会被迫等待。

从Java 8开始引入了`CompletableFuture`，它针对`Future`做了改进，可以传入回调对象，当异步任务完成或者发生异常时，自动调用回调对象的回调方法。



```
public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newSingleThreadExecutor();
        Future<String> stringFuture = executor.submit(new Callable<String>() {
            @Override
            public String call() throws Exception {
                Thread.sleep(2000);
                return "future thread";
            }
        });
        Thread.sleep(1000);
        System.out.println("main thread");
        System.out.println(stringFuture.get());

}
```



#### 2.2 CompletableFuture 异步执行任务

##### 2.2.1 异步任务接口

```
//接受Runnable，无返回值，使用ForkJoinPool.commonPool()线程池
public static CompletableFuture<Void> runAsync(Runnable runnable)

//接受Runnable，无返回值，使用指定的executor线程池  
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
  
//接受Supplier，返回U，使用ForkJoinPool.commonPool()线程池 这个线程池默认线程数是 CPU 的核数。
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
  
//接受Supplier，返回U,使用指定的executor线程池 
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)

```

DEMO：

```
public CompletableFuture<String> getCompletableFutureData(){    CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(()->{        return  "return world";    }, executor);    return  completableFuture;}
```



```
public void runAsync(){    CompletableFuture.runAsync(()->{       //do something async    }, executor);;}
```



##### 2.2.2 设置任务结果

```
public boolean complete(T value);public boolean completeExceptionally(Throwable ex);CompletableFuture.completedFuture();
```

设置结果DEMO1：

```
public void runAsync() throws ExecutionException, InterruptedException {
        CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(()->{
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "我正在返回异步结果";
        },executor);
        completableFuture.complete("单独设置返回结果");
        System.out.println(completableFuture.get());

        TimeUnit.SECONDS.sleep(6);
        System.out.println(completableFuture.get());

    }
```

输出结果：

```
单独设置返回结果
单独设置返回结果
```

设置结果DEMO2：

```
public void runAsync() throws ExecutionException, InterruptedException {
    CompletableFuture<String> completableFuture =CompletableFuture.completedFuture("我完成了任务");
    completableFuture.complete("单独设置返回结果");
    System.out.println(completableFuture.get());
    TimeUnit.SECONDS.sleep(6);
    System.out.println(completableFuture.get());

}
```

输出结果：

```
我完成了任务
我完成了任务
```

需要注意点：

```
一旦 complete 设置成功，CompletableFuture 返回结果就不会被更改，即使后续 CompletableFuture 任务执行结束。

同样，申明一个完成任务的future（CompletableFuture.completedFuture("我完成了任务")），后续再对其操作也不起作用。
```

异常任务设置DEMO3：

```
public void runAsync() throws ExecutionException, InterruptedException {
    CompletableFuture<String> completableFuture =CompletableFuture.supplyAsync(()->{
      return "我正在运行任务";
    },executor);
    completableFuture.completeExceptionally(new RuntimeException("我发生了异常"));
    System.out.println(completableFuture.get());

}
```

输出结果中将不会得到任务结果，将会返回异常信息，项目中可以根据任务结果设置自定义异常信息，便于统一处理任务结果或者做任务监控。

##### 2.2.3 串行关系

```
//有返回值
CompletableFuture#thenApply
//无返回值
CompletableFuture#thenAccept
//异步执行
CompletableFuture#thenApplyAsync
CompletableFuture#thenAcceptAsync(Consumer<? super T> action);
```

代码DEMO：

```
   CompletableFuture completableFuture = CompletableFuture.supplyAsync(()->{
       try {
           TimeUnit.SECONDS.sleep(5);
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
       return " supplyAsync start --->";
   },executor).thenApply((res)->{
       //收到上层传递结果，继续传递结果
       return res+"thenApply continue ---> ";
   }).thenAccept(res->{
       // 收到上层传递结果，不继续传递结果，返回null
       System.out.println(res+"thenAccept done ");
   }).thenAccept(res->{
       //收到结果null
       System.out.println(res+"thenAccept done ");
   });
```

当使用同步执行的时候，需要等到所有串行结果执行完毕future才能获取到值。调用thenApply方法执行第二个任务时，则第二个任务和第一个任务是共用同一个线程池。调用thenApplyAsync执行第二个任务时，则第一个任务使用的是你自己传入的线程池，如果没有传入线程池，第二个任务使用的是ForkJoin线程池，当获取completableFuture任务结果时，也需要等待所有串行任务执行完毕才行。



注意点：

当我们完成了一个异步任务，还需要操作一些与异步任务相关的其他操作，如刷缓存，写日志等。则可以采用如下方式实现

```
public CompletableFuture runAsync() throws ExecutionException, InterruptedException {
    CompletableFuture<String> completableFuture = CompletableFuture.supplyAsync(() -> {
        return " supplyAsync start --->";
    }, executor);
	
    //链路已经断开，后续不影响completableFuture 返回值，业务方获取到的completableFuture信息为“supplyAsync start --->”
    completableFuture.thenAcceptAsync((res) -> {
    //执行任务  
    }, executor);

    return completableFuture;

}

```



##### 2.2.4 并行执行关系

由于异步执行同上类似，不再进行展示

```
//等待两个对象执行完毕，获取返回结果
CompletableFuture#thenCombine

//等待两个对象执行完毕，不获取返回结果
CompletableFuture#runAfterBoth

//所有future对象执行完毕，不获取返回结果
CompletableFuture#allOf
 
```



代码DEMO：

```
public void runAsync() {
    CompletableFuture<Integer> moneyFutrue = CompletableFuture.supplyAsync(() -> {
        System.out.println("查询钱包余额");
        return 1000;
    }, executor);

    CompletableFuture<Integer> foodCostFuture = CompletableFuture.supplyAsync(() -> {
        System.out.println("查询食物价格");
        return 100;
    }, executor);

    CompletableFuture<Integer> resData = moneyFutrue.thenCombine(foodCostFuture, (money, foodCost) -> {
        System.out.println("剩余额度：");
        return money - foodCost;
    });
    
    CompletableFuture<Integer> resData = moneyFutrue.thenCombine(foodCostFuture, (money, foodCost) -> {
        System.out.println("剩余额度：");

        System.out.println(money - foodCost); 
    });


}
```

代码DEMO2: allof 执行 获取所有任务结果，对于实际的项目中，可以定义需要的对象去接收传递参数。

```
    public CompletableFutur<List<Integer>> runAsync() {


        CompletableFuture<Integer> moneyFutrue = CompletableFuture.supplyAsync(() -> {
            System.out.println("去王庄收的食物数量：");
            return 1000;
        }, executor);

        CompletableFuture<Integer> foodCostFuture = CompletableFuture.supplyAsync(() -> {
            System.out.println("去李庄收的食物数量：");
            return 100;
        }, executor);

        List<CompletableFuture<Integer>> futures = new ArrayList<>();
        futures.add(moneyFutrue);
        futures.add(foodCostFuture);

        CompletableFuture<Void> completableFuture = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));

        List<Integer> results = new ArrayList<>();
        CompletableFuture<List<Integer>> resFutur=  completableFuture.thenApply((res) -> {
            for (CompletableFuture<Integer> cmplfuture:futures){
                //如果取不到，设置一个默认值
                results.add(cmplfuture.getNow(0));
            }
            return results;
        });

        return resFutur;

    }
```

此时可以用resFutur中的get或者join方法去获取异步任务执行结果。

##### 2.2.5 OR执行关系

```
//等待任何一个对象执行完毕，获取返回结果CompletableFuture#acceptEither//等待任何对象执行完毕，不获取返回结果CompletableFuture#runAfterEither//所有future对象执行完毕，不获取返回结果CompletableFuture#anyOf
```

代码DEMO：

```
public void runAsync() {

    CompletableFuture<String> cf
            = CompletableFuture.supplyAsync(() -> {
        sleep(5, TimeUnit.SECONDS);
        return "坐公交车";
    });// 1

    CompletableFuture<String> cf2 = cf.supplyAsync(() -> {
        sleep(3, TimeUnit.SECONDS);
        return "坐地铁";
    });
    CompletableFuture<String> cf3 = cf2.applyToEither(cf, s -> s);

    System.out.println(cf2.join());

}

```

输出结果：

```
坐地铁
```

代码DEMO2：

```
public void runAsync() throws ExecutionException, InterruptedException {

    CompletableFuture<String> cf
            = CompletableFuture.supplyAsync(() -> {
        sleep(5, TimeUnit.SECONDS);
        return "坐公交车";
    });// 1

    CompletableFuture<String> cf2 = cf.supplyAsync(() -> {
        sleep(3, TimeUnit.SECONDS);
        return "坐地铁";
    });

    ArrayList<CompletableFuture<String>> futures = new ArrayList<CompletableFuture<String>>();
    futures.add(cf);
    futures.add(cf2);

    CompletableFuture<Object> res = CompletableFuture.anyOf(cf,cf2);

    System.out.println(res.get());
}
```

数据结果：

```
坐地铁
```

##### 2.2.6 异常处理

```
//whenComplete 与 handle 方法就类似于 try..catch..finanlly 中 finally 代码块。无论是否发生异常，都将会执行的。这两个方法区别在于 handle 支持返回结果。
CompletableFuture#handle

CompletableFuture#whenComplete

//使用方式类似于 try..catch 中 catch 代码块中异常处理。
CompletableFuture#exceptionally 
```

代码DEMO：

```
public void runAsync() throws ExecutionException, InterruptedException {

    CompletableFuture<String> cf
            = CompletableFuture.supplyAsync(() -> {
        int a = 1 / 0;
        sleep(5, TimeUnit.SECONDS);
        return "坐公交车";
    });

    cf.exceptionally(ex->{
        return null;
    }).thenAccept(res->{
        System.out.println(res);
    });

}
```

数据结果：

```
null
```

代码DEMO2：

```
public void runAsync() throws ExecutionException, InterruptedException {

    CompletableFuture<String> cf
            = CompletableFuture.supplyAsync(() -> {
        int a = 1 / 0;
        sleep(5, TimeUnit.SECONDS);
        return "坐公交车";
    });


    CompletableFuture<String> cf2 = cf.supplyAsync(() -> {
        sleep(3, TimeUnit.SECONDS);
        return "坐地铁";
    });

    List<CompletableFuture<String>> futures = new ArrayList<>();
    futures.add(cf);
    futures.add(cf);
    
    //如果不捕获异常，是无法执行到thenApply去取结果的
    CompletableFuture<List<String>> allRes= CompletableFuture.allOf(cf, cf2).exceptionally((ex) -> {
        return null;
    }).thenApply(res -> {
        List<String> result = new ArrayList<>();
        for (CompletableFuture<String> future : futures) {
            result.add(future.getNow(""));
        }
        return result;
    });


}

```

代码DEMO3：

```
public void runAsync() throws ExecutionException, InterruptedException {

    CompletableFuture<String> cf
            = CompletableFuture.supplyAsync(() -> {
        int a = 1 / 0;
        sleep(5, TimeUnit.SECONDS);
        return "坐公交车";
    });


    CompletableFuture<String> cf2 = cf.supplyAsync(() -> {
        sleep(3, TimeUnit.SECONDS);
        return "坐地铁";
    });

    List<CompletableFuture<String>> futures = new ArrayList<>();
    futures.add(cf);
    futures.add(cf);
    CompletableFuture<List<String>> allRes= CompletableFuture.allOf(cf, cf2).whenComplete((res,ex)->{
        System.out.println(res);
    }).thenApply(res -> {
        List<String> result = new ArrayList<>();
        for (CompletableFuture<String> future : futures) {
            result.add(future.getNow(""));
        }
        return result;
    });
    //先异常捕获，再进行任务处理，
   CompletableFuture<List<String>> allRes= CompletableFuture.allOf(cf, cf2).exceptionally(ex->{
        return null;
    }).thenApply(res -> {
        List<String> result = new ArrayList<>();
        for (CompletableFuture<String> future : futures) {
            result.add(future.getNow(""));
        }
        return result;
    }).whenComplete((res,ex)->{
        System.out.println("fasdfdsf");
    });


}
```

如上代码中：当任务中有错误的时候，thenApply是无法执行的，除非进行异常捕获 ，如果任务中没有错误，thenApply可以继续执行，实际项目中不推荐使用，正常业务逻辑处理中，可以先对批量任务进行异常捕获，然后再对结果进行处理。



##### 2.2.7 超时时间控制

CompletableFuture 超时时间控制可以采用两种方式进行控制:

1. 对CompletableFuture设置超时时间
2. 业务方通过CompletableFuture获取结果时候设置超时时间

DEMO代码1 如下：

```
public void runAsync() throws ExecutionException, InterruptedException, TimeoutException {

    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        sleep(5, TimeUnit.SECONDS);
        return "我超时了";
    }).orTimeout(1,TimeUnit.SECONDS);
    sleep(2,TimeUnit.SECONDS);
    System.out.println(cf.isDone());
    sleep(6,TimeUnit.SECONDS);
    System.out.println(cf.get());
    
}
```

输出结果：

```
true
Exception in thread "main" java.util.concurrent.ExecutionException: java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.CompletableFuture.reportGet(CompletableFuture.java:395)
	at java.base/java.util.concurrent.CompletableFuture.get(CompletableFuture.java:1999)
	at com.idsidslilang.go.future.FutureService.runAsync(FutureService.java:25)
	at com.idsidslilang.go.future.FutureService.main(FutureService.java:33)
Caused by: java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.CompletableFuture$Timeout.run(CompletableFuture.java:2792)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)
```

可以看出，设置了超时时间以后，在运行超过了两秒时，future任务就已经终止。



第二种方式：

```
public void runAsync() {

    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        sleep(5, TimeUnit.SECONDS);
        return "我超时了";
    });

    try {
        sleep(2,TimeUnit.SECONDS);
        System.out.println(cf.isDone());
        System.out.println(cf.get(2,TimeUnit.SECONDS));
    }catch (TimeoutException ex){
        ex.printStackTrace();
    }


}
```

输出结果：

```
false
java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1886)
	at java.base/java.util.concurrent.CompletableFuture.get(CompletableFuture.java:2021)
	at com.idsidslilang.go.future.FutureService.runAsync(FutureService.java:23)
	at com.idsidslilang.go.future.FutureService.main(FutureService.java:37)
```





##### 2.2.8 超时时间生效判断：

CompletableFuture从设置超时时间处开始进行计时，定义future时就已经在进行异步计算。



代码DEMO：

```
public void runAsync() throws ExecutionException, InterruptedException, TimeoutException {

    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        sleep(6, TimeUnit.SECONDS);
        return "我超时了";
    }).orTimeout(1,TimeUnit.SECONDS);
    System.out.println(cf.isDone());
    System.out.println(cf.get());;

}
```

输出结果：

```
false
Exception in thread "main" java.util.concurrent.ExecutionException: java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.CompletableFuture.reportGet(CompletableFuture.java:395)
	at java.base/java.util.concurrent.CompletableFuture.get(CompletableFuture.java:1999)
	at com.idsidslilang.go.future.FutureService.runAsync(FutureService.java:20)
	at com.idsidslilang.go.future.FutureService.main(FutureService.java:27)
Caused by: java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.CompletableFuture$Timeout.run(CompletableFuture.java:2792)
	at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
	at java.base/java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:264)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)
```



代码DEMO2：

```
public void runAsync() throws ExecutionException, InterruptedException, TimeoutException {

    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        sleep(4, TimeUnit.SECONDS);
        return "我超时了";
    });
    System.out.println(cf.isDone());
    sleep(5,TimeUnit.SECONDS);
    cf.orTimeout(1,TimeUnit.SECONDS);
    System.out.println(cf.get());;

}
```

输出结果：

```
false
我超时了
```

代码DEMO3:

```
public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        sleep(4, TimeUnit.SECONDS);
        return "我超时了";
    });
    sleep(2,TimeUnit.SECONDS);
    System.out.println(cf.get(1,TimeUnit.SECONDS));
}
```

输出结果：

```
Exception in thread "main" java.util.concurrent.TimeoutException
	at java.base/java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1886)
	at java.base/java.util.concurrent.CompletableFuture.get(CompletableFuture.java:2021)
	at com.idsidslilang.go.future.FutureService.main(FutureService.java:33)
```

代码DEMO4：

```
public static void main(String[] args) throws ExecutionException, InterruptedException, TimeoutException {
    CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> {
        sleep(4, TimeUnit.SECONDS);
        return "我超时了";
    });
    sleep(4,TimeUnit.SECONDS);
    System.out.println(cf.get(1,TimeUnit.SECONDS));
}
```

输出结果：

```
我超时了
```