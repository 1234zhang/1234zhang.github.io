---
layout: '[layout]'
title: 集合框架源码之list(一)
date: 2020-01-09 14:43:31
tags:
    - Java集合
categories:
    - Java
    - 集合框架
    - ArrayList
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
```java
    // 删除指定位置的某个元素
    public E remove(int index) {
        //检查index是否合法
        Objects.checkIndex(index, size);
        final Object[] es = elementData;

        @SuppressWarnings("unchecked") 
        E oldValue = (E) es[index];
        fastRemove(es, index);

        return oldValue;
    }
    // 删除在列表中第一次出现的指定元素。如果列表不含有指定元素，则不变。
    //
    public boolean remove(Object o) {
        final Object[] es = elementData;
        final int size = this.size;
        int i = 0;
        // break:标号 -> 表示：终止结束到标签
        found: {
            if (o == null) {
                for (; i < size; i++)
                    if (es[i] == null)
                        break found;
            } else {
                for (; i < size; i++)
                    if (o.equals(es[i]))
                        break found;
            }
            return false;
        }
        fastRemove(es, i);
        return true;
    }
    // 专用的删除方法，跳过边界检查并且不返回删除的值
    private void fastRemove(Object[] es, int i) {
        modCount++;
        final int newSize;
        if ((newSize = size - 1) > i)
            // 删除指定位置之后，将后续元素向前移动
            System.arraycopy(es, i + 1, es, i, newSize - i);
        es[size = newSize] = null;
    }
```

### 删除所有元素
```java
// 清除所有元素
    public void clear() {
        modCount++;
        final Object[] es = elementData;
        for (int to = size, i = size = 0; i < to; i++)
            es[i] = null;
    }
```
### 删除某个指定范围内的元素
```java
    /*
        删除指定范围内的元素，将删除后的元素向左移动(减少其索引)
        通过使用(fromIndex - toIndex)元素来缩短列表。
    */
    protected void removeRange(int fromIndex, int toIndex) {
        if (fromIndex > toIndex) {
            throw new IndexOutOfBoundsException(
                    outOfBoundsMsg(fromIndex, toIndex));
        }
        modCount++;
        shiftTailOverGap(elementData, fromIndex, toIndex);
    }
    // 通过向左移动，来减少从lo到li的距离
    private void shiftTailOverGap(Object[] es, int lo, int hi) {
        System.arraycopy(es, hi, es, lo, size - hi);
        for (int to = size, i = (size -= hi - lo); i < to; i++)
            es[i] = null;
    }
```

### 删除某个包含在某个集合中的所有元素
```java
    // 集合c就是要在elementData数组中要删除的元素
    // 该集合不能为空
    public boolean removeAll(Collection<?> c) {
        return batchRemove(c, false, 0, size);
    }
    // 该方法表示删除不在集合c中的元素
    // 同样该集合c不能允许空元素
     public boolean retainAll(Collection<?> c) {
        return batchRemove(c, true, 0, size);
    }
```

## 修改元素
```java
    public E set(int index, E element) {
        // 确定修改元素的索引是否合法
        Objects.checkIndex(index, size);
        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
```

## 查找元素
```java
    // 查看元素是否在列表中
    public boolean contains(Object o) {
        return indexOf(o) >= 0;
    }

    // 返回元素第一次出现的索引，如果不包含则返回-1
    public int indexOf(Object o) {
        return indexOfRange(o, 0, size);
    }
    // 在数组中查找元素，返回第一次出现的位置
    int indexOfRange(Object o, int start, int end) {
        Object[] es = elementData;
        if (o == null) {
            for (int i = start; i < end; i++) {
                if (es[i] == null) {
                    return i;
                }
            }
        } else {
            for (int i = start; i < end; i++) {
                if (o.equals(es[i])) {
                    return i;
                }
            }
        }
        return -1;
    }
    // 返回最后元素最后一次出现的位置。
    public int lastIndexOf(Object o) {
        return lastIndexOfRange(o, 0, size);
    }

    int lastIndexOfRange(Object o, int start, int end) {
        Object[] es = elementData;
        if (o == null) {
            for (int i = end - 1; i >= start; i--) {
                if (es[i] == null) {
                    return i;
                }
            }
        } else {
            for (int i = end - 1; i >= start; i--) {
                if (o.equals(es[i])) {
                    return i;
                }
            }
        }
        return -1;
    }
```

## 序列化
```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // 写出元素计数以及任何隐藏的内容
        int expectedModCount = modCount;
        //执行默认的反序列化/序列化过程。将当前类的非静态和非瞬态字段写入此流
        s.defaultWriteObject();

        // 写出大小作为与clone()行为兼容的容量
        s.writeInt(size);

        // 按照正确的顺序写出元素
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {

        // 读入大小以及隐藏元素
        s.defaultReadObject();

        // 读入容量
        s.readInt(); // ignored

        if (size > 0) {
            // like clone(), allocate array based upon size not capacity
            SharedSecrets.getJavaObjectInputStreamAccess().checkArray(s, Object[].class, size);
            Object[] elements = new Object[size];

            // 按照顺序读入所有元素
            for (int i = 0; i < size; i++) {
                elements[i] = s.readObject();
            }

            elementData = elements;
        } else if (size == 0) {
            elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new java.io.InvalidObjectException("Invalid size: " + size);
        }
    }
```

关于列表的序列化中，没有使用默认的序列化，而是自己实现了序列化方法是因为，动态数组在操作的时候可能会造成大量的空元素，如果所有空元素都序列化了，会造成一定的资源浪费。

## 为什么说ArrayList不适合频繁插入和删除操作
在增加删除ArrayList中经常会调用System.arraycopy这个效率很低的操作来复制数组，所以导致ArrayList在插入和删除操作中效率不高

(小小说一句：这个是2020年第一篇博客。许个小小的愿望，希望自己2020年能坚持写博客，希望能拿到一个大厂的正式offer，稳定的搬砖。。。)
![](https://brandonxcc.top/日出.jpg)