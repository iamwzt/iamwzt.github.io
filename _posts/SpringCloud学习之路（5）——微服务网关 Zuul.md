本系列文章用于记录SpringCloud的学习历程，计划会先简单地过一遍各个组件，分别写个小Demo，混个眼熟；然后再去看一下源码，理解原理。

本系列文章导航：
- [SpringCloud学习之路（1）——注册中心 Eureka](https://iamwzt.github.io/2019/08/22/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-1-%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83-Eureka/)
- [SpringCloud学习之路（2）——负载均衡 Ribbon](https://iamwzt.github.io/2019/08/23/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-2-%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1-Ribbon/)
- [SpringCloud学习之路（3）——断路器 Hystrix](https://iamwzt.github.io/2019/08/23/SpringCloud%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF-3-%E6%96%AD%E8%B7%AF%E5%99%A8-Hystrix/)
- [SpringCloud学习之路（4）——声明式REST客户端 Feign]()
- **SpringCloud学习之路（5）——微服务网关 Zuul**

---

## Zuul 简介
Zuul 是netflix开源的一个 API Gateway 服务器, 本质上是一个 web servlet 应用。

Zuul 的主要功能是路由转发和过滤器。路由功能是微服务的一部分，比如/api/user转发到到用户服务，/api/address转发到到地址服务。

Zuul 默认和 Ribbon 结合实现了负载均衡的功能。

---

## 前期准备
**与本系列文章所使用的软件版本略有不同，本文使用的版本为：**
1. SpringBoot (2.0.1.RELEASE)
2. SpringCloud (Finchley.RELEASE)

这是由于使用Zuul要求SpringBoot和SpringCloud的版本必须成一定组合，否则可能会出现下面这个问题：
```
Description:
The bean 'proxyRequestHelper', defined in class path resource [org/springframework/cloud/netflix/zuul/ZuulProxyAutoConfiguration$NoActuatorConfiguration.class], could not be registered. 
A bean with that name has already been defined in class path resource [org/springframework/cloud/netflix/zuul/ZuulProxyAutoConfiguration$EndpointConfiguration.class] and overriding is disabled.
 
Action:
Consider renaming one of the beans or enabling overriding by setting spring.main.allow-bean-definition-overriding=true

```

项目采用maven构建，相关pom文件如下：
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.1.RELEASE</version>
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

在前文中，我们已经有了：
1. Eureka服务端
2. 多个服务提供者（provider）
3. 一个具备负载均衡能力的服务调用者（invoker）

本文将使用 Zuul 新增一个网关，并配置路由。

---

## 使用Zuul
首先需要引入以下依赖：
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
    </dependency>
</dependencies>
```

在启动类上加 `@EnableZuulProxy` 注解：
```java
@EnableZuulProxy
@SpringBootApplication
public class ZuulApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZuulApplication.class, args);
    }
}
```

配置文件：
```yaml
spring:
  application:
    name: API-Gateway-Zuul
server:
  port: 20000
eureka:
  client:
    service-url:
        defaultZone: http://localhost:8761/eureka/
# 重点是下面这些        
zuul:
  routes:
    provider:
      path: /api-provider/**
      serviceId: service-provider1
    invoker:
      path: /api-invoker/**
      serviceId: service-invoker1
```
在配置文件中，我们配置了**provider** 和 **invoker**的路由，分别将 `service-provider1` 服务路由到了 `/api-provider/**`，以及将 `service-invoker1` 路由到了 `/api-invoker/**`。

到此，我们便完成了一个简易的网关服务，启动后在浏览器输入 `http://localhost:20000/api-provider/hello` 和 `http://localhost:20000/api-invoker/hello`，都能看到如下结果：
```
hello Sara From Provider, Port: 21003
```

## 拦截器功能
使用Zuul可以方便地定义拦截器，继承 `ZuulFilter`类并实现接口即可：
```java
@Component
public class MyFilter extends ZuulFilter {
    @Override
    public String filterType() {
        //  pre：路由之前
        //  routing：路由之时
        //  post： 路由之后
        //  error：发送错误调用
        return "pre";
    }

    @Override
    public int filterOrder() {
        // 拦截的顺序
        return 0;
    }

    @Override
    public boolean shouldFilter() {
        // 拦截的逻辑，此处为永远拦截
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        RequestContext cxt = RequestContext.getCurrentContext();
        HttpServletRequest request = cxt.getRequest();
        System.out.println(request.getMethod() + ": " + request.getRequestURI());
        String username = request.getParameter("username");
        // 参数中的username为admin时拦截，其他放行，到相应的服务
        if (Objects.equals(username, "admin")) {
            cxt.setSendZuulResponse(false);
            try {
                cxt.getResponse().setContentType("text/html;charset=utf-8");
                cxt.getResponse().getWriter().write("拦截到用户为admin");
            } catch (Exception e) {
                e.printStackTrace();
            }
            return null;
        }
        return null;
    }
}
```

