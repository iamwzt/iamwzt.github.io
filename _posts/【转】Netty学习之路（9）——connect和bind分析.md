转自[Netty 源码解析（九）: connect 过程和 bind 过程分析](https://www.javadoop.com/post/netty-part-9)

上面我们介绍的 register 操作非常关键，它建立起来了很多的东西，它是 Netty 中 NioSocketChannel 和 NioServerSocketChannel 开始工作的起点。

这一节，我们来说说 register 之后的 connect 操作和 bind 操作。这节非常简单。

### connect 过程分析
对于客户端 NioSocketChannel 来说，前面 register 完成以后，就要开始 connect 了，这一步将连接到服务端。

```java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    // 这里完成了 register 操作
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    if (regFuture.isDone()) {
        if (!regFuture.isSuccess()) {
            return regFuture;
        }
        // 这里是连接的关键
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    } else {
        // ...
    }
}
```
这里一路点进去：
最后，我们会来到 AbstractChannel 的 connect 方法：
```java
@Override
public ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
    return pipeline.connect(remoteAddress, promise);
}
```
connect 操作是交给 pipeline 来执行的。进入 pipeline 中，我们会发现，connect 这种 Outbound 类型的操作，是从 pipeline 的 tail 开始的：
> 前面介绍的 register 操作是 Inbound 的，是从 head 开始的

```java
@Override
public final ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
    return tail.connect(remoteAddress, promise);
}
```
接下来就是 pipeline 的操作了，从 tail 开始，执行 pipeline 上的 Outbound 类型的 handlers 的 connect(...) 方法，那么真正的底层的 connect 的操作发生在哪里呢？还记得我们的 pipeline 的图吗？
![](https://www.javadoop.com/blogimages/netty-source/22.png)
从 tail 开始往前找 out 类型的 handlers，每经过一个 handler，都执行里面的 connect() 方法，最后会到 head 中，因为 head 也是 Outbound 类型的，我们需要的 connect 操作就在 head 中，它会负责调用 unsafe 中提供的 connect 方法：
```java
@Override
public void connect(
        ChannelHandlerContext ctx,
        SocketAddress remoteAddress, SocketAddress localAddress,
        ChannelPromise promise) {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```
接下来，我们来看一看 connect 在 unsafe 类中所谓的底层操作：
```java
@Override
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }
    try {
        if (connectPromise != null) {
            throw new ConnectionPendingException();
        }
        boolean wasActive = isActive();

        // 这一步会做 JDK 底层的 SocketChannel connect，然后设置 interestOps 为 SelectionKey.OP_CONNECT
        // 返回值代表是否已经连接成功        
        if (doConnect(remoteAddress, localAddress)) {
            // 处理连接成功的情况
            fulfillConnectPromise(promise, wasActive);
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // 下面这块代码，在处理连接超时的情况，代码很简单
            // 这里用到了 NioEventLoop 的定时任务的功能
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                    @Override
                    public void run() {
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        ConnectTimeoutException cause =
                                new ConnectTimeoutException("connection timed out: " + remoteAddress);
                        if (connectPromise != null && connectPromise.tryFailure(cause)) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) {
                        if (connectTimeoutFuture != null) {
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise());
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}
```
如果上面的 doConnect 方法返回 false，那么后续是怎么处理的呢？

在上一节介绍的 register 操作中，channel 已经 register 到了 selector 上，只不过将 interestOps 设置为了 0，也就是什么都不监听。

而在上面的 doConnect 方法中，我们看到它在调用底层的 connect 方法后，会设置 interestOps 为 `SelectionKey.OP_CONNECT`。

剩下的就是 NioEventLoop 的事情了，还记得 NioEventLoop 的 run() 方法吗？也就是说这里的 connect 成功以后，这个 TCP 连接就建立起来了，后续的操作会在 NioEventLoop.run() 方法中被 processSelectedKeys() 方法处理掉。

### bind 过程分析
说完 connect 过程，我们再来简单看下 bind 过程：
```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();
    if (regFuture.cause() != null) {
        return regFuture;
    }

    if (regFuture.isDone()) {
        ChannelPromise promise = channel.newPromise();
        // 看这里
        doBind0(regFuture, channel, localAddress, promise);
        return promise;
    } else {
        final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
        regFuture.addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                Throwable cause = future.cause();
                if (cause != null) {
                    promise.setFailure(cause);
                } else {
                    promise.registered();

                    doBind0(regFuture, channel, localAddress, promise);
                }
            }
        });
        return promise;
    }
}
```
然后一直往里看，会看到，bind 操作也是要由 pipeline 来完成的：
> AbstractChannel.java

```java
@Override
public ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return pipeline.bind(localAddress, promise);
}
```
bind 操作和 connect 一样，都是 Outbound 类型的，所以都是 tail 开始：
```java
@Override
public final ChannelFuture bind(SocketAddress localAddress, ChannelPromise promise) {
    return tail.bind(localAddress, promise);
}
```
最后的 bind 操作又到了 head 中，由 head 来调用 unsafe 提供的 bind 方法：

```java
@Override
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) {
    unsafe.bind(localAddress, promise);
}
```
感兴趣的读者自己去看一下 unsafe 中的 bind 方法，非常简单，bind 操作也不是什么异步方法，我们就介绍到这里了。
