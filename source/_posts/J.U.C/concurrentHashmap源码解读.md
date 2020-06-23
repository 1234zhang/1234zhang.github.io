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

            }
        }
    }
}
```