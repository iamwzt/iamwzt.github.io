再来看看前面的 register0(promise) 方法。
在NioEventLoop工作流程中讲到，这个 register 任务进入到了 NioEventLoop 的 taskQueue 中，然后会启动 NioEventLoop 中的线程。
该线程会轮询这个 taskQueue，然后执行这个 register 任务。

注意，此时执行该方法的是 eventLoop 中的线程：
> AbstractChannel.java

```java
private void register0(ChannelPromise promise) {
    try {
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
        boolean firstRegistration = neverRegistered;
        // *** 进行 JDK 底层的操作：Channel 注册到 Selector 上 ***
        doRegister();
        neverRegistered = false;
        registered = true;

        // 到这里，就算是 registered 了

        // 这一步也很关键，因为这涉及到了 ChannelInitializer 的 init(channel)
        // 我们之前说过，init 方法会将 ChannelInitializer 内部添加的 handlers 添加到 pipeline 中
        pipeline.invokeHandlerAddedIfNeeded();
        // 设置当前 promise 的状态为 success
        //   因为当前 register 方法是在 eventLoop 中的线程中执行的，需要通知提交 register 操作的线程
        safeSetSuccess(promise);
        // 当前的 register 操作已经成功，该事件应该被 pipeline 上
        //   所有关心 register 事件的 handler 感知到，往 pipeline 中扔一个事件        
        pipeline.fireChannelRegistered();

        // 这里 active 指的是 channel 已经打开
        if (isActive()) {
            // 如果该 channel 是第一次执行 register，那么 fire ChannelActive 事件
            if (firstRegistration) {
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                // 该 channel 之前已经 register 过了，
                // 这里让该 channel 立马去监听通道中的 OP_READ 事件
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```
我们先说掉上面的 doRegister() 方法，然后再说 pipeline。
```java
@Override
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            // 附 JDK 中 Channel 的 register 方法：
            // public final SelectionKey register(Selector sel, int ops, Object att) {...}
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            // ...
        }
    }
}
```
这里做了 JDK 底层的 register 操作，将 SocketChannel(或 ServerSocketChannel) 注册到 Selector 中，并且可以看到，这里的监听集合设置为了 0，也就是什么都不监听。

> 当然，也就意味着，后续一定有某个地方会需要修改这个 selectionKey 的监听集合，不然啥都干不了

我们重点来说说 pipeline 操作，我们之前在介绍 NioSocketChannel 的 pipeline 的时候介绍到，我们的 pipeline 现在长这个样子：
![](https://www.javadoop.com/blogimages/netty-source/20.png)
> 现在，我们将看到这里会把 LoggingHandler 和 EchoClientHandler 添加到 pipeline。

继续看代码，register 成功以后，执行了以下操作：
```java
pipeline.invokeHandlerAddedIfNeeded();
```
在这里打个断点，一路跟踪一下，会执行到 pipeline 中 ChannelInitializer 实例的 handlerAdded 方法，在这里会执行它的 init(context) 方法：
```java
@Override
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    if (ctx.channel().isRegistered()) {
        if (initChannel(ctx)) {
            removeState(ctx);
        }
    }
}
```
然后我们看下 initChannel(ctx)，这里终于来了我们之前介绍过的 init(channel) 方法：
```java
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.add(ctx)) { // Guard against re-entrance.
        try {
            // 1. 将把我们自定义的 handlers 添加到 pipeline 中
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            exceptionCaught(ctx, cause);
        } finally {
            ChannelPipeline pipeline = ctx.pipeline();
            if (pipeline.context(this) != null) {
                // 2. 将 ChannelInitializer 实例从 pipeline 中删除
                pipeline.remove(this);
            }
        }
        return true;
    }
    return false;
}
```
我们前面也说过，ChannelInitializer 的 init(channel) 被执行以后，那么其内部添加的 handlers 会进入到 pipeline 中，然后上面的 finally 块中将 ChannelInitializer 的实例从 pipeline 中删除，那么此时 pipeline 就算建立起来了，如下图：
![](https://www.javadoop.com/blogimages/netty-source/21.png)

> 其实这里还有个问题，如果我们在 ChannelInitializer 中添加的是一个 ChannelInitializer 实例呢

pipeline 建立了以后，然后我们继续往下走，会执行到这一句：
```java
pipeline.fireChannelRegistered();
```
只要摸清楚了 fireChannelRegistered() 方法，以后碰到其他像 fireChannelActive()、fireXxx() 等就知道怎么回事了，它们都是类似的。我们来看看这句代码会发生什么：
> DefaultChannelPipeline.java

```java
@Override
public final ChannelPipeline fireChannelRegistered() {
    // 传的是 head
    AbstractChannelHandlerContext.invokeChannelRegistered(head);
    return this;
}
```
