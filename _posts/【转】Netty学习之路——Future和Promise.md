转自[Netty 源码解析（三）: Netty 的 Future 和 Promise](https://www.javadoop.com/post/netty-part-3)

Netty 中非常多的异步调用，所以在介绍更多 NIO 相关的内容之前，我们来看看它的异步接口是怎么使用的。

前面我们在介绍 Echo 例子的时候，已经用过了 ChannelFuture 这个接口了：
![](https://www.javadoop.com/blogimages/netty-source/6.png)

关于 `Future` 接口，用得最多的就是在使用 Java 的线程池 ThreadPoolExecutor 的时候。
在 submit 一个任务到线程池中的时候，返回的就是一个 Future 实例。
通过它来获取提交的任务的执行状态和最终的执行结果，我们最常用它的 `isDone()` 和 `get()` 方法。

Netty中的Future继承了JDK中的Future，增加了一些方法：

![](https://wzt-img.oss-cn-chengdu.aliyuncs.com/Netty%23Future.png)

主要增加了一些关于 `Listener`的方法，用于监听回调，那样就不用阻塞去获取；

还有 `sync` 和 `wait` 方法：
sync() 内部会先调用 await() 方法，等 await() 方法返回后，会检查下这个任务是否失败，如果失败，重新将导致失败的异常抛出来。
也就是说，如果使用 await()，任务抛出异常后，await() 方法会返回，但是不会抛出异常，而 sync() 方法返回的同时会抛出异常。

接下来，我们来看 Future 接口的子接口 ChannelFuture，这个接口用得最多，它将和 IO 操作中的 Channel 关联在一起了，用于异步处理 Channel 中的事件。

```java
public interface ChannelFuture extends Future<Void> {
    // ChannelFuture 关联的 Channel
    Channel channel();

    // 覆写以下几个方法，使得它们返回值为 ChannelFuture 类型 
    @Override
    ChannelFuture addListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelFuture addListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);
    @Override
    ChannelFuture removeListener(GenericFutureListener<? extends Future<? super Void>> listener);
    @Override
    ChannelFuture removeListeners(GenericFutureListener<? extends Future<? super Void>>... listeners);
    @Override
    ChannelFuture sync() throws InterruptedException;
    @Override
    ChannelFuture syncUninterruptibly();
    @Override
    ChannelFuture await() throws InterruptedException;
    @Override
    ChannelFuture awaitUninterruptibly();
    // 用来标记该 future 是 void 的，
    // 这样就不允许使用 addListener(...), sync(), await() 以及它们的几个重载方法
    boolean isVoid();
}
```
我们看到，ChannelFuture 接口相对于 Future 接口，除了将 channel 关联进来，没有增加什么东西。还有个 isVoid() 方法算是不那么重要的存在吧。
其他几个都是方法覆写，为了让返回值类型变为 ChannelFuture，而不是原来的 Future。

好，这里先将`Future`放一放，来看看`Promise`，Promise 接口和 ChannelFuture 一样，也继承了 Netty 的 Future 接口，然后加了一些 Promise 的内容：
```java
public interface Promise<V> extends Future<V> {
    // 标记该 future 成功及设置其执行结果，并且会通知所有的 listeners。
    // 如果该操作失败，将抛出异常(失败指的是该 future 已经有了结果了，成功的结果，或者失败的结果)
    Promise<V> setSuccess(V result);

    // 和 setSuccess 方法一样，只不过如果失败，它不抛异常，返回 false
    boolean trySuccess(V result);

    // 标记该 future 失败，及其失败原因。
    // 如果失败，将抛出异常(失败指的是已经有了结果了)
    Promise<V> setFailure(Throwable cause);

    // 和 setFailure 方法一样，只不过如果失败，它不抛异常，返回 false
    boolean tryFailure(Throwable cause);

    // 标记该 future 不可以被取消
    boolean setUncancellable();
    // 这里和 ChannelFuture 一样，对这几个方法进行覆写，目的是为了返回 Promise 类型的实例
    @Override
    Promise<V> addListener(GenericFutureListener<? extends Future<? super V>> listener);
    @Override
    Promise<V> addListeners(GenericFutureListener<? extends Future<? super V>>... listeners);
    @Override
    Promise<V> removeListener(GenericFutureListener<? extends Future<? super V>> listener);
    @Override
    Promise<V> removeListeners(GenericFutureListener<? extends Future<? super V>>... listeners);
    @Override
    Promise<V> await() throws InterruptedException;
    @Override
    Promise<V> awaitUninterruptibly();
    @Override
    Promise<V> sync() throws InterruptedException;
    @Override
    Promise<V> syncUninterruptibly();
}
```
Promise 实例内部是一个任务，任务的执行往往是异步的，通常是一个线程池来处理任务。
Promise 提供的 setSuccess(V result) 或 setFailure(Throwable t) 将来会被某个执行任务的线程在执行完成以后调用。
同时那个线程在调用 setSuccess(result) 或 setFailure(t) 后会回调 listeners 的回调函数（当然，回调的具体内容不一定要由执行任务的线程自己来执行，它可以创建新的线程来执行，也可以将回调任务提交到某个线程池来执行）。
而且，一旦 setSuccess(...) 或 setFailure(...) 后，那些 await() 或 sync() 的线程就会从等待中返回。

所以这里就有两种编程方式，一种是用 await()，等 await() 方法返回后，得到 promise 的执行结果，然后处理它；
另一种就是提供 Listener 实例，我们不太关心任务什么时候会执行完，只要它执行完了以后会去执行 listener 中的处理方法就行。

接下来，我们再来看下 ChannelPromise，它继承了前面介绍的 ChannelFuture 和 Promise 接口。

![ChannelPromise](https://wzt-img.oss-cn-chengdu.aliyuncs.com/ChannelPromise.png)

`ChannelPromise` 中没有什么新接口，只是重写了一下，使方法的返回类型变为`ChannelPromise`。
下图是几个接口的主要方法：
![](https://www.javadoop.com/blogimages/netty-source/7.png)

接下来，我们需要来一个实现类，这样才能比较直观地看出它们是怎么使用的，因为上面的这些都是接口定义，具体还得看实现类是怎么工作的。

下面，我们来介绍下 `DefaultPromise` 这个实现类，这个类很常用，它的源码也不短，我们先介绍几个关键的内容，然后介绍一个示例使用。

首先，我们看下它有哪些属性：
```java
public class DefaultPromise<V> extends AbstractFuture<V> implements Promise<V> {
    // 保存执行结果
    private volatile Object result;
    // 执行任务的线程池，promise 持有 executor 的引用
    private final EventExecutor executor;
    // 监听者，回调函数，任务结束后（正常或异常结束）执行
    private Object listeners;
    // 等待这个 promise 的线程数(调用sync()/await()进行等待的线程数量)
    private short waiters;
    // 是否正在唤醒等待线程，用于防止重复执行唤醒，不然会重复执行 listeners 的回调方法
    private boolean notifyingListeners;
    // more
}
```

可以看出，此类实现了 Promise，但是没有实现 ChannelFuture，所以它和 Channel 联系不起来。

别急，我们后面会碰到另一个类 DefaultChannelPromise 的使用，这个类是综合了 ChannelFuture 和 Promise 的，但是它的实现其实大部分都是继承自这里的 DefaultPromise 类的。

说完上面的属性以后，大家可以看下 setSuccess(V result) 、trySuccess(V result) 和 setFailure(Throwable cause) 、 tryFailure(Throwable cause) 这几个方法：
![](https://www.javadoop.com/blogimages/netty-source/8.png)

上面几个方法都非常简单，先设置好值，然后执行监听者们的回调方法。
notifyListeners() 方法感兴趣的读者也可以看一看，不过它还涉及到 Netty 线程池的一些内容，我们还没有介绍到线程池，这里就不展开了。
上面的代码，在 setSuccess0 或 setFailure0 方法中都会唤醒阻塞在 sync() 或 await() 的线程。

另外，就是可以看下 sync() 和 await() 的区别，其他的我觉得随便看看就好了。
```java
@Override
public Promise<V> sync() throws InterruptedException {
    await();
    rethrowIfFailed();
    return this;
}
```

写个Demo试验一下：
```java
public static void main(String[] args) {
    // 构造线程池
    EventExecutor executor = new DefaultEventExecutor();
    // 创建 DefaultPromise 实例
    Promise promise = new DefaultPromise(executor);
    // 下面给这个 promise 添加两个 listener
    promise.addListener(new GenericFutureListener<Future<Integer>>() {
        @Override
        public void operationComplete(Future future) throws Exception {
            if (future.isSuccess()) {
                System.out.println("任务结束，结果：" + future.get());
            } else {
                System.out.println("任务失败，异常：" + future.cause());
            }
        }
    }).addListener(new GenericFutureListener<Future<Integer>>() {
        @Override
        public void operationComplete(Future future) throws Exception {
            System.out.println("任务结束，balabala...");
        }
    });
    // 提交任务到线程池，五秒后执行结束，设置执行 promise 的结果
    executor.submit(new Runnable() {
        @Override
        public void run() {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
            }
            // 设置 promise 的结果
            // promise.setFailure(new RuntimeException());
            promise.setSuccess(123456);
        }
    });

    // main 线程阻塞等待执行结果
    try {
        promise.sync();
    } catch (InterruptedException e) {
    }
}
```
运行结果，5秒后输出：
```
任务结束，结果：123456
任务结束，balabala...
```
回过头来再看一下这张图，看看大家是不是看懂了本文内容：
![](https://www.javadoop.com/blogimages/netty-source/6.png)

可以猜测一下：
main 线程调用 b.bind(port) 这个方法会返回一个 ChannelFuture。
bind() 是一个异步方法，当某个执行线程执行了真正的绑定操作后，那个执行线程一定会标记这个 future 为成功（我们假定 bind 会成功）。
然后这里的 sync() 方法（main 线程）就会返回了。

如果 bind(port) 失败，我们知道，sync() 方法会将异常抛出来，然后就会执行到 finally 块了。 

一旦绑定端口 bind 成功，进入下面一行，f.channel() 方法会返回该 future 关联的 channel。

channel.closeFuture() 也会返回一个 ChannelFuture，然后调用了 sync() 方法。
这个 sync() 方法返回的条件是：有其他的线程关闭了 NioServerSocketChannel，往往是因为需要停掉服务了，然后那个线程会设置 future 的状态（ setSuccess(result) 或 setFailure(cause) ），这个 sync() 方法才会返回。
