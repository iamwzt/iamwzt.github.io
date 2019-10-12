## 相关类

### Watcher 接口

任何一个事件处理类都必须实现`Watcher` 接口，它有一个内部接口 `Event`，以及 `process`方法。前者定义了**ZK的状态**和**事件类型**的枚举，后者则定义了事件的**回调方法**。
```java
@InterfaceAudience.Public
public interface Watcher {
    @InterfaceAudience.Public
    public interface Event {
        @InterfaceAudience.Public
        public enum KeeperState {/*...*/}
        
        @InterfaceAudience.Public
        public enum EventType {/*...*/}
    }
    
    abstract public void process(WatchedEvent event);
}
```

回调方法 `process`只有一个参数 `WatchedEvent`，这个类也很简单，只有三个参数，分别是通知状态、事件类型，以及节点路径。
```java
public class WatchedEvent {
    final private KeeperState keeperState;
    final private EventType eventType;
    private String path;
    /*...*/
}
```

和 `WatchedEvent` 紧密相关的还有 `WatcherEvent`，两者都是对ZK服务端事件的封装，不过后者实现了序列化接口，可用于网络传输。
ZK服务端会将 `WatchedEvent` 包装成 `WatcherEvent`进行传输，客户端则需逆向处理，解包装成`WatchedEvent`来处理事件。

可以看到，`WatchedEvent` 和 `WatcherEvent` 都只有简单的事件本身的信息，而不包含具体的内容，因此需要客户端主动去获取感兴趣的最新数据。

---

## 工作机制

### 客户端注册

客户端可以通过 `getData()`、`getChildren()`和`exist()`三个接口来向服务端注册 Watcher。注册的原理都是相同的，这里仅以 `getData()` 为例进行分析。

`getData` 有两个重载方法，区别在于第二个参数。
```java
// 使用自定义的 watcher
public byte[] getData(final String path, Watcher watcher, Stat stat)
// 是否使用默认的 watcher
public byte[] getData(String path, boolean watch, Stat stat)
```
默认的 watcher 在实例化 `ZooKeeper` 的时候指定：

```java
public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher)
```

`ZooKeeper` 实例维护了一个 `ZKWatchManager` 类的实例，该类实现了`ClientWatchManager`接口，是客户端的 watcher 管理器。
默认 watcher 就保存在 `ZKWatchManager` 的 `defaultWatcher` 中。
```java
private final ZKWatchManager watchManager = new ZKWatchManager();

private static class ZKWatchManager implements ClientWatchManager {
    private final Map<String, Set<Watcher>> dataWatches =
        new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> existWatches =
        new HashMap<String, Set<Watcher>>();
    private final Map<String, Set<Watcher>> childWatches =
        new HashMap<String, Set<Watcher>>();

    private volatile Watcher defaultWatcher;
    /* ... */
}
```

回到`getData`，传递 watcher 参数后，首先会被封装成 `WatchRegistration` 对象，并设置 request 对象为“使用watcher监听”：
```java
public byte[] getData(final String path, Watcher watcher, Stat stat)
    throws KeeperException, InterruptedException {
    /* ... */
    WatchRegistration wcb = null;
    if (watcher != null) {
        wcb = new DataWatchRegistration(watcher, clientPath);
    }
    /* ... */
    request.setWatch(watcher != null);
    ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);
    /* ... */
}
```

然后在客户端连接对象cnxn（`ClientCnxn`）的 `submitRequest()`方法中，又被封装成 `Packet` 对象进行网络传输。

（PS：`Packet`在ZK中可以被看作最小的通信协议单元）

```java
public ReplyHeader submitRequest(RequestHeader h, Record request,
        Record response, WatchRegistration watchRegistration)
        throws InterruptedException {
    /* ... */
    Packet packet = queuePacket(h, r, request, response, null, null, null,
                null, watchRegistration);
    /* ... */
}
```

发送完请求后，由`ClientCnxn`的 `SendThread`线程的 `readResponse()`方法负责接收服务端的响应。
```java
void readResponse(ByteBuffer incomingBuffer) throws IOException {
    /* ... */
    finishPacket(packet);
}
```
在 `finishPacket()` 中注册 watcher 到 `ZKWatchManager` 的 `dataWatches` 中。
```java
private void finishPacket(Packet p) {
    // 注册 watcher
    // p.replyHeader.getErr() 为0，则代表服务端响应成功。
    if (p.watchRegistration != null) {
        p.watchRegistration.register(p.replyHeader.getErr());
    }
    /* ... */
}
```
> ZooKeeper.java
```java
public void register(int rc) {
    if (shouldAddWatch(rc)) {
        // 获取已有的 watcher 映射列表，若无则新增
        Map<String, Set<Watcher>> watches = getWatches(rc);
        synchronized(watches) {
            Set<Watcher> watchers = watches.get(clientPath);
            if (watchers == null) {
                watchers = new HashSet<Watcher>();
                watches.put(clientPath, watchers);
            }
            watchers.add(watcher);
        }
    }
}
```

`dataWatches` 是一个 `Map<String, Set<Watcher>>`，保存了节点路径和watcher的映射关系。

到此，客户端注册 Watcher 完毕。稍微总结一下：
1. 调用客户端API，传入 watcher；
2. 标记 request，封装 watcher 到 WatcherRegistration;
3. 向服务端发送 request；
4. 若响应成功，则注册 watcher 到 ZKWatcherManager 中进行管理；

值得一提的是，packet 对象在序列化时，并没有把 watcher 对象也一并序列化，以此降低网络传输的成本，以及服务端的内存压力。

### 服务端处理

从客户端接收到请求后，服务端会在 `FinalRequestProcessor.processRequest()` 方法中判断是否需要注册 watcher：
```java
public void processRequest(Request request) {
    /* ... */
    switch (request.type) {
        /* ... */
        case OpCode.getData: {
            /* ... */
            byte b[] = zks.getZKDatabase().getData(getDataRequest.getPath(), stat,
                    getDataRequest.getWatch() ? cnxn : null);
            rsp = new GetDataResponse(b, stat);
            break;
        }
    }
    /* ... */
}
```
可以看到，当 `getDataRequest.getWatch()` 为 true 时（也就是在客户端标记的），就会将当前的 `ServerCnxn` 对象传入 `ZKDatabase.getData()`方法中。

在这里 `ServerCnxn` 是 `Watcher` 接口的实现类，可以看做一个 `Watcher`：
```java
public abstract class ServerCnxn implements Stats, Watcher {
    /* ...*/
}
```

> ZKDatabase.getData()
> 
> 这个类维护了 ZK 的内存数据库，包括Session、DataTree和committed logs

```java
public byte[] getData(String path, Stat stat, Watcher watcher) 
throws KeeperException.NoNodeException {
    return dataTree.getData(path, stat, watcher);
}
```
> DataTree.getData()

```java
public byte[] getData(String path, Stat stat, Watcher watcher)
        throws KeeperException.NoNodeException {
    DataNode n = nodes.get(path);
    if (n == null) {
        throw new KeeperException.NoNodeException();
    }
    synchronized (n) {
        n.copyStat(stat);
        // 在这里将 watcher 添加到了 WatcherManager 的 dataWatches 中了
        if (watcher != null) {
            dataWatches.addWatch(path, watcher);
        }
        return n.data;
    }
}
```
> WatchManager.addWatch()
>
> WatchManager 从两个维度来保存 watcher

```java
public synchronized void addWatch(String path, Watcher watcher) {
    HashSet<Watcher> list = watchTable.get(path);
    if (list == null) {
        // don't waste memory if there are few watches on a node
        // rehash when the 4th entry is added, doubling size thereafter
        // seems like a good compromise
        list = new HashSet<Watcher>(4);
        watchTable.put(path, list);
    }
    list.add(watcher);

    HashSet<String> paths = watch2Paths.get(watcher);
    if (paths == null) {
        // cnxns typically have many watches, so use default cap here
        paths = new HashSet<String>();
        watch2Paths.put(watcher, paths);
    }
    paths.add(path);
}
```

到这里就完成了服务端的watcher注册和存储了。

再来看看是怎么触发的。

前面是注册了监听某节点数据变动的watcher，所以当在该节点修改数据时，会触发`NodeDataChanged`事件：
> DataTree.setData()

```java
public Stat setData(String path, byte data[], int version, long zxid,
        long time) throws KeeperException.NoNodeException {
    Stat s = new Stat();
    DataNode n = nodes.get(path);
    if (n == null) {
        throw new KeeperException.NoNodeException();
    }
    byte lastdata[] = null;
    synchronized (n) {
        lastdata = n.data;
        n.data = data;
        n.stat.setMtime(time);
        n.stat.setMzxid(zxid);
        n.stat.setVersion(version);
        n.copyStat(s);
    }
    // now update if the path is in a quota subtree.
    String lastPrefix;
    if((lastPrefix = getMaxPrefixWithQuota(path)) != null) {
      this.updateBytes(lastPrefix, (data == null ? 0 : data.length)
          - (lastdata == null ? 0 : lastdata.length));
    }
    // 在这里， 由 WatcherManager 触发
    dataWatches.triggerWatch(path, EventType.NodeDataChanged);
    return s;
}
```
> WatchManager.triggerWatch()

```java
public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    // 封装一个 WatchedEvent 对象
    WatchedEvent e = new WatchedEvent(type,
            KeeperState.SyncConnected, path);
    HashSet<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);
        if (watchers == null || watchers.isEmpty()) {
            if (LOG.isTraceEnabled()) {
                ZooTrace.logTraceMessage(LOG,
                        ZooTrace.EVENT_DELIVERY_TRACE_MASK,
                        "No watchers for " + path);
            }
            return null;
        }
        for (Watcher w : watchers) {
            HashSet<String> paths = watch2Paths.get(w);
            if (paths != null) {
                paths.remove(path);
            }
        }
    }
    for (Watcher w : watchers) {
        if (supress != null && supress.contains(w)) {
            continue;
        }
        // 这里执行 ServerCnxn 的 process() 方法
        w.process(e);
    }
    return watchers;
}
```
> ServerCnxn.process()

```java
@Override
synchronized public void process(WatchedEvent event) {
    // 请求头的xid（第一个参数）为“-1”，代表是一个通知
    ReplyHeader h = new ReplyHeader(-1, -1L, 0);
    /* ... */
    // 前面说过，要将 WatchedEvent 包装成 WatcherEvent
    WatcherEvent e = event.getWrapper();
    sendResponse(h, e, "notification");
}
```

可以看到，服务端触发 watcher 的逻辑是比较简单的。

### 客户端回调

在客户端，由`SendThread.readResponse()`处理服务端的的响应：
```java
void readResponse(ByteBuffer incomingBuffer) throws IOException {
    /* ... */
    // -1 ，代表是个通知
    if (replyHdr.getXid() == -1) {
        WatcherEvent event = new WatcherEvent();
        // 反序列化
        event.deserialize(bbia, "response");
    
        // ChrootPath 处理
        if (chrootPath != null) {
            String serverPath = event.getPath();
            if(serverPath.compareTo(chrootPath)==0)
                event.setPath("/");
            else if (serverPath.length() > chrootPath.length())
                event.setPath(serverPath.substring(chrootPath.length()));
            else {
                LOG.warn("Got server path " + event.getPath()
                        + " which is too short for chroot path "
                        + chrootPath);
            }
        }
        // 这里将 WatcherEvent 还原
        WatchedEvent we = new WatchedEvent(event);
        // 交给 EventThread，在下个轮询周期回调
        eventThread.queueEvent( we );
        return;
    }
    /* ... */
}
```

`EventThread` 是ZK中专门用来处理服务端通知事件的线程：
```java
public void queueEvent(WatchedEvent event) {
    if (event.getType() == EventType.None
            && sessionState == event.getState()) {
        return;
    }
    sessionState = event.getState();

    // 根据事件的各种属性，取出所有watcher
    WatcherSetEventPair pair = new WatcherSetEventPair(
            watcher.materialize(event.getState(), event.getType(),
                    event.getPath()),
                    event);
    // queue the pair (watch set & event) for later processing
    waitingEvents.add(pair);
}
```
客户端识别事件的类型，从响应的Watcher存储中去除watcher，并加到结果中：
```java
public Set<Watcher> materialize(Watcher.Event.KeeperState state,
                                Watcher.Event.EventType type,
                                String clientPath)
{
    Set<Watcher> result = new HashSet<Watcher>();
    switch (type) {
        /* ... */
        case NodeDataChanged:
        case NodeCreated:
            synchronized (dataWatches) {
                addTo(dataWatches.remove(clientPath), result);
            }
            synchronized (existWatches) {
                addTo(existWatches.remove(clientPath), result);
            }
            break;
        /* ... */
    }
    return result;
}
```

在 `EventThread` 的 `run()`方法中，将会不断地处理 waitingEvents 队列中的watcher。


### 小结
由此，便完成了Watcher机制源码的简要分析，从中可以发现Watcher的几个特性：
1. 一次性，无论是客户端还是服务端，一旦触发一个watcher，就将被移除；
2. 客户端串行执行，所有的watcher回调都是在一个队列中串行执行的，要注意不要因为一个watcher的处理逻辑影响了整体的回调；
3. 轻量，服务端不会讲事件的具体内容告知客户端，客户端注册watcher的时候也不会发送真实的watcher对象。

