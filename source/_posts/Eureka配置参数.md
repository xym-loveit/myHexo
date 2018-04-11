---
title: Eureka配置参数
date: 2018-04-08 10:16:09
categories: eureka配置
tags: [eureka配置]
description: eureka配置
---

### 服务注册 
* `eureka.client.register-with-eureka`: 是否支持注册功能，默认为true
* `eureka.server.enable-self-preservation`: 服务注册中心自我保护机制，默认为true（服务注册中心红色警告）
* `eureka.client.region`: 设置微服务应用的Region，默认为default
* `eureka.client.availability-zones`: 设置微服务应用的zone，多个可以采用“，”分割，默认为defaultZone，和region是一对多的关系，即一个region下有多个zone
* `http://<username>:<password>@localhost:1111/eureka/`,配置安全的注册中心地址，其中<username>为安全校验信息的用户名，<passoword>为该用户的密码。

### 服务续约
* `eureka.instance.lease-renewal-interval-in-seconds`: 定义服务续约任务的调用间隔时间，默认30秒
* `eureka.instance.lease-expiration-duration-in-seconds`: 定义服务失效的时间，默认90秒

### 获取服务
* `eureka.client.fetch-registry`: 提供获取服务功能，默认为true
* `eureka.client.registry-fetch-interval-seconds`: 服务注册中心，服务清单缓存更新时间间隔（更新频率），默认30秒

下面整理了`org.springframework.cloud.netflix.eureka.EurekaClientConfigBean`中定义的常用配置参数以及对应的说明和默认值,这些参数均以`eureka.client`为前缀。

参数名|说明|默认值
---|---|---
enabled|启用eureka客户端|true
registryFetchIntervalSeconds|从eureka服务端获取注册信息的间隔时间，单位为秒|30
instanceInfoReplicationIntervalSeconds|更新实例信息的变化到Eureka服务端的间隔时间，单位为秒|30
initialInstanceInfoReplicationIntervalSeconds|初始化实例信息到Eureka服务端的间隔时间，单位为秒|40
eurekaServiceUrlPollIntervalSeconds|轮询Eureka服务端地址更改的间隔时间，单位为秒。当我们与`Spring Cloud Config`配合，动态刷新`Eureka`的`ServiceURL`地址时需要关注该参数|5*60=300(5分钟)
eurekaServerReadTimeoutSeconds|读取Eureka Server信息的超时时间，单位为秒|8
eurekaServerConnectTimeoutSeconds|连接Eureka Server的超时时间，单位为秒|5
eurekaServerTotalConnections|从Eureka客户端到所有Eureka服务端的连接总数|200
eurekaServerTotalConnectionsPerHost|从Eureka客户端到每个Eureka服务端主机的连接总数|50
eurekaConnectionIdleTimeoutSeconds|Eureka服务端连接的空闲关闭时间，单位为秒|30
heartbeatExecutorThreadPoolSize|心跳连接池的初始化线程数|2
heartbeatExecutorExponentialBackOffBound|心跳超时重试延迟时间的最大乘数值|10
cacheRefreshExecutorThreadPoolSize|缓存刷新线程池的初始化线程数|2
cacheRefreshExecutorExponentialBackOffBound|缓存刷新重试延迟时间的最大乘数值|10
useDnsForFetchingServiceUrls|使用DNS来获取Eureka服务端的serviceURL|false
registerWithEureka|是否要将自身的实例信息注册到Eureka服务端|true
preferSameZoneEureka|是否偏好使用处于相同Zone的Eureka服务端|true
filterOnlyUpInstances|获取实例时是否过滤，仅保留UP状态的实例|true
fetchRegistry|是否从Eureka服务端获取注册信息|true


下面整理了一些`org.springframework.cloud.netflix.eureka.EurekaInstanceConfigBean`中定义的配置参数以及对应的说明和默认值，这些参数均以`eureka.instance`为前缀。

参数名|说明|默认值
---|---|---
preferIpAddress|是否优先使用IP地址作为主机名的标识|false
leaseRenewalIntervalInSeconds|Eureka客户端向服务端发送心跳的时间间隔，单位为秒|30
leaseExpirationDurationInSeconds|Eureka服务端在收到最后一次心跳之后等待的时间上限，单位为秒。超过该时间之后服务端会将该服务实例从服务清单中剔除，从而禁止服务调用请求被发送到该实例上|90
nonSecurePort|非安全的通信端口号|80
securePort|安全的通信端口号|443
nonSecurePortEnabled|是否启用非安全的通信端口号|true
securePortEnabled|是否启用安全的端口号|false
appname|服务名，默认取spring.application.name的配置值，如果没有则为unknow|
hostname|主机名，不配置的时候将根据操作系统的主机名来获取|






