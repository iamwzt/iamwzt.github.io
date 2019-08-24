---
layout:     post
title:      SpringCloud学习之路（3）——断路器 Hystrix
date:       2019-08-23
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - SpringCloud
    - Hystrix
---

本系列文章用于记录SpringCloud的学习历程，计划会先简单地过一遍各个组件，分别写个小Demo，混个眼熟；然后再去看一下源码，理解原理。

本系列文章导航：
- [SpringCloud学习之路（1）——注册中心 Eureka](https://iamwzt.github.io/2019/08/22/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-1-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83-Eureka/)
- [SpringCloud学习之路（2）——负载均衡 Ribbon](https://iamwzt.github.io/2019/08/23/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-2-%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1-Ribbon/)
- **SpringCloud学习之路（3）——断路器 Hystrix**
- [SpringCloud学习之路（4）——声明式REST客户端 Feign](https://iamwzt.github.io/2019/08/24/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-4-%E5%A3%B0%E6%98%8E%E5%BC%8FREST%E5%AE%A2%E6%88%B7%E7%AB%AF-Feign/)

---

## Hystrix 简介
> 在微服务场景中，通常会有很多层的服务调用。如果一个底层服务出现问题，故障会被向上传播给用户。
我们需要一种机制，当底层服务不可用时，可以阻断故障的传播。这就是断路器的作用。它是系统服务稳定性的最后一重保障。
>
> 在springcloud中，断路器组件就是Hystrix。Hystrix也是Netflix套件的一部分。
它的功能是，当对某个服务的调用在**一定的时间内**，有**超过一定次数**并且**失败率超过一定值**，该服务的断路器会打开。返回一个由开发者设定的fallback。
> 
> **Hystrix被设计的目标是**：
>  
> - 对通过第三方客户端库访问的依赖项（通常是通过网络）的延迟和故障进行保护和控制。
> - 在复杂的分布式系统中阻止级联故障。
> - 快速失败，快速恢复。
> - 回退，尽可能优雅地降级。
> - 启用近实时监控、警报和操作控制。
>
> **Hystrix的实现原理**：
> - 用一个HystrixCommand 或者 HystrixObservableCommand （这是命令模式的一个例子）包装所有的对外部系统（或者依赖）的调用，典型地它们在一个单独的线程中执行
> - 调用超时时间比你自己定义的阈值要长。有一个默认值，对于大多数的依赖项你是可以自定义超时时间的。
> - 为每个依赖项维护一个小的线程池(或信号量)；如果线程池满了，那么该依赖性将会立即拒绝请求，而不是排队。
> - 调用的结果有这么几种：成功、失败（客户端抛出异常）、超时、拒绝。
> - 在一段时间内，如果服务的错误百分比超过了一个阈值，就会触发一个断路器来停止对特定服务的所有请求，无论是手动的还是自动的。
> - 当请求失败、被拒绝、超时或短路时，执行回退逻辑。
> - 近实时监控指标和配置变化。

更多介绍参考[Hystrix-wiki](https://github.com/Netflix/Hystrix/wiki)

---

## 前期准备
本系列文章所使用的软件版本如下：
1. SpringBoot (2.1.3.RELEASE)
2. SpringCloud (Finchley.RELEASE)

项目采用maven构建，相关pom文件如下：
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.1.3.RELEASE</version>
    <relativePath/>
</parent>
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 使用Hystrix
Hystrix是在服务调用者上生效的，所以首先需要引入**依赖**，现在Invoker的全部依赖如下所示：
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!--eureka客户端-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <!--负载均衡-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    </dependency>
    <!--断路器-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-hystrix</artifactId>
    </dependency>
    <!--在Spring Boot 2 中需要手动引入，否则无法使用 @HystrixCommand 注解-->
    <dependency>
        <groupId>com.netflix.hystrix</groupId>
        <artifactId>hystrix-javanica</artifactId>
        <version>RELEASE</version>
    </dependency>    
</dependencies>
```

在启动类中打开开关（`@EnableCircuitBreaker`）：
```java
//@EnableDiscoveryClient
@SpringBootApplication
@EnableCircuitBreaker
public class InvokerApplication {
    public static void main(String[] args) {
        SpringApplication.run(InvokerApplication.class, args);
    }
}
```

完成一个Hystrix命令接口，这里我们将原先在Controller中的逻辑后移到Service中：
```java
// 简单起见，就不实现接口了
@Service
public class InvokerService {
    @Autowired
    private RestTemplate restTemplate;
    
    // 关键的注解，fallbackMethod参数需要指定发生fallback时调用的方法，方法签名需要和正常方法一致
    @HystrixCommand(fallbackMethod = "helloFallback")
    public String hello() {
        String url = "http://service-provider1/hello";
        return restTemplate.getForObject(url, String.class);
    }

    public String helloFallback() {
        return "error_hello!";
    }
}
```

万事具备，将服务提供方关闭，来调用一下服务调用方的接口 `http://localhost:22001/hello`，可以看到以下结果：
```
error_hello!
```
重新将服务起起来，过一会调用结果就又恢复正常了。

（完）
