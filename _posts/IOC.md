### 一个简单的开始

```java
public static void main(String[] args) {
    ApplicationContext context = 
        new ClassPathXmlApplicationContext("classpath:application.xml");
}
```

其中 <font color=red>`ApplicationContext`</font> 是Spring中配置应用的核心接口，继承了多个接口，使其拥有了众多功能：
1. `ListableBeanFactory`，赋予了它**Bean工厂**的能力；
2. `ResourceLoader`，赋予了它**加载文件资源**的能力；
3. `ApplicationEventPublisher `，赋予了它**向注册的监听器发布事件**的能力；
4. `MessageSource`，赋予了它**处理消息，支持国际化**的能力；
5. 各上下文类成“父子”关系，“子”上下文中的定义优先于“父”上下文的，一整个web项目都可以用一个上下文，而其下的每一个servlet都可以有自己的上下文

在上面代码中，`ClassPathXmlApplicationContext`是在类路径中寻找`xml`格式的配置文件，然后根据其内容构建应用上下文。其大致的继承结构如图所示：
![ApplicationContext继承树](https://wzt-img.oss-cn-chengdu.aliyuncs.com/ApplicationContext-tree.png)

该树的三个叶节点分别是：
- `FileSystemXml...`：从系统路径中找xml
- `ClassPathXml...`：从类路径找xml
- `AnnotationConfig...`：根据注解配置
