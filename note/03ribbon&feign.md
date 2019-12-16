ribbon&feign
===
## ribbon调用远程服务
```java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

Map<String,Object> map = restTemplate.getForObject("http://product-service/api/v1/product/findById?id="+productId, Map.class);
```
## Ribbon负载均衡@LoadBanlanced
* 默认轮训
* 自定义负载均衡
```yml
# application.yml
product-service:
	  ribbon:
        NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule 
        # 随机策略

# WeightedResponseTimeRule 权重策略
```
## feign 
* 加入依赖
```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
```
* 启动类增加@EnableFeignClients
* 增加一个接口 并@FeignClient(name="product-service")
* 注意点
    * 路径
    * Http方法必须对应
    * 使用requestBody，应该使用@PostMapping
    * 多个参数的时候，通过@RequestParam（"id") int id)方式调用
* 超时配置
    * 默认optons readtimeout是60，但是由于hystrix默认是1秒超时
    * 修改调用超时时间
    ```yml
		feign:
		  client:
		    config:
		      default:
		        connectTimeout: 2000
                readTimeout: 2000
    ```
## ribbon 和 feign区别
* 选择feign
* 默认集成了ribbon
* 写起来更加思路清晰和方便
* 采用注解方式进行配置，配置熔断等方式方便
