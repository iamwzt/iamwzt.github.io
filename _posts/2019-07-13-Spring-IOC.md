## 一个简单的开始：`ApplicationContext`

```java
public static void main(String[] args) {
    ApplicationContext context = 
        new ClassPathXmlApplicationContext("classpath:application.xml");
}
```

其中 **`ApplicationContext`** 是Spring中配置应用的核心接口，继承了多个接口，使其拥有了众多功能：
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

现在加点实际的东西。

### 通过上下文容器，来获得一个对象。

#### 首先定义一个接口及其实现类
```java
public interface MessageService {
    String getMsg();
}

public class MessageServiceImpl implements MessageService {
    public String getMsg() {
        return "Hello Spring!";
    }
}
```

#### 然后是配置文件`application.xml`
在项目目录的`resources`下新建一个`application.xml`，内容如下：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd" default-autowire="byName">
    <!--这句是重点-->
    <bean id="messageService" class="com.w.springlearning.example.MessageServiceImpl"/>
</beans>
```

#### 然后在程序中使用上下文来获取对象实例并调用其接口
```java
public class Demo {
    public static void main(String[] args) {
        ApplicationContext context = 
            new ClassPathXmlApplicationContext("classpath:application.xml");
        MessageService messageService = context.getBean(MessageService.class);
        System.out.println(messageService.getMsg());
    }
}
```

例子很简单，原理就复杂了，下面将分析Spring是如何通过配置文件启动`ApplicationContext`，并在启动过程中创建`Bean`，注入依赖等。

### `BeanFactory`简介
在上文对`ApplicationContext`的介绍中，我们知道`ApplicationContext`继承了`ListableBeanFactory`接口，使其拥有了**Bean工厂**的能力，换句话说，`ApplicationContext`就是一个**BeanFactory**。

先来看一下`BeanFactory`的继承关系：
![BeanFactory继承关系](https://www.javadoop.com/blogimages/spring-context/2.png)
1. 顶层的`BeanFactory`都是一些基本的、获取**单个**Bean的方法；
2. 第二层的`ListableBeanFactory`，主要提供了一些获取**多个**Bean的方法；`HierarchicalBeanFactory`，顾名思义，提供了将多个BeanFactory设为“父子”关系的方法；`AutowireCapableBeanFactory`，提供了自动装配Bean的能力；
3. `ApplicationContext`，我们的主角，继承了第二层的中的两个，没继承的那个，也通过方法`getAutowireCapableBeanFactory()`持有了；
4. `ConfigurableListableBeanFactory `继承了第二层中的所有接口，也是个特殊的接口，先提一嘴；
5. 其它的类就不一一提了。

## 启动过程分析
先从`ClassPathXmlApplicationContext`的构造方法说起：
```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    @Nullable
    private Resource[] configResources;
    
    // 已经有ApplicationContext，要设置父子关系，用这个方法
    public ClassPathXmlApplicationContext(ApplicationContext parent) {
        super(parent);
    }
	
    // 全参数的构造方法
    public ClassPathXmlApplicationContext(
            String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
            throws BeansException {
        super(parent);
        
        // 根据提供的路径，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
        setConfigLocations(configLocations);
        if (refresh) {
            refresh(); // 核心方法
        }
    }
}	
```
其中核心方法是`refresh()`，之所以叫refresh而不是init之类的，是因为在ApplicationContext建立起来之后，
是可以通过该方法重建的：**销毁原有的，建立新的**。

### `refresh()`
> AbstractApplicationContext.java
```java
public void refresh() throws BeansException, IllegalStateException {
    // 加锁，保证同时只能进行一个refresh
    // startupShutdownMonitor 这个对象就是为refresh和destory操作加锁存在的
    synchronized (this.startupShutdownMonitor) {
	    
        // 准备工作，记录下容器的启动时间、标记“已启动”状态、处理配置文件中的占位符
        prepareRefresh();
        
        // 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
        // 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
        // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
        // 注意，这里的beanFactory就是上面曾提了一嘴的继承了2层三个接口的那个。        
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        
        // 设置 BeanFactory 的类加载器，添加几个 BeanPostProcessor，手动注册几个特殊的 bean
        prepareBeanFactory(beanFactory);
        
        try {
            // 【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
            // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。】
        
            // 这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
            // 具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事 subclasses.
            postProcessBeanFactory(beanFactory);
        
            // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
            invokeBeanFactoryPostProcessors(beanFactory);
        
            // 注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
            // 此接口两个方法: postProcessBeforeInitialization 和 postProcessAfterInitialization
            // 两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化
            registerBeanPostProcessors(beanFactory);
        
            // 初始化当前 ApplicationContext 的 MessageSource，国际化这里就不展开说了，不然没完没了了
            initMessageSource();
        
            // 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
            initApplicationEventMulticaster();
        
            // 从方法名就可以知道，典型的模板方法(钩子方法)，
            // 具体的子类可以在这里初始化一些特殊的 Bean（在初始化 singleton beans 之前）
            onRefresh();
        
            // 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
            registerListeners();
        
            // 重点，重点，重点
            // 初始化所有的 singleton beans
            //（lazy-init 的除外）
            finishBeanFactoryInitialization(beanFactory);
        
            // 最后，广播事件，ApplicationContext 初始化完成
            finishRefresh();
        }
        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }
        
            // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
            destroyBeans();
        
            // Reset 'active' flag.
            cancelRefresh(ex);
        
            throw ex;
        }
        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

#### 创建容器前的准备工作：`prepareRefresh()`
> AbstractApplicationContext.java
```java
protected void prepareRefresh() {
	// 记录启动时间.
	this.startupDate = System.currentTimeMillis();
	// 设置两个标识，两个都是 AtomBoolean类型的
	this.closed.set(false);
	this.active.set(true);

	if (logger.isDebugEnabled()) {
		if (logger.isTraceEnabled()) {
			logger.trace("Refreshing " + this);
		}
		else {
			logger.debug("Refreshing " + getDisplayName());
		}
	}

	// Initialize any placeholder property sources in the context environment.
	initPropertySources();

	// 校验 XML 配置文件
	getEnvironment().validateRequiredProperties();

	// Store pre-refresh ApplicationListeners...
	if (this.earlyApplicationListeners == null) {
		this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
	}
	else {
		// Reset local application listeners to pre-refresh state.
		this.applicationListeners.clear();
		this.applicationListeners.addAll(this.earlyApplicationListeners);
	}

	// Allow for the collection of early ApplicationEvents,
	// to be published once the multicaster is available...
	this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

### 注册Bean到工厂中：`obtainFreshBeanFactory()`
> AbstractApplicationContext.java
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 关闭旧的BeanFactory（若有）
    // 创建新的BeanFactory，并加载Bean定义且注册等
    refreshBeanFactory();
    return getBeanFactory();
}
```
> AbstractRefreshableApplicationContext.java
```java
protected final void refreshBeanFactory() throws BeansException {
    // 如果 ApplicationContext 中已经加载过 BeanFactory 了，销毁所有 Bean，关闭 BeanFactory
    // 注意，应用中 BeanFactory 本来就是可以多个的，这里可不是说应用全局是否有 BeanFactory，而是当前
    // ApplicationContext 是否有 BeanFactory
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    try {
        // 初始化一个 DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 用于 BeanFactory 的序列化，大部分情况下用不到吧
        beanFactory.setSerializationId(getId());
        
        // 设置 BeanFactory 的两个配置属性：
        // 是否允许 Bean 覆盖、是否允许循环引用
        customizeBeanFactory(beanFactory);
        
        // 加载 Bean 到 BeanFactory 中
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
    }
}
```
到这里其实可以看到，虽然前面提到说 `ApplicationContext`继承了两个 `BeanFactory`接口，
但是其本身不实现相应的方法，而是通过（委托）内部持有的 `DefaultListableBeanFactory`来完成BeanFactory相关的操作。

为啥是 `DefaultListableBeanFactory`？此处再看一下BeanFactory的继承树：
![BeanFactory继承关系](https://www.javadoop.com/blogimages/spring-context/2.png)
`DefaultListableBeanFactory`在图中最下方，实现了可见的所有接口，功能最全。
> 如果你想要在程序运行的时候动态往 Spring IOC 容器注册新的 bean，就会使用到这个类。那我们怎么在运行时获得这个实例呢？
之前我们说过 ApplicationContext 接口能获取到 AutowireCapableBeanFactory，就是最右上角那个，然后它向下转型就能得到 DefaultListableBeanFactory 了。

在进一步向下之前，我们需要了解一下什么是`BeanDefinition`。上文提到，将Bean注册到BeanFactory，
可以简单理解为是一个完成 BeanName 到 BeanDefinition 的Map过程，这里的 BeanDefinition 保存了 Bean 的信息，
比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等等。
简单来说，Bean和BeanDefinition 类似于 实例和类的关系吧。
