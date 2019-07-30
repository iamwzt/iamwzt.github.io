### 首先来看一下Netty实现的Echo是什么样的
所谓Echo，中文名“回声”，就是客户端向服务端发送什么，服务端就给客户端回什么。

它的服务端程序是这样的：
```java
public class EchoServer {
    private final int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            System.err.println("Usage: " + EchoServer.class.getSimpleName() + " <port> ");
            return;
        }
        int port = Integer.parseInt(args[0]);
        new EchoServer(port).start();
    }

    public void start() throws Exception {
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(port))
                    .handler(new LoggingHandler(LogLevel.INFO))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline()
                                    .addLast(new LoggingHandler(LogLevel.INFO))
                                    .addLast(new EchoServerHandler());
                            ch.pipeline().remove(this);
                        }
                    });
            ChannelFuture future = bootstrap.bind().sync();
            future.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();
        }
    }
}
```
然后是客户端的代码：
```java
public class EchoClient {
    private final int port;
    private final String host;

    public EchoClient(String host, int port) {
        this.port = port;
        this.host = host;
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            System.err.println("Usage: " + EchoClient.class.getSimpleName() + "<host> <port>");
            return;
        }
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        new EchoClient(host, port).start();
    }

    public void start() throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .remoteAddress(new InetSocketAddress(host, port))
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline()
                                    .addLast(new LoggingHandler(LogLevel.INFO))
                                    .addLast(new EchoClientHandler());
                        }
                    });
            ChannelFuture future = bootstrap.connect().sync();
            future.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();
        }
    }
}
```
将两者的`start`方法提出来比较一下：
![这里有图](https://www.javadoop.com/blogimages/netty-source/5.png)

可以发现流程很相似，不同点也在图里标出来了。

上面的代码基本就是模板代码，每次使用都是这一个套路，唯一需要我们开发的部分是 `handler(…)` 和 `childHandler(…)` 方法中指定的各个 handler，如 `EchoServerHandler` 和 `EchoClientHandler`。
当然 Netty 源码也给我们提供了很多的 handler，比如上面的 `LoggingHandler`，它就是 Netty 源码中为我们提供的，需要的时候直接拿过来用就好了。

来看一下上面代码中涉及到的一些内容：
1. `ServerBootstrap` 用于创建服务器实例，`Bootstrap`用于创建客户端实例；

2. 两个 `EventLoopGroup`：bossGroup 和 workerGroup，它们涉及的是 Netty 的线程模型，可以看到服务端有两个 group，而客户端只有一个，它们就是 Netty 中的线程池。

3. **Netty 中的 Channel**，没有直接使用 Java 原生的 `ServerSocketChannel` 和 `SocketChannel`，而是包装了 `NioServerSocketChannel` 和 `NioSocketChannel` 与之对应。

4. 左边 handler(…) 方法指定了一个 handler（LoggingHandler），这个 handler 是给服务端收到新的请求的时候处理用的。右边 handler(...) 方法指定了客户端处理请求过程中需要使用的 handlers。

5. 左边 childHandler(…) 指定了 childHandler，这边的 handlers 是给新创建的连接用的，我们知道服务端 ServerSocketChannel 在 accept 一个连接以后，需要创建 SocketChannel 的实例，childHandler(…) 中设置的 handler 就是用于处理新创建的 SocketChannel 的，而不是用来处理 ServerSocketChannel 实例的。

6. **pipeline**：handler 可以指定多个（需要上面的 ChannelInitializer 类辅助），它们会组成了一个 pipeline，它们其实就类似拦截器的概念，现在只要记住一点，每个 NioSocketChannel 或 NioServerSocketChannel 实例内部都会有一个一个 pipeline 实例。pipeline 中还涉及到 handler 的执行顺序。

7. **ChannelFuture**：这个涉及到 Netty 中的异步编程，和 JDK 中的 Future 接口类似。
