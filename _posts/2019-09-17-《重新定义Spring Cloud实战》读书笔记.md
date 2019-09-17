---
layout:     post
title:      《重新定义Spring Cloud实战》读书笔记
subtitle:   Eureka篇
date:       2019-09-17
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - SpringCloud
    - eureka
---

## 第二章 Spring Cloud Eureka 上篇

### 服务发现的由来

- 单体架构时代：依赖外部服务较少，通过配置域名的方式访问；
- SOA架构时代：通过 Nginx 来维护域名和IP的映射；
- 微服务时代：
    - 不引入服务注册中心，采用手动或脚本的方式，在部署时更新Nginx的配置文件；
    - 引入服务注册中心；
    
### 技术选型
    
常见的注册中心有Zookeeper、Consul、Etcd、Eureka等，各有特色。这里比较一下Consul和Eureka：

Eureka是采用不保证成功的复制操作来达到弱一致性（最终一致性），Client在Server端的注册信息有一个租期，并其需要通过心跳来保证连接。

Consul则是通过Raft算法来保证强一致性，并且客户端agent采用gossip协议来协助完成心跳检测。

Spring Cloud 选用 Eureka的原因：
1. AP而非CP；
2. Java体系；
3. 是Netflix的一部分，和Zuul、ribbon等整合得更好。

### 入门案例
见[SpringCloud学习之路-1-注册中心-Eureka](https://iamwzt.github.io/2019/08/22/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-1-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83-Eureka/)

### REST API 简介
Eureka Server 提供了REST API，允许非Java语言的其他应用服务通过HTTP REST的方法接入Eureka的服务发现中。

如：查询所有实例，可访问： `http://hostname:port/eureka/apps`

---

## 第三章 Spring Cloud Eureka 下篇

### Eureka的核心类
- `InstanceInfo`：代表注册的服务实例
- `LeaseInfo`：代表应用服务的租约信息
- `ServiceInstance`：对Service discovery的实例信息的抽象接口，在Spring Cloud中的实现类为 `EurekaRegistration`
- `InstanceStatus`：标识服务实例的状态

### 服务的核心操作
- 服务注册（register）
- 服务下线（cancel）
- 服务租约（renew）
- 服务剔除（evict）

围绕这几个操作，Eureka设计了几个核心操作类，其中和操作定义相关的两个接口是：
- `LeaseManager`：定义了上述的几个操作的方法；
- `LookupService`：定义了客户端从服务中心获取服务实例的查询方法；

### Eureka的设计理念

作为服务发现和注册中心，需要解决一下几个问题：
1. 如何注册；
    ```
    1. 调用相应的REST API；
    2. 若是Spring Cloud的应用，基于Spring Boot的自动配置，会自动实现注册。
    ```
2. 如何剔除；
    ```
    1. 若是正常关闭，会调用钩子方法或其他生命周期毁掉方法去删除注册信息；
    2. 若超过一定时间没有发送心跳，也会被server主动剔除。
    ```
3. 服务实例的注册信息如何保持一致性
    - AP 优于 CP
    ```
    由于Eureka是在部署在AWS的背景下设计的，设计者认为在云端，失败是不可避免的。
    此时应该在网络分区的时候，还能提供正常的服务，哪怕是过期的数据。
    ```
    - Peer to Peer 架构
    ```
    分布式系统多个副本之间的复制方式可以分为：主从复制和对等复制
    主从复制的写操作压力都在主副本上，是整个系统的瓶颈。
    对等复制的副本之间不分主从，每个副本都可以写，但是副本之间的数据同步及冲突是个难题。
    1、Server在执行复制操作时，会使用 HEADER_REPLICATION 请求头来区分普通操作，避免接收到请求的peer节点继续复制，产生死循环；
    2、Eureka 采用 lastDirtyTimeStamp 来解决数据不一致的问题（类似版本号）；
    3、复制有风险，可能会失败，需要应用实例定时的heartbeat来纠正数据不一致问题。
    ```
    - Zone 和 Region 设计
    ```
    这里的设计基于 AWS 的 Region 和 Zone 的基础设施之上。
    Region 之间资源隔离，默认的复制只在一个 Region 下的多个 AvailabilityZone 进行。
    Eureka 支持 preferSameZone，即优先拉取和应用实例同处一个AZ下的 Eureka Server 列表。
    ```
    - SELF PRESERVATION 设计
    ```
    心跳功能由于网络偶尔抖动或短暂不可用，可能会造成误判。
    或者 Server 和 Client 出现了网络分区，可能导致server清空部分实例列表，影响其“A”的属性。
    由于上两者原因，引入SELF PRESERVATION 设计：
    即存在一个阈值，当一定时间内收到的心跳数少于该阈值时，则关闭租约失效剔除机制，保护注册信息。
    ```

### Eureka 参数调优及监控

Eureka 的参数在 Client 端和 Server 端有些不同，这里不作一一说明。只针对几个新手常见问题来举例说明：

1. 服务下线了，Server接口返回的实例信息还在？
```
Eureke Server 并非强一致， 因此 registry 中会有过期的实例信息，有以下几个原因：
1 - 应用实例异常挂掉，需要等 EvictionTask 去剔除；
    可以调整 EvictionTask 频率（默认为60秒）：eureka.server.eviction-interval-timer-in-ms=5000
    
2 - 应用实例正常下线，但是Eureka Server有 response cache，需要等缓存过期；
    可以考虑关闭response cache：eureka.server.use-read-only-response-cache=false
    或者调整readWriteCacheMap过期时间：eureka.server.response-cache-auto-expiration-in-seconds=60
    
3 - 引入并开启了SELF PRESERVATION，导致 registry 中信息不会因为过期而被剔除，直到退出该模式。
    可以考虑关闭该模式：eureka.server.enable-self-preservation=false
```
2. 服务上线了，Client不能及时获取到？
```
适当提高Client拉取Server注册信息的频率：
eureka.client.registry-fetch-interval-seconds=10
```

### Eureka 的高可用
Client端的高可用：
1. 若Client启动之前没有Server，可通过配置`eureka.client.backup-registry-impl`从备份 registry 获取关键服务的信息；
2. 若Client启动后Server全部挂掉，本地内存有之前的数据，由于server挂掉不会更新；
3. 若Client启动后部分Server挂掉，可以人工介入删除配置文件中挂掉的Server地址，也可放任不管，Client维护了一张不可用Server的列表

Server端高可用：
1. 采用了Peer to Peer的机制，高可用都在Client端完成了；
2. 支持跨Region的高可用；
3. SELF PRESERVATION机制。
