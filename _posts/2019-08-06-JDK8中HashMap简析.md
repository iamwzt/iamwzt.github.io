---
layout:     post
title:      JDK8中 HashMap 源码简析
date:       2019-08-06
author:     iamwzt
header-img: img/home-bg-o.jpg
catalog: true
tags:
    - HashMap
---

本文主要介绍JDK8中的HashMap中的get()、put()和resize()方法。

### 一、几个常量值
先介绍一下在源码中常见到的几个常量值。
```java
// 默认初始容量：就是数组长度默认为16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

// 最大容量：就是数组最大可达到的长度
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认加载因子：用于计算扩容阈值 threshold
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 树化阈值：一个链表中的节点达到该值就转化为红黑树
static final int TREEIFY_THRESHOLD = 8;

// 去树化阈值：红黑树中节点个数减少至该值就转化为链表
static final int UNTREEIFY_THRESHOLD = 6;

// 树化最小容量，≥ 4 × TREEIFY_THRESHOLD
// 数组长度小于该值时，某个节点的链表长度超出树化阈值，选择扩容，而非树化
static final int MIN_TREEIFY_CAPACITY = 64;
```
---

### 二、构造方法
HashMap 中一共有4种构造方法，列举如下： 

**注意：**  
除了第4个以其它map作入参的构造方法外，其余的都**不初始化table数组**！  
初始化工作是在put第一个元素时完成的。
```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    // 这个方法里头初始化数组
    putMapEntries(m, false);
}
```


---

### 三、get方法
```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

#### 3.1 getNode()
```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    
    // table 里头有元素， 并且相应下标位置不为 null 
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        
        // 看头节点是不是咯，不是再往下找
        if (first.hash == hash && 
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
        
            // 红黑树的查找
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### 3.2 hash()
这里顺带一提 jdk8 中 HashMap 的 hash()方法实现：
- key 为null，则为0，因此 `put(null, obj)` 时，都放在数组的第一个位置；
- 高低16位异或，称为hash扰动，低16位保留高16位的信息，减少冲突

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
---

### 四、put方法
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

#### 4.1 putVal()
```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 若还未初始化，则先初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 情况一：若数组对应下标还没有节点，则直接插入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        
        // 情况二：数组相应下标下有节点，且头结点的key和要put的key一样
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // 情况三：若是树节点，则用树的套路
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // 情况四：是链表则在这里遍历
        else {
            // 遍历列表，这里有两种情况：
            // 1. 若不存在则添到链尾，并判断是否需要树化
            // 2. 若存在则保存在变量 e 中
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 这里 -1 是因为还有个头节点
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // e 不为null，应该新值替换旧值
        if (e != null) { 
            V oldValue = e.value;
            // onlyIfAbsent 为 true时，则只有map中不存在才添加，存在不作改变
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    // HashMap 结构性的变化次数 +1，为fast-fail服务
    ++modCount;
    
    // 检查是否需要扩容（插入后再检查）
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

#### 4.2 treeifyBin()
这个方法不细讲，只是提一下树化的另一个条件（在几个常量中介绍过）：
`tab.length >= MIN_TREEIFY_CAPACITY`
```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 这里可以看到，并不是达到树化阈值就要变成红黑树的
    // 当数组tab长度小于64（默认值）时，选择扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```
#### 4.3 put 小结
可以看到put过程中有以下几种情况：
1. 数组的指定下标位置还为null
2. 数组的指定下标位置不为null，但是头节点就和待插入值就相同
3. 数组的指定下标位置不为null，头节点也不一样，而且是个红黑树节点
4. 数组的指定下标位置不为null，头节点也不一样，而且是个正常的链表
    1. 链表里不存在key相同的，则添加到**链表尾**（JDK7 中是添加到链表头）
    2. 链表内存在相同key，则替换

还有两点需要注意的是：
1. 先插入，后检查是否需要扩容；
2. 并不是达到树化阈值就要树化。

put 的流程图如下：
![put流程图](https://wzt-img.oss-cn-chengdu.aliyuncs.com/HashMap-putVal.png)

---

### 五、resize方法
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 若原数组长度 > (1<<30)， 则不再扩容，只将扩容阈值设为Integer的最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新数组长度 = 原数组长度 ×2 后，仍 < (1<<30)
        // 同时，原数组长度 > 16,
        // 则将新的扩容阈值 ×2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    // 到这说明还没初始化，就用默认值去设置
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    // 到这为止，就是计算扩容后新数组的长度，以及新的扩容阈值
    
    // 然后开始迁移数据
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 情况一：若没有hash冲突形成链表或树，则直接移到新的位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 情况二：对于节点是红黑树的处理
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 情况三：节点是链表的处理
                else { // preserve order
                    // 这里用两个链去分别保存要留在原下标，和去新下标的Node节点
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    
                    do {
                        next = e.next;
                        // 这里等于0，说明该节点在新数组的位置还是老数组的下标
                        // 如老数组大小为16，则二进制为 10000，比较第5个位是1或0来判断留原地还是迁移
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    // 迁移数据
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
在resize时，新的Node数组长度是原先的2倍。
由于长度都是2的指数，以及hash的特点，原数组中节点的要么在**原地保留**，要么移动到 **（原位置 + 原数组长度）** 的地方：

假设原数组长度为oldCap = 16，扩容后newCap = 32；
某Node节点的hash值为h的末5位为`10101`，则 h&(oldCap-1) = 0101，在下标为5的槽位；
扩容后，h&(newCap-1) = 10101，在下标为(5+16 = 21)的槽位。

由于这个特性，扩容后每个Node链表中，有一部分需要到新的下标位置，一部分还在原来的下标位置。

所以这里用了**lo**和**hi**两个链表，来分别存放这两部分数据。然后一把把这些数据迁移到新的数组中。

--over--

