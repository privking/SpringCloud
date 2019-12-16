服务降级熔断Hystrix
===
* 系统负载过高，突发流量或者网络等各种异常情况介绍，常用的解决方案
* 在一个分布式系统里，一个服务依赖多个服务，可能存在某个服务调用失败，比如超时、异常等，如何能够保证在一个依赖出问题的情况下，不会导致整体服务失败，通过Hystrix就可以解决
* 提供了熔断、隔离、Fallback、cache、监控等功能
## How to Use
* 依赖
```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    </dependency>
```
* 添加启动类注解，启动类里面增加注解`@EnableCircuitBreaker`
* api方法上增加 @HystrixCommand(fallbackMethod = "saveOrderFail")编写fallback方法实现，方法签名一定要和api方法签名一致（注意点！！！）
```java
@RestController
@RequestMapping("api/v1/order")
public class OrderController {
    @Autowired
    private OrderService orderService;

    @RequestMapping("save")
    @HystrixCommand(fallbackMethod = "saveOrderFail")
    public Object save(@RequestParam("userId") int userId, @RequestParam("productId") int productId){
        Map<String,Object> map  = new HashMap<>();
        map.put("code", "0");
        map.put("data", orderService.save(userId, productId));
        return map ;
    }

    private Object saveOrderFail(int userId,int productId){
        Map<String,Object> map  = new HashMap<>();
        map.put("code", "-1");
        map.put("msg", "出现错误");
        return map;
    }
}
```

## feignClient整合HyStrix 
```java
@FeignClient(name="product-service",fallback = ProductClientFallback.class)
public interface ProductClient {
    @GetMapping("api/v1/product/findById")
    String findById(@RequestParam("id")int id);
}
```
```java
@Component
public class ProductClientFallback implements ProductClient {
    @Override
    public String findById(int id) {
        System.out.println("调用异常");
        return null;
    }
}
```

## Hystrix 配置
* execution.isolation.strategy   隔离策略
	* THREAD 线程池隔离 （默认）
	* SEMAPHORE 信号量
	* 信号量适用于接口并发量高的情况，如每秒数千次调用的情况，导致的线程开销过高，通常只适用于非网络调用，执行速度快
* 超时时间调整
```yml
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 4000
```
* https://github.com/Netflix/Hystrix/wiki/Configuration#execution.isolation.strategy
## Dashboard仪表盘
* http://localhost:8781/hystrix
* Hystrix Dashboard输入： http://localhost:8781/actuator/hystrix.stream



## 配置再讲解

```java
// 熔断器在整个统计时间内是否开启的阀值
 @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),
 // 至少有3个请求才进行熔断错误比率计算
 /**
  * 设置在一个滚动窗口中，打开断路器的最少请求数。
  比如：如果值是20，在一个窗口内（比如10秒），收到19个请求，即使这19个请求都失败了，断路器也不会打开。
  */
 @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "3"),
 //当出错率超过50%后熔断器启动
 @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
 // 熔断器工作时间，超过这个时间，先放一个请求进去，成功的话就关闭熔断，失败就再等一段时间
 @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000")
@HystrixProperty(name = "coreSize", value = "30"),
 /**
  * BlockingQueue的最大队列数，当设为-1，会使用SynchronousQueue，值为正时使用LinkedBlcokingQueue。
  */
 @HystrixProperty(name = "maxQueueSize", value = "101"),
 /**
  * 设置存活时间，单位分钟。如果coreSize小于maximumSize，那么该属性控制一个线程从实用完成到被释放的时间.
  */

/**
  我们知道，线程池内核心线程数目都在忙碌，再有新的请求到达时，线程池容量可以被扩充为到最大数量。
等到线程池空闲后，多于核心数量的线程还会被回收，此值指定了线程被回收前的存活时间，默认为 2，即两分钟。
*/
 @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"),
 /**
  * 设置队列拒绝的阈值,即使maxQueueSize还没有达到
  */
 @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),

// 滑动统计的桶数量
 /**
  * 设置一个rolling window被划分的数量，若numBuckets＝10，rolling window＝10000，
  *那么一个bucket的时间即1秒。必须符合rolling window % numberBuckets == 0。默认1
  */
 @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "10"),
 // 设置滑动窗口的统计时间。熔断器使用这个时间
 /** 设置统计的时间窗口值的，毫秒值。
  circuit break 的打开会根据1个rolling window的统计来计算。
  若rolling window被设为10000毫秒，则rolling window会被分成n个buckets，
  每个bucket包含success，failure，timeout，rejection的次数的统计信息。默认10000
  **/
 @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "10000")


//fallbackMethod：方法执行时熔断、错误、超时时会执行的回退方法，需要保持此方法与 Hystrix 方法的签名和返回值一致。

//defaultFallback：默认回退方法，当配置 fallbackMethod 项时此项没有意义，另外，默认回退方法不能有参数，返回值要与 Hystrix方法的返回值相同。
```

### 降级
```java
@RestController
@RequestMapping("/hystrix1")
@DefaultProperties(defaultFallback = "defaultFail")
public class HystrixController1 {

    @HystrixCommand(fallbackMethod = "fail1")
    @GetMapping("/test1")
    public String test1() {
        throw new RuntimeException();
    }

    private String fail1() {
        System.out.println("fail1");
        return "fail1";
    }

    @HystrixCommand(fallbackMethod = "fail2")
    @GetMapping("/test2")
    public String test2() {
        throw new RuntimeException();
    }

    @HystrixCommand(fallbackMethod = "fail3")
    private String fail2() {
        System.out.println("fail2");
        throw new RuntimeException();
    }

    @HystrixCommand
    private String fail3() {
        System.out.println("fail3");
        throw new RuntimeException();
    }

    private String defaultFail() {
        System.out.println("default fail");
        return "default fail";
    }


}

//当访问/hystrix1/test1抛出异常，服务降级返回fail1。
//当访问/hystrix1/test2抛出异常，服务不断降级返回default fail。
```

### 熔断
```java
@RequestMapping("/hystrix2")
@DefaultProperties(defaultFallback = "defaultFail")
public class HystrixController2 {

    @HystrixCommand(commandProperties =
            {
                    // 熔断器在整个统计时间内是否开启的阀值
                    @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),
                    // 至少有3个请求才进行熔断错误比率计算
                    /**
                     * 设置在一个滚动窗口中，打开断路器的最少请求数。
                     比如：如果值是20，在一个窗口内（比如10秒），收到19个请求，即使这19个请求都失败了，断路器也不会打开。
                     */
                    @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "3"),
                    //当出错率超过50%后熔断器启动
                    @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),
                    // 熔断器工作时间，超过这个时间，先放一个请求进去，成功的话就关闭熔断，失败就再等一段时间
                    @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000"),
                    // 统计滚动的时间窗口
                    @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "10000")
            })
    @GetMapping("/test1")
    public String test1(@RequestParam("id") Integer id) {
        System.out.println("id:" + id);

        if (id % 2 == 0) {
            throw new RuntimeException();
        }
        return "test_" + id;
    }

    private String defaultFail() {
        System.out.println("default fail");
        return "default fail";
    }
}
//当访问1次/hystrix2/test1?id=1和2次/hystrix2/test1?id=2，错误率达66%超过了设置的50%。服务进入熔断。
//如果在5s之后，下一次请求成功，会关闭熔断，服务恢复。


```

```java
@RestController
@RequestMapping("/hystrix3")
@DefaultProperties(defaultFallback = "defaultFail")
public class HystrixController3 {

    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "500")
    })
    @GetMapping("/test1")
    public String test1() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(1000);
        return "test1";
    }

    @HystrixCommand(commandProperties = {
            @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1500"),
            // 滑动统计的桶数量
            /**
             * 设置一个rolling window被划分的数量，若numBuckets＝10，rolling window＝10000，
             *那么一个bucket的时间即1秒。必须符合rolling window % numberBuckets == 0。默认1
             */
            @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "10"),
            // 设置滑动窗口的统计时间。熔断器使用这个时间
            /** 设置统计的时间窗口值的，毫秒值。
             circuit break 的打开会根据1个rolling window的统计来计算。
             若rolling window被设为10000毫秒，则rolling window会被分成n个buckets，
             每个bucket包含success，failure，timeout，rejection的次数的统计信息。默认10000
             **/
            @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "10000")},
            threadPoolProperties = {
                    @HystrixProperty(name = "coreSize", value = "15"),
                    /**
                     * BlockingQueue的最大队列数，当设为-1，会使用SynchronousQueue，值为正时使用LinkedBlcokingQueue。
                     */
                    @HystrixProperty(name = "maxQueueSize", value = "15"),
                    /**
                     * 设置存活时间，单位分钟。如果coreSize小于maximumSize，那么该属性控制一个线程从实用完成到被释放的时间.
                     */
                    @HystrixProperty(name = "keepAliveTimeMinutes", value = "2"),
                    /**
                     * 设置队列拒绝的阈值,即使maxQueueSize还没有达到
                     */
                    @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),
                    @HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "10"),
                    @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "10000")
            })
    @GetMapping("/test2")
    public String test2() throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(1000);
        return "test2";
    }

    private String defaultFail() {
        System.out.println("default fail");
        return "default fail";
    }
}
//当.\ab -c 30 -n 30 http://localhost:8082/hystrix3/test1并发30个请求。
//由于没有显式配置maxQueueSize。Hystrix不会向阻塞队列里面放任务。当任务的数量超过了线程池负载的阈值，会采用拒绝任务策略。
//核心线程数量+初始数量=11个线程。11个线程由于线程阻塞进入了Timeout状态，后续无法正常继续执行其他任务，采取拒绝任务策略(拒绝了30-11=19个任务)。

//maxQueueSize：作业队列的最大值，默认值为 -1，设置为此值时，队列会使用 SynchronousQueue，此时其 size 为0。
//Hystrix 不会向队列内存放作业。如果此值设置为一个正的 int 型，队列会使用一个固定 size 的 LinkedBlockingQueue。
//此时在核心线程池内的线程都在忙碌时，会将作业暂时存放在此队列内，但超出此队列的请求依然会被拒绝。

//queueSizeRejectionThreshold：由于 maxQueueSize 值在线程池被创建后就固定了大小，
//如果需要动态修改队列长度的话可以设置此值，
//即使队列未满，队列内作业达到此值时同样会拒绝请求。
//此值默认是 5，所以有时候只设置了 maxQueueSize 也不会起作用。

//当.\ab -c 100 -n 100 http://localhost:8082/hystrix3/test2并发100个请求。核心线程数是15个，加上初始的1个。
//核心线程数应该是16个，阻塞队列里面有15个。所以executions为31个。
//当线程数超过了阻塞队列的阈值，就拒绝任务，所以Rejected的数量是100-31=69个。
//默认情况下，在滑动窗口内，请求数量超过了20个，才计算熔断错误比率。当熔断错误比率超过了50%，采用熔断策略。

```





