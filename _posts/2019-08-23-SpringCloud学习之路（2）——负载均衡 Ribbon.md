---
layout:     post
title:      SpringCloud学习之路（2）——负载均衡 Ribbon
date:       2019-08-23
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - SpringCloud
    - Ribbon
---

本系列文章用于记录SpringCloud的学习历程，计划会先简单地过一遍各个组件，分别写个小Demo，混个眼熟；然后再去看一下源码，理解原理。

本系列文章导航：
- [SpringCloud学习之路（1）——注册中心 Eureka](https://iamwzt.github.io/2019/08/22/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-1-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83-Eureka/)
- **SpringCloud学习之路（2）——负载均衡 Ribbon**
- [SpringCloud学习之路（3）——断路器 Hystrix](https://iamwzt.github.io/2019/08/23/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-3-%E6%96%AD%E8%B7%AF%E5%99%A8-Hystrix/)
- [SpringCloud学习之路（4）——声明式REST客户端 Feign](https://iamwzt.github.io/2019/08/24/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-4-%E5%A3%B0%E6%98%8E%E5%BC%8FREST%E5%AE%A2%E6%88%B7%E7%AB%AF-Feign/)
- [SpringCloud学习之路（5）——微服务网关 Zuul](https://iamwzt.github.io/2019/08/27/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-5-%E5%BE%AE%E6%9C%8D%E5%8A%A1%E7%BD%91%E5%85%B3-Zuul/)

---

## Ribbon简介
Ribbon是Netflix发布的负载均衡器，它有助于控制HTTP和TCP的客户端的行为。
为Ribbon配置服务提供者地址后，Ribbon就可基于**某种负载均衡算法**，自动地帮助服务消费者去请求。
Ribbon默认为我们提供了很多负载均衡算法，例如轮询、随机等。当然，我们也可为Ribbon实现自定义的负载均衡算法。
在Spring Cloud中，当Ribbon与Eureka配合使用时，Ribbon可自动从Eureka Server获取服务提供者地址列表，并基于负载均衡算法，请求其中一个服务提供者实例。

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

完成一个Ribbon的Demo，还需要以下这些组件：
1. 注册中心 Eureka
2. 多个服务提供者
3. 一个服务调用者

在 [SpringCloud学习之路（1）——注册中心 Eureka](https://iamwzt.github.io/2019/08/22/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-1-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83-Eureka/) 一文中，我们已经有了这些组件了，具体可以去再过一遍，这里需要做的改造只有将一个服务提供者扩充为多个。

---

## 多个服务提供者
在这里我们采用不同的端口来完成多个服务提供者的扩充，可以在idea启动时指定参数：
```
-Dserver.port=xxx
```

为了区分不同服务提供者，对其服务方法也进行一点改造，具体如下：
```java
@RestController
public class ProviderCtl {
    @Value("${server.port}")
    private String port;

    @GetMapping("hello")
    public String sayHello() {
        return "hello from Provider, Port: " + port;
    }
}
```
改造完成！

---

## 服务调用者
负载均衡是在服务调用者这边实现的，在[SpringCloud学习之路（1）——注册中心 Eureka](https://iamwzt.github.io/2019/08/22/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-1-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83-Eureka/)中其实已经实现，这里再作一下说明。

在配置 `RestTemplate` 实例的时候，我们引入了一个注解 `@LoadBalanced`，这将会在RestTemplate中加入LoadBalancerInterceptor这个拦截器，
把 `RestTemplate` 配置成一个具有负载均衡能力的客户端：
```java
@Configuration
public class HttpClient {
    @Bean("restTemplate")
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}
```

多次访问 `http://loclahost:22001/hello`，可以看到以下结果轮流出现：
```
hello from Provider, Port: 21001
hello from Provider, Port: 21002
```

也可通过 `LoadBalancerClient`， 来获取选择的服务的一些信息：
```java
@RestController
public class InvokerCtl {
    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @GetMapping("hello")
    public String sayHello() {
        String url = "http://service-provider1/hello";
        return restTemplate.getForObject(url, String.class);
    }
    @GetMapping("info")
    public String serviceInfo() {
        ServiceInstance choose = loadBalancerClient.choose("service-provider1");
        return "HOST: " + choose.getHost() + " PORT: " + choose.getPort();
    }
}
```

上面讲到，Ribbon默认为我们提供了很多负载均衡算法，例如轮询、随机等。
而从demo的结果来看，其默认的算法为轮询，如果想要变成其他算法，比如随机，可以修改配置文件：
```yaml
# 相应的服务名
service-provider1:
  ribbon:
    # 负载均衡算法
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```
重新起服务，可以看到结果变成了随机出现了。

（完）
