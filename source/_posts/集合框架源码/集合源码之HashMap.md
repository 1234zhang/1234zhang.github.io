---
layout: '[layout]'
title: 集合源码之HashMap
date: 2020-04-18 15:21:10
tags:
    - Java集合
categories:
    - Java
    - 集合框架
    - HashMap
---

# 前言
同样作为Java集合框架中使用频率最多的数据结构之一，hashmap的地位可谓是相当的高。在面试中出现的频率也是相当的高，所以这篇重在讨论hashmap相关的源码。同时对比jdk7与jdk8的实现有什么异同。

# jdk7的实现
jdk7中的结构如下图：
![avatar](https://brandonxcc.top/jdk7的hashmap.png)
大方向上，hashmap里面是一个数组，然后数组中的元素是一个单向链表，用于解决hash冲突。

链表元素是一个实体类Entry，这个类主要包含四个属性(K值，Value值，指向下一个节点的位置，hash值)

capacity: 当前数组的容量，始终保持2^n，可以扩容，扩容后的大小是现数组容量的两倍

loadFacto: 负载因子，默认为0.75

threshold：阈值，等于loadFactor * capacity；
## put过程
### put
```java
    public V put(K key, V value){
        // 如果数组为空，首先初始化数组
        if(table == EMPTY_TABLE){
            inflateTable(threshold)
        }
        // 如果key为null， 则调用putForNullKey()这个方法，将entry放到table[0]中。
        if(key == null){
            return putForNullKey(value);
        }
        // 1.计算key的hash值
        int hash = hash(key);
        // 2.找到对应的下标
        int i = indexFor(hash, table.length);
        // 3. 遍历一下对应下标处的链表，查看是否有重复的key值，如果有则直接覆盖，put方法返回旧值
        for(Entry<K, V> e = table[i]; e != null; e = e.next){
            Object k;
            if(e.hash == hash && ((k = e.key) == key || key.equals(k))){
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        // 4. 不存在重复的key值，将此entry加入到对应数组下标的链表中
        addEntry(hash, key, value, i);
        return null;
    }
```

### 初始化数组
在第一个元素插入HashMap的时候做一次数组的初始化，就是先确定初始数组的大小，并计算数组扩容的阈值
```java
private void inflateTable(int toSize){
    // 保证数组大小一定是2的n次方
    // 比如new HashMap(20),那么这个数组的大小则是32
    int capacity = roundUpToPowerOf2(toSize);
    // 计算扩容阈值： capacity * loadFactor
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    // 初始化数组
    table = new Entry[capacity];
    initHashSeedAsNeeded(capacity); // 初始化hash种子，如果在需要的情况下。
}
```
**这里有一个疑问：为什么hashmap要将数组大小保证为2的N次方**
在计算Entry数组下标的时候
```java
static int indexFor(int h, int length){
    return h & (length - 1);
}
```
这里的h是 `int hash = hash(key.hashCode())`，也就是根据key的hashCode再进行一次hash计算出来的，length是Entry数组的长度。

一般我们利用hash计算一个数组的索引的时候，常用的方式是`h % length`，也就是求余数的方式，但是效率并不高。SUN的大师们经过实验发现，当容量在2的次方的时候，`h&(length - 1) == h % length`按位运算特别快。

### 计算数组的具体位置
```java
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return hash & (length-1);
}
```
取hash值的低n位，作为数组下标的位置。

### 添加节点到链表中
找到数组下标之后，会先进行key判重，如果没有重复，就准备将新值放入到链表的表头
```java
    void addEntry(int hash, K key, V value, int bucketIndex){
        // 如果当前的HashMap大小已经达到了阈值，并且新值要插入的数组位置已经有了元素，那么要扩容
        if((size >= threshold) && (null != table[bucketIndex])){
            // 扩容
            resize(2 * table.length);
            // 扩容之后重新计算hash值
            hash = (null != key) ? hash(key) : 0;
            // 重新计算扩容后的下标值
            bucketIndex = indexFor(hash, table.length);
        }
        // 将新的值放到链表的头部，然后size++
        createEntry(hash, key, value, bucketIndex);
    }
```
添加节点的主要逻辑是：先判断是否需要扩容，需要的话先扩容，然后再将这个新的数据加入到扩容后的数组的相应位置处的链表的表头

### 数组扩容
在插入新值的时候，如果当前的size已经达到了阈值，并且要插入的数组位置上已经有元素，那么就会触发扩容，扩容之后，数组大小为原来的2倍。
```java
void resize(int newCapacity){
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if(oldCapacity == MAXIMUM_CAPACITY){
        threshold = Integer.MAX_VALUE;
        return;
    }
    Entry[] newTable = new Entry[newCapacity];
    // 将原来数组中的值迁移到新的更大的数组中
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```
扩容就是用一个新的大的数组替换原来的小数组，并将原来数组中的值迁移到新的数组中。
由于是双倍扩容，迁移过程中，会将原来table[i]中的链表的所有节点，分拆到新数组的newTable[i]和newTable[i + oldLength]位置上。
## get
相对于put过程，get的过程更加简单
- 根据key计算hash值
- 找到相应的数组下标：hash & (length - 1)
- 遍历该数组位置处的链表，直到找到相等的key
```java
    public V get(Object key){
        // 如果key为null的话，就会被放到table[0]中，所以只要遍历table[0]处的链表就可以
        if(key == null){
            return getForNullKey();
        }
        Entry<K,V> entry = getEntry(key);
        return null == entry ? null : entry.getValue();
    }

    final Entry<K, V> getEntry(Object key){
        if(size == 0){
            return null;
        }
        int hash = (key == null) ? 0 : hash(key);
        
        for(Entry<K, V> e = table[indexFor(hash, table.length)]; e != null; e.next){
            Object K;
            if(e.hash == hash && ((k = e.key) == key || (key !+ null && key.equals(k)))){
                return e;
            }
            return null;
        }
    }
```

# jdk8的实现
jdk8对HashMap进行了一些修改，最大的不同就是引入了红黑树，所以其由`数组+链表+红黑树`组成

下面是jdk8的hashmap的结构：
![avatar](https://brandonxcc.top/jdk8的hashmap.png)

# jdk8 hashmap的一些简介
hashmap通常是采用桶装的hashtable即是使用数组加链表的方式，当链表中的元素过多的时候，就会将链表转换为红黑树。红黑树能够提供更快的查找方式，将时间复杂度从O(n) 降低到O(logn)。

hashmap中两个重要的参数Capacity和loadFactor。初始的桶大小和负载因子。负载因子是hash table允许扩容的大小(阈值, 一般是2的n次幂)。当数组中的元素个数大于了阈值之后，就会进行扩容，并rehash。
**为什么负载因子是0.75**
0.75很好的维持了时间与空间的平衡。太小了浪费空间，会造成频繁的resize()扩容操作，但是太大了hash冲突增加导致性能下降。

**为什么链表的长度大于等于8的时候链表会转换为红黑树**
根据泊松分布方程式： (exp(-0.5) * pow(0.5, k)/factorial(k))。在阈值为0.75的情况下，泊松分布概率函数中的参数λ=0.5。泊松分布用来估算在一段特定时间或者特定空间内发生成功事件的概率，即是长度为length的数组中放入hash地放入0.75*length数量的数据，数组中某一个下标放入k个数据的概率。

虽然引入TreeNode能够有效的提高hashmap的增删改查的性能，但是节点大小是链表节点的两倍。根据泊松分布考虑，每个节点大小为8的概率为0.00000006，千万分支一还小，这样的开销是值得的。

**当一棵红黑树的节点数小于等于6的时候会从红黑树转换为链表**
避免在一个节点为8的位置上频繁删除然后增加情况下，频繁的树转换和链表相互转换。

**最小树化容量**
如果entry的数组大小大于等于64的时候，就会被树化。64的原因是避免在扩容与树化之间产生冲突。
# put的过程
## putVal
所有的put方法都会调用这个私有方法。
```java
    /**
        @param hash key的hash值
        @param key key值
        @param value put的值
        @param onlyIfAbsent if true 只有在key不存在的时候才能进行put操作
        @param evict if false 这个数组处于建造模式(the table is in creation mode) 
    */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
        // 首次put的时候，会初始化数组的长度
        // 但是在jdk8中，首次扩容与后续扩容有些不一样，因为这次是数组从null初始化到默认的16或者自定义的初始值。
            n = (tab = resize()).length;
            // 找到具体的数组下标，如果这个位置没有值，那么直接初始化一下，Node并放置在这个位置。
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            // 如果这个位置有值
            Node<K,V> e; K k;
            // 判断这个位置的key和要存入的key值是不是相等，如果是则取出这个值
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
                // 如果该节点是红黑树的节点，调用红黑树的插入方法。
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 这里说明是链表的节点
                for (int binCount = 0; ; ++binCount) {
                    // jdk8中使用尾插法 
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 如果新插入的值是链表的第八个
                        // 会触发下面的方法，也就是将链表转为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 如果该链表找到了相等的key
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        //此时break，那么e为链表中[与要插入的新值的key相等]的node
                        break;
                    p = e;
                }
            }
            // e != null 说明新插入的key在原来链表中已经存在
            // 下面就是进行值覆盖，然后返回旧的值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        // 记录这个操作。
        ++modCount;
        // 如果这个插入操作之后，超过了阈值，则要进行扩容操作。
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

jdk7与jdk8的不一样的地方就是，Java7是先扩容再插入，Java8是先插入再扩容。

## 数组扩容
resize() 方法用于初始化数组或者数组扩容，每次扩容之后，容量就要变成原来的两倍，并进行数据迁移。
```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        // 原数组已经被初始化过
        if (oldCap > 0) {
            // 如果原来数组的长度大于最大容量，则将阈值扩大到最大
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            // 将长度扩大到原来的两倍，而且小于极限长度，而且原来长度大于最小长度
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                     // 将阈值扩大到原来的两倍
                newThr = oldThr << 1; // double threshold
        }
        // 对应使用new HashMap(int initialCapacity)初始化之后。
        // 使用这个构造方法，只会初始化阈值，不会初始化长度
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
        // 对应使用 new HashMap()之后，第一次put
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        // 用新数组大小初始化新数组
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
            // 如果是第一次put，即是没有老数组，则这里就直接返回了，不会进行数据的迁移
        table = newTab;
        // 下面开始老数组的数据迁移
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    // 如果该位置上只有一个链表只有一个元素，那么直接计算hash位置，然后复制就可以。
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                        // 如果该位置上是一颗红黑树
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                    // 下面要进行链表的拆分
                    // 拆分成两个链表，放到新数组中，并且要保留原来的先后顺序
                    // lo前缀的对应一条链表，hi前缀的对应另外一条链表
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            // 如果扩容之后还在原来位置，则添加到第一条链表的尾部
                            if ((e.hash & oldCap) == 0) { 
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            // 如果扩容之后不在原来位置，而是移动了2次幂的位置，则添加到第二条链表的尾部
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null); // 遍历这条链表
                        if (loTail != null) {
                            // 原索引放到bucket
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            // 原索引+oldCap放到bucket处
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
元素在重新计算了hash之后，因为n变成了2倍，那么n - 1的mask范围在高位多1bit，因此index位置就会发生变化。

## get过程分析
1. 计算key的hash值，根据hash值找到对应数组的下标
2. 判断数组位置处的元素是否是我们要找的元素，如果不是继续
3. 判断该元素的类型是否是TreeNode，如果是，用红黑树的方式获取数据
4. 如果不是红黑树，遍历链表找到相等的key
```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 判断第一个节点是不是就是需要的
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 判断是否是红黑树
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);

            // 链表遍历
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
## 红黑树与AVL树的区别
AVL树比红黑树保持更加严格的平衡，AVL树从树根到最深的的路径最多为1.44lg(n + 2)，而在红黑树中最多为2lg(n + 1), 越深说明花费的时间越多。从这里可以看出AVL树的查找效率更高。但是这是以更多的旋转导致的更慢的插入和删除操作为代价的。因此如果希望查找更多，则使用AVL树
而红黑树更加通用，在添加、删除和查找方面有更好的表现。AVL树查找更快，代价是删除和添加速度较慢。


## 参考文章连接
1. [resize之后的rehash](https://blog.csdn.net/qq_27093465/article/details/52270519)
2. [Java7/8 中的 HashMap 和 ConcurrentHashMap 全解析](https://www.javadoop.com/post/hashmap)
3. [为什么负载因子是0.75，以及为什么链表转红黑树的阈值是8](https://blog.csdn.net/reliveIT/article/details/82960063)
4. [为什么初始化长度是2的n次方](https://blog.csdn.net/sybnfkn040601/article/details/73194613)
5. [为什么使用红黑树而不使用AVL树](https://blog.csdn.net/21aspnet/article/details/88939297)

![avatar](https://brandonxcc.top/20200418.jpg)