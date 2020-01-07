分布式链路追踪系统Sleuth和ZipKin
===
## Sleuth
* 什么是Sleuth
	* 一个组件，专门用于记录链路数据的开源组件
* `[order-service,96f95a0dd81fe3ab,852ef4cfcdecabf3,false]`
    * 第一个值，spring.application.name的值
    * 第二个值，96f95a0dd81fe3ab ，sleuth生成的一个ID，叫Trace ID，用来标识一条请求链路，一条请求链路中包含一个Trace ID，多个Span ID
    * 第三个值，852ef4cfcdecabf3、spanid 基本的工作单元，获取元数据，如发送一个http
    * 第四个值：false，是否要将该信息输出到zipkin服务中来收集和展示。

## 链路追踪组件Zipkin+Sleuth
* pom.xml
```xml
    <dependency>
	    <groupId>org.springframework.cloud</groupId>
	    <artifactId>spring-cloud-starter-zipkin</artifactId>
    </dependency>
```
* 配置zipkin.base-url
* 配置采样百分闭spring.sleuth.sampler