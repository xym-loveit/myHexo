---
title: Spring Cloud常见问题总结
date: 2018-04-11 21:01:13
categories: Spring-Cloud系列
tags: [Spring-Cloud,SpringCloud常见问题,Eureka 配置,Ribbon 配置,Hystrix 配置,Turbine 配置,Spring Cloud 配置手册]
description: 在使用SpringCloud的过程中，可能会遇到一些问题。本文对常见问题做一些总结。
---

## Eureka常见问题
### Eureka注册服务慢
默认情况下，服务注册到Eureka Server的过程慢。在开发或测试时，常常希望能够加速这一过程，从而提升工作效率。Spring Cloud官方详细描述了该问题的原因并提供了解决方案：  

> 简单翻译：服务注册涉及到周期性心跳，默认30秒一次（通过客户端配置ServiceUrl）。只有当实例、服务端和客户端的本地缓存中的元数据都相同时，服务才能被其他客户端发现（所以可能需要3次心跳）。可以使用参数`eureka.instance.leaseRenewalntervalInSeconds`修改时间间隔，从而加快客户端连接到其他服务的过程。在生产环境中最好坚持使用默认值，因为在服务器内部有一些计算，他们会对续约作出假设。

综上，要想解决服务注册慢的问题，只须将`eureka.instance.leaseRenewalIntervalInSeconds`设定一个更小的值。该配置用于设置Eureka Client向Eureka Server发送心跳的时间间隔，默认30秒。在生产环境中，建议坚持使用默认值。

> 原文来自：[https://cloud.spring.io/spring-cloud-netflix/multi/multi__service_discovery_eureka_clients.html#_why_is_it_so_slow_to_register_a_service](https://cloud.spring.io/spring-cloud-netflix/multi/multi__service_discovery_eureka_clients.html#_why_is_it_so_slow_to_register_a_service)


### 已停止的微服务节点注销慢或不注销
在开发环境下，常常希望Eureka Server能迅速有效地注销已停止的微服务实例，然而，由于Eureka Server清理无效节点周期长（默认90秒），以及自我保护模式等原因，可能会遇到微服务注销慢甚至不注销的问题。解决方案如下：  
* Eureka Server 端：  
配置关闭自我保护，并按需配置Eureka Server清理无效节点的时间间隔。  
```
eureka.server.enable-self-preservation
#设为false，关闭自我保护，从而保证会注销微服务
eureka.server.eviction-interval-timer-in-ms
#清理间隔（单位毫秒，默认为60*1000）

```
* Eureka Client 端
配置开启健康检查，并按需配置续约更新时间和到期时间。  
```
eureka.client.healthcheck.enabled
#设置为true,开启健康检查（需要spring-boot-starter-actuator依赖）
eureka.instance.lease-renewal-interval-in-seconds
#续约更新时间间隔（默认30秒）
eureka.instance.lease-expiration-duration-in-seconds
#续约到期时间（默认90秒）

```
值得注意的是，这些配置仅建议在开发或测试时使用，生产环境建议坚持使用默认值。

**示例**

* Eureka Server配置：  
```
eureka:
  server:
    enable-self-preservation: false
    eviction-interval-timer-in-ms: 4000

```
* Eureka Client配置：
```
eureka:
  client:
    healthcheck:
      enabled: true
  instance:
    lease-expiration-duration-in-seconds: 30
    lease-renewal-interval-in-seconds: 10

```
> 修改Eureka的续约频率可能会打破Eureka的自我保护特性，详见：[https://github.com/spring-cloud/spring-cloud-netflix/issues/373](https://github.com/spring-cloud/spring-cloud-netflix/issues/373)。这意味着在生产环境中，如果想要使用Eureka的自我保护特性，应该坚持使用默认配置。


### 如何自定义微服务的Instance ID。
Instance ID用于唯一标识注册到Eureka Server上的微服务实例。在Eureka Server首页可以直观地看到各个微服务的Instance ID。如下图： 
 
![]()

在Spring Cloud中，服务的Instance ID的默认值是`${spring.cloud.client.hostname}:${spring.application.name}:${server.port}`。如果想要自定义这部分内容，只须在微服务中配置`eureka.instance.instance-id`属性即可，例如：

```
spring:
  application:
    name: microservice-provider-user
eureka:
  instance:
    #将Instance ID设置成 IP:端口的形式 
    instance-id: ${spring.cloud.client.ipAddress}:${server.port}

```
这样，就可将微服务`microservice-provider-user`的Instance ID设为IP:端口的形式。这样设置后，效果下图所示：  

![]()

> Spring Cloud初始化Instance ID的相关代码：
> org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration
> org.springframework.cloud.commons.util.IdUtils.getDefaultInstanceId(PropertyResolver resolver);
>org.springframework.cloud.netflix.eureka.EurekaInstanceConfigBean.getInstanceId()

### Eureka的UNKNOWN问题总结
注册信息unknown，是新手常会遇到的问题。如下图，有二种UNKNOWN的情况，一种是应用名称UNKNOWN,另一种是应用状态UNKNOWN。下面分别说明这二种情况。

![]()

**应用名称UNKNOWN**

应用名称UNKNOWN显然不合适，首先微服务的名称不够语义化，无法直接观看出这是哪个服务；更重要的是，我们常常使用应用名称消费对应微服务接口。
一般来说，有二种情况会导致该问题的发生：  
* 未配置`spring.application.name`或者`eureka.instance.appname`属性。如果这两个属性均不配置，就会导致应用名称UNKNOWN的问题。
* 某些版本的SpringFox会导致该问题，例如SpringFox2.6.0。建议使用SpringFox2.6.1或更新版本

**微服务实例状态UNKNOWN**

微服务实例的状态UNKNOWN同样很麻烦。一般来讲，只会请求状态是UP的微服务。该问题一般由健康检查导致。`eureka.client.healthcheck.enabled=true`必须设置在`application.yml`中，而不能设置在`bootstrap.yml`中，否则一些场景下会导致应用状态UNKNOWN的问题。

> SpringFox是一款基于Spring和Swagger的开源的API文档框架，前身是swagger-springmvc。官网：[http://springfox.github.io/springfox/](http://springfox.github.io/springfox/)。

## Hystrix/Feign整合Hystrix后首次请求失败
某些场景下，Feign或Ribbon整合Hystrix后，会出现首次调用失败的问题。

### 原因分析
Hystrix默认的超时时间是1秒，如果在1秒内得不到响应，就会进入`fallback`逻辑。由于Spring的懒加载机制，首次请求往往会比较慢，因此在某些机器（特别是低端的机器）上，首次请求需要的时间可能就会大于1秒。

### 解决方案

* 方法一：延长Hystrix的超时时间，示例：
`hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 5000`该配置让Hystrix的超时时间改为5秒 
* 方法二：禁用Hystrix的超时，示例：
`hystrix.command.default.execution.timeout.enabled: false`
* 方法三：对于Feign，还可为Feign禁用Hystrix，示例：
`feign.hystrix.enabled: false`
这样即可为Feign全局禁用Hystrix支持，该方式比较极端，一般不建议使用。

## Turbine集合数据不完整

在某些版本的Spring Cloud（例如Brixton SR5）中，Turbine会发生该问题。该问题直观体现是：使用Turbine聚合了多个微服务，但在Hystrix Dashboard上只能看到部分微服务的监控数据。

例如Turbine配置如下：
```
turbine:
  app-config: microservice-consumer-movie,microservice-consumer-movie-feign-hystrix-fallback-stream
  cluster-name-expression: "'default'"
  
```
Turbine理应聚合`microservice-consumer-movie`和`microservice-consumer-movie-feign-hystrix-fallback-stream`这两个微服务的监控数据，然而打开Hystrix Dashboard时，会发现Hystrix Dashboard只显示部分微服务的监控数据。

### 解决方案
当Turbine集合的微服务部署在同一台主机时，就会出现该问题。

* 方法一：为各个微服务配置不同的`hostname`。并将`preferIpAddress`设为false或者不设置。
```
eureka:
  client:
    service-url:
      defaultZone: http://discovery:8761/eureka/
  instance:
    hostname: ribbon #配置hostname

```
* 方法二：设置`turbine.combine-host-port=true`。
```
turbine:
  app-config: microservice-consumer-movie,microservice-consumer-movie-feign-hystrix-fallback-stream
  cluster-name-expression: "'default'"
  combine-host-port: true

```
* 方法三：升级Spring Cloud到Camden或更新版本。当然，也可单独升级Spring Cloud Netflix到1.2或更新版本（一般不建议单独升级Spring Cloud Netflix，因为可能会跟Spring Cloud其他组件冲突）。这是因为老版本中的`combine-host-port`默认值是false。Spring Cloud已经意识到该问题，所以在新的版本中将该属性的默认值修改为true。该方案和方法二本质上是一致的。

相关代码:  
> org.springframework.cloud.netflix.turbine.TurbineProperties.combineHostPort
> org.springframework.cloud.netflix.turbine.CommonsInstanceDiscovery.getInstance(String hostname, String port, String cluster, Boolean status)
> 相关issue：[https://github.com/spring-cloud/spring-cloud-netflix/issues/1087](https://github.com/spring-cloud/spring-cloud-netflix/issues/1087)

## Spring Cloud各组件配置属性
Spring Cloud中的大部门问题都可以使用配置属性的方式来解决。下面将配置的地址罗列出来，方便查阅。

### Spring Cloud的配置
Spring Cloud的所有组件配置都在其官方文档的附录，地址如下：  

[http://cloud.spring.io/spring-cloud-static/Camden.SR7/#_appendix_compendium_of_configuration_properties](http://cloud.spring.io/spring-cloud-static/Camden.SR7/#_appendix_compendium_of_configuration_properties)

### 原生配置
Spring Cloud 整合了很多类库，例如：`Eureka`、`Ribbon`、`Feign`等。这些组件自身也有一些配置属性，如下：  
* Eureka的配置：[https://github.com/Netflix/eureka/wiki/Configuring-Eureka](https://github.com/Netflix/eureka/wiki/Configuring-Eureka)
* Ribbon的配置：[https://github.com/Netflix/ribbon/wiki/Programmers-Guide](https://github.com/Netflix/ribbon/wiki/Programmers-Guide)
* Hystrix的配置：[https://github.com/Netflix/Hystrix/wiki/Configuration](https://github.com/Netflix/Hystrix/wiki/Configuration)
* Turbine的配置：[https://github.com/Netflix/Turbine/wiki/Configuration-(1.x)](https://github.com/Netflix/Turbine/wiki/Configuration-(1.x))


## Spring Cloud定位问题的思路

### 1、排查配置问题
排查配置有无问题，举几个例说明。

* YAML缩进是否正确
    * 项目启动报错，错误指向yml文件
* 配置属性是否正确
    * 可以借助IDE提示功能来排查，当IDE不提示或给出警告时，应格外注意。  
* 配置属性的位置是否正确
    * 应当配置在`Eureka Client`项目上的属性，配置在了`Eureka Server`项目上
    * 应当写在`bootstrap.yml`中的属性，写在了`application.yml`中，如：`spring.cloud.config.uri=http://localhost:8080`     
    * 应当写在`application.yml`的属性，写在了`bootstrap.yml`中，如：`eureka.client.healthcheck.enabled-true`
### 2、排查环境问题
如确认配置无误，即可考虑运行环境是否存在问题。

* 环境变量
    * 当前使用的`spring boot maven`插件版本需要jdk8支持,在jdk7下报错，需要指定插件版本号，添加`<version>`配置
* 依赖下载是否完整
    * 启动前使用`mvn clean package`,确认依赖完整性。
* 网络问题
    * 微服务之间通过网络保持通信，因此，网络常常是排查问题的关键。当问题发生时候，可优先排查网络问题。

### 3、排查代码问题
经过以上步骤，依然没有定位到问题，那么可能是编写代码出了问题。很多时候，常常因为少了某个注解，或是依赖缺失，而导致了各种异常。许多场景下，设置合理的日志级别，会对问题的定位有奇效。

### 4、排查Spring Cloud自身问题

如果确定不是自身问题，就可以debug一下Spring Cloud的代码。同时，可在github等平台给项目提交issue，然后参考官方回复，尝试规避相应问题。

**可参考资源：**
> 各项自身的github，例如：Eureka的Github：[https://github.com/netflix/eureka](https://github.com/netflix/eureka)
> Spring Cloud对应的项目Github，例如Eureka项目在 Spring Cloud Netflix中：[https://github.com/spring-cloud/spring-cloud-netflix](https://github.com/spring-cloud/spring-cloud-netflix)

> Spring Cloud的StackOverflow：[https://stackoverflow.com/questions/tagged/spring-cloud](https://stackoverflow.com/questions/tagged/spring-cloud)
> Spring Cloud的 Gitter[https://gitter.im/spring-cloud/spring-cloud](https://gitter.im/spring-cloud/spring-cloud)
> Spring Cloud中国社区 [http://www.spring4all.com](http://www.spring4all.com)
















