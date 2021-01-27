# 一文掌握Java本地缓存神器—Caffeine
## 前言
&ensp;&ensp;&ensp;&ensp;Caffeine是基于Java8的高性能缓存库，参考了Google guava的API，基于Guava Cache和ConcurrentLinkedHashMap的经验改进而来。
### 性能对比
&ensp;&ensp;&ensp;&ensp;以下是官方的性能测试对比，官方地址:https://github.com/ben-manes/caffeine/wiki/Benchmarks
1. 8个线程读，100%的读操作
![](https://mmbiz.qpic.cn/mmbiz_png/FcNb6Xk2BKCUOFt46Ydqx52K7vXugiaSx8nFhBBQDOjnBPiaoFnTD7WTI9UEibMwsL8DqVcDLw8P46ph2H8bNXfXg/0?wx_fmt=png)


2. 6个线程读，2个线程写，也就是75%的读操作，25%的写操作。
![](https://mmbiz.qpic.cn/mmbiz_png/FcNb6Xk2BKCUOFt46Ydqx52K7vXugiaSxiahzhJQCAbJTsTkcGiaDohMAEZtXkq02my7cTDGw4pSZP0plQh7jGAtw/0?wx_fmt=png)


3. 8个线程写，100%的写操作
![](https://mmbiz.qpic.cn/mmbiz_png/FcNb6Xk2BKCUOFt46Ydqx52K7vXugiaSxDPHnmRTxFCOrGCyWSdwqPVoichxqEhUNt0t6bNLd8c7IrMCiaTb2BXhQ/0?wx_fmt=png)


结论：从以上对比来看，其他缓存框架相比较Caffeine就是个渣渣。  
### Caffeine特性
1. 限制缓存大小  
2. 通过异步自动加载实体到缓存中  
3. 基于大小的回收策略  
4. 基于时间的回收策略  
5. 基于引用的回收策略  
6. 当向缓存中一个已经过时的元素进行访问的时候将会进行异步刷新  
7. key自动封装虚引用  
8. value自动封装弱引用或软引用  
9. 实体过期或被删除的通知  
10. 写入通知，可以将其传播到其他数据源中  
11. 统计累计访问缓存  
### 最佳实践
#### 添加依赖  
```
<dependency>
       <groupId>com.github.ben-manes.caffeine</groupId>
       <artifactId>caffeine</artifactId>
       <version>2.8.2</version>
  </dependency>
```
#### 1. 加载策略
&ensp;&ensp;&ensp;&ensp;Caffeine提供了四种缓存添加策略：手动加载，自动加载，手动异步加载和自动异步加载。
##### (1) 手动添加
```
public static void manualLoad() {
     Cache<String, String> cache = Caffeine.newBuilder()
             .expireAfterWrite(10, TimeUnit.MINUTES)
             .maximumSize(10_000)
             .build();
     //查找一个缓存元素，没有查找到的时候返回null
     String name = cache.getIfPresent("name");
     System.out.println("name:" + name);
     //查找缓存，如果缓存不存在则生成缓存元素,如果无法生成则返回null
     name = cache.get("name", k -> "小明");
     System.out.println("name:" + name);
     //添加或者更新一个缓存元素
     cache.put("address", "深圳");
     String address = cache.getIfPresent("address");
     System.out.println("address:" + address);
 }
```
##### (2) 自动加载
```
private static void autoLoad() throws InterruptedException {
     LoadingCache<String, String> cache = Caffeine.newBuilder()
             .maximumSize(10_000)
             .expireAfterWrite(10, TimeUnit.MINUTES)
             .build(new CacheLoader<String, String>() {
                 @Nullable
                 @Override
                 public String load(@NonNull String s) throws Exception {
                     System.out.println("load:" + s);
                     return "小明";
                 }

                 @Override
                 public @NonNull Map<String, String> loadAll(@NonNull Iterable<? extends String> keys) throws Exception {
                     System.out.println("loadAll:" + keys);
                     Map<String, String> map = new HashMap<>();
                     map.put("phone", "13866668888");
                     map.put("address", "深圳");
                     return map;
                 }
             });
     //查找缓存，如果缓存不存在则生成缓存元素,如果无法生成则返回null
     String name = cache.get("name");
     System.out.println("name:" + name);
     //批量查找缓存，如果缓存不存在则生成缓存元素
     Map<String, String> graphs = cache.getAll(Arrays.asList("phone", "address"));
     System.out.println(graphs);
  }
```
##### (3) 手动异步加载
```
private static void manualAsynLoad() throws ExecutionException, InterruptedException {
   AsyncCache<String, String> cache = Caffeine.newBuilder()
          .expireAfterWrite(10, TimeUnit.MINUTES)
          .maximumSize(10_000)
          //可以用指定的线程池
          .executor(Executors.newSingleThreadExecutor())
          .buildAsync();
   //查找缓存元素，如果不存在，则异步生成
  CompletableFuture<String> graph = cache.get("name", new Function<String, String>() {
      @SneakyThrows
      @Override
      public String apply(String key) {
          System.out.println("key:" + key+",当前线程:"+Thread.currentThread().getName());
          //模仿从数据库获取值
          Thread.sleep(1000);
          return "小明";
      }
    });
  System.out.println("获取name之前_time:"+System.currentTimeMillis()/1000);
  String name = graph.get();
  System.out.println("获取name:"+name+",time:"+System.currentTimeMillis()/1000);
 }
```
##### (4) 自动异步加载
```
private static void autoAsynLoad() throws ExecutionException, InterruptedException {
    AsyncLoadingCache<String, String> cache = Caffeine.newBuilder()
           .maximumSize(10_000)
           .expireAfterWrite(10, TimeUnit.MINUTES)
           //你可以选择:去异步的封装一段同步操作来生成缓存元素
           .buildAsync(new AsyncCacheLoader<String, String>() {
               @Override
               public @NonNull CompletableFuture<String> asyncLoad(@NonNull String key, @NonNull Executor executor) {
                     System.out.println("自动异步加载_key:" + key+",当前线程:"+Thread.currentThread().getName());
                     return CompletableFuture.completedFuture("小明");
                 }
              });
           //也可以选择:构建一个异步缓存元素操作并返回一个future
           //.buildAsync((key, executor) -> createExpensiveGraphAsync(key, executor));
           //查找缓存元素，如果其不存在，将会异步进行生成
           cache.get("name").thenAccept(name->{
           System.out.println("name:" + name);
       });
    }

 private static CompletableFuture<String> createExpensiveGraphAsync(String key, Executor executor) {
     return CompletableFuture.supplyAsync(new Supplier<String>() {
         @Override
         public String get() {
             System.out.println(executor);
             System.out.println("key:" + key+",当前线程:"+Thread.currentThread().getName());
             return "小明";
         }
      }, executor);
}
```
#### 2. 回收策略
&ensp;&ensp;&ensp;&ensp;Caffeine提供了三种回收策略：基于容量回收、基于时间回收、基于引用回收。
##### (1) 基于容量回收策略
&ensp;&ensp;&ensp;&ensp;基于大小回收策略有两种：一种是基于容量大小，一种是基于权重大小。两者只能取其一。
##### ①　基于容量--maximumSize
&ensp;&ensp;&ensp;&ensp;为缓存容量指定特定的大小，Caffeine.maximumSize(long)。当缓存容量超过指定的大小，缓存将尝试逐出最近或经常未使用的条目。
```
public static void main(String[] args) throws InterruptedException {
   Cache<String, String> cache = Caffeine.newBuilder()
           .maximumSize(1)
           .removalListener((String key, Object value, RemovalCause cause) ->
                   System.out.printf("Key %s was removed (%s)%n", key, cause))
           .build();
   cache.put("name", "小明");
   System.out.println("name:" + cache.getIfPresent("name") + ",缓存容量:" + cache.estimatedSize());
   cache.put("address", "中国");
   Thread.sleep(2000);
   System.out.println("name:" + cache.getIfPresent("name") + ",缓存容量:" + cache.estimatedSize());
}
```
##### ②　基于权重--maximumWeight
&ensp;&ensp;&ensp;&ensp;用Caffeine.maximumWeight(long)指定权重大小，通过Caffeine.weigher(Weigher)方法自定义计算权重方式。
```
public static void main(String[] args) throws InterruptedException {
   //初始化缓存，设置最大权重为20
   Cache<Integer, Integer> cache = Caffeine.newBuilder()
           .maximumWeight(20)
           .weigher(new Weigher<Integer, Integer>() {
               @Override
               public @NonNegative int weigh(@NonNull Integer key, @NonNull Integer value) {
                   System.out.println("weigh:"+value);
                   return value;
               }
            })
            .removalListener((Integer key, Object value, RemovalCause cause) ->
                    System.out.printf("Key %s was removed (%s)%n", key, cause))
            .build();

   cache.put(100, 10);
   //打印缓存个数，结果为1
   System.out.println(cache.estimatedSize());
   cache.put(200, 20);
   //稍微休眠一秒
   Thread.sleep(1000);
   //打印缓存个数，结果为1
   System.out.println(cache.estimatedSize());
}
```
##### (2) 基于时间策略
##### ①　写入时间--expireAfterWrite
&ensp;&ensp;&ensp;&ensp;在最后一次写入开始计时，到达指定的时间后过期清除。如果一直写入，那么一直不会过期。
```
private static void writeFixedTime() throws InterruptedException {
   //在最后一次访问或者写入后开始计时，在指定的时间后过期。
   LoadingCache<String, String> graphs = Caffeine.newBuilder()
           .expireAfterWrite(1, TimeUnit.SECONDS)
           .removalListener((String key, Object value, RemovalCause cause) ->
                   System.out.printf("Key %s was removed (%s)%n", key, cause))
           .build(key -> createExpensiveGraph(key));
   String name = graphs.get("name");
   System.out.println("第一次获取name:" + name);
   name = graphs.get("name");
   System.out.println("第二次获取name:" + name);
   Thread.sleep(2000);
   name = graphs.get("name");
   System.out.println("第三次延迟2秒后获取name:" + name);
}
private static String createExpensiveGraph(String key) {
     System.out.println("重新自动加载数据");
     return "小明";
}
```
##### ②　写入和访问时间--expireAfterAccess
&ensp;&ensp;&ensp;&ensp;在最后一次写入或访问开始计时，在指定时间后过期清除。如果一直访问或写入，那么一直不会过期。
```
private static void accessFixedTime() throws InterruptedException {
   //在最后一次访问或者写入后开始计时，在指定的时间后过期。
  LoadingCache<String, String> graphs = Caffeine.newBuilder()
          .expireAfterAccess(3, TimeUnit.SECONDS)
          .removalListener((String key, Object value, RemovalCause cause) ->
                   System.out.printf("Key %s was removed (%s)%n", key, cause))
          .build(key -> createExpensiveGraph(key));
  String name = graphs.get("name");
  System.out.println("第一次获取name:" + name);
  name = graphs.get("name");
  System.out.println("第二次获取name:" + name);
  Thread.sleep(2000);
  name = graphs.get("name");
  System.out.println("第三次延迟2秒后获取name:" + name);
}

private static String createExpensiveGraph(String key) {
    System.out.println("重新自动加载数据");
    return "小明";
}
```
##### ③　自定义时间--expireAfter
&ensp;&ensp;&ensp;&ensp;自定义策略，由Expire实现独自计算时间。分别计算新增、更新、读取时间。
```
private static void customTime() throws InterruptedException {
    LoadingCache<String, String> graphs = Caffeine.newBuilder()
          .removalListener((String key, Object value, RemovalCause cause) ->
                  System.out.printf("Key %s was removed (%s)%n", key, cause))
          .expireAfter(new Expiry<String, String>() {
              @Override
              public long expireAfterCreate(@NonNull String key, @NonNull String value, long currentTime) {
                  //这里的currentTime由Ticker提供，默认情况下与系统时间无关，单位为纳秒
                  System.out.println(String.format("expireAfterCreate----key:%s,value:%s,currentTime:%d", key, value, currentTime));
                  return TimeUnit.SECONDS.toNanos(10);
              }

              @Override
              public long expireAfterUpdate(@NonNull String key, @NonNull String value, long currentTime, @NonNegative long currentDuration) {
                  //这里的currentTime由Ticker提供，默认情况下与系统时间无关，单位为纳秒
                  System.out.println(String.format("expireAfterUpdate----key:%s,value:%s,currentTime:%d,currentDuration:%d", key, value, currentTime,currentDuration));
                  return TimeUnit.SECONDS.toNanos(3);
              }

              @Override
              public long expireAfterRead(@NonNull String key, @NonNull String value, long currentTime, @NonNegative long currentDuration) {
                  //这里的currentTime由Ticker提供，默认情况下与系统时间无关，单位为纳秒
                  System.out.println(String.format("expireAfterRead----key:%s,value:%s,currentTime:%d,currentDuration:%d", key, value, currentTime,currentDuration));
                  return currentDuration;
               }
          })

          .build(key -> createExpensiveGraph(key));
    String name = graphs.get("name");
    System.out.println("第一次获取name:" + name);
    name = graphs.get("name");
    System.out.println("第二次获取name:" + name);
    Thread.sleep(5000);
    name = graphs.get("name");
    System.out.println("第三次延迟5秒后获取name:" + name);
    Thread.sleep(5000);
    name = graphs.get("name");
    System.out.println("第五次延迟5秒后获取name:" + name);
}
    
private static String createExpensiveGraph(String key) {
    System.out.println("重新自动加载数据");
    return "小明";
}
```
##### (3) 基于引用策略
&ensp;&ensp;&ensp;&ensp;异步加载的方式不支持引用回收策略
##### ①　软引用
&ensp;&ensp;&ensp;&ensp;当GC并且内存不足时，会触发软引用回收策略。  
&ensp;&ensp;&ensp;&ensp;设置jvm启动时-XX:+PrintGCDetails -Xmx100m 参数，可以看GC日志打印会触发软引用的回收策略。
```
private static void softValues() throws InterruptedException {
    //当进行GC的时候进行驱逐
    LoadingCache<String, byte[]> cache = Caffeine.newBuilder()
           .softValues()
           .removalListener((String key, Object value, RemovalCause cause) ->
                   System.out.printf("Key %s was removed (%s)%n", key, cause))
           .build(key -> loadDB(key));
    System.out.println("1");
    cache.put("name1", new byte[1024 * 1024*50]);
    System.gc();
    System.out.println("2");
    Thread.sleep(5000);
    cache.put("name2", new byte[1024 * 1024*50]);
    System.gc();
    System.out.println("3");
    Thread.sleep(5000);
    cache.put("name3", new byte[1024 * 1024*50]);
    System.gc();
    System.out.println("4");
    Thread.sleep(5000);
    cache.put("name4", new byte[1024 * 1024*50]);
    System.gc();
    Thread.sleep(5000);
}

private static byte[] loadDB(String key) {
    System.out.println("重新自动加载数据");
    return new byte[1024*1024];
}
```
##### ②　弱引用
&ensp;&ensp;&ensp;&ensp;当GC时，会触发弱引用回收策略。 　　
&ensp;&ensp;&ensp;&ensp;设置jvm启动时-XX:+PrintGCDetails -Xmx100m 参数，可以看GC日志打印会触发弱引用的回收策略。
```
private static void weakKeys() throws InterruptedException {
    LoadingCache<String, byte[]> cache = Caffeine.newBuilder()
           .weakKeys()
           .weakValues()
           .removalListener((String key, Object value, RemovalCause cause) ->
                   System.out.printf("Key %s was removed (%s)%n", key, cause))
           .build(key -> loadDB(key));
    System.out.println("添加name1");
    cache.put("name1", new byte[1024 * 1024]);
    System.gc();
    System.out.println("添加name2");
    Thread.sleep(5000);
    cache.put("name2", new byte[1024 * 1024]);
    System.gc();
    System.out.println("添加name3");
    Thread.sleep(5000);
    cache.put("name3", new byte[1024 * 1024]);
    System.gc();
    Thread.sleep(5000);
}

private static byte[] loadDB(String key) {
    System.out.println("重新自动加载数据");
    return new byte[1024*1024];
}
```
#### 3. 刷新策略
&ensp;&ensp;&ensp;&ensp;刷新策略可以通过LoadingCache.refresh(K)方法，异步为key对应的缓存元素刷新一个新的值。与回收策略不同的是，在刷新的时候如果查询缓存元素，其旧值将仍被返回，直到该元素的刷新完毕后结束后才会返回刷新后的新值。
```
public static void refreshLoad() throws InterruptedException {
    LoadingCache<Integer, String> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        //设置在写入或者更新之后1分钟后，调用 CacheLoader 重新加载
        .refreshAfterWrite(1, TimeUnit.SECONDS)
        .build(new CacheLoader<Integer, String>() {
        @Override
        public String load(Integer key) throws Exception {
            String values = queryData(key);
            log("load刷新,key:" + key + ",查询的数据库值:" + values);
            return values;
        }

        @Nullable
        @Override
        public String reload(@NonNull Integer key, @NonNull String oldValue) throws Exception {
            String values = queryData(key);
            log("reload刷新,key:" + key + ",旧值:" + oldValue + ",查询的数据库值:" + values);
            return values;
        }
    });
    Thread thread1 = startLoadingCacheQuery("client1", cache);
    Thread thread2 = startLoadingCacheQuery("client2", cache);
    Thread.sleep(2000);
    Thread thread3 = startLoadingCacheQuery("client3", cache);
    Thread.sleep(1000);
    Thread thread4 = startLoadingCacheQuery("client4", cache);
}

private static Thread startLoadingCacheQuery(String clientName, LoadingCache<Integer, String> cache) {
     Thread thread = new Thread(() -> {
          log("异步从缓存中查询数据开始");
          String values = cache.get(1);
          log("查询的key为:" + 1 + ",值为:" + values);
          log("异步从缓存中查询数据结束");
     });
     thread.setName(clientName);
     thread.start();
     return thread;
}

private static String queryData(Integer key) throws InterruptedException {
     String value = System.currentTimeMillis() + "";
     return value;
}

private static void log(String msg) {
     System.out.println(String.format("当前时间:%d,线程名称:%s,msg:%s", System.currentTimeMillis(), Thread.currentThread().getName(), msg));
}
```
#### 4. 缓存写入传播
&ensp;&ensp;&ensp;&ensp;CacheWriter给缓存提供了充当底层资源的门面的能力，当其与CacheLoader一起使用的时候，所有的读和写操作都可以通过缓存向下传播。Writers提供了原子性的操作，包括从外部资源同步的场景。这意味着在缓存中，当一个key的写入操作在完成之前，后续其他写操作都是阻塞的，同时在这段时间内，尝试获取这个key对应的缓存元素的时候获取到的也将都是旧值。如果写入失败那么之前的旧值将会被保留同时异常将会被传播给调用者。
```
public static void write() throws InterruptedException {
    Cache<String, String> cache = Caffeine.newBuilder()
    //.expireAfterWrite(1, TimeUnit.SECONDS)
    .writer(new CacheWriter<String, String>() {
        @SneakyThrows
        @Override
        public void write(String key, String graph) {
            // 持久化或者次级缓存
            hread.sleep(2000);
            System.out.println(String.format("写入时间:%d,key:%s,value:%s", System.currentTimeMillis(), key, graph));

        @Override
        public void delete(String key, String value, RemovalCause cause) {
            // 从持久化或者次级缓存中删除                                                 
            System.out.println(String.format("delete事件,key:%s,value:%s,原因:%s:",                                           key,value,cause.toString()));
        }
    })
    .removalListener(new RemovalListener<String, String>() {
        @Override
        public void onRemoval(@Nullable String key, @Nullable String value,                                        @NonNull RemovalCause removalCause) {            
            System.out.println(String.format("remove事件,key:%s,value:%s,原因:%s:", 
                      key,value,removalCause.toString()));
        }
    })
    .build();
    cache.put("name", "小明");
    System.out.println(String.format("时间:%d,name:%s",System.currentTimeMillis(),cache.getIfPresent("name")));
    //在这里获取name会同步等待"小明"写入完成
    cache.put("name", "小强");
    System.out.println(String.format("时间:%d,name:%s",System.currentTimeMillis(),cache.getIfPresent("name")));
    Thread.sleep(2000);
    System.out.println(String.format("时间:%d,name:%s",System.currentTimeMillis(),cache.getIfPresent("name")));
}
```   
#### 5. 统计
&ensp;&ensp;&ensp;&ensp;通过使用Caffeine.recordStats()方法可以打开数据收集功能。Cache.stats()方法将会返回一个CacheStats对象，其将会含有一些统计指标，比如：
* hitRate(): 查询缓存的命中率  
* evictionCount(): 被驱逐的缓存数量  
* averageLoadPenalty(): 新值被载入的平均耗时  
```
public static void statistics() throws InterruptedException {
    LoadingCache<String, String> cache = Caffeine.newBuilder()
            .maximumSize(10)
            .recordStats()
            .build(key->"小明");
    int i=0;
    while(i<1000){
        i++;
        cache.put("name"+i,"小明");
    }
    Thread.sleep(10000);
    //缓存命中率
    System.out.println("缓存命中率:"+cache.stats().hitRate());
    //回收的缓存数量
    System.out.println("回收的缓存数量:"+cache.stats().evictionCount());
    //新值被载入的平均耗时
    System.out.println("新值被载入的平均耗时:"+cache.stats().averageLoadPenalty());
}
```
欢迎大家关注微信公众号：CodingTao
![](https://mmbiz.qpic.cn/mmbiz_jpg/FcNb6Xk2BKCUOFt46Ydqx52K7vXugiaSxB1a72ic1Bq05Xc32jBREn1LiaicAAtZyLRUbU35bl8rJWSz35IYIWGypw/0?wx_fmt=jpeg)

