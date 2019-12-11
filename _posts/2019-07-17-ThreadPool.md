---
layout:     post
title:      线程池源码解析
subtitle:   
date:       2019-07-18
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - ThreadPool
---
### 一、线程池总览
先看一下线程池的几个“直系”关系：

![线程池直系关系图](https://wzt-img.oss-cn-chengdu.aliyuncs.com/ThreadPoolExecutor-Simple.png)

- `Executor`位于最顶层，也是最简单的，只有一个`execute(Runnable)`方法;
- `ExecutorService`继承了`Executor`接口，在其基础上增加了诸如`shutdown()`的管理方法；
- `AbstractExecutorService`则是实现了`ExecutorService`的抽象类，实现了一些有用的方法供子类直接调用；
- `ThreadPoolExecutor`是直接可以拿来用的类，提供了关于线程池的丰富的功能。

此外，和线程池有关的还有以下这些类：
![线程池沾亲带故的类](https://www.javadoop.com/blogimages/java-thread-pool/others.png)

- `Executors`，一见类名中的‘s’，就知道是个工具类了，可以方便地创建一些线程池；
- `Future`这边的类，则是用来支持线程执行返回结果的，在线程池中，每个任务都是包装成`FutureTask`给线程执行的；
- `BlockingQueue`是线程池中的等待队列，其具体作用在下文源码分析中会讲到。

对相关的类有了一些概念后，下面便以线程池的几个直系类或接口，开始源码之旅。

### 二、Executor 接口
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

### 三、ExecutorService 接口
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

### 四、AbstractExecutorService 抽象类
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

### 五、ThreadPoolExecutor 类

#### 5.1 构造方法
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
1. `corePoolSize`：核心线程数
2. `maximumPoolSize`：最大线程数
3. `keepAliveTime`：空闲线程存活时间，当线程超过这个时间都没有任务，将会被注销，默认对非核心线程起作用
4. `unit`：存活时间的单位
5. `workQueue`：任务队列，核心线程都忙时，新提交的线程放在这里
6. `threadFactory`：线程工厂，用于生成新线程
7. `handler`：拒绝策略，当所有线程都忙，队列也满时，对新提交任务的处理策略

除了上述7个参数外，还有几个重要参数，也在这一并罗列：
#### 5.2 ctl: 状态 + 线程数
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
1. `RUNNING`：运行状态，此时能接收并处理任务，
2. `SHUTDOWN`：不接收新任务，但会执行等待队列里的任务
3. `STOP`：不接收新任务，不执行等待队列的任务，中断正在执行的任务
4. `TIDYING`：任务执行完，线程数为0，将执行钩子方法terminated()
5. `TERMINATED`：terminated()执行完毕后的状态

几种状态的转换：
- `RUNNING` -> `SHUTDOWN`：调用了`shutdown()`方法
- (`RUNNING` or `SHUTDOWN`) -> `STOP`：调用了 `shutdownNow()`
- `SHUTDOWN` -> `TIDYING`：等待队列及线程池为空
- `STOP` -> `TIDYING`：线程池为空
- `TIDYING` -> `TERMINATED`：钩子方法`terminated()`执行完毕

如图所示：

![线程池状态图](https://wzt-img.oss-cn-chengdu.aliyuncs.com/ThreadPoolStatus.png)

#### 5.3 Worker: 工作线程
在线程池内部，有一个重要的类，其名为 `Worker`，是用来执行任务的线程。
`Worker`实现了`Runnable`接口，还继承了`AbstractQueuedSynchronized`抽象类。可以看到在JUC包中AQS真是无处不在啊。
```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    // 该类永不会被序列化，有这个参数只是为了消除javac的warning
    private static final long serialVersionUID = 6138294804551838833L;

    // 线程在此
    final Thread thread;
    // 创建线程时若没有指定任务，就为null
    Runnable firstTask;
    // 线程完成的任务数
    volatile long completedTasks;

    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    // 调用了外部的方法
    public void run() {
        runWorker(this);
    }

    // 省略AQS相关的方法
    ...
}
```
#### 5.4 execute()
有了上面的铺垫，接下来我们来看一下线程池的核心方法`execute()`。
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();
    
    // 情况一：线程数 < 核心线程数
    // 创建一个新的线程（Worker），并将当前任务作为firstWork
    if (workerCountOf(c) < corePoolSize) {
        // 如果添加成功，就返回了
        // 添加失败说明线程池不允许提交任务了
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // 情况二：
    // 1、线程数 > 核心线程数；
    // （或）2、线程池不允许提交线程
    
    // 如果线程池处于RUNNING，那就将任务放入任务队列中
    if (isRunning(c) && workQueue.offer(command)) {
        // 任务提交成功
        int recheck = ctl.get();
        // 再检查线程池是否还是RUNNING，不是就从队列中移除该任务，并执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 若工作线程数为0，则创建新线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    // 情况三：加入队列失败
    // 说明线程池已经SHUTDOWN，或者队列已经饱和，执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

```java
// 可能由于线程池STOP、SHUTDOWN，或者线程工厂创建线程失败，或者OOM异常，导致失败返回false
// firstTask可以为null
// core为true，则以corePoolSize作为线程数上限，否则以maximumPoolSize为线程数上限
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 两种情况会创建新的worker：
        // 1. 线程池处于RUNNING状态；
        // 2. 线程池为SHUTDOWN，firstTask为null，且任务队列不为空
        // 第1个好理解，第2个可以从SHUTDOWN这个状态的语义去理解：
        // 即不允许提交任务，但是还会执行队列中的任务
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            // 线程数大于物理的上限，或者大于设定的上限，就返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 线程数CAS+1，若成功则跳出外层循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            // 若状态和开始时不一致，则重试
            if (runStateOf(c) != rs)
                continue retry;
            // 到这说明CAS失败了，在内层重试
        }
    }

    // worker是否启动
    boolean workerStarted = false;
    // worker是否已添加到workers这个HashSet中
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 持有这个线程池的全局锁，才能避免在下面的过程中，线程池被干掉了
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 若worker里的线程已经被启动，就抛出异常
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    // largestPoolSize 用来记录线程池中线程数的最大值
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            // 添加成功就启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 若启动失败就做一些清理动作
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
简单看下 `addWorkerFailed`
```java
// 1. 移除Worker
// 2. 有效线程数-1
// 3. 有worker线程移除，可能是最后一个线程退出需要尝试终止线程池
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        decrementWorkerCount();
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```
刚才在介绍Worker时说到，启动worker线程时会执行`runWorker()`方法：
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();

            // 若线程池状态大于等于STOP，则要中断该线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 钩子方法，留给需要的子类实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 钩子方法，留给需要的子类实现
                    afterExecute(task, thrown);
                }
            } finally {
                // task置空，准备获取下个任务
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        // 执行到这，说明是没有初始任务，队列中也没有任务了，正常退出
        completedAbruptly = false;
    } finally {
        // 根据completedAbruptly可以知道是正常退出或异常
        processWorkerExit(w, completedAbruptly);
    }
}
```
再来看一下获取任务的方法`getTask()`
```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 线程池状态为STOP、TIDYING、TERMINATED
        // 或为SHUTDOWN，且队列为空
        // 则将线程数-1，并返回null，让worker正常退出
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 若允许销毁核心线程，或线程数大于核心线程，则当前线程可以回收
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 线程数 > 最大线程数（重新设置，改小了最大线程数），或者超时了
        // 同时线程数 > 1 并且队列为空
        // 就要CAS减少线程数并返回null了，CAS失败就重来这一流程吧
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            // 若当前线程允许回收，则用带超时的取任务方式
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
总结一下，`getTask()`有三种情况会返回null
1. 线程池状态为STOP、TIDYING、TERMINATED，或为SHUTDOWN，且队列为空，则返回null；
2. 调小了最大线程数，则超出这个数值的线程将被关闭，要返回null；
3. 超时了，并且允许销毁当前线程（默认核心线程数内的线程不会被销毁），则返回null。

返回null后，会导致runWorker()方法中跳出while循环，然后调用`processWorkerExit`进行将worker移除。

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果runWorker()中是因为中断意外退出循环，则在这里需要处理worker数量
    // 正常退出的话，已经在getTask()里处理了
    if (completedAbruptly)
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 统计总任务处理数
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate();

    int c = ctl.get();
    // 若线程池还没停止（RUNNING或SHUTDOWN），还可以执行任务
    if (runStateLessThan(c, STOP)) {
        // 若worker并非意外退出
        if (!completedAbruptly) {
            // 计算线程最小数，若允许核心线程超时则为0，否则为核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            // 若还有任务，且允许核心线程超时，则最小数为1
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            // 若当前线程数不小于最小数，则直接返回
            // 否则需要再创一个worker来执行任务
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        addWorker(null, false);
    }
}
```
添加worker失败，会尝试terminate线程池：
```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 三种情况下会直接返回
        // 1. 还在RUNNING；
        // 2. TIDYING或TERMINATED，其它线程再终结；
        // 3. SHUTDOWN，但任务队列还不为空
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        
        // 线程数不为0，则中断1个空闲worker（获取任务时阻塞的）
        // 并借由此将其移除，同时扩散terminate信号
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 将状态设置为TIDYING状态
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 供用户实现
                    terminated();
                } finally {
                    // 调完terminated()就将状态设置为TERMINTAED状态了
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // CAS失败则重试
    }
}
```

#### 5.5 拒绝策略

在`ThreadPoolExecutor`中，已经有默认实现的四种拒绝策略了：
```java
// 抛异常
public static class AbortPolicy implements RejectedExecutionHandler {
    public AbortPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}

// 忽略
public static class DiscardPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}

// 丢弃最早的任务
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public DiscardOldestPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}

// 提交任务的线程自己执行
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public CallerRunsPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```
其中提交任务的线程自己执行这一策略可以实现一定程度的系统平滑降级，why？

- 当线程池已经饱和，无能力继续向其提交新任务时，由提交任务的线程（T1）自己执行；
- 在执行过程中，T1无法产生新的任务，线程池也可以在此期间减轻负荷量；
- 若仍有源源不断的新的TCP请求到来，将会阻塞在TCP的缓冲区；
- 若TCP的缓冲区满，客户端才会收到报错信息

#### 5.6 关闭线程池
```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 若有安全管理器，则需要校验是否有权限
        checkShutdownAccess();
        // 将线程池状态设置为SHUTDOWN状态，若状态已经更大，则不用管
        advanceRunState(SHUTDOWN);
        // 中断空闲的工作线程
        interruptIdleWorkers();
        // 为 ScheduledThreadPoolExecutor 准备的钩子方法
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```
```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();
        // 从任务队列中获取未开始执行的任务列表
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```
-- over --
