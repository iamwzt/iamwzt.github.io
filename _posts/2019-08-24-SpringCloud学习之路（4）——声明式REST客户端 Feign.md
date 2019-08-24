本系列文章用于记录SpringCloud的学习历程，计划会先简单地过一遍各个组件，分别写个小Demo，混个眼熟；然后再去看一下源码，理解原理。

本系列文章导航：
- [SpringCloud学习之路（1）——注册中心 Eureka](https://iamwzt.github.io/2019/08/22/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-1-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83-Eureka/)
- [SpringCloud学习之路（2）——负载均衡 Ribbon](https://iamwzt.github.io/2019/08/23/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-2-%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1-Ribbon/)
- [SpringCloud学习之路（3）——断路器 Hystrix](https://iamwzt.github.io/2019/08/23/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-3-%E6%96%AD%E8%B7%AF%E5%99%A8-Hystrix/)
- **SpringCloud学习之路（4）——声明式REST客户端 Feign**

---

## Feign简介
Feign是一个**声明式**的**REST客户端**，它集成了Ribbon，是SpringCloud中调用服务的一种方式。

在前面的文章中，我们在服务调用者（invoker）中都是用 `RestTemplate` + `@LoadBalancer` 进行服务调用的。
本文要介绍的Feign便是另一种方式，它的特点是**声明式**，那么何为声明式，下面来看。

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

在前面我们已经有了：
1. Eureka服务端
2. 多个服务提供者（provider）
3. 一个具备负载均衡能力的服务调用者（invoker）

下面我们将引入 **Feign**，对服务调用者（invoker）进行改造。

---

## 使用Feign

1. 首先需要引入Feign的**依赖**：
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2. 在启动类上加 `@EnableFeignClients` 注解：
```java
//@EnableDiscoveryClient
@SpringBootApplication
//@EnableCircuitBreaker
@EnableFeignClients
public class InvokerApplication {
    public static void main(String[] args) {
        SpringApplication.run(InvokerApplication.class, args);
    }
}
```

3. 新建一个接口 `IMyFeignClient`：
```java
@FeignClient("service-provider1")
public interface IMyFeignClient {
    @GetMapping("hello")
    String hello();
}
```
注意这里的两个注解：
- `@FeignClient("service-provider1")`，**声明**该类的方法请求的服务名；
- `@GetMapping("hello")`，**声明**该方法请求的接口；

4. 然后在Controller里调用：
```java
@RestController
public class InvokerCtl {
    @Autowired
    private IMyFeignClient myFeignClient;

    @GetMapping("hello")
    public String sayHello() {
        return myFeignClient.hello();
    }
}
```

5. 启动服务，测试 
输入`http://loclahost:22001/hello`，可以看到浏览器轮流显示以下内容：
```
hello from Provider, Port: 21001
hello from Provider, Port: 21002
hello from Provider, Port: 21003
```

可见Feign已成功使用，并且是集成了负载均衡Ribbon的。这里我们也可以通过配置来改变负载均衡策略：
```yaml
service-provider1:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```

---

## 总结
通过上面的这个小小小demo，应该可以对**声明式**有点概念了，我们只要按以下方式两步走：
1. 在接口上加 `@FeignClient("service-name")` 声明要调用的服务名；
2. 添加需要调用的服务接口的各个方法（方法签名要和被调用的一致）

无需去手动实现接口，就可以在Controller类里访问服务的接口了。

此外，Feign集成了Ribbon，可以方便地使用负载均衡的各种功能。

（完）
