zuul网关
===
* 什么是网关
	* API Gateway，是系统的唯一对外的入口，介于客户端和服务器端之间的中间层，处理非业务功能 提供路由请求、鉴权、监控、缓存、限流等功能
	* 统一接入
		* 智能路由
		* AB测试、灰度测试
		* 负载均衡、容灾处理	
		* 日志埋点（类似Nignx日志）
	* 流量监控
		* 限流处理
		* 服务降级
	* 安全防护
		* 鉴权处理
		* 监控
		* 机器网络隔离
## 主流的网关
* zuul：是Netflix开源的微服务网关，和Eureka,Ribbon,Hystrix等组件配合使用，Zuul 2.0比1.0的性能提高很多
* kong: 由Mashape公司开源的，基于Nginx的API gateway
* nginx+lua：是一个高性能的HTTP和反向代理服务器,lua是脚本语言，让Nginx执行Lua脚本，并且高并发、非阻塞的处理各种请求
## SpringCloud的网关组件zuul基本使用
* 加入依赖
```xml
	<!-- eureka discovery -->
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
	</dependency>
	<!-- zuul -->
	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-starter-netflix-zuul</artifactId>
	</dependency>
```
* 启动类加注解 `@EnableZuulProxy` 
* 默认集成断路器  `@EnableCircuitBreaker`
* 默认访问规则 
	* `http://gateway:port/service-id/**`
	* `http://localhost:9000/apigetway/api/v1/order/save?userId=2&productId=1`
```yml
server:
  port: 9000

spring:
  application:
    name: api-getway

# 注册中心地址
eureka:
  client:
    serviceUrl:
	  defaultZone: http://localhost:8761/eureka/

zuul:
  routes:
  # 自定义路由
	order-service: /apigetway/**
 # 内外网隔离
 # 忽略整个服务
  ignored-services: order-service
  # 匹配过滤
  ignored-patterns: /*-service/api/v1/order/save
  # 默认网关不转发Cookie, Set-Cookie, Authorization
  # private Set<String> sensitiveHeaders = new LinkedHashSet<>(
  # 		Arrays.asList("Cookie", "Set-Cookie", "Authorization"));
  sensitive-headers: 
```

## 自定义Zuul过滤器
1. 新建一个filter包
2. 新建一个类，继承ZuulFilter，重写里面的方法
3. 在类顶部加注解，`@Component`,让Spring扫描

<img src="imgs/filters.png" width="900" height="600">

```java
/**
检查含order的url是否有token的请求头或参数
*/
@Component
public class LoginFilter extends ZuulFilter {
	 /**
     * 指定该Filter的类型
     * ERROR_TYPE = "error";
     * POST_TYPE = "post";
     * PRE_TYPE = "pre";
     * ROUTE_TYPE = "route";
     */
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

	/**
     * 指定该Filter执行的顺序（Filter从小到大执行）
     * DEBUG_FILTER_ORDER = 1;
     * FORM_BODY_WRAPPER_FILTER_ORDER = -1;
     * PRE_DECORATION_FILTER_ORDER = 5;
     * RIBBON_ROUTING_FILTER_ORDER = 10;
     * SEND_ERROR_FILTER_ORDER = 0;
     * SEND_FORWARD_FILTER_ORDER = 500;
     * SEND_RESPONSE_FILTER_ORDER = 1000;
     * SIMPLE_HOST_ROUTING_FILTER_ORDER = 100;
     * SERVLET_30_WRAPPER_FILTER_ORDER = -2;
     * SERVLET_DETECTION_FILTER_ORDER = -3;
     */
    @Override
    public int filterOrder() {
        return 0;
    }

  	/**
     * 指定需要执行该Filter的规则
     * 返回true则执行run()
     * 返回false则不执行run()
     */
    @Override
    public boolean shouldFilter() {
        RequestContext requestContext = RequestContext.getCurrentContext();
        String uri = requestContext.getRequest().getRequestURI();
        System.out.println(uri);
        if(uri.contains("order")){
            return true;
        }
        return false;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext requestContext = RequestContext.getCurrentContext();
        HttpServletRequest request = requestContext.getRequest();
        String token = request.getHeader("token");
        if(StringUtils.isBlank(token)){
            token = request.getParameter("token");
        }
        if(StringUtils.isBlank(token)){
            requestContext.setSendZuulResponse(false);
            requestContext.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
        }
        return null;
    }
}

```

```java
/**
 * 网关层限流限流
 */
@Component
public class RateLimiteFilter extends ZuulFilter {
	//guava
    //com.google.common.util.concurrent.RateLimiter
    private static final RateLimiter RATE_LIMITER = RateLimiter.create(1000);


    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return -4;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext requestContext = RequestContext.getCurrentContext();
        String uri = requestContext.getRequest().getRequestURI();
        System.out.println(uri);
        if(uri.contains("order")){
            return true;
        }


        return false;
    }

    @Override
    public Object run() throws ZuulException {
        if(!RATE_LIMITER.tryAcquire()){
            RequestContext requestContext = RequestContext.getCurrentContext();
            requestContext.setSendZuulResponse(false);
            requestContext.setResponseStatusCode(HttpStatus.TOO_MANY_REQUESTS.value());
        }
        return null;
    }
}
```

## 微服务网关Zull集群搭建
### lvs+keepalived+nginx高性能负载均衡集群
* LVS是一个开源的软件，可以实现传输层四层负载均衡。LVS是Linux Virtual Server的缩写，意思是Linux虚拟服务器。目前有三种IP负载均衡技术（VS/NAT、VS/TUN和VS/DR）；八种调度算法（rr,wrr,lc,wlc,lblc,lblcr,dh,sh）。
Keepalived作用
* LVS可以实现负载均衡，但是不能够进行健康检查，比如一个rs出现故障，LVS 仍然会把请求转发给故障的rs服务器，这样就会导致请求的无效性。keepalive 软件可以进行健康检查，而且能同时实现 LVS 的高可用性，解决 LVS 单点故障的问题，其实 keepalive 就是为 LVS 而生的。
