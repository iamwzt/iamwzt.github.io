本系列文章用于记录SpringCloud的学习历程，计划会先过一遍各个组件的基本使用，写一个小Demo，然后再去看一下核心的源码。废话不多说，开始。

### Eureka简介

> Eureka是Netflix开发的服务发现框架，本身是一个基于REST的服务，主要用于定位运行在AWS域中的中间层服务，以达到负载均衡和中间层服务故障转移的目的。SpringCloud将它集成在其子项目spring-cloud-netflix中，以实现SpringCloud的服务发现功能。
>
> Eureka包含两个组件：Eureka Server和Eureka Client。
>
> Eureka Server提供服务注册服务，各个节点启动后，会在Eureka Server中进行注册，这样EurekaServer中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观的看到。
>
> Eureka Client是一个java客户端，用于简化与Eureka Server的交互，客户端同时也就是一个内置的、使用轮询(round-robin)负载算法的负载均衡器。
>
> 在应用启动后，将会向Eureka Server发送心跳,默认周期为30秒，如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，Eureka Server将会从服务注册表中把这个服务节点移除(默认90秒)。
>
> Eureka Server之间通过复制的方式完成数据的同步，Eureka还提供了客户端缓存机制，即使所有的Eureka Server都挂掉，客户端依然可以利用缓存中的信息消费其他服务的API。综上，Eureka通过心跳检查、客户端缓存等机制，确保了系统的高可用性、灵活性和可伸缩性。

以上内容来自百度百科。

### 前期准备
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

### Eureka 服务端
Eureka 服务端需要引入的**依赖**有：
```xml
<!-- Eureka server 中已经包含了 starter-web -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```
**配置文件**为：
```yaml
server:
  port: 8761
eureka:
  client:
    # 不注册信息到服务器
    register-with-eureka: false
    # 不从服务器拉取注册信息
    fetch-registry: false
```
注意这里有两个参数设置为false，如果不设置默认为true，那么服务端也会作为一个客户端去寻找一个服务端进行注册和拉取注册信息。

**启动类**如下：
```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```
比较简单，在SpringBoot的基础上增加一个`@EnableEurekaServer`的注解。

OK，到这里一个简单的Eureka服务端便已完成，可在浏览器输入 `http://localhost:8761` 查看控制台。

### Eureka 客户端
Eureka 客户端可以简单地分成两类：服务提供者和调用者；当然服务提供者同时可能是另一个服务的调用者，反之亦然。

#### 服务提供者
首先需要引入**依赖**：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 注意这里和服务端的区别 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

然后是**配置文件**：
```yaml
server:
  port: 21001
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    hostname: localhost
spring:
  application:
    name: service-provider1
```
这里有两点需要说明下：
1. `eureka.client.service-url` 配置eureka服务端的地址
2. `spring.application.name` 配置本服务的名称

**一个服务类**：
```java
@RestController
public class ProviderCtl {
    @GetMapping("hello")
    public String sayHello() {
        return "hello";
    }
}
```

**启动类**：
```java
//@EnableEurekaClient
@SpringBootApplication
public class ProviderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ProviderApplication.class, args);
    }
}
```
`@EnableEurekaClient`注解用于启动服务注册，但实际上这里注释掉也是可以注册成功的，查了下网上的资料，说是有没有无所谓，默认是开启的，主要配置文件中配置了相关的参数即可注册。此处不纠结这点。

#### 服务调用者
引入**依赖**：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!-- 这里多了一个ribbon，这是负载均衡相关的包 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

**配置文件和启动类**与服务提供者基本一致，只有端口和应用名修改一下，这里就不贴出来了；

**一个服务类**：
```java
@RestController
public class InvokerCtl {
    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("hello")
    public String sayHello(){
        // 这里的域名就是服务提供者的服务名
        String url = "http://service-provider1/hello";
        return restTemplate.getForObject(url, String.class);
    }
}
```
**RestTemplate 配置类**：
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
这里用到了 `@LoadBalanced` 注解，作用是提供了负载均衡的能力，该注解来自`ribbon`依赖，具体的等我们介绍`ribbon`时再说，这里只是露个脸。

---

到此，就完成了一个简单的“注册中心——服务提供者——服务调用者”的Demo了。
打开 `http://localhost:8761`可以看到已经注册了两个服务：
![此处有图]()
输入 `http://localhost:22001/hello` 可以看到浏览器上显示 “hello”。完工。
