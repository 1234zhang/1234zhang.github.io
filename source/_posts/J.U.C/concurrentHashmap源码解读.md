---
layout: '[layout]'
title: concurrentHashmap源码解读
date: 2020-05-17 15:58:52
tags:
    - java并发
categories:
    - Java
    - JUC
---
# 线程不安全的hashmap
总所周知，hashmap是一个线程不安全的集合结构。在多线程的情况下，链表会成环。

## 多线程的hashmap为什么会成环
这个在jdk8以下会出现的问题，因为jdk7采用的是头插法，在进行扩容的时候会对原有的元素进行rehash。rehash的结果会出现链表的拆分，原有的链表会被拆成两份。在并发的环境之下，某个节点先保存了next，另外的节点又在next后面添加了e，第一个线程再次执行，先头插了next，再头插了e。由于第二个线程再next后面加上了e节点，继续头插，e又跑到了next前面。就出现了环。这个情况再jdk8以后有所解决，jdk8中采用了尾插法，rehash之后每个元素的相对位置不会发生改，所以jdk8发生环的概率会小很多。但是jdk8还是线程不安全的，在并发的条件下，还是建议使用接下来要介绍的concurrenthashmap。

# ConcurrentHashMap
在jdk7和jdk8中，跟hashMap一样，在实现上有一些差异，下面分别来讲一讲jdk7和jdk8的不同

## ConcurrentHashMap在jdk7的实现

![jdk7的concurrentHashMap](https://www.javadoop.com/blogimages/map/3.png)

**concurrencyLevel** : 并行级别或者segment数量，默认是16个。也就是说ConcurrentHashMap有16和Segment，所以理论上这个时候可以支持16个线程并发写，只要他们的操作分别分布在不同的segment上。这个值可以在初始化的时候设置为其他的值，但是一旦初始化之后，是不可以扩容的。

**initialCapacity** : 初始容量，这个值指的是整个ConcurrentHashMap的初始容量，实际操作的时候需要平均分给每个Segment
**loadFactor** : 负载因子，因为Segment是不可扩容的，所以这个负载因子是给每个Segment内部使用的。

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (initialCapacity <= 0 || initialCapacity < 0 || concurrencyLevel <= 0) 
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS) 
        concurrencyLevel = MAX_SEGMENTS;
    int sshift = 0;
    int ssize = 1;
    // 计算并行级别sszie，保持并行级别是2的n次方
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 这里我们先使用默认值 concurrencyLevel 为16，sshift为4
    // 可以计算出segmentShift为28，segmentMask为15
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;

    if (initialCapacity > MAXIMUM_CAPACITY) initialCapacity = MAXIMUM_CAPACITY;
    // initialCapacity是设置整个map初始的大小
    // 根据initialCapacity计算segment数组中每个位置可以分到的大小
    // 如 initialCapacity 为64，那么每个Segment可以分到4个。
    int c = initialCapacity / ssize;
    if (c * ssize < initilaCapacity) ++c;
    // 默认MIN_SEGMETN_TABLE_CAPACITY是2， 也是有讲究的，因为这样的话，对于具体的segment上
    // 插入一个元素不会扩容，插入两个元素才会扩容
    int cap = c;
    while (cap < c) {
        cap <<= 1;
    }
    // 创建segment数组
    // 并创建数组的第一个元素，segment[0]
    Segment<K,V> s0 = new Segment<K,V>(loadFactor, (int)(cap*loadFactor),
                                        (HashEntry<K,V>()) new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[]) new Segment[ssize];
    // 往数组里面写入segment[0]
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segment[0]
    this.segments = ss;
}
```
初始化完成了， 我们得到一个Segment数组。
这里我们可以认为是使用 new ConcurrentHashMap() 无参构造器函数进行初始化的，那么初始化完成之后
- Segment数组长度为16，不可扩容
- Segment[i]的默认长度是2， 负载因子是0.75，得出阈值是1.5。那么在插入一个元素的时候不会扩容，插入两个元素的时候就会扩容。
- 这里初始化了segment[0]，其他位置还是null
- 当前的segmentShift的值为32 - 4 = 28， segmentMask的值是16 - 1 = 15，姑且把他们当作移位数和掩码。

### put过程分析
首先从put的主流程开始
```java
public V put(K key, V value) {
    Segment<K, V> s;
    if (value == null) throw new NullPointerException();
    // 1. 计算key的hash值
    int hash = hash(key);
    // 2. 根据hash值找到Segment数组中的位置j
    //      hash是32位，无符号有移segmentShift(28)位，剩下高4位
    //      然后和segmentMask(15)做一次与操作，也就是j是hash高4位就是segment数组的下标位置
    int j = (hash >>> segmentShift) & segmentMask;
    // 刚刚初始化的时候，只初始化了segment[0]，但是其他位置上还是null
    // ensureSegment(j) 对segment[j]进行初始化
    if ((s = (Segment<K, V>) UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);
    // 3. 插入新值到槽s中
    return s.put(key, hash, value, false);
}
```
这一层逻辑很简单，就是根据hash值找到相应的Segment，之后就是Segment内部的put操作

Segment内部是由数组加链表组成的
```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 在往该segment写入之前，需要先获得该segment的独占锁
    HashEntry<K, V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // 这个是segment内部的数组
        HashEntry<K, V>[] tab = table;
        // 利用hash值，找到应该放置的数组下标
        int index = (tab.length - 1) & hash;

        // first 是数组该位置出的链表的表头
        HashEntry<K, V> first = entryAt(tab, index);
        // 下面的循环主要处理该位置没有任何元素，和已经存在一个链表两种情况
        for (HashEntry<K, V> e = first;;) {
            if (e != null) {
                K k;
                if ((k = e.key) == key || (e.hash == hash & key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        // 覆盖旧值
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 继续顺着链表走
                e = e.next;
            } else {
                // node 是否为null，要看获取锁的过程(tryLock())
                // 如果不为空，则直接将node设置为表头
                // 如果为空，初始化并设置为表头
                if (node != null) node.setNext(first);
                else node = new HashEntry<K, V>(hash, key, value, first);
                int c = count + 1;
                // 如果超过了该segment的阈值，这个segment需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPCITY)
                    rehash(node); // 扩容操作
                else 
                    // 没有达到阈值，将node放到数组tab的index位置
                    // 其实就是将新的节点设置成为原链表的表头
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 解锁
        unlock();
    }
    return oldValue;
}
```
由于有独占锁的保护，所以segment内部的操作并不复杂。

接下来介绍put操作中的关键步骤

**初始化segment：ensureSegment**
ConcurrentHashMap 初始化的时候会初始化第一个槽segment[0]，对于其他的槽来说，在插入第一个值的时候，需要进行初始化。

这里需要考虑并发，因为很可能会有多个线程同时进来初始化同一个槽segment[k]，只要有一个成功就可以了。
```java
private Segment<K, V> ensureSegment(int k) {
    final Segment<K, V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE;
    Segment<K, V> seg;
    if ((seg = (Segment<K, V>) UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 这里看到为什么之前要初始化segment[0]了，
        // 使用segment[0]处的数组长度和负载因子来初始化segment[k]
        // 为什么要用“当前”， 因为segment[0]可能早就扩容过了
        Segment<K,V> proto = ss[0];
        int cap = proto.table.length;
        float lf = proto.loadFactory;
        int threshold = (int)(cap * lf);
        // 初始化segment[k]内部数组
        HashEntry<K, V> tab = (HashEntry<K, V>[]) new HashEntry[cap];
        if ((seg = (Segment<K, V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
            // 再次检查是否被其他线程初始化了
            Segment<K, V> s = new Segment<K, V>(lf, threshold, tab);
            // 使用while循环，内部用cas，当前线程成功设值或其他线程成功设置后，退出
            while ((seg = (Segment<K, V>) UNSAFE.getObjectVolatile(ss, u)) == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```
总的来说，ensureSegment(int k)比较简单，对于并发操作使用CAS进行控制。上面的while循环是为了cas失败后，将seg赋值返回。

**获取写入锁：scanAndLockForPut**
在往某个segment中put的时候，首先会调用node = tryLock() ? null : scanAndLockForPut(key, hash, value)，也就是说先进行一次tryLock()快速获取该segment的独占锁，如果失败，那么进入到scanAndLockForPut这个方法来获取锁。
```java
private HashEntry<K, V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K, V> first = entryForHash(this, hash);
    HashEntry<K, V> e = first;
    HashEntry<K, V> node =null;
    int retries = -1;

    // 循环获取锁
    while (!tryLock()) {
        HashEntry<K, V> f;
        if (retries < 0) {
            if (e == null) {
                if (node == null) {
                    // 进到这里说明该位置的链表是空的，没有任何元素
                    // 当然，进到这里的另外一个原因是tryLock()失败，所以该槽存在并发，不一定是该位置
                    node = new HashEntry<K, V>(hash, key, value, null);
                }
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                e = e.next;
        }
        // 重试次数如果超过MAX_SCAN_RETRIES(单核1 多核64)，那么不抢了，
        // 进入到阻塞队列等待锁，lock()是阻塞方法，直到获取锁之后返回。
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        // 到这里了，说明有新的元素进入到了链表，成为了新的表头
        // 所以这边的策略是，相当于重新走一遍这个scanAndLockForPut方法
        else if ((retries & 1) == 0 && (f = entryForHash(this, hash)) != first) {
            e = first = f;
            retries = -1;
        }
    }
    return node;
}
```
这个方法有两个出口，一个是tryLock()成功了，循环终止，另一个就是重试次数超过了MAX_SCAN_RETRIES，进到lock() 方法，此方法会阻塞等待，直到成功拿到独占锁。

这个方法看似复杂，其实就做了一件事请，就是获取该segment的独占锁，如果需要的话，顺便实例化一下node。

**扩容：rehash**
segment数组不能扩容，扩容是segment数组某个位置内部的数组HashEntry<K, V>[]进行扩容，扩容之后容量为原来的两倍。

首先我们要回顾一下触发扩容的地方，put的时候，如果判断该值的插入会导致该segment的元素个数超过阈值，那么先进行扩容，再插值。可以这个时候回去看一眼put的过程

该方法不需要考虑并发，因为到这里的时候，是持有该segment的独占锁的。
```java
// 方法参数中的node是这次扩容之后，需要添加到新数组的数据
private void rehash(HashEntry<K, V> node) {
    HashEntry<K, V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 2倍
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    // 创建新数组
    HashEntry<K, V>[] newTable = (HashEntry<K,V>[]) new HashEntry[newCapacity];
    // 新的掩码，例如从16扩容到32，那么sizeMask为31，对应二进制'000....0011111'
    int sizeMask = newCapacity - 1;
    // 遍历原数组，将原数组位置i处的链表拆分到新数组位置i和i + oldCap两个位置
    for (int i = 0; i < oldCapacity; i++) {
        // e是链表的第一个元素
        HashEntry<K, V> e = oldTable[i];
        if (e != null) {
            HashEntry<K, V> next = e.next;
            // 计算应该放置在新数组中的位置
            // 假设原数组长度为16， e在oldTable[3]处，那么idx只可能是3或者是3 + 16 = 19
            int idx = e.hash & sizeMask;
            if (next == null) {
                newTable[idx] = e;
            } else {
                // e 是链表的头节点
                HashEntry<K, V> lastRun = e;
                // idx是当前链表头节点e的新位置
                int lastIdx = idx;
                // 下面这个for循环会找到一个lastRun节点
                // 这个节点之后的所有元素是将要放到一起的
                for(HashEntry<K,V> last = next; last != null; last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                // 将lastrun及其之后的所有节点组成的这个链表放到lastIdx这个位置
                newTab[lastIdx] = lastRun;
                // 下面的操作是处理lastRun之前的节点
                // 这些节点分配在另一个链表中，也可能分配到上面的那个链表中
                for (HashEntry<K, V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K, V> n = newTable[k];
                    newTable[k] = new HashEntry<K, V>(h, p.key, v, n);
                }
            }
        }
    }
    // 将新来的node放到新数组中刚刚的两个链表之一的头部
    int nodeIndex = node.hash & sizeMask;
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = ndoe;
    table = newTable;
}
```
这里有两个for循环挨到一起的，但是我们仔细看了一下，如果没有第一个循环也是可以工作的。但是这个循环下来，如果lastRun的后面还有比较多的节点，那么这次循环就是值得的。因为我们只需要克隆lastRun前面的节点，后面的一串节点跟着lastRun走就行了，不用进行任何操作。但是如果lastRun都是链表最后一个节点的或者很靠后的节点，那么这次遍历就有点浪费了，不过doug lea说了，根据统计，如果使用默认值，大约只有1/6的节点需要克隆。

### get过程分析
相对于put过程，get过程就简单很多了。
1. 计算hash值，找到segment数组中具体位置
2. segment里面也是一个数组，根据hash找到数组中具体位置
3. 顺着链表进行查找即可。
```java
public V get(Object key) {
    Segment<K, V> s;
    HashEntry<K, V>[] tab;
    // 1. hash值
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 2. 根据hash找到对应的segment
    if ((s = (Segment<K, V> UNSAFE.getObjectVolatile(segments, u)) != null) && (tab = s.table) != null) {
        // 3. 找到segment内部相应的链表，遍历
        for (HashEntry<K, V> e = (HashEntry<K, V> UNSAFE.getObjectVolatile
        (tab, (long)(((tab.length - 1) & hash)) << TSHIFT) + TBASE);
         e != null; e = e.next) {
             K k;
             if ((k = e.key) == key || (e.hash == h && key.equals(k))) {
                 return e.value;
             }
         }
    }
    return null;
}
```

### 并发问题分析
在get的时候，没有加锁，所以我们就需要考虑并发问题。

添加节点的操作put和删除节点的操作remove都是加了独占锁的，所以这两个操作之间是不会有影响的。我们要考虑的问题是get的时候同一个segment发生了put或者remove的情况。

1. put操作的线程安全性
    - 初始化槽，使用CAS来初始化Segment中的数组
    - 添加节点到链表的操作是插入到表头的，所以，如果这个时候get操作在链表遍历的过程已经到来中间，是不会影响的。另一个并发问题就是get操作在put之后，需要保证刚刚插入表头的节点被读取，这个依赖于setEntryAt方法中使用的UNSAFE.putOrderedObject.
    - 扩容。扩容是新创建了数组，然后进行迁移数据，最后将newTable设置给属性table。所以，如果get操作此时也在进行，那么也没有关系，如果get先行，那么就是在旧的table上做查询操作；而put先行，那么put操作的可见性保证就是table使用了volatile关键字。
2. remove操作的线程安全性
    
    如果remove操作破坏的节点get操作已经过去了，那么这里不存在任何问题

    如果remove先破坏了一个节点，分两种情况考虑
    1. 如果此节点是头节点，那么需要将头节点的next设置为数组该位置的元素，table虽然使用了volatile修饰，但是volatile并不能提供数组内部操作的可见性保证，所以源码中使用了UNSAFE来操作数组，关于这部分的源码在setEntryAt
    2. 如果要删除的节点不是头节点，它会将要删除节点的后继节点接到前驱节点中，这里的并发保证就是next属性是volatile的。


# JAVA8关于concurrentHashMap的相关源码
jdk8中的结构如下：
![4](https://www.javadoop.com/blogimages/map/4.png)

jdk7中使用分段锁的方式进行并发控制，最大并发量是segment的数量。但是jkd8为了提高并发量，直接使用了一个较大的数组。同时也引入了红黑树这个数据结构，来解决hash冲突。在结构上和jdk8的hashmap结构相同，但是要保证并发性，所以代码会更加复杂一些。

## 初始化
**无参数的初始化**
```java
    public ConcurrentHashMap() {

    }
```

**指定初始容量的初始化**
```java
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
        throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
```
通过提供容量，计算来sizeCtl的大小，sizeCtl = [(1.5 * initialCapacity) 向上取得最近的2的n次方数]。如果initialCapacity是10，那么sizeCtl就是16，如果是11，那么就是32

## put过程分析
```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }
```

```java
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        // 取得hash值
        int hash = spread(key.hashCode());
        // 用于记录链表长度
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh; K fk; V fv;
            // 如果数组为空，或者长度为0
            if (tab == null || (n = tab.length) == 0)
            // 则初始化数组。
                tab = initTable();
            // 使用hash值，找到对应下标位置，得到第一个节点f
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果该位置为空，那么使用cas将元素放入这个位置
                // 如果cas成功，那么这次的put操作完成
                // 如果cas失败，则说明有竞争，那么就要进行下一次循环。
                if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                    break;                   // no lock when adding to empty bin
            }
            // 如果hash == MOVED，那么就要进行扩容
            else if ((fh = f.hash) == MOVED)
                // 帮助数据迁移。
                tab = helpTransfer(tab, f);
            // 如果onlyIfAbsent == true(说明要替换)
            // 而且key == 首节点的key
            // 那么我们就要替换这个value值。
            else if (onlyIfAbsent // check first node without acquiring lock
                     && fh == hash
                     && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                     && (fv = f.val) != null)
                return fv;
            else {
            // 到这里说明f是首节点，而且不为空
                V oldVal = null;
                // 获取首节点的监视器锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        // 判断首节点的hash值>0，则说明是链表
                        if (fh >= 0) {
                            // 用于累加，记录链表长度。
                            binCount = 1;
                            // 遍历链表
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果遇到相同key的值，判断是否需要替换
                                // 如果要替换，则替换之后，退出
                                // 如果不需要替换，则直接退出。
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                // 遍历到了链表的末端，则将这个值添加到末尾。
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key, value);
                                    break;
                                }
                            }
                        }
                        // 如果是树节点
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            // 调用红黑树的方式插入到末尾节点。
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                        // 如果key为空的话，就会初始化一个ReservationNode进行占为。
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
                if (binCount != 0) {
                    // 判断链表长度是否到达或者超过阈值，进行树化操作。和HashMap相同，也是8
                    if (binCount >= TREEIFY_THRESHOLD)
                        // 这个方法和hashMap有一些不同。就是不一定会进行树化
                        // 如果当前数组长度小于64的话，则会进行扩容，而不会进行树化。
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
上面是put的主流程，其中还有几个小流程没有顾及到，分别是数组初始化，树化，扩容和数据迁移等。
### 数组初始化
```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // 有其他的线程正在进行数据初始化。
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            // 如果没有其他线程进行数组初始化，那么使用cas将sizeCtl设置为-1，然后开始数组初始化。
            else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // DEFAULT_CAPACITY默认值为16。
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        // 初始化数组，设置为默认值16或者指定初始化的长度。
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        // 将这个数组赋值给table，table是使用volatile修饰的。
                        table = tab = nt;
                        // 将sc的值，设置为 n * 0.75
                        sc = n - (n >>> 2);
                    }
                } finally {
                    // 将sizeCtl设置为sc的值。
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

### 链表转红黑树
```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n;
    if (tab != null) {
        // 当数组长度小于MIN_TREEIFY_CAPACITY时候，即是小于64，则会进行扩容，而不会树化。
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        // b是头节点，要判断头节点是否为空。
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 对这个头节点进行加锁。
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    // 遍历链表，建立红黑树
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
                    // 将红黑树放到数组对应对位置上。
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

### 扩容和数据迁移操作
```java
// 在参数传入进来的时候，已经size翻了倍了。
private final void tryPresize(int size) {
    // c = size * 1.5 + 1 然后再向上取最接近2的n次方的那个数。
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        // 下面这个分支和数组初始化相同。
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
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
            if (U.compareAndSetInt(this, SIZECTL, sc,
                                    (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```

这个方法的核心在于sizeCtl的操作。首先将其设置为一个负数，然后再调用tranfer方法。整个tryPresize方法中可能会调用多次transfer这个方法。
接下来就要去看看这个方法里面到底做了啥
### 数据迁移：transfer
这个方法主要是将原来的tab数组的元素迁移到新的nextTab中。

虽然之前的tryPresize方法中多次调用transfer这个方法，但是不涉及多线程。其他的地方也在调用这个方法，比如put方法中有一个helpeTransfer方法，这个方法中就涉及到transfer方法的调用。

这里的并发机制是，愿数组有n那么长，就有n个任务，每个线程执行一个任务是最简单的，每做完一个任务再检测是否还有其他的任务没有做完，帮助迁移就可以了。而Doung lea使用了一个stride，每个线程执行一部分的任务。所以我们需要一个统筹调度者，来规划那个线程执行哪些任务，这个就是transferIndex的作用。

第一个发起数据迁移任务的线程会将transferIndex指向数组的末端，然后从后往前移动stride个位置，这stride个任务就会被分配给第一个线程。然后transferIndex指向新的位置。下一次分配stride个任务的起点就是transferIndex的新位置，然后再分配给下一个线程（其实也可能是同一个线程，如果这个线程把stride个任务执行完了的话）

```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // stride在单核的条件下等于n，多核模式下等于 (n >>> 3) / NCPU（NCPU是cpu的核心数）。stride的最小值是16
        // stride可以理解为步长，有n个位置要迁移。
        // 将n个任务分为多个任务包，任务包中有stride个小任务。
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        // 如果nextTab为null，则需要初始化这个数组
        // 外围调用这个方法的时候会保证第一次进行数据迁移的时候nextTab可以为空
        // 之后参与数据迁移的线程调用的时候nextTab不能为空
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            // nextTable是concurrenthashmap中的属性
            nextTable = nextTab;
            // transfer也是concurrenthashmap中的属性，用于控制数据迁移的位置
            transferIndex = n;
        }

        int nextn = nextTab.length;
        // ForwardingNode这个对象的构造方法，会生成一个node，其中key、value和next都为null
        // 但是这个node的hash值是MOVED
        // 在后面的步骤中，我们可以看到当i这个位置的数据完成迁移之后
        // 就会将i这个位置的值设置为这个node，用来提醒其他线程，这个位置数据迁移已经完成了。
        // 所以ForwardingNode这个对象，更多的是像一个标志位。
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        // advance指的是做完了一个位置的迁移工作，可以做下一个位置的了。
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        // i是位置索引，bound是边界值，注意是从后往前的。
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            // 下面这个while循环中
            // advance为true表示可以进行下一个位置的迁移
            // i = transferIndex； bound = transferIndex - stride
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                // 如果transfer <= 0，说明原数组的所有位置都有相应的线程进行处理了。
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSetInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    // bound是迁移任务的边界，可以看到nextBound的赋值位置而且是从后往前的。
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    // 所有迁移操作已经完成
                    nextTable = null;
                    // 将nextTab的值赋给table，完成迁移
                    table = nextTab;
                    // 设置sizeCtl为新数组长度的0.75倍。
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                // sizeCtl在进行数据迁移之间会将值设置为(rs << RESIZE_STAMP_SHIFT) + 2
                // 然后每个参与数据迁移对线程，在调用前都会将sizeCtl进行加一，表示线程领取了这个任务
                // 然后在执行数据迁移之后，会将sizeCtl的值减一，表示完成了这个任务。
                if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    // 到这里说明 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT
                    // 这样就会执行上面到if语句块，然后方法执行结束。
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            // 如果i位置是空的，那么就会将ForwardingNode这个node放到这个位置，表示迁移完成
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 如果这个位置的hash值是MOVED说明这个位置已经迁移完成了。
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                // 对该位置的链表头节点加锁，开始数据迁移。
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        // 头节点的hash值大于等于0， 说明是链表节点
                        if (fh >= 0) {
                            // 下面的数据迁移和jdk7差不多
                            // 找到原链表中的lastRun，lastRun后面的节点一起进行迁移
                            // lastRun前面的节点进行克隆，然后分到两个链表中。
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // 其中一个链表放在位置i
                            setTabAt(nextTab, i, ln);
                            // 另外一个链表放在i + n的位置
                            setTabAt(nextTab, i + n, hn);
                            // 然后将i的位置设置为fwd
                            // 其他线程看到该位置是MOVED，就不会进行数据迁移了。
                            setTabAt(tab, i, fwd);
                            // 将advance设置为true，表示数据迁移完成。
                            advance = true;
                        }
                        // 红黑树的迁移。
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
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof ReservationNode)
                            throw new IllegalStateException("Recursive update");
                    }
                }
            }
        }
    }
```
说起来，transfer这个方法并没有完成所有的迁移任务，每次调用这个方法只实现了transferIndex往前stride个位置的迁移。其他的还是需要外围函数的帮忙。

## put过程分析
1. 计算 hash 值
2. 根据 hash 值找到数组对应位置: (n - 1) & h
3. 根据该位置处结点性质进行相应查找
    - 如果该位置为 null，那么直接返回 null 就可以了
    - 如果该位置处的节点刚好就是我们需要的，返回该节点的值即可
    - 如果该位置节点的 hash 值小于 0，说明正在扩容，或者是红黑树，后面我们再介绍 find 方法
    - 如果以上 3 条都不满足，那就是链表，进行遍历比对即可

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // 判断头结点是否就是我们需要的节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 如果头结点的 hash 小于 0，说明 正在扩容，或者该位置是红黑树
        else if (eh < 0)
            // 参考 ForwardingNode.find(int h, Object k) 和 TreeBin.find(int h, Object k)
            return (p = e.find(h, key)) != null ? p.val : null;

        // 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```