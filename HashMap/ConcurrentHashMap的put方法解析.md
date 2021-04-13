# ConcurrentHashMap的put方法解析

本文主要对ConcurrentHashMap的原理进行解析，主要是对比JDK1.7和JDK1.8的不同实现方式。

## JDK1.7put实现

JDK1.7中引入Segment，Segment类通过继承ReentrantLock类，进行加锁，从而控制整个插入过程。
Segment数组也是一种数组加链表的结构方式，每个segement[i]都有一把锁，当某对<key,value>想要进行插入操作，首先要找对应segment数组对应的index，并获取锁，才能对HashEntry进行操作。 
![](E:\githubWork\JoneWangwz.github.io\image\20210313215706304.png)

```java
public V put(K key, V value) {
        Segment<K,V> s;//创建segment
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);//获取hash
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);//确定要segment[j]
        return s.put(key, hash, value, false);//插入
    }
```

第一步：首先通过ConcurrentHashMap中的put方法，计算key.hash的值，根据hash值定位到segment索引；

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);//①上锁
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;//②获取hash值
                HashEntry<K,V> first = entryAt(tab, index);//③确定插入到HashEntry[index]的首节点first
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {//3.1 此节点是否是null，若不是则需要遍历
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {//3.2遍历完成或此节点为空，创建HashEntry，插入；3.3此处要注意判断当前segement的count数量，是否需要进行rehash。
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();//④ 释放锁
            }
            return oldValue;
        }
```

第二步：进入到segment的`put()`方法，1.要上锁；2.计算出当前key.hash；3.确定hashEntry的位置，找到此hashEntry[i]的头节点first；3.1 此节点是否是null，若不是则需要遍历，3.2遍历完成或此节点为空，创建HashEntry，插入；3.3此处要注意判断当前segement的count数量，是否需要进行rehash；4 释放锁。

## JDK1.8put实现

JDK1.8较之前版本进行了改进，采用分段+CAS锁的方式保证线程安全，分段锁是基于synchronized关键字实现的。

`private static final float LOAD_FACTOR = 0.75f;`负载因子，默认值75%。当table使用率达到75%，为减少hash碰撞，将table的长度扩容一倍。
`static final int TREEIFY_THRESHOLD = 8;`默认值为8。当链表的长度大于8时，结构由链表转为红黑树。
`static final int UNTREEIFY_THRESHOLD = 6;`默认值为6，红黑树转为链表的阈值。
`private static final int MIN_TRANSFER_STRIDE = 16;`默认值16.当table扩容时，每个线程最少迁移table的槽位数。
`static final int MOVED = -1;`值为-1。当Node.hash为MOVED时，代表table正在扩容。
`static final int TREEBIN = -2;` 值为-2。代表此元素后接红黑树
`private transient volatile Node<K,V>[] nextTable;`table迁移过程的临时变量，在迁移过程中将元素全都迁移到nextTable上。
`private transient volatile int sizeCtl;`用来标志table初始化或扩容，不通的取值代表着不通的含义：
0：table没有初始化
-1:table正在初始化
小于-1：实际值为resizeStamp(n)<<RESIZE_STAMP_SHIFT+2，表明table正在扩容
大于0：初始化完成后，代表table最大存放元素的个数，默认0.75*n.
`private transient volatile int transferIndex;` 表示table容量从n扩容到2n时，是从索引n->1元素开始迁移， transferIndex代表当前已经迁移的元素的下标。
`ForwardingNode` ：一个特殊的Node节点，其hashcode=MOVED，代表此时table正在做扩容。当下一个线程向这个元素插入数据时，检查hashcode=MOVED，就会帮着扩容。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());//得到hash值
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                synchronized (f) {//f为当前索引第一个节点
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {//此处为值覆盖
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {//假设此链表有多个数据，会需要循环多次才能执行插入操作
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {//红黑树
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
                if (binCount != 0) {//在此处需要判断链表是否大于8，如果大于，则直接转红黑树。
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```



### 直接插入

![](E:\githubWork\JoneWangwz.github.io\image\20210313215748435.png)

在多线程中，通过使用synchronized加锁，实现同步。当前节点已存在数据，此时需要将key(a)设为`Node<K,V> f`，采用尾部插入法将key(c)插入到entry[1]中。

### 新创建表插入

![](E:\githubWork\JoneWangwz.github.io\image\20210313204638955.png)

第一步：当前concurrentHashMap为空时，需要进行表的初始化，假设线程T1抢到锁，首先创建大小为`DEFAULT_CAPACITY=16`的Node[]，第二轮循环，此时f指向key(a)要插入的地址，创建Node插入key(a)；
第二步：此时线程T2要插入数据，恰好也是此位置，此时f指向key(a)，e指向key(a)，接着`Node<K,V> pred = e;`，pred指向key(a)；
第三步：此时`(e = e.next) == null`成立，创建新节点并插入key(b)。最后判断链表值`if (binCount >= TREEIFY_THRESHOLD)`，如果满足，则需要将链表转化为红黑树。

### 插入过程扩容

鄙人此块源码实在是看不下去了，所以就简单描述一下其中的原理吧，请大家见谅。当concurrentHashMap正在扩容时，多线程可以帮助进行扩容，通常情况下通过设定`MAX_RESIZERS`设定最大帮助线程数量。插入过程扩容是比较复杂的一种插入数据的过程。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);//得到标志符号
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {//小于0说明还在扩容
                //判断是否标志发生了变化 ||扩容结束了 ||达到最大帮助线程||判断扩容转移下标是否在调整   
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                //将帮助线程增加1
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

当线程T3想要插入key(c)时，此时正好触发扩容，当前table[1]的`(fh = f.hash) == MOVED`，当前线程进行插入到table[1]时，将会触发帮忙扩容，当`if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))`帮助线程增加1的时候，触发`transfer(tab, nextTab);`帮忙转移，假设由32扩容到64当前由T1、T2、T3三个线程，那么可以设定T1转移0-15槽位，T2转移16-31，T3转移32-47，以此类推。