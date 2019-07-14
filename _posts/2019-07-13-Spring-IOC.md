## 开始之前
> 源码版本信息
> - spring-context: 5.1.5.RELEASE

节省篇幅起见，删掉了源码中打印日志相关的代码。

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

#### 注册Bean到工厂中：`obtainFreshBeanFactory()`
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
但是其本身不实现相应的方法，而是通过**（委托）内部持有的 `DefaultListableBeanFactory`来完成BeanFactory相关的操作**。

为啥是 `DefaultListableBeanFactory`？此处再看一下BeanFactory的继承树：
![BeanFactory继承关系](https://www.javadoop.com/blogimages/spring-context/2.png)
`DefaultListableBeanFactory`在图中最下方，实现了可见的所有接口，功能最全。
> 如果你想要在程序运行的时候动态往 Spring IOC 容器注册新的 bean，就会使用到这个类。那我们怎么在运行时获得这个实例呢？
之前我们说过 ApplicationContext 接口能获取到 AutowireCapableBeanFactory，就是最右上角那个，然后它向下转型就能得到 DefaultListableBeanFactory 了。

在进一步向下之前，我们需要了解一下什么是`BeanDefinition`。上文提到，将Bean注册到BeanFactory，
可以简单理解为是一个完成 BeanName 到 BeanDefinition 的Map过程，这里的 BeanDefinition 保存了 Bean 的信息，
比如这个 Bean 指向的是哪个类、是否是单例的、是否懒加载、这个 Bean 依赖了哪些 Bean 等等。

简单来说，Bean和BeanDefinition 类似于 实例和类的关系吧。

##### `BeanDefinition`定义
```java
// 节省篇幅，省略所有set方法对应的get方法
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {
    // Spring 默认支持的两种模式：SINGLETON、PROTOTYPE
    // 还有 request, session, globalSession, application, websocket 这几种
    // 属于web 的扩展
    String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;
    String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
    
    // 暂且跳过
    int ROLE_APPLICATION = 0;
    int ROLE_SUPPORT = 1;
    int ROLE_INFRASTRUCTURE = 2;
    
    // 父Bean名称，涉及Bean继承（非Java的继承，仅继承配置信息）
    void setParentName(@Nullable String parentName);
    
    // Bean的类名称
    void setBeanClassName(@Nullable String beanClassName);
    
    // Bean的scope
    void setScope(@Nullable String scope);
    
    // 懒加载
    void setLazyInit(boolean lazyInit);
    
    // 依赖
    void setDependsOn(@Nullable String... dependsOn);
    
    // 是否可注入其他bean，只对根据类型注入有效
    void setAutowireCandidate(boolean autowireCandidate);
    
    // 优先注入，若有多个相同类型的bean，spring优先选择该值为true的
    void setPrimary(boolean primary);
    
    // 生成该Bean的工厂名称
    void setFactoryBeanName(@Nullable String factoryBeanName);
    // 生成该Bean的工厂方法名称
    void setFactoryMethodName(@Nullable String factoryMethodName);
    
    // Bean的构造方法参数
    ConstructorArgumentValues getConstructorArgumentValues();
    // Bean的属性值
    MutablePropertyValues getPropertyValues();
    
    // Bean的初始化方法
    void setInitMethodName(@Nullable String initMethodName);
    // Bean的销毁方法
    void setDestroyMethodName(@Nullable String destroyMethodName);
    
    // 上面跳过的三个Role
    void setRole(int role);
    
    // 人可读的描述信息
    void setDescription(@Nullable String description);
    
    boolean isSingleton();
    boolean isPrototype();
    // 不可被实例化，只可作为父bean
    boolean isAbstract();
    @Nullable
    String getResourceDescription();
    @Nullable
    BeanDefinition getOriginatingBeanDefinition();    
```
众多依赖信息，先求个眼熟。但是可以看到，里头并没有`getInstance`之类的方法来获取实例，这点后面会说。

介绍完 `BeanDefinition`，再回到之前的`refreshBeanFactory`方法。
在有一个BeanFactory之后，还有两步要走：
```java
    // 设置 BeanFactory 的两个配置属性：
    // 是否允许 Bean 覆盖、是否允许循环引用
    customizeBeanFactory(beanFactory);
    // 加载 Bean 到 BeanFactory 中
    loadBeanDefinitions(beanFactory);
```
先看第一个：
##### `customizeBeanFactory`
> 这个比较简单，就是设置两个Boolean值
```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    // 是否允许Bean的定义覆盖
    // 默认同一文件覆盖会报错，不同文件可以覆盖
    if (this.allowBeanDefinitionOverriding != null) {
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    // 是否允许循环引用
    if (this.allowCircularReferences != null) {
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```

##### 重头戏：`loadBeanDefinitions`
> AbstractXmlApplicationContext.java
```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 通过一个 XmlBeanDefinitionReader 来加载Bean.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // Configure the bean definition reader with this context's
    // resource loading environment.
    beanDefinitionReader.setEnvironment(this.getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // 提供给子类覆写来实现自定义的初始化（貌似没有子类用）
    initBeanDefinitionReader(beanDefinitionReader);
    // 重点来了
    loadBeanDefinitions(beanDefinitionReader);
}


// 通过Reader加载XML文件
// 两个分支殊途同归，最终都会根据Resource来加载
// 因此我们就跟着第一个分支走
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
    Resource[] configResources = getConfigResources();
    if (configResources != null) {
        reader.loadBeanDefinitions(configResources);
    }
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        reader.loadBeanDefinitions(configLocations);
    }
}
```
> AbstractBeanDefinitionReader.java
```java
@Override
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int count = 0;
    // 每个文件是一个Resource，循环去加载
    for (Resource resource : resources) {
        count += loadBeanDefinitions(resource);
    }
    // 返回一共加载的BeanDefinition数量
    return count;
}
```
> XmlBeanDefinitionReader.java
```java
@Override
public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    
    // private final ThreadLocal<Set<EncodedResource>> resourcesCurrentlyBeingLoaded；
    // 用一个ThreadLocal来存放文件资源
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
                "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    try {
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            
            // 关键方法
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
                "IOException parsing XML document from " + encodedResource.getResource(), ex);
    }
    finally {
        currentResources.remove(encodedResource);
        if (currentResources.isEmpty()) {
            this.resourcesCurrentlyBeingLoaded.remove();
        }
    }
}


protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
        throws BeanDefinitionStoreException {
    try {
        // 将 xml 文件转换为 Document 对象
        Document doc = doLoadDocument(inputSource, resource);
        // 关键方法：注册BeanDefinition
        int count = registerBeanDefinitions(doc, resource);
        return count;
    }
    // 各种异常处理
    catch (...) {
        ...
    }
}

public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 关键方法，继续向下
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    // 返回从当前文件加载了多少BeanDefinition
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
> DefaultBeanDefinitionDocumentReader.java
```java
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    // 从 xml 根节点开始解析文件
    doRegisterBeanDefinitions(doc.getDocumentElement());
}

@SuppressWarnings("deprecation")  // for Environment.acceptsProfiles(String...)
protected void doRegisterBeanDefinitions(Element root) {
    // 我们看名字就知道，BeanDefinitionParserDelegate 必定是一个重要的类，它负责解析 Bean 定义，
    // 这里为什么要定义一个 parent? 看到后面就知道了，是递归问题，
    // 因为 <beans /> 内部是可以定义 <beans /> 的，所以这个方法的 root 其实不一定就是 xml 的根节点，也可以是嵌套在里面的 <beans /> 节点，
    // 从源码分析的角度，我们当做根节点就好了
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);

    if (this.delegate.isDefaultNamespace(root)) {
        // 这块说的是根节点 <beans ... profile="dev" /> 中的 profile 是否是当前环境需要的，
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                    profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);

            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                // 如果当前环境配置的 profile 不包含此 profile，那就直接 return 了，不对此 <beans /> 解析
                return;
            }
        }
    }

    preProcessXml(root);
    // 继续 ↓
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);

    this.delegate = parent;
}

protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    // default namespace 涉及到的就四个标签 <import />、<alias />、<bean /> 和 <beans />
                    parseDefaultElement(ele, delegate);
                }
                else {
                    // 解析其他 namespace 的元素
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```
可以看到解析BeanDefinition分成两种：
1. Default Element: 即` xmlns="http://www.springframework.org/schema/beans"`下的四个标签 `<import />`、`<alias />`、`<bean />` 和 `<beans />`
2. Custom Element: 其他标签，如我们经常会使用到的 `<mvc />`、`<task />`、`<context />`、`<aop />`等。

如果要解析这些非Default的标签，就要在XML头部引入相应的namespace及.xsd的文件路径，如下所示，同时代码中需要提供相应的 parser 来解析，
如 MvcNamespaceHandler、TaskNamespaceHandler、ContextNamespaceHandler、AopNamespaceHandler 等。
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"       
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd"
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd
           http://www.springframework.org/schema/mvc   
           http://www.springframework.org/schema/mvc/spring-mvc.xsd  
       default-autowire="byName">
</beans>
```

回头再看看处理default标签的方法：
> DefaultBeanDefinitionDocumentReader.java
```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    // 下面重点讲一下解析<bean />标签的方法
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        processBeanDefinition(ele, delegate);
    }
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        // recurse
        doRegisterBeanDefinitions(ele);
    }
}

protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 将 <bean /> 节点中的信息提取出来，然后封装到一个 BeanDefinitionHolder 中，细节往下看
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 解析自定义属性
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 注册Bean
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // 发送注册事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```
在解析`<bean />`前，回顾一下Bean都有哪些属性
1. class
2. name：可指定 id、name(用逗号、分号、空格分隔)
3. scope
4. constructor-arg
5. properties
6. autowiring mode：no(默认值)、byName、byType、 constructor
7. lazy-initialization mode：是否懒加载(如果被非懒加载的bean依赖了那么其实也就不能懒加载了)
8. initialization method：bean 属性设置完成后，会调用这个方法
9. destruction method： 	bean 销毁后的回调方法

这些属性其实就是与`BeanDefinition`这个类里出现过的get/set方法对应的。例子如下：
```java
<bean id="exampleBean" name="name1, name2, name3" class="com.javadoop.ExampleBean"
      scope="singleton" lazy-init="true" init-method="init" destroy-method="cleanup">

    <!-- 可以用下面三种形式指定构造参数 -->
    <constructor-arg type="int" value="7500000"/>
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg index="0" value="7500000"/>

    <!-- property 的几种情况 -->
    <property name="beanOne">
        <ref bean="anotherExampleBean"/>
    </property>
    <property name="beanTwo" ref="yetAnotherBean"/>
    <property name="integerProperty" value="1"/>
</bean>
```

回到解析`<bean />`的方法中：
> BeanDefinitionParserDelegate.java
```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}

public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
    String id = ele.getAttribute(ID_ATTRIBUTE);
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

    // 按 ",; " 三种符号来切分name属性值，将其全部放入别名列表
    List<String> aliases = new ArrayList<>();
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    // 将id作为bean名称，若没有，则将第一个别名作为bean名称
    String beanName = id;
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
    }

    // 校验bean名称和别名在当前层级的<bean />的唯一性
    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }

    // 根据<bean >...<bean />中的配置，生成一个BeanDefinition实例
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    if (beanDefinition != null) {
        // 若bean名称未被设置：↓
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                            beanDefinition, this.readerContext.getRegistry(), true);
                }
                else {
                    // beanName为“类全路径名#数字”
                    // beanClassName为“类全路径名”
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                            beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                            !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }
            }
            catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }
    return null;
}
```
来看看是怎么根据配置文件来生成BeanDefinition实例的
```java
public AbstractBeanDefinition parseBeanDefinitionElement(
        Element ele, String beanName, @Nullable BeanDefinition containingBean) {

    this.parseState.push(new BeanEntry(beanName));

    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    String parent = null;
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }

    try {
        // 创建 bd，然后设置类信息、父bean信息
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);
        // 设置 bd的属性，在AbstractBeanDefinition里
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
        
        // 下面是解析<bean />内各子元素的
        // <meta />
        parseMetaElements(ele, bd);
        // <lookup-method />
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        // <replaced-method />
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
        // <constructor-arg />
        parseConstructorArgElements(ele, bd);
        // <property />
        parsePropertyElements(ele, bd);
        // <qualifier />
        parseQualifierElements(ele, bd);

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));

        return bd;
    }
    catch (...) { ... }
    finally {
        this.parseState.pop();
    }
    return null;
}
```
到此，完成了一个`<bean />`到 `BeanDefinitionHolder`，回到解析`<bean />`的方法上：
> DefaultBeanDefinitionDocumentReader.java
```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 将 <bean /> 节点中的信息提取出来，然后封装到一个 BeanDefinitionHolder 中，细节往下看
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 解析自定义属性
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 用给定的bean工厂注册Bean
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // 发送注册事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

这个时候我们已经有了一个`BeanDefinitionHolder`，里头有：
```java
public class BeanDefinitionHolder implements BeanMetadataElement {
	private final BeanDefinition beanDefinition;
	private final String beanName;
	@Nullable
	private final String[] aliases;
	...
}
```
先来看看怎么注册Bean：
> BeanDefinitionReaderUtils.java
```java
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // 用bean名称注册Bean.
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // 注册bean别名
    // 保存alias -> beanName 的关系，根据alias找到beanName，再找到Bean
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```
> DefaultListableBeanFactory.java
```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    Assert.hasText(beanName, "Bean name must not be empty");
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");

    if (beanDefinition instanceof AbstractBeanDefinition) {
        try {
            ((AbstractBeanDefinition) beanDefinition).validate();
        }
        catch (BeanDefinitionValidationException ex) {
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                    "Validation of bean definition failed", ex);
        }
    }
    // 从beanDefinitionMap中取出叫该beanName的bd，此处涉及到bean覆盖
    BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
    if (existingDefinition != null) {
        // 不允许覆盖则抛异常
        if (!isAllowBeanDefinitionOverriding()) {
            throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
        }
        // 日志：旧bd角色小于新bd
        // ROLE_APPLICATION < ROLE_SUPPORT < ROLE_INFRASTRUCTURE
        else if (existingDefinition.getRole() < beanDefinition.getRole()) {...}
        // 日志：新（不同的）bd换旧bd
        else if (!beanDefinition.equals(existingDefinition)) {...}
        // 日志：同个bd更新
        else {...}
        this.beanDefinitionMap.put(beanName, beanDefinition);
    }
    else {
        // 判断是否已经有其他的 Bean 开始初始化了.
        // 注意，"注册Bean" 这个动作结束，Bean 依然还没有初始化，我们后面会有大篇幅说初始化过程，
        // 在 Spring 容器启动的最后，会 预初始化 所有的 singleton beans        
        if (hasBeanCreationStarted()) {
            // Cannot modify startup-time collection elements anymore (for stable iteration)
            synchronized (this.beanDefinitionMap) {
                this.beanDefinitionMap.put(beanName, beanDefinition);
                List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
                updatedDefinitions.addAll(this.beanDefinitionNames);
                updatedDefinitions.add(beanName);
                this.beanDefinitionNames = updatedDefinitions;
                if (this.manualSingletonNames.contains(beanName)) {
                    Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
                    updatedSingletons.remove(beanName);
                    this.manualSingletonNames = updatedSingletons;
                }
            }
        }
        // 正常情况下进这个分支：没覆盖，没初始化
        // 仍在spring启动注册阶段
        else {
            // 将 bd 放到这个 map 中，这个 map 保存了所有的 bd
            // private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
            // 默认为256的大小
            this.beanDefinitionMap.put(beanName, beanDefinition);
            // 按照注册顺序保存beanName
            // private volatile List<String> beanDefinitionNames = new ArrayList<>(256);
            this.beanDefinitionNames.add(beanName);
            // 这是个 LinkedHashSet，代表的是手动注册的 singleton bean，
            // 注意这里是 remove 方法，到这里的 Bean 当然不是手动注册的
            // 手动指的是通过调用以下方法注册的 bean ：
            //     registerSingleton(String beanName, Object singletonObject)
            // 这不是重点，解释只是为了不让大家疑惑。Spring 会在后面"手动"注册一些 Bean，
            // 如 "environment"、"systemProperties" 等 bean，我们自己也可以在运行时注册 Bean 到容器中的            
            this.manualSingletonNames.remove(beanName);
        }
        // 这个不重要，在预初始化的时候会用到
        this.frozenBeanDefinitionNames = null;
    }

    if (existingDefinition != null || containsSingleton(beanName)) {
        resetBeanDefinition(beanName);
    }
}
```
OK，到这里为止，已经注册了各bd，并发送了注册事件。下面我们来看看`refresh`的第3步：

#### 准备Bean容器`prepareBeanFactory`
> AbstractApplicationContext.java
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置 BeanFactory 的类加载器，BeanFactory 需要加载类，也就需要类加载器，
    // 这里设置为加载当前 ApplicationContext 类的类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 添加一个 BeanPostProcessor，这个 processor 比较简单：
    // 实现了 Aware 接口的 beans 在初始化的时候，这个 processor 负责回调，
    // 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
    // 注意：它不仅仅回调 ApplicationContextAware，
    // 还会负责回调 EnvironmentAware、ResourceLoaderAware 等，看下源码就清楚了
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    
    // 下面几行的意思就是，如果某个 bean 依赖于以下几个接口的实现类，在自动装配的时候忽略它们，
    // Spring 会通过其他方式来处理这些依赖。
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值，
    // 之前我们说过，"当前 ApplicationContext 持有一个 BeanFactory"，这里解释了第一行
    // ApplicationContext 还继承了 ResourceLoader、ApplicationEventPublisher、MessageSource
    // 所以对于这几个依赖，可以赋值为 this，注意 this 是一个 ApplicationContext
    // 那这里怎么没看到为 MessageSource 赋值呢？那是因为 MessageSource 被注册成为了一个普通的 bean
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 这个 BeanPostProcessor 也很简单，在 bean 实例化后，如果是 ApplicationListener 的子类，
    // 那么将其添加到 listener 列表中，可以理解成：注册 事件监听器
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 这里涉及到特殊的 bean，名为：loadTimeWeaver，这不是我们的重点，忽略它
    // tips: ltw 是 AspectJ 的概念，指的是在运行期进行织入，这个和 Spring AOP 不一样
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // spring默认注册一些的环境相关的bean：
    // environment、systemProperties、systemEnvironment
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

#### 初始化所有的 singleton beans：`finishBeanFactoryInitialization`
到目前为止，应该说 BeanFactory 已经创建完成，并且所有的实现了 `BeanFactoryPostProcessor` 接口的 Bean 都已经初始化，
并且其中的 `postProcessBeanFactory(factory)` 方法已经得到回调执行了。
而且 Spring 已经“手动”注册了一些特殊的 Bean，如 `environment`、`systemProperties` 等。

（这里我们跳过了`refresh`中的一大段，直接进入重点关注对象）
> AbstractApplicationContext.java
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // 初始化名为 conversionService 的bean
    // 该bean最有用的场景就是前端传来的值转化成后端Controller需要的参数
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
            beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
                // 这个是用来实例化的，下面看
                beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    // Register a default embedded value resolver if no bean post-processor
    // (such as a PropertyPlaceholderConfigurer bean) registered any before:
    // at this point, primarily for resolution in annotation attribute values.
    if (!beanFactory.hasEmbeddedValueResolver()) {
        beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
    }

    // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(null);

    // 没什么别的目的，因为到这一步的时候，Spring 已经开始预初始化 singleton beans 了，
    // 肯定不希望这个时候还出现 bean 定义解析、加载、注册。
    beanFactory.freezeConfiguration();

    // 实例化所有非懒加载的单例bean
    beanFactory.preInstantiateSingletons();
}
```
> DefaultListableBeanFactory.java
```java
@Override
public void preInstantiateSingletons() throws BeansException {
    // 获取所有的 BeanName
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // 触发所有的非懒加载的 singleton beans 的初始化操作
    for (String beanName : beanNames) {
        // 合并父 Bean 中的配置，就是 <bean id="" class="" parent="" /> 中的 parent
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 非抽象、非懒加载的单例Bean
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // FactoryBean适用于一些比较复杂的Bean，如数据库连接池的创建
            if (isFactoryBean(beanName)) {
                // 在bean名前加“&”再实例化
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                    final FactoryBean<?> factory = (FactoryBean<?>) bean;
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                        ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
            }
            // 普通的Bean，直接调用getBean()
            else {
                getBean(beanName);
            }
        }
    }

    // 到这里说明所有的非懒加载的 singleton beans 已经完成了初始化
    // 如果我们定义的 bean 是实现了 SmartInitializingSingleton 接口的，那么在这里得到回调，忽略
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```
接下来讲一下getBean()
> AbstractBeanFactory.java
```java
@Override
public Object getBean(String name) throws BeansException {
    return doGetBean(name, null, null, false);
}

// 该方法在bean已经初始化过了时直接从容器返回，否则先初始化再返回
@SuppressWarnings("unchecked")
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    // 获取一个 “正统的” beanName，处理两种情况，一个是前面说的 FactoryBean(前面带 ‘&’)，
    // 一个是别名问题，因为这个方法是 getBean，获取 Bean 用的，你要是传一个别名进来，是完全可以的
    final String beanName = transformedBeanName(name);
    
    Object bean;    // 这个是返回值

    // 检查是否已经创建过了
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 如果是FactoryBean，则返回它创建的那个对象
        // 否则直接返回sharedInstance
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    else {
        // 创建过了此 beanName 的 prototype 类型的 bean，那么抛异常，
        // 往往是因为陷入了循环引用
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 检查该bd在容器中是否存在
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // 若没有，在父容器中找找看
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                        nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }
        
        // 这里typeCheckOnly是false，将beanName放入Set集合alreadyCreated中
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }

        // 稍稍总结一下：
        // 到这里的话，要准备创建 Bean 了，对于 singleton 的 Bean 来说，容器中还没创建过此 Bean；
        // 对于 prototype 的 Bean 来说，本来就是要创建一个新的 Bean。
        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);

            // Guarantee initialization of beans that the current bean depends on.
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                    }
                    registerDependentBean(dep, beanName);
                    try {
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
                    }
                }
            }

            // Create bean instance.
            if (mbd.isSingleton()) {
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        // Explicitly remove instance from singleton cache: It might have been put there
                        // eagerly by the creation process, to allow for circular reference resolution.
                        // Also remove any beans that received a temporary reference to the bean.
                        destroySingleton(beanName);
                        throw ex;
                    }
                });
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }

            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                            "Scope '" + scopeName + "' is not active for the current thread; consider " +
                            "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                            ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            if (logger.isTraceEnabled()) {
                logger.trace("Failed to convert bean '" + name + "' to required type '" +
                        ClassUtils.getQualifiedName(requiredType) + "'", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```
