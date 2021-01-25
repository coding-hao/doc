# 前言
&ensp;&ensp;&ensp;&ensp;绝大多数系统都是读多写少的，总所周知，内存的访问速度很快，是磁盘访问速度的数十倍，如果不使用缓存，都通过数据库访问硬盘，对于双十一这样大的交易量是不可想象的。有人专门写了一篇《让 CPU 告诉你硬盘和网络到底有多慢》，将磁盘、内存、网络对数据的处理速度站在人类的角度来感知表述。
* 1.	从内存中读取 1MB 的连续数据，耗时大约为 250us，换算成人类时间是 7.5天。
* 2.	从 SSD 读取 1MB 的顺序数据，大约需要 1ms，换算成人类时间是 1个月。
由此可见，内存和硬盘的差距，相当于拖拉机和法拉利的差距。
# 为什么要用Caffeine
&ensp;&ensp;&ensp;&ensp;在[Java本地缓存神器---Caffeine（一）](https://mp.weixin.qq.com/s/8OWS1asuZ7dWxMimHOQ-TQ)里面已经介绍了caffeine和其他缓存框架的性能对比，结果显示，caffeine的性能远远强于其他缓存框架。如果想要了解关于caffeine更多的知识，可以看[Java本地缓存神器---Caffeine（一）](https://juejin.cn/post/6919817359343484935)，
[Java本地缓存神器---Caffeine（二）](https://mp.weixin.qq.com/s/DiT6cWoH6Gl4pEVvHRbfMg)这两篇文章。  

# SpringBoot整合caffeine
&ensp;&ensp;&ensp;&ensp;SpringBoot默认使用的是SimpleCacheConfiguration，即使用ConcurrentMapCacheManager来实现缓存，ConcurrentMapCache实质是一个ConcurrentHashMap集合对象java内置，如果需要使用caffeine，那么需要引用caffeine的依赖。      
## 1. 配置caffeine的maven依赖
  ```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```
## 2.在SpringBoot中开启缓存
```
@SpringBootApplication
//开启缓存
@EnableCaching
public class DemoApplication {
	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```
## 3. 通过bean装配caffeine

```
@Configuration
public class CaffeineConfig {

    @Autowired
    BookDao bookDao;

    @Autowired
    CacheLoader cacheLoader;

    @Bean
    @Primary
    public CacheManager caffeineCache() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        Caffeine caffeine = Caffeine.newBuilder()
                //cache的初始容量值
                .initialCapacity(10)
                //maximumSize用来控制cache的最大缓存数量，maximumSize和maximumWeight不可以同时使用，
                .maximumSize(100)
                .expireAfterAccess(5, TimeUnit.SECONDS)
                .removalListener((Integer key, Object value, RemovalCause cause) ->
                        System.out.printf("时间:%d,Key:%d,value:%s,移除原因(%s)%n",System.currentTimeMillis()/1000, key,value.toString(),cause)
                )
                //使用refreshAfterWrite必须要设置cacheLoader
                .refreshAfterWrite(5, TimeUnit.SECONDS);
        cacheManager.setCaffeine(caffeine);
        //缓存加载策略，当key不存在或者key过期之类的都可以通过CacheLoader来重新获得数据
        cacheManager.setCacheLoader(cacheLoader);
        return cacheManager;
    }

    @Bean
    public CacheLoader<Object, Object> cacheLoader() {
        CacheLoader<Object, Object> cacheLoader = new CacheLoader<Object, Object>() {
            @Override
            public Object load(Object key) throws Exception {
                System.out.println("重新从数据库加载数据:" + key);
                return bookDao.getBookById((int)key);
            }

            // 达refreshAfterWrite所指定的时候会触发这个事件方法
            @Override
            public Object reload(Object key, Object oldValue) throws Exception {
                //可以在这里处理重新加载策略，本例子，没有处理重新加载，只是返回旧值。
                return oldValue;
            }
        };
        return cacheLoader;
    }

}

```
## 4. 定义实体对象
```
@Getter
@Setter
@ToString
public class BookBean implements Serializable {
    private static final long serialVersionUID = -6585766340444705937L;
    private int bookId;
    private String bookName;
    private String author;
    private float price;
}
```
## 5. 定义访问数据库类
```
@Component
public class BookDao {
    /**
     * 模拟数据库
     */
    private static Map<Integer, BookBean> bookMap = new HashMap<>();
    @PostConstruct
    private void initDB() {
        BookBean xiyou = new BookBean();
        xiyou.setBookId(1);
        xiyou.setBookName("西游记");
        xiyou.setAuthor("吴承恩");
        xiyou.setPrice(55.5f);
        bookMap.put(1, xiyou);
        BookBean honglou = new BookBean();
        honglou.setBookId(2);
        honglou.setBookName("红楼梦");
        honglou.setAuthor("曹雪芹");
        honglou.setPrice(66.6f);
        bookMap.put(2, honglou);
    }

    public BookBean getBookById(int bookId) {
        return bookMap.get(bookId);
    }

    public BookBean update(BookBean bookBean) {
        bookMap.put(bookBean.getBookId(), bookBean);
        return bookMap.get(bookBean.getBookId());
    }

    public BookBean save(BookBean bookBean) {
        bookMap.put(bookBean.getBookId(), bookBean);
        return bookMap.get(bookBean.getBookId());
    }

    public void delete(int bookId) {
        bookMap.remove(bookId);
    }
}
```

## 6. 定义服务接口
```
public interface BookService {

    public BookBean getBookById(int bookId);

    public BookBean updata(BookBean bookBean);

    public BookBean save(BookBean bookBean);

    public String delete(int bookId);

}
```

## 7. 定义服务实现类
```
@Service
//可以在此处统一指定cacheNames的名称
@CacheConfig(cacheNames = {"book"})
public class BookServiceImpl implements BookService {

    @Autowired
    BookDao bookDao;

    /**
     * 如果缓存存在，直接读取缓存值；如果缓存不存在，则调用目标方法，并将结果放入缓存
     * value、cacheNames：两个等同的参数（cacheNames为Spring 4新增，作为value的别名），用于指定缓存存储的集合名
     * key：缓存对象存储在Map集合中的key值，非必需，默认按照函数的所有参数组合作为key值，若自己配置需使用SpEL表达式，比如：@Cacheable(key = “#p0”)：使用函数第一个参数作为缓存的key值
     *
     * @param bookId
     * @return
     */
    @Cacheable(cacheNames = {"book"}, key = "#bookId")
//    @Cacheable(value = "book" ,key = "targetClass + methodName +#p0")
    @Override
    public BookBean getBookById(int bookId) {
        System.out.println("查询数据库");
        return bookDao.getBookById(bookId);
    }

    @CachePut(cacheNames = {"book"}, key = "#bookBean.bookId")
    @Override
    public BookBean updata(BookBean bookBean) {
        System.out.println("更新数据库:" + bookBean.toString());
        return bookDao.update(bookBean);
    }

    @CachePut(cacheNames = {"book"}, key = "#bookBean.bookId")//写入缓存，key为user.id,一般该注解标注在新增方法上
    @Override
    public BookBean save(BookBean bookBean) {
        System.out.println("保存至数据库:" + bookBean.toString());
        return bookDao.save(bookBean);
    }

    /**
     * CacheEvict 用来从缓存中移除相应数据
     *  allEntries=true:方法调用后清空所有cacheName为book的缓存
     *  beforeInvocation=true:方法调用前清空所有缓存
     *
     * @param bookId
     * @return
     */

    @CacheEvict(cacheNames = {"book"})
    @Override
    public String delete(int bookId) {
        bookDao.delete(bookId);
        return "成功";
    }
}
```

## 8. 定义controller
```
@RestController
@RequestMapping("book")
public class BookController {

    @Autowired
    BookService bookService;

    @GetMapping("get/{bookId}")
    @ResponseBody
    public BookBean getBook(@PathVariable("bookId") int bookId) {
        System.out.println("时间:"+System.currentTimeMillis()/1000+",查询book接口被调用");
        return bookService.getBookById(bookId);
    }

    @PostMapping("updata")
    @ResponseBody
    public BookBean updata(@RequestBody BookBean bookBean) {
        System.out.println("时间:"+System.currentTimeMillis()/1000+",更新book接口被调用");
        return bookService.updata(bookBean);
    }
    @PostMapping("save")
    @ResponseBody
    public BookBean save(@RequestBody BookBean bookBean) {
        System.out.println("时间:"+System.currentTimeMillis()/1000+",保存book接口被调用");
        return bookService.save(bookBean);
    }

    @PostMapping("delete/{bookId}")
    @ResponseBody
    public String delete(@PathVariable("bookId") int bookId) {
        System.out.println("时间:"+System.currentTimeMillis()/1000+",删除book接口被调用");
        return bookService.delete(bookId);
    }
}
```
## 9. 测试
### 1.查询测试
#### (1) 连续两次访问 http://localhost:8080/book/get/2
![](https://mmbiz.qpic.cn/mmbiz_png/FcNb6Xk2BKCj9Gg6ApicOVuPUOWoaMenDRz1APd4gN2Vc8icvDeicuFCV1HMyDwp9vAibKtGwFBcSEXDLicAqEBA4ww/0?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/FcNb6Xk2BKCj9Gg6ApicOVuPUOWoaMenDDnkVLoCL6tadcE8exPdVZdGpODDyn8cHWMfB8Zr3AeYseiaoneHIvMw/0?wx_fmt=png)  
&ensp;&ensp;&ensp;&ensp;可以看到第二次访问的时候，是直接从缓存中查询的数据，并没有从数据库重新加载
#### (2) 超过五秒之后再访问 http://localhost:8080/book/get/2
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b735fbf944e4b18aceffaa2a53ea070~tplv-k3u1fbpfcp-watermark.image)  
我们能看到会重新从数据库加载数据，并且会caffeine访问时间的回收策略
#### (3) 测试修改对缓存的影响
&ensp;&ensp;&ensp;&ensp; 先连续两次访问 http://localhost:8080/book/get/2 ，五秒之内再
调用http post方法访问 http://localhost:8080/book/updata，修改《红楼梦》价格为99.9
实体数据为：
{
    "bookId": 2,
    "bookName": "红楼梦",
    "author": "曹雪芹",
    "price":99.9
}，再访问http://localhost:8080/book/get/2
![](https://mmbiz.qpic.cn/mmbiz_png/FcNb6Xk2BKCj9Gg6ApicOVuPUOWoaMenD10icjynS47SWUegMibibZpWNywmKEuFjIrnmI9QU1mxNW0auZnPayAo7A/0?wx_fmt=png)
![](https://mmbiz.qpic.cn/mmbiz_png/FcNb6Xk2BKCj9Gg6ApicOVuPUOWoaMenDpaBt1wcWmCYPm3GlULricibpefGkerpHrMjSmicvsnwOdSKdQ4rqwvkFA/0?wx_fmt=png)  
我们可以看到，第一次调接口查询，因为触发了回收策略，会从数据库重新加载数据；第二次调接口查询时，会从缓存中获取数据；第三次调接口修改，会更新数据库，并且更新缓存；第四次调接口查询，会直接从缓存中获取数据。

# 源码
[https://github.com/coding-hao/spring-learn/tree/main/spring_cache](url)

#公众号
更多相关文章，请关注公众号：CodingTao
![](http://mmbiz.qpic.cn/mmbiz_jpg/FcNb6Xk2BKAoGc3st39zWJAGGFwKp34b7zBIkN8QUmKDOFRuMMkl7Vt9stMFM5S8m8kCUyvNOpGTKFFKv6s6Tg/640?wx_fmt=jpeg)


