转自[Netty 源码解析（四）: Netty 的 ChannelPipeline](https://www.javadoop.com/post/netty-part-4)

前面我们说了，使用 Netty 的时候，我们通常就只要写一些自定义的 handler 就可以了，我们定义的这些 handler 会组成一个 pipeline，用于处理 IO 事件，这个和我们平时接触的 Filter 或 Interceptor 表达的差不多是一个意思。

每个 Channel 内部都有一个 pipeline，pipeline 由多个 handler 组成，handler 之间的顺序是很重要的，因为 IO 事件将按照顺序顺次经过 pipeline 上的 handler，这样每个 handler 可以专注于做一点点小事，由多个 handler 组合来完成一些复杂的逻辑。

![pipeline](https://www.javadoop.com/blogimages/netty-source/11.png)

从图中，我们知道这是一个**双向链表**。

首先，我们看两个重要的概念：**Inbound** 和 **Outbound**。在 Netty 中，IO 事件被分为 Inbound 事件和 Outbound 事件。

Outbound 的 out 指的是 出去，有哪些 IO 事件属于此类呢？
比如 connect、write、flush 这些 IO 操作是往外部方向进行的，它们就属于 Outbound 事件。

其他的，诸如 accept、read 这种就属于 Inbound 事件。

服务端的 childHandler 中，可能出现下面这段略具“迷惑性”的代码：
```java
pipeline.addLast(new StringDecoder());  // #1
pipeline.addLast(new StringEncoder());  // #2
pipeline.addLast(new BizHandler());     // #3
```
初学者肯定都纳闷，以为这个顺序写错了，应该是先 decode 客户端过来的数据，然后用 BizHandler 处理业务逻辑，最后再 encode 数据然后返回给客户端，所以添加的顺序应该是 1 -> 3 -> 2 才对。

其实这里的三个 handler 是**分组**的，分为 Inbound（1 和 3） 和 Outbound（2）：
- 客户端连接进来的时候，读取（read）客户端请求数据的操作是 Inbound 的，所以会先使用 1，然后是 3 对处理进行处理；
- 处理完数据后，返回给客户端数据的 write 操作是 Outbound 的，此时使用的是 2。

所以虽然添加顺序有点怪，但是执行顺序其实是按照 1 -> 3 -> 2 进行的。

如果我们在上面的基础上，加上下面的第四行，这是一个 OutboundHandler：
```java
pipeline.addLast(new OutboundHandlerA());   // #4
```
那么执行顺序是不是就是 1 -> 3 -> 2 -> 4 呢？答案却是否定的。

对于 Inbound 操作，按照添加顺序执行每个 Inbound 类型的 handler；
而对于 Outbound 操作，是反着来的，从后往前，顺次执行 Outbound 类型的 handler。

所以，上面的顺序应该是先 1 后 3，它们是 Inbound 的，然后是 4，最后才是 2，它们两个是 Outbound 的。

前面说过，pipeline是个双向链表，可以理解成 **InBound** 是从队首到队尾，而 **OutBound** 是从队尾到队首的顺序。

下面我们来介绍它们的接口使用。
![](https://www.javadoop.com/blogimages/netty-source/9.png)
定义处理 Inbound 事件的 handler 需要实现 ChannelInboundHandler，定义处理 Outbound 事件的 handler 需要实现 ChannelOutboundHandler。
最下面的三个类，是 Netty 提供的适配器，特别的，如果我们希望定义一个 handler 能同时处理 Inbound 和 Outbound 事件，可以通过继承中间的 ChannelDuplexHandler 的方式。

有了 Inbound 和 Outbound 的概念以后，我们来开始介绍 Pipeline 的源码。

我们说过，一个 Channel 关联一个 pipeline，NioSocketChannel 和 NioServerSocketChannel 在执行构造方法的时候，都会走到它们的父类 AbstractChannel 的构造方法中：
```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    // 给每个 channel 分配一个唯一 id
    id = newId();
    // 每个 channel 内部需要一个 Unsafe 的实例
    unsafe = newUnsafe();
    // 每个 channel 内部都会创建一个 pipeline
    pipeline = newChannelPipeline();
}
```
上面的三行代码中，id 比较不重要，Netty 中的 Unsafe 实例其实挺重要的，这里简单介绍一下。

在 JDK 的源码中，sun.misc.Unsafe 类提供了一些底层操作的能力。
它设计出来不是给我们的代码使用的，而是给 JDK 中的源码使用的，比如 AQS、ConcurrentHashMap 等。
但若需要的话，我们也是可以获取它的实例的。

> Unsafe 类的构造方法是 private 的，但是它提供了 getUnsafe() 这个静态方法：
> ```java
> Unsafe unsafe = Unsafe.getUnsafe();
> ```
> 大家可以试一下，上面这行代码编译没有问题，但是执行的时候会抛 java.lang.SecurityException 异常，因为它就不是给我们的代码用的。
>
> 但是如果就是想获取 Unsafe 的实例，可以通过下面这个代码获取到:
> ```java
> Field f = Unsafe.class.getDeclaredField("theUnsafe");
> f.setAccessible(true);
> Unsafe unsafe = (Unsafe) f.get(null);
> ```

Netty 中的 Unsafe 也是同样的意思，它封装了 Netty 中会使用到的 JDK 提供的 NIO 接口，比如将 channel 注册到 selector 上，比如 bind 操作，比如 connect 操作等，这些操作都是稍微偏底层一些。
Netty 同样也是不希望我们的业务代码使用 Unsafe 的实例，它是提供给 Netty 中的源码使用的。

关于 Unsafe，我们后面用到了再说，这里只要知道，它封装了大部分需要访问 JDK 的 NIO 接口的操作就好了。这里我们继续将焦点放在实例化 pipeline 上：
```java
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}

protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
在这里初始化了tail 和 head 这两个 handler。
tail 实现了 ChannelInboundHandler 接口，而 head 实现了 ChannelOutboundHandler 和 ChannelInboundHandler 两个接口，并且最后两行代码将 tail 和 head 连接起来:

![](https://www.javadoop.com/blogimages/netty-source/12.png)

> 注意，在不同的版本中，源码也略有差异，head 不一定是 in + out，大家知道这点就好了。
>
> 还有，从上面的 head 和 tail 我们也可以看到，其实 pipeline 中的每个元素是 ChannelHandlerContext 的实例，而不是 ChannelHandler 的实例，context 包装了一下 handler，但是，后面我们都会用 handler 来描述一个 pipeline 上的节点，而不是使用 context，希望读者知道这一点。

这里只是构造了 pipeline，并且添加了两个固定的 handler 到其中（head + tail），还不涉及到自定义的 handler 代码执行。我们回过头来看下面这段代码：
![](https://www.javadoop.com/blogimages/netty-source/13.png)

