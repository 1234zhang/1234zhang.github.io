---
layout: '[layout]'
title: Java内存模型(四)
date: 2019-08-05 16:29:09
categories:
    - Java
    - 《java并发编程艺术》读书笔记
tags:
    - Java并发编程
---
## 简介
这篇主要是关于volatile相关的概念，比如volatile的内存语义，以及内存语义的实现

<!-- more -->

## volatile的特性
### 可见性
对一个volatile变量的读，总是能看到(任意线程)对这个volatile变量最后的写入。一个变量的单个读/写操作，与一个普通变量的读/写操作都是使用同一个锁来同步，他们之间的执行效果相同。
### 原子性
对任意单个volatile变量的读/写具有原子性，包括64位的long型和double型变量。但类似volatile++这种复合操作不具有原子性
## volatile写-读建立的happens-before关系
### 内存语义角度
volatile的读-写与锁的释放-获取具有相同效果。
### 实例代码与图解
```java
class VolatileExample{
    int a = 0;
    volatile boolean flag = false;

    public void writer(){
        i = 1;                     //1
        flag = true;               //2
    }
    public void reader(){
        if(flag){                  // 3
            int i = a;             // 4
        }
    }
}
```
happens-before关系
![](https://brandon-blog.oss-cn-beijing.aliyuncs.com/JMM/JavaVolatile%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.png)

## volatile读-写的内存语义
### 写一个volatile变量
当写一个volatile变量时，JMM会把该线程对应的本地内存种的共享变量值刷新到主内存。
### 读一个volatile变量
当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

## volatile内存语义的实现
### JMM实现volatile写/读的内存语义
JMM为了实现volatile内存语义，JMM会分别限制编译器重排序和处理器重排序。
volatile重排序表：
![avatar](https://brandon-blog.oss-cn-beijing.aliyuncs.com/JMM/volatile%E9%87%8D%E6%8E%92%E5%BA%8F%E8%A7%84%E5%88%99%E8%A1%A8.png)

### 内存屏障的插入
为了实现volatile的内存语义，编译器在生成字节码的时候，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。下面是基于保守策略的JMM内存屏障插入策略
- 在每个volatile写操作的前面插入一个StoreStore屏障
- 在每个volatile写操作的后面插入一个StoreLoad屏障
- 在每个volatile读操作的后面插入一个LoadLoad屏障
- 在每个volatile读操作的后面插入一个LoadStore屏障

volatile写插入内存屏障后生成的指令序列示意图
![avatar](https://brandon-blog.oss-cn-beijing.aliyuncs.com/JMM/voaltile%E6%8C%87%E4%BB%A4%E5%BA%8F%E5%88%97%E5%9B%BE.png)

### volatile写后面的StoreLoad屏障
此屏障的作用是避免volatile写与后面可能有的volaitle读/写操作重排序。因为编译器常常无法准确判断在一个volaitle写的后面是否需要插入一个StoreLoad屏障(比如一个voaltile写之后方法立即reture)，为了保证能正确实现volatile的内存语义，在每个voaltile写的后面插入一个StoreLoad屏障。因为volatile写-读内存语义的常见使用模式是：一个线程写volatile变量，多个读线程读同一个volatile变量。当读线程数量大大超过写线程时，选择在volatile写之后插入StoreLoad屏障将带来可观的执行效率的提升。

### 复杂volatile读写指令序列图
![avatar](https://brandon-blog.oss-cn-beijing.aliyuncs.com/JMM/%E5%A4%8D%E6%9D%82%E6%8C%87%E4%BB%A4%E5%BA%8F%E5%88%97%E5%9B%BE.png)