---
layout:     post
title:      @Value都能注入哪些值
date:       2019-09-01
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - Spring
---

在实践中，`@Value`注解常用于注入配置文件以及环境变量中的值，但除此之外，其还有许多其他的用法，本文将对其做一个总结。

### 1. 普通字符串
```java
@Value("oneNormalString")
private String normal;
```
### 2. 系统变量
```java
@Value("#{systemProperties['os.name']}")
private String osName;
```

### 3. 表达式结果
```java
@Value("#{ T(java.lang.Math).random() * 100.0 }")
private double randomNumber;
```

### 4. 文件资源
```java
@Value("classpath:com/wzt/spring/configuration/config.txt")
private Resource resourceFile; 
```

### 5. URL资源
```java
@Value("http://www.baidu.com")
private Resource testUrl;
```

### 6. 其他Bean的字段
```java
@Value("#{fip.bandwidth}")
private String fromAnotherBean;
```
这个叫做“fip”的bean长这样“
```java
@Bean("fip")
public Fip fip(){
    Fip fip = new Fip();
    fip.setBandwidth("123MB");
    return fip;
}
```

### 7. 环境变量
```java
@Value("${fromEnv}")
private String fromEnv;
```

### 8. application.yaml配置文件的变量

> application.yaml/properties等配置文件是默认的，可以直接注入。
```java
@Value("${spring.application.name}")
private String appName;
```

### 9. 其他配置文件

> 除了8中的默认配置文件，还可以配合`@PropertySource`注解配置自定义的配置文件

首先建两个配置文件：

第一个：config.properties
```properties
config.name=myConfig
# 这是另一个配置文件的名称
anotherConfig.file=myConfig2
```
第二个：myConfig2.properties
```properties
config2.name=myConfig2
```
然后用 `@PropertySource`注解在类上进行配置：
```java
@RestController
@PropertySource({"classpath:config.properties",
        "classpath:${anotherConfig.file}.properties"})
public class ValueTestController {
    // ...
}
```
这里注意第二个配置文件的名称是用了第一个配置文件里的变量值`${anotherConfig.file}`.

然后使用这两个值：
```java
@Value("${config.name}")
private String configName;
@Value("${config2.name}")
private String config2Name;
```
### 10. 统一测试
```java
@RestController
@PropertySource({"classpath:config.properties",
        "classpath:${anotherConfig.file}.properties"})
public class ValueTestController {
    @Value("oneNormalString")
    private String normal;
    @Value("#{systemProperties['os.name']}")
    private String osName;
    @Value("#{ T(java.lang.Math).random() * 100.0 }")
    private double randomNumber;
    @Value("classpath:com/wzt/spring/configuration/config.txt")
    private Resource resourceFile;
    @Value("http://www.baidu.com")
    private Resource testUrl;
    @Value("#{fip.bandwidth}")
    private String fromAnotherBean;
    @Value("${fromEnv}")
    private String fromEnv;

    @Value("${spring.application.name}")
    private String appName;

    @Value("${config.name}")
    private String configName;

    @Value("${config2.name}")
    private String config2Name;
    @Autowired
    private Environment env;

    @GetMapping(value = "sout", produces = "application/json")
    public String sout() throws Exception {
        return "{\"Normal\": \"" + normal +
                "\" , \"OsName\": \"" + osName +
                "\" , \"RandomNumber\": \"" + randomNumber +
                "\" , \"FromAnotherBean\": \"" + fromAnotherBean +
                "\" , \"resourceFile\": \"" + resourceFile.getFilename() +
                "\" , \"testUrl\": \"" + testUrl.getURI() +
                "\" , \"fromEnv\": \"" + fromEnv +
                "\" , \"from_configuration_file\": {" +
                "\"appName\": \"" + appName +
                "\" , \"configName\": \"" + configName +
                "\" , \"config2Name\": \"" + config2Name +
                "\"}}";
    }
}
```
访问该接口，返回：
```
{
    "Normal": "oneNormalString",
    "OsName": "Windows 7",
    "RandomNumber": "53.982013924795446",
    "FromAnotherBean": "123MB",
    "resourceFile": "config.txt",
    "testUrl": "http://www.baidu.com",
    "fromEnv": "from_env",
    "from_configuration_file": {
        "appName": "VALUE_TEST",
        "configName": "myConfig",
        "config2Name": "myConfig2"
    }
}
```

(END)
