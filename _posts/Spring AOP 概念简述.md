转自 [Spring AOP 源码分析系列文章导读](https://www.cnblogs.com/nullllun/p/9197043.html)

所谓AOP的原理，简单来说就是通过**代理模式**为目标对象生成代理对象，并将横切逻辑加到目标方法的前后。
原理说起来很简单，具体到Spring里实际实现上，可就没那么简单了。

本文就AOP中的几个概念，看看Spring中相应的代码实现。

### 连接点 JoinPoint
**连接点**指的是程序执行过程中的一些点，如方法调用、异常处理等。在Spring中，仅支持方法级别的连接点，即每个方法调用都是一个连接点。

具体相关的接口如下：
```java
public interface Joinpoint {
    // 执行拦截器链的下一个拦截器逻辑
    Object proceed() throws Throwable;
    // 获取使用这个连接点的对象（Spring中就是实际方法调用的对象实例）
    Object getThis();
    
    AccessibleObject getStaticPart();
}
```
这个 `Joinpoint` 接口中，`proceed` 方法是核心，该方法用于执行拦截器逻辑。
关于拦截器这里简单说一下吧，以前置通知拦截器为例。
在执行目标方法前，该拦截器首先会执行前置通知逻辑，如果拦截器链中还有其他的拦截器，则继续调用下一个拦截器逻辑。
直到拦截器链中没有其他的拦截器后，再去调用目标方法。

`Joinpoint` 接口的继承关系很简单：
（这里有个图）

从图中可以再次看到，Spring中的连接点只有**方法调用-Invocation**

### 切点 Pointcut
**切点**是用来选择连接点的。通过代码来看看怎么过滤：
```java
public interface Pointcut {
    // 返回类过滤器
    ClassFilter getClassFilter();
    // 返回方法过滤器
    MethodMatcher getMethodMatcher();
    
    Pointcut TRUE = TruePointcut.INSTANCE;
}
```
再看看**类过滤器**和**方法过滤器**。
```java
public interface ClassFilter {
    boolean matches(Class<?> clazz);
    ClassFilter TRUE = TrueClassFilter.INSTANCE;
}
```
```java
public interface MethodMatcher {
	boolean matches(Method method, Class<?> targetClass);
	boolean isRuntime();
	boolean matches(Method method, Class<?> targetClass, Object... args);
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
```
上面的两个接口均定义了 `matches` 方法，用户只要实现该方法，即可对连接点进行选择。
在日常使用中，大家通常是用 `AspectJ` 表达式对连接点进行选择。
Spring 中提供了一个 AspectJ 表达式切点类 - `AspectJExpressionPointcut`，下面我们来看一下这个类的继承体系图：
(这里有图)

如上所示，这个类最终实现了 `Pointcut`、`ClassFilter` 和 `MethodMatcher` 接口，因此该类具备了通过 AspectJ 表达式对连接点进行选择的能力。
那下面我们不妨写一个表达式对上一节的连接点进行选择，比如下面这个表达式：
```
execution(* *.find*(..))
```
该表达式用于选择以 find 的开头的方法。

### 通知 Advice
**通知**即我们定义的横切逻辑。在Spring中，根据调用横切逻辑的时机，定义了以下几种类型的通知：
1. 前置通知（Before advice）- 在目标方法调用前执行通知
2. 后置通知（After advice）- 在目标方法完成后执行通知
3. 返回通知（After returning advice）- 在目标方法执行成功后，调用通知
4. 异常通知（After throwing advice）- 在目标方法抛出异常后，执行通知
5. 环绕通知（Around advice）- 在目标方法调用前后均可执行自定义逻辑

在Spring中，通知的接口长这样：
```java
public interface Advice {
}
```
嗯。。。是个空的。它的所有方法定义都在子接口中，来看看继承关系：
（此处有图）

### 切面 Aspect
**切面**整合了切点和通知两个模块，切点解决了 `where` 问题，通知解决了 `when` 和 `how` 问题。
切面把两者整合起来，就可以解决 对什么方法（where）在何时（when - 前置还是后置，或者环绕）执行什么样的横切逻辑（how）的三连发问题。
在 Spring AOP 中，切面只是一个概念，并没有一个具体的接口或类与此对应。
不过 Spring 中倒是有一个接口的用途和切面很像，我们不妨了解一下，这个接口就是切点通知器 `PointcutAdvisor`。
我们先来看看这个接口的定义，如下：
```java
public interface Advisor {
	Advice EMPTY_ADVICE = new Advice() {};
	Advice getAdvice();
	boolean isPerInstance();
}
public interface PointcutAdvisor extends Advisor {
	Pointcut getPointcut();
}
```
`PointcutAdvisor`和它的父接口`Advisor`一个能获取切点，一个能获取通知，和切面的功能很像了。
但是切点和通知是一对多的关系，而`PointcutAdvisor`的实现是一对一的。

### 织入 Weaving
现在我们有了**连接点**、**切点**、**通知**，以及**切面**等，可谓万事俱备，但是还差了一股东风。
这股东风是什么呢？没错，就是**织入**。所谓织入就是在切点的引导下，将通知逻辑插入到方法调用上，使得我们的通知逻辑在方法调用时得以执行。
说完织入的概念，现在来说说 Spring 是通过何种方式将通知织入到目标方法上的。
先来说说以何种方式进行织入，这个方式就是通过实现后置处理器 `BeanPostProcessor` 接口。
该接口是 Spring 提供的一个拓展接口，通过实现该接口，用户可在 bean 初始化前后做一些自定义操作。
那 Spring 是在何时进行织入操作的呢？答案是在 bean 初始化完成后，即 bean 执行完初始化方法（init-method）。
Spring通过切点对 bean 类中的方法进行匹配。若匹配成功，则会为该 bean 生成代理对象，并将代理对象返回给容器。
容器向后置处理器输入 bean 对象，得到 bean 对象的代理，这样就完成了织入过程。

（完）
