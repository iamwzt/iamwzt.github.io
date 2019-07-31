转自[Netty 源码解析（二） - Netty 中的 Channel](https://www.javadoop.com/post/netty-part-2)
### Netty中的Channel

话接上文，Netty 中的 Channel 没有直接使用 Java 原生的 ServerSocketChannel 和 SocketChannel，而是包装了 NioServerSocketChannel 和 NioSocketChannel 与之对应。

在 Bootstrap（客户端） 和 ServerBootstrap（服务端） 的启动过程中都会调用 channel(…) 方法：

![有图]()

现在看一下源码：
```java
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}
```
可以看到就是设置了 `channelFactory` 为 `ReflectiveChannelFactory` 的一个实例。

> ReflectiveChannelFactory.java

```java
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {

    private final Constructor<? extends T> constructor;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        ObjectUtil.checkNotNull(clazz, "clazz");
        try {
            this.constructor = clazz.getConstructor();
        } catch (NoSuchMethodException e) {
            throw new IllegalArgumentException("Class " + StringUtil.simpleClassName(clazz) +
                    " does not have a public non-arg constructor", e);
        }
    }

    @Override
    public T newChannel() {
        try {
            return constructor.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + constructor.getDeclaringClass(), t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(ReflectiveChannelFactory.class) +
                '(' + StringUtil.simpleClassName(constructor.getDeclaringClass()) + ".class)";
    }
}
```
代码很简单，在构造方法里获取 Channel 的无参构造方法，然后在 `newChannel` 方法里实例化。

此处我们只要知道 ChannelFactory 的 `newChannel` 方法什么时候会被调用就可以了:
- 对于 NioSocketChannel ，由于是充当客户端的功能，在 `connect(…)`时创建；
- 对于 NioServerSocketChannel ，它充当服务端功能，在绑定端口 `bind(…)` 时创建。

接下来追踪一下充当客户端的 Bootstrap 中 NioSocketChannel 的创建过程，看看 NioSocketChannel 是怎么和 JDK 中的 SocketChannel 关联在一起的：
```java
public ChannelFuture connect() {
    // 只是校验一下各个参数是不是正确设置了
    validate();
    SocketAddress remoteAddress = this.remoteAddress;
    if (remoteAddress == null) {
        throw new IllegalStateException("remoteAddress not set");
    }
    return doResolveAndConnect(remoteAddress, config.localAddress());
}
```
```java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
    // 接着看这里
    final ChannelFuture regFuture = initAndRegister();
    final Channel channel = regFuture.channel();

    if (regFuture.isDone()) {
        if (!regFuture.isSuccess()) {
            return regFuture;
        }
        return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
    } else {
        // ...
    }
}
```
```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    try {
        // 喏，就是这里了
        channel = channelFactory.newChannel();
        init(channel);
    } catch (Throwable t) {
        // ...
    }
    // ...
}
```
然后看一下 `NIOSocketChannel` 是怎么用无参构造方法实例化的：
```java
public NioSocketChannel() {
    this(DEFAULT_SELECTOR_PROVIDER);
}
// 此处的 DEFAULT_SELECTOR_PROVIDER：
// private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();

public NioSocketChannel(SelectorProvider provider) {
    this(newSocket(provider));
}

private static SocketChannel newSocket(SelectorProvider provider) {
    try {
        // 在这里创建了 SocketChannel
        return provider.openSocketChannel();
    } catch (IOException e) {
        throw new ChannelException("Failed to open a socket.", e);
    }
}
```
到此，创建了一个JDK里的 SocketChannel。 `NioServerSocketChannel`同理，从 `bind(...)`进去可以看到相似的套路。

然后再看一下`NioSocketChannel`的构造方法：
```java
public NioSocketChannel(SelectorProvider provider) {
    this(newSocket(provider));
}

public NioSocketChannel(SocketChannel socket) {
    this(null, socket);
}

public NioSocketChannel(Channel parent, SocketChannel socket) {
    super(parent, socket);
    config = new NioSocketChannelConfig(this, socket.socket());
}
```
第二行代码很简单，实例化了内部的 NioSocketChannelConfig 实例，它用于保存 channel 的配置信息。

第一行调用父类构造器，除了设置属性外，还设置了 SocketChannel 的非阻塞模式：
```java
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    // 客户端关心的是 OP_READ 事件
    super(parent, ch, SelectionKey.OP_READ);
}

protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    // 我们看到这里只是保存了 SelectionKey.OP_READ 这个信息，在后面的时候会用到
    this.readInterestOp = readInterestOp;
    try {
        // 设置非阻塞模式
        ch.configureBlocking(false);
    } catch (IOException e) {
        try {
            ch.close();
        } catch (IOException e2) {
            if (logger.isWarnEnabled()) {
                logger.warn(
                        "Failed to close a partially initialized socket.", e2);
            }
        }
        throw new ChannelException("Failed to enter non-blocking mode.", e);
    }
}
```

NioServerSocketChannel 的构造方法类似，也设置了非阻塞，然后设置服务端关心的 `OP_ACCEPT` 事件：
```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

这节关于 Channel 的内容我们先介绍这么多，主要就是实例化了 JDK 层的 SocketChannel 或 ServerSocketChannel，然后设置了非阻塞模式，我们后面再继续深入下去。
