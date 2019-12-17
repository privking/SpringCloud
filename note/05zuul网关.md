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
```