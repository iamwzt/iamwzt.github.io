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
```java
public interface ExecutorService extends Executor {
    // 关闭线程池，不再接受新任务，已提交的任务继续执行
    void shutdown();
    // 相比上一个，会尝试去停止正在执行的任务，并返回还没执行的任务
    List<Runnable> shutdownNow();
    // 是否已关闭
    boolean isShutdown();
    // 调用关闭线程池的方法后，所有任务结束，则返回true
    // 注意：必须在关闭线程池的两个方法后调用才可能返回true
    boolean isTerminated();
    // 阻塞，直到所有任务执行完，或超时，或当前线程被中断
    // 在超时前所有任务执行完返回true，否则false
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
    // 提交一个Callable任务
    <T> Future<T> submit(Callable<T> task);
    // 提交一个Runable任务，并将result作为Future的内容
    <T> Future<T> submit(Runnable task, T result);
    // 提交一个Runable任务
    Future<?> submit(Runnable task);
    // 执行所有任务
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    // 执行所有任务，带超时机制
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
    // 任一任务完成，则返回其结果
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    // 上个方法的超时机制版
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
可以看到已经有较丰富的方法了，经常用这个接口作为线程池的静态类型。

来看一个两段式的线程池关闭方式：
```java
void shutdownAndAwaitTermination(ExecutorService pool) {
  pool.shutdown(); // 停止接收新任务
  try {
    // 等待结束
    if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
      pool.shutdownNow(); // 若还未结束，强行结束
      // 继续等待
      if (!pool.awaitTermination(60, TimeUnit.SECONDS))
          System.err.println("Pool did not terminate");
    }
  } catch (InterruptedException ie) {
    // 线程被中断的处理
    pool.shutdownNow();
    // Preserve interrupt status
    Thread.currentThread().interrupt();
  }
}
```

### AbstractExecutorService 抽象类
AbstractExecutorService 实现了 ExecutorService 中的`submit`/`invokeAll`/`invokeAny`方法。

先来看一下几个`submit`方法：
```java
// 将Runnable 参数和 T泛型类封装成 FutureTask
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
// 将Callable 参数封装成 FutureTask
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}

public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}

public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```
可以看到，几个submit方法均是把参数给封装成 `FutureTask`任务类后，再执行`execute`方法。

这里讲到了`FutureTask`就顺便提一下是如何封装的：
```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```
其中state参数是`volitle`类型的。
当传`Runable`时，需要通过`Executors`的工具方法将其转化成`Callable`类型，如下：
```java
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}

static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```
显然，适配器模式。

// TODO: 几个invoke的方法以后再讲

### ThreadPoolExecutor 类

#### 构造方法
`ThreadPoolExecutor` 是JDK中的线程池实现，可以说是最重要的类了。
在初学线程池的时候，可能都用过`Executors`的几个静态工厂方法来创建线程池，如：
```java
Executors.newFixedThreadPool(int);
Executors.newCachedThreadPool();
Executors.newSingleThreadExecutor();
...
```
这些方法最终都是调用了 `ThreadPoolExecutor`类的构造方法：
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
在这个构造方法中，出现了以下7个参数：
1. corePoolSize：核心线程数
2. maximumPoolSize：最大线程数
3. keepAliveTime：空闲线程存活时间，当线程超过这个时间都没有任务，将会被注销，默认对非核心线程起作用
4. unit：存活时间的单位
5. workQueue：任务队列，核心线程都忙时，新提交的线程放在这里
6. threadFactory：线程工厂，用于生成新线程
7. handler：拒绝策略，当所有线程都忙，队列也满时，对新提交任务的处理策略

除了上述7个参数外，还有几个重要参数，也在这一并罗列：
#### 状态 + 线程数 = ctl
`ThreadPoolExecutor`用一个32位数来同时表示线程池的状态及线程数量：
```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```
其高3位表示状态，低29位表示线程数（最多能达到5亿多吧）：
```java
private static final int COUNT_BITS = Integer.SIZE - 3;
// 000 11111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// 111 00000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
// 000 00000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 001 00000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
// 010 00000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
// 011 00000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

// 将低29位置0，便是状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
// 将高3位置0，便是线程数
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }

// 几个状态的比较的方法，无需关注低29位
private static boolean runStateLessThan(int c, int s) {
    return c < s;
}
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```
这里讲到了线程池的几个状态：
1. RUNNING：运行状态，此时能接收并处理任务，
2. SHUTDOWN：不接收新任务，但会执行等待队列里的任务
3. STOP：不接收新任务，不执行等待队列的任务，中断正在执行的任务
4. TIDYING：任务执行完，线程数为0，将执行钩子方法terminated()
5. TERMINATED：terminated()执行完毕后的状态

几种状态的转换：
- RUNNING -> SHUTDOWN：调用了`shutdown()`方法
- (RUNNING or SHUTDOWN) -> STOP：调用了 `shutdownNow()`
- SHUTDOWN -> TIDYING：等待队列及线程池为空
- STOP -> TIDYING：线程池为空
- TIDYING -> TERMINATED：钩子方法`terminated()`执行完毕

