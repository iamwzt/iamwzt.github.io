本文主要分析 Spring AOP 是如何为目标 bean 筛选出合适的通知器（Advisor）。

Spring 是如何将 AOP 和 IOC 模块整合到一起的？通过拓展点 `BeanPostProcessor` 接口。

Spring AOP 抽象代理创建器实现了 `BeanPostProcessor` 接口，并在 bean 初始化后置处理过程中向 bean 中织入通知。
下面我们就来看看相关源码：

### AOP入口方法
```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
    @Override
    public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                // 若有需要，为 bean 生成代理对象
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }
    
    // 若有需要，为 bean 生成代理对象
    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
            return bean;
        }
        if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
            return bean;
        }
        // 如果是基础设施类（Pointcut、Advice、Advisor 等接口的实现类），或是应该跳过的类
        // 则不应该生成代理，此时直接返回 bean
        if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            // 将 <cacheKey, FALSE> 键值对放入缓存中，供上面的 if 分支使用
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return bean;
        }
    
        // 为目标 bean 查找合适的通知器
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
        // 若 specificInterceptors 数组不为 null ，则为 bean 生成代理
        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // 创建代理
            Object proxy = createProxy(
                    bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
            this.proxyTypes.put(cacheKey, proxy.getClass());
            // 返回代理对象，此时 IOC 容器输入 bean，得到 proxy。此时，
            // beanName 对应的 bean 是代理对象，而非原始的 bean
            return proxy;
        }
    
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        // specificInterceptors == null，直接返回 bean
        return bean;
    }
}
```
以上就是 Spring AOP 创建代理对象的入口方法分析，过程比较简单，这里简单总结一下：
1. 若 bean 是 AOP 基础设施类型，则直接返回
2. **为 bean 筛选合适的通知器**
3. **如果通知器数组不为空，则为 bean 生成代理对象，并返回该对象**
4. 若数组为空，则返回原始 bean

### 筛选合适的通知器
在向目标 bean 中织入通知之前，我们先要为 bean 筛选出合适的通知器（通知器持有通知）。

如何筛选呢？方式有很多，比如我们可以通过正则表达式匹配方法名，当然更多的时候用的是 AspectJ 表达式进行匹配。
那下面我们就来看一下使用 AspectJ 表达式筛选通知器的过程。
```java
protected Object[] getAdvicesAndAdvisorsForBean(
        Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```
```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    // 获取所有的通知器列表
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    
    // 筛选可应用在 beanClass 上的 Advisor，通过 ClassFilter 和 MethodMatcher
    // 对目标类和方法进行匹配
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    // 拓展操作
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```

### 创建代理对象
在筛选到合适的通知器后，接着就是创建代理对象了。
```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {
    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }
    
    // 创建 ProxyFactory 实例
    ProxyFactory proxyFactory = new ProxyFactory();
    proxyFactory.copyFrom(this);
    
    // 配置文件可配置此参数：proxy-target-class
    // true，则不管有没有接口，都是用CGLib来生成代理类
    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            // 有接口则往代理工厂里添加所有的接口
            // 否则设置 ProxyTargetClass 为 true，即还是用cglib来代理
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }
    
    // 这个方法会返回匹配了当前 bean 的 advisors 数组
    // 注意：如果 specificInterceptors 中有 advice 和 interceptor，它们也会被包装成 advisor
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);
    
    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    
    return proxyFactory.getProxy(getProxyClassLoader());
}
```
我们看到，这个方法主要是在内部创建了一个 ProxyFactory 的实例，然后 set 了一大堆内容。
剩下的工作就都是这个 ProxyFactory 实例的了，通过这个实例来创建代理: `getProxy(classLoader)`。

转换战场。
#### `ProxyFactory`详解
> ProxyFactory#getProxy

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```
> ProxyCreatorSupport#createAopProxy

```java
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
	return getAopProxyFactory().createAopProxy(this);
}
```
在创建AopProxy之前需要一个AopProxy工厂
```java
public ProxyCreatorSupport() {
    this.aopProxyFactory = new DefaultAopProxyFactory();
}
```
来看看 `DefaultAopProxyFactory`的`createAopProxy`方法是个什么鬼。
```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // (我也没用过这个optimize，默认false) || (proxy-target-class) || (没有接口)
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        // 如果要代理的类本身就是接口，也会用 JDK 动态代理
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        // 如果有接口，会跑到这个分支
        return new JdkDynamicAopProxy(config);
    }
}
```
到这里，我们知道 `createAopProxy` 方法有可能返回 `JdkDynamicAopProxy` 实例，也有可能返回 `ObjenesisCglibAopProxy` 实例，这里总结一下：
- 如果设置了 proxy-target-class="true"，那么都会使用 CGLIB;
- 如果被代理的目标类实现了一个或多个自定义的接口，那么就会使用 JDK 动态代理，否则会使用 CGLIB 实现代理。

**JDK 动态代理**基于接口，所以只有接口中的方法会被增强；而 **CGLIB** 基于类继承，需要注意就是如果方法使用了 final 修饰，或者是 private 方法，是不能被增强的。

有了 AopProxy 实例以后，我们就回到ProxyFactory的getProxy了：
```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```
来分别看一下**动态代理**和**CGLIB**的 `getProxy`方法：
