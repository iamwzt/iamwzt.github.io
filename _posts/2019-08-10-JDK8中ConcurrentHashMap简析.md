---
layout:     post
title:      JDK8中ConcurrentHashMap简析
date:       2019-08-10
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - ConcurrentHashMap
---

相关链接：[JDK8中 HashMap 源码简析](https://iamwzt.github.io/2019/08/06/JDK8%E4%B8%ADHashMap%E7%AE%80%E6%9E%90/)

本文主要介绍JDK8中ConcurrentHashMap的get()、put()方法，其中put()方法涉及到扩容，较为复杂，需要用心看下。

### 几个常量值
相对HashMap，ConcurrentHashMap 多了很多常量，这里先罗列一些，混个眼熟，后面用到的再细讲。

这几个和HashMap是一样的
```java
private static final int MAXIMUM_CAPACITY = 1 << 30;
private static final int DEFAULT_CAPACITY = 16;
private static final float LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;
static final int UNTREEIFY_THRESHOLD = 6;
static final int MIN_TREEIFY_CAPACITY = 64;
```
这几个是独有的：
```java
// 创建map的快照时用
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
// 这个是老版本的遗留，分16个segment
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
// 最小步长，数据迁移时用，可以理解成一个线程最少负责迁移这么多任务吧
private static final int MIN_TRANSFER_STRIDE = 16;
// 下两个和sizeCtl有关
private static int RESIZE_STAMP_BITS = 16;
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
// 最大参与扩容的线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
// 几种特殊的hash值，代表不同的节点状态
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
// 系统的CPU核数
static final int NCPU = Runtime.getRuntime().availableProcessors();
```
一头雾水？那就对了，先有个眼熟，往下接着看。

### 几个Node内部类
ConcurrentHashMap 里有几个Node的子类，分别表示几种不同状态下的节点。如图所示：
![这里有图]()

这里的状态就是上面常量中的那几个`MOVED`/`TREEBIN`/`RESERVED`。
#### Node：普通的节点
Node是常规链表状态下的节点
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
    // 省略其它
}
```
#### TreeNode：红黑树节点
```java
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    // 省略其它
}    
```
#### TreeBin：持有红黑树根节点的一个东东
TreeBin 不保存 key 和 value ，而是持有红黑树的根节点；它的hash值是`TREEBIN`（-2）
```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // values for lockState
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock
    // 省略其它
}    
```
#### ForwardingNode：代表数据正在迁移
ForwardingNode 代表正在迁移数据（扩容中），持有扩容后的table表；它的hash值是`MOVED`（-1）
```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    // 省略其它
}    
```

#### ReservationNode：占位节点
computeIfAbsent 和 compute 方法中用到的占位节点，hash值为`RESERVED`（-2）
```java
static final class ReservationNode<K,V> extends Node<K,V> {
    ReservationNode() {
        super(RESERVED, null, null, null);
    }

    Node<K,V> find(int h, Object k) {
        return null;
    }
}
```
---

### get方法
```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        // 这里是通过Unsafe类的方法来获取指定下标的节点
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 正常的链表节点是不会小于0的，只有在扩容、树节点的根节点、反序列化时才会小于0
        else if (eh < 0)
            // 在上述的三种情况下，find方法也各有实现
            return (p = e.find(h, key)) != null ? p.val : null;
        // 这里是普通的链表搜索了
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```
#### spread()
- 和HashMap一样的是：同样需要高低位扰动（spread）；
- 不一样的是：1、没有对key为null的处理；2、和HASH_BITS与后确保符号位为0；

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```
---

### put方法
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```

### putVal方法
```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 不同于HashMap，不接收key或value为null的节点
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            // 初始化table表
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 指定下标没有节点时，则不加锁，直接CAS插入
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 指定下标的节点hash为MOVED时，说明正在扩容，就帮忙扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        // 正常情况下，就来到这个分支了
        else {
            V oldVal = null;
            // 先加个锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        // 和HashMap不同，这里是1起步的
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // 这里是红黑树的插值
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 若大于树化阈值，就树化
                // 这里和HashMap一样，不一定就树化的，还可能扩容
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 这里会将binCount+1，若需要，就扩容
    addCount(1L, binCount);
    return null;
}
```

#### initTable()
插入第一个节点时，需要初始化table数组
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl < 0，说明已有其它线程在初始化了，自旋等待一下
        if ((sc = sizeCtl) < 0)
            Thread.yield(); 
        // 将 sizeCtl 设为 -1，代表本线程要初始化了，锁住
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    // sc > 0，说明有个初始值，否则就用默认值
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 下面这句等价于 sc = n * 0.75
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

#### treeifyBin()
```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        // 当table表太小（默认小于64），会选择扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```
#### tryPresize()
有两个地方会调用这个方法：
1. putAll()方法，被put的数量不定；
2. treeifyBin()，此时是扩容，size必是2的指数

```java
private final void tryPresize(int size) {
    // 扩容时进来已是2的指数，如果大于等于最大容量的一半，就直接设为最大容量值
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        // 调整为 2 的指数
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        // 这里和初始化数组的逻辑一样
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
            // 2、后续进到这个分支，nextTab为新建的数组nt
            if (sc < 0) {
                Node<K,V>[] nt;
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            // 1、会先进这个分支，此时nextTab为null
            // 这里将 SIZECTL 设置为 (rs << RESIZE_STAMP_SHIFT) + 2
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

#### transfer()
```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 单线程下的任务数就是整个数组
    // 多线程：
    //  1、 每个线程的任务数 = 数组长度n/8/CPU数
    //  2、 最小值为 MIN_TRANSFER_STRIDE（16）
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;
    
    // 第一次进来时是null，进行初始化
    // 第一个发起数据迁移的线程进这个方法时还没有 nextTab
    if (nextTab == null) {
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        // nextTable 是 ConcurrentHashMap 中的属性
        nextTable = nextTab;
        // transferIndex 也是 ConcurrentHashMap 的属性，用于控制迁移的位置
        // 从数组的最后一个元素开始往前
        transferIndex = n;
    }
    int nextn = nextTab.length;
    
    // ForwardingNode 翻译过来就是正在被迁移的 Node
    // 这个构造方法会生成一个Node，key、value 和 next 都为 null，关键是 hash 为 MOVED
    // 后面我们会看到，原数组中位置 i 处的节点完成迁移工作后，
    //    就会将位置 i 处设置为这个 ForwardingNode，用来告诉其他线程该位置已经处理过了
    //    所以它其实相当于是一个标志。    
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    
    // advance 指的是做完了一个位置的迁移工作，可以准备做下一个位置的了
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    
    // i 是位置索引，bound 是边界，注意是从后往前
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        
        // 此处循环就是为当前线程划出一块任务区域
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;

            // 这里 transferIndex 一旦小于等于 0，说明原数组的所有位置都有相应的线程去处理了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }

            // 看括号中的代码，nextBound 是这次迁移任务的边界，注意，是从后往前
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 所有的迁移操作已经完成
            if (finishing) {
                nextTable = null;
                table = nextTab;
                // 重新计算 sizeCtl：n 是原数组长度，所以 sizeCtl 得出的值将是新数组长度的 0.75 倍
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            // 之前在tryPresize()方法中说过，sizeCtl 在迁移前会设置为 (rs << RESIZE_STAMP_SHIFT) + 2
            // 然后，每有一个线程参与迁移就会将 sizeCtl 加 1（下面的helpTransfer()方法），
            // 这里使用 CAS 操作对 sizeCtl 进行减 1，代表做完了属于自己的任务
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                // 任务结束，方法退出
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                // 所有的迁移任务都做完了，下个循环就会进入到上面的 if(finishing){} 分支了
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果位置 i 处是空的，没有任何节点，那么放入刚刚初始化的 ForwardingNode “空节点”
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 该位置处是一个 ForwardingNode，代表该位置已经迁移过了
        else if ((fh = f.hash) == MOVED)
            advance = true;
        // 此处，便是要开始迁移了
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // hash值大于0，则为普通的Node链表的迁移
                    if (fh >= 0) {
                        // 这个runBit就是为了区分这个链表中的节点应该移到新数组的哪个位置
                        // 如原数组长度为16（10000），则runBit除了第5位可能为1或0，其它位都为0
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        // 这个循环是为了从链表里找到一个节点（lastRun）
                        // 在此之后的所有节点都将在新数组的原下标，或（原下标+原数组长度）
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 为0代表需要在新数组的原下标，放在ln中(low)
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        // 否则方法hn(high)中
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        // 处理链表中lastRun之前的节点
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 将low和high两个链表分别放在新数组的指定下标下
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        // 设置原数组的该下标位置为ForwardingNode，Hash值为MOVED（-1），代表已迁移
                        setTabAt(tab, i, fwd);
                        // advance 设置为 true，代表该位置已经迁移完毕
                        advance = true;
                    }
                    // 红黑树的迁移
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        // 如果一分为二后，节点数少于 8，那么将红黑树转换回链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

#### helpTransfer()
```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    // 判断节点是否是 ForwardingNode 类型
    // 一般来说，扩容时会把头结点变成这个类型，并且hash值为MOVED（-1）
    // 相比普通的Node节点，它还持有将要扩容的新数组对象
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            // 每多一个线程参与迁移，SIZECTL 就 CAS 加1
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

最后放一张put的简易流程图：
![put流程图](https://wzt-img.oss-cn-chengdu.aliyuncs.com/ConcurrentHashMap-putVal.png)
