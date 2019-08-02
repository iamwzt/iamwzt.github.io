
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
    else {               // 到这说明还没初始化，就用默认值去设置
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    
    // 到这为止，就是计算扩容后新数组的长度，和新的扩容阈值
    
    // 建一个新数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 若没有hash冲突形成链表或树，则直接移到新的位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 对于节点是红黑树的处理
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 这里用两个链去分别保存要留在原下标，和去新下标的Node节点
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    
                    do {
                        next = e.next;
                        // 这里等于0，说明该节点在新数组的位置还是这个原先的下标
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

所以这里用了**lo**和**hi**两个链表，来分别存放这两部分数据。然后一把把这些数据放到新的数组中。

