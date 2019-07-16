### 线程池总览
先看一下线程池的几个“直系”关系：

![线程池直系关系图](https://wzt-img.oss-cn-chengdu.aliyuncs.com/ThreadPoolExecutor-Simple.png)

- `Executor`位于最顶层，也是最简单的，只有一个`executor(Runnable)`方法;
- `ExecutorService`继承了`Executor`接口，在其基础上增加了诸如`shutdown()`的管理方法；
- `AbstractExecutorService`则是实现了`ExecutorService`的抽象类，实现了一些有用的方法供子类直接调用；
- `ThreadPoolExecutor`是直接可以拿来用的类，提供了关于线程池的丰富的功能。

此外，和线程池有关的还有以下这些类：
![线程池沾亲带故的类](https://www.javadoop.com/blogimages/java-thread-pool/others.png)

- `Executors`，一见类名中的‘s’，就知道是个工具类了，可以方便地创建一些线程池；
- `Future`这边的类，则是用来支持线程执行返回结果的，在线程池中，每个任务都是包装成`FutureTask`给线程执行的；
- `BlockingQueue`是线程池中的等待队列，其具体作用在下文源码分析中会讲到。

### Executor 接口
```java
public interface Executor {
    void execute(Runnable command);
}
```
简单到无需多言。
不过还是根据其注释唠一唠这个接口的设计思想。

有了线程池后我们不需要再手动去new一个Thread去执行任务，而是向线程池提交即可，就像这样：
```java
Executor executor = anExecutor;
executor.execute(new RunnableTask());
```
正常情况下，每个任务进来就会有一个线程去执行，就像这样：
```java
class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();  // 每个任务都用一个新的线程来执行
    }
}
```
但若希望线程池串行执行任务，也可以这么来（谁又会这么搞呢？）：
```java
class DirectExecutor implements Executor {
    public void execute(Runnable r) {
        r.run();// 这里不是用的new Thread(r).start()，也就是说没有启动任何一个新的线程。
    }
}
```
总的来说，`Executor`这个接口只能提交任务。如果想要更丰富的功能，如想知道执行结果、管理线程池等，
就需要继承了这个接口的 `ExecutorService`了。

### ExecutorService 接口
