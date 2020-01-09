---
layout: '[layout]'
title: 集合框架源码之list
date: 2020-01-09 14:43:31
tags:
---
# 写在前面的话
众所周知，在面试的时候，集合框架是一定会考察的知识点。这个也是Java基础中很重要的一部分，在特定的场景之下选择合适的数据结构，能够提高代码的效率。比如这篇博客将要总结的ArrayList和LinkedList。这个是很鲜明的对比，由于内部实现的不同，在查找频繁的场景之下使用ArrayList,在插入频繁的情况之下使用LinkedList。所以了解其内部实现，能够有利于我们选择合适的工具，去高效完成我们的需求。

接下来我将分别介绍ArrayList和LinkedList内部源码的实现

# ArrayList
ArrayList实现了list接口,可调节数组大小实现，具有全部可选择列表操作。并允许所有元素包括(null)。

```java
/*
1.ArrayList内部使用Object[]数组来存储ArrayList元素。
从这个可以得出该集合类不能存储基本数据类型。
这里涉及Java的泛型。
2. 初始化时， 首先将该数组设置为空数组，
在插入第一个元素的时候，再设置为默认长度
*/
transient Object[] elementData;
```
## 构造函数
主要内部提供一下三种类型的构造函数
```java
//共享数组实例，用于空实例
private static final Object[] EMPTY_ELEMENTDATA = {};
//在初始化时，提供初始长度，这个可以使用于事先知道存储大小，可以避免频繁扩容所导致的性能损失
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
//初始化不提供初始长度，则初始化为一个长度为0的数组。
public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
// 将一个集合类转换为列表，使用Arrays.copyOf。如果提供的集合类长度为0，则初始化为一个长度为0的数组。
 public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // 使用c.toArray可能不会返回一个正确的数组，所以我们使用Arrays.copyOf
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

```

## 扩容
在ArrayList中的属性中，关于容量的默认值分别如下
```java
//默认长度
private static final int DEFAULT_CAPACITY = 10;

/*
这里设置将容量的长度设置为Integer最大值减8的原因是：
这里是要分配的数组最大大小，由于一些虚拟机中在数组会保存一些数据头。
当扩容之后的容量大于MAX_ARRAY_SIZE，比较最小容量是否还是大于MAX_ARRAY_SIEZE，如果是则分配Integer.MAX_VALUE，否则分配MAX_ARRAY_SIZE长度

*/
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

### 自定义list容器的容量
```java
/*
    自定义容量的大小，参数minCapactity要满足
    1. 大于elementData本身长度
    2. elementData不是一个空数组
    3. minCapacity要大于默认值(10)
*/
public void ensureCapacity(int minCapacity) {
        if (minCapacity > elementData.length
            && !(elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
                 && minCapacity <= DEFAULT_CAPACITY)) {
            modCount++;
            grow(minCapacity);
        }
    }
```

### 扩容操作
将elementData容量扩展为至少可以容纳的最小容量指定的元素数。
```java
    private Object[] grow(int minCapacity) {
        return elementData = Arrays.copyOf(elementData,
                                           newCapacity(minCapacity));
    }
    private Object[] grow() {
        return grow(size + 1);
    }
```
### 扩容算法
```java
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        // 将原来容量扩展为原来的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果新的长度小于指定的最小长度。
        if (newCapacity - minCapacity <= 0) {
            // 判断ElementData是否是一个空数组
            if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
                return Math.max(DEFAULT_CAPACITY, minCapacity);
            if (minCapacity < 0) // overflow
            // 如果minCapacity是小于零的，抛出OOM异常
                throw new OutOfMemoryError();
            // 返回最小长度
            return minCapacity;
        }
        /*如果新的长度大于指定的最小长度，再判断newCapacity是否大于该数组允许的最大长度
        如果小于则扩容为新的长度
        如果大于则扩容为指定的长度。
         */
        return (newCapacity - MAX_ARRAY_SIZE <= 0)
            ? newCapacity
            : hugeCapacity(minCapacity);
    }

    // 查看自定义的值是否合法
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();

        return (minCapacity > MAX_ARRAY_SIZE)
            ? Integer.MAX_VALUE
            : MAX_ARRAY_SIZE;
    }

```

## 插入元素

### 每次插入单个元素
```java
    // 在指定的位置插入元素
     public void add(int index, E element) {
         // 判断index的值是否合法。
        rangeCheckForAdd(index);
        // 该属性与iterator和list iterator有关
        // 非预期修改情况下，会触发快速失败
        modCount++;
        final int s;
        Object[] elementData;
        if ((s = size) == (elementData = this.elementData).length)
            elementData = grow();
        // 对原来数组进行复制操作，空出index的位置。并插入element，size - index后的数据统一后移一位
        System.arraycopy(elementData, index,
                         elementData, index + 1,
                         s - index);
        elementData[index] = element;
        size = s + 1;
    }
    // 在数组尾部插入元素
    public boolean add(E e) {
        modCount++;
        add(e, elementData, size);
        return true;
    }
```

### 每次插入一个集合的元素
```java
    /*
        将一个集合的元素插入到该elementData中，
        按照插入集合的顺序返回
    */
    public boolean addAll(Collection<? extends E> c) {
        // 将插入的集合转换为数组
        Object[] a = c.toArray();
        modCount++;
        int numNew = a.length;
        if (numNew == 0)
            return false;
        Object[] elementData;
        final int s;
        if (numNew > (elementData = this.elementData).length - (s = size))
            elementData = grow(s + numNew);
        System.arraycopy(a, 0, elementData, s, numNew);
        size = s + numNew;
        return true;
    }
    /*
        在指定的位置插入某个集合的元素
    */
    public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);
        Object[] a = c.toArray();
        modCount++;
        int numNew = a.length;
        if (numNew == 0)
            return false;
        Object[] elementData;
        final int s;
        if (numNew > (elementData = this.elementData).length - (s = size))
            elementData = grow(s + numNew);

        int numMoved = s - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index,
                             elementData, index + numNew,
                             numMoved);
        System.arraycopy(a, 0, elementData, index, numNew);
        size = s + numNew;
        return true;
    }
```

## 删除元素

