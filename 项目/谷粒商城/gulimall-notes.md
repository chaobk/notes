## 1.整合SpringCache简化缓存开发

### 1.1 引入依赖

```java
spring-boot-starter-cache
spring-boot-starter-data-redis(使用redis作为缓存实现)
```



### 1.2 写配置

1）自动配置了哪些

CacheAutoConfiguration会导入RedisCacheConfiguration；自动配好了缓存管理器RedisCacheManager

2） 配置使用redis作为缓存

```yaml
spring:
	cache:
		type: redis
```



### 1.3 缓存注解

>@Cacheable：触发将数据保存到缓存的操作
>
>@CacheEvict：触发将数据从缓存删除的操作
>
>@CachePut：不影响方法执行更新缓存
>
>@Caching：组合以上多个操作
>
>@CacheConfig：在类级别共享缓存的相同配置

 ### 1.4 使用缓存

1）开启缓存

启动配上加@EnableCaching

2）只需使用注解就能完成缓存操作

```java
    @Cacheable("category")
    @Override
    public List<CategoryEntity> getLevel1Categorys() {
        System.out.println("getLevel1Categorys。。。。。。被调用");
        List<CategoryEntity> entities = baseMapper.selectList(new QueryWrapper<CategoryEntity>().eq("parent_cid", 0));
        return entities;
    }
```

3）默认行为

* 如果缓存命中，方法不会被调用
* key默认生成：缓存的名字::SimpleKey [] （自主生成的key之）
* 缓存的value值，默认使用jdk序列化机制，将序列化后的数据存到redis
* 默认ttl时间 -1

4）自定义

* 指定生成的缓存使用的key

  * ```java
     @Cacheable(value = "category", key = "'level1Categorys'")
    ```

* 指定缓存的数据的存活时间

  * ```yaml
    spring:
      cache:
        type: redis
        redis:
          time-to-live: 3600000
    ```

* 将数据保存为json格式 

  * 默认情况下是Hex格式
  * ![image-20230917111429465](https://chaobk-img-repo.oss-cn-shanghai.aliyuncs.com/image-20230917111429465.png)

  * 原理

    * CacheAutoConfiguration -> RedisCacheConfiguration -> 自动配置了RedisCacheManager -> 初始化所有的缓存 -> 每个缓存决定使用什么配置 -> 如果redisCacheConfiguration有就用已有的，没有就用默认配置 -> 想改缓存的配置，只需要给容器中放一个RedisCacheConfiguration即可 -> 就会应用到当前RedisCacheManager管理的所有缓存分区中

    * 在代码中进行配置

    * ```java
      
      @Configuration
      @EnableCaching
      @EnableConfigurationProperties(CacheProperties.class)
      public class MyCacheConfig {
      
              /**
               * 配置文件中的ttl失效了
               *
               * 是因为配置文件的东西没有用上
               * 1、原来和配置文件绑定的配置类是这样子的
               *      @ConfigurationProperties(prefix = "spring.cache")
               *      public class CacheProperties
               * 2、要让配置生效
               *      @EnableConfigurationProperties(CacheProperties.class)
               *
               *
               * @return
               */
              @Bean
              RedisCacheConfiguration redisCacheConfiguration(CacheProperties cacheProperties) {
                      RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig();
      
                      config = config.serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()));
                      config = config.serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericFastJsonRedisSerializer()));
      
      
                      // 将配置文件的配置生效，否则配置文件中的ttl就没有用
                      CacheProperties.Redis redisProperties = cacheProperties.getRedis();
                      if (redisProperties.getTimeToLive() != null) {
                              config = config.entryTtl(redisProperties.getTimeToLive());
                      }
      
                      if (redisProperties.getKeyPrefix() != null) {
                              config = config.prefixKeysWith(redisProperties.getKeyPrefix());
                      }
      
                      if (!redisProperties.isCacheNullValues()) {
                              config = config.disableCachingNullValues();
                      }
      
                      if (!redisProperties.isUseKeyPrefix()) {
                              config = config.disableKeyPrefix();
                      }
      
                      return config;
              }
      
      } 
      ```

### 1.5 同时失效多个缓存

1）方法一

```java
    @Caching(evict = {
            @CacheEvict(value = "category", key = "'getLevel1Categorys'"),
            @CacheEvict(value = "category", key = "'getCatalogJson'")
    })
```

2）方法二

```java
@CacheEvict(value = "category", allEntries = true)
```

```java
public class Main {
    
}
```



### 1.6 SpringCache不足

1）读模式

* 缓存穿透
  * 查询一个null数据
  * 解决：缓存空数据： cache-null-values=true
* 缓存击穿
  * 大量并发进来同时查询一个正好过期的数据
  * 解决：默认无锁，可以使用注解的sync属性，本地锁
* 缓存雪崩
  * 大量的key同时过期
  * 解决：加随机时间。加上过期时间 spring.cache.time-to-live

2）写模式（缓存与数据库一致）

* 读写加锁
* 引入Canal，该知道MySQL的更新去更新数据库
* 读多写多，直接去数据库查询，不经缓存



## 2.使用多线程方式

* 继承Thread类

* 实现Runable接口

* 实现Callable接口
* 通过线程池，提交任务过去



## 3.构建线程池

### 3.1 七大参数 

> corePoolSize – the number of threads to keep in the pool, even if they are idle, unless allowCoreThreadTimeOut is set 
>
> maximumPoolSize – the maximum number of threads to allow in the pool 
>
> keepAliveTime – when the number of threads is greater than the core, this is the maximum time that excess idle threads will wait for new tasks before terminating. 
>
> unit – the time unit for the keepAliveTime argument 
>
> workQueue – the queue to use for holding tasks before they are executed. This queue will hold only the Runnable tasks submitted by the execute method. 
>
> threadFactory – the factory to use when the executor creates a new thread
>
> handler – the handler to use when execution is blocked because the thread bounds and queue capacities are reached



### 3.2 工作顺序

1）线程池创建，准备好core数量的核心线程，准备接受任务

2） 新的任务过来，首先使用空的core进行任务接收

3）core满了，就将再进来的任务放入阻塞队列中。空闲的core就会自己去阻塞队列获取任务执行

4）阻塞队列满了，就直接开新线程执行，最大只能开到max指定的数量

5）max都执行好了，max-core数量空闲的线程会在keepAliveTime指定的事件后自动销毁。最终保持到core大小

6）如果线程数开到了max的数量，还有新任务进来，就会使用reject指定的拒绝策略进行处理

 

### 3.3 使用线程池的优点

* 降低资源的消耗
* 提高响应速度
* 提高线程的可管理性

### 3.4 CompletableFuture

#### 3.4.1 一般使用

```java
//        CompletableFuture.runAsync(() -> {
//            System.out.println("Runable start:" + Thread.currentThread().getId());
//            int i = 10 / 2;
//            System.out.println("Runable end: " + i);
//        }, executor);

//        CompletableFuture<Integer> res = CompletableFuture.supplyAsync(() -> {
//            System.out.println("Runable start:" + Thread.currentThread().getId());
//            int i = 10 / 2;
//            System.out.println("Runable end: " + i);
//            return i;
//        }, executor);
//        System.out.println(res.get());


//        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
//            System.out.println("Runable start:" + Thread.currentThread().getId());
//            int i = 10 / 0;
//            System.out.println("Runable end: " + i);
//            return i;
//        }, executor).whenComplete((res, exception) -> {
//            // 虽然能得到异常信息，但是没有办法返回数据
//            System.out.println("异步任务完成了,结果是:" + res + " ;异常是：" + exception);
//        }).exceptionally(throwable -> {
//            // 可以感知到异常，同时返回默认值
//            return 10;
//        });
//
//        System.out.println(future.get());

        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("Runnable start: " + Thread.currentThread().getId());
            int i = 10 / 0;
            System.out.println("Runanble end: " + i);
            return i;
        }, executor).handle((res, thr) -> {
            if (res != null) {
                return res * 2;
            }
            if (thr != null) {
                return 0;
            }
            return 0;
        });
        System.out.println(future.get());
```



#### 3.4.2 线程串行化

```java
public class FutureTest {

    static ThreadPoolExecutor executor = new ThreadPoolExecutor(5,
            100,
            10,
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(10000),
            Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());

    public static void main(String[] args) throws ExecutionException, InterruptedException {

        // 1.thenRun获取不到上个线程的执行结果，无返回值
//        CompletableFuture<Void> future2 = CompletableFuture.supplyAsync(() -> {
//            System.out.println("Runnable start: " + Thread.currentThread().getId());
//            int i = 10 / 3;
//            System.out.println("Runanble end: " + i);
//            return i;
//        }, executor).thenRunAsync(() -> {
//            System.out.println("任务2启动了");
//        }, executor);

        // 2.能接收上一步的执行结果，无返回值
//        CompletableFuture.supplyAsync(() -> {
//            System.out.println("Runnable start: " + Thread.currentThread().getId());
//            int i = 10 / 3;
//            System.out.println("Runanble end: " + i);
//            return i;
//        }, executor).thenAcceptAsync(res -> {
//            System.out.println("测试的线程id: " + Thread.currentThread().getId());
//            System.out.println("上一步的的执行结果: " + res);
//        }, executor);
//
//        System.out.println("main...end...");


        // 3.thenApplyAsync 能接收上一步结果，有返回值
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            System.out.println("Runnable start: " + Thread.currentThread().getId());
            int i = 10 / 3;
            System.out.println("Runanble end: " + i);
            return i;
        }).thenApplyAsync((res) -> {
            System.out.println("任务2启动了: " + res);
            return "hello";
        }, executor);
        System.out.println(future.get());
        System.out.println("main...end...");
    }
}
```

#### 3.4.3 两任务组合，都要完成

```java
         /**
         * 两个都完成
         */
        CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            System.out.println("任务1Runnable start: " + Thread.currentThread().getId());
            int i = 10 / 3;
            System.out.println("任务1Runanble end: " + i);
            return i;
        }, executor);

        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("任务2Runnable start: " + Thread.currentThread().getId());
            int i = 10 / 3;
            System.out.println("任务2Runanble end: " + i);
            return "hello";
        }, executor);

        //         runAfterBoth 获取不到前面流程的执行结果
//        future1.runAfterBothAsync(future2, () -> {
//            System.out.println("任务3 start");
//        }, executor);

        // 可以接收到前面的结果
        //    /**
        //     * Performs this operation on the given arguments.
        //     * @param t the first input argument
        //     * @param u the second input argument
        //     */
        //    void accept(T t, U u);
//        future1.thenAcceptBothAsync(future2, (f1, f2) -> {
//            System.out.println("任务3 Runnable start: " + Thread.currentThread().getId());
//            System.out.printf("%s %s", f1, f2);
//        }, executor);

        // 可以有返回值
//        System.out.println(future1.thenCombine(future2, (f1, f2) -> {
//            return f1 + ": " + f2 + " -> hello";
//        }).get());
```



#### 3.4.4 两任务组合，有一个完成

```java
        /**
         * runAfterEitherAsync 不感知处理结果
         */
//        future1.runAfterEitherAsync(future2, () -> {
//            System.out.println("任务3看是");
//        }, executor);

        /**
         * 可以感知到处理结果，但是不能返回结果
         */
//        future1.acceptEitherAsync(future2, (res) -> {
//            System.out.println(res);
//            System.out.println("3");
//        }, executor);

        System.out.println(future1.applyToEitherAsync(future2, (res) -> {
            return res + "333";
        }, executor).get());

        executor.shutdown();
```



#### 3.4.5 多任务组合



```java
// 全执行完
CompletableFuture<Void> future = CompletableFuture.allOf(future1, future2);
// 等待所有结果做完
future.get();

// 任意一个执行完
CompletableFuture.anyOf(future1, future2);
...get..
```



## 4.视图映射

通过SpringMVC提供的功能

```java
@Configuration
public class GulimallWebConfig implements WebMvcConfigurer {

    @Override
    public void addViewControllers(ViewControllerRegistry registry) {

        /**
         * 替代
         * @GetMapping("/login.html")
         * public String loginPage() { return "login"; }
         */
        registry.addViewController("/login.html").setViewName("login");
        registry.addViewController("/reg.html").setViewName("reg");
    }
}

```



## 5.model数据在重定向的数据丢失问题

非转发，重定向的时候model数据会丢失

因此需要使用另外的RedirectAttributes

```java
    @PostMapping("/regist")
    public String regist(@Valid UserRegistVo vo, BindingResult result, RedirectAttributes redirectAttributes) {
        if(result.hasErrors()) {
            Map<String, String> errors = result.getFieldErrors().stream().collect(Collectors.toMap(FieldError::getField, DefaultMessageSourceResolvable::getDefaultMessage));
            redirectAttributes.addFlashAttribute("errors", errors);
            // model在重定向的时候数据会丢失
            // model.addAttribute("errors", errors);

            // Request Method 'POST' not supported
            // 用户注册 -> /regist[post] -> 转发/reg.html （路径映射默认都是get方式访问的)，
            // 为了避免这个错误，可以不使用forward，而是交给系统自己拼接

            // 校验出错，回到注册页
//            return "forward:/reg.html";
//            return "reg";
            return "redirect:/reg.html";
        }

        return "redirect:/login.html";
    }
```

不过上面这种重定向方式，会重定向到当前服务的的端口，例如localhost:port/reg.html

并不会重定向到域名下，例如auth.gulimall.com/reg.html



## 6.Java存储Redis数据的奇怪数据问题

```java
public class LoginServiceImpl implements LoginService {

    @Autowired
    private ThirdPartyFeignService thirdPartyFeignService;

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Override
    public R sendCode(String phone) {
        String redisSaveName = AuthConstant.SMS_CODE_CACHE_PREFIX + phone;
        String redisCode = redisTemplate.opsForValue().get(redisSaveName);
        if (redisCode != null) {
            long codeTime = Long.parseLong(redisCode.split("_")[1]);
            if (System.currentTimeMillis() - codeTime < 60 * 1000) {
                // 60s内不能再发
                return R.error(BizCodeEnum.SMS_CODE_EXCEPTION.getCode(), BizCodeEnum.SMS_CODE_EXCEPTION.getMsg());
            }
        }

        // 1.接口防刷 TODO

        // 2.验证码校验 redis
        // 2.1 存key key-phone, value-code  sms:code:phone -> 123456
        String originCode = UUID.randomUUID().toString().substring(0, 5);
        String code = originCode + "_" + System.currentTimeMillis();
        redisTemplate.opsForValue().set(redisSaveName, code, 10, TimeUnit.MINUTES);

        thirdPartyFeignService.sendCode(phone, originCode);
        return R.ok();
    }
}
```

代码中，RedisTemplate中的泛型是否指定是有影响的，不指定泛型，存储的数据会有一个奇怪的前缀存储在Redis中，例如：

```java
\xac\xed\x00\x05t\x00\x14sms:code:17393116826
```

如果把泛型指定了，也就是RedisTemplate<String, String>这种，就可以避免前缀的影响：

```java
sms:code:17393116826
```



## 7.Gitee OAuth使用

### 7.1 接入第三方认证

[Gitee OAuth 文档](https://gitee.com/api/v5/oauth_doc#/)



### 7.2 换取access_token

code换取access_token的操作只能有一次

同一个用户的accessToken一段时间是不会变化的，即使多次获取



### 7.3 Session共享问题

![image-20231119150924996](https://chaobk-img-repo.oss-cn-shanghai.aliyuncs.com/image-20231119150924996.png)



#### 7.3.1 分布式下session共享问题

* 同一个服务，复制多分，session不同步问题
* 不同服务，session不能共享问题



## 8.单点登录

TODO



## 
