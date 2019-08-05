---
layout: '[layout]'
title: Java内存模型((六)
date: 2019-08-05 20:17:56
categories:
    - Java
    - 《java并发编程艺术》读书笔记
tags:
    - Java并发编程
---
## 简介
final域的读和写更像是普通的变量访问
<!-- more -->
## final域的重排序规则
### 编译器和处理器要遵守的两个规则
- 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序
- 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。
```java
public class FinalExmaple{
    int i;    // 普通变量
    final int j;    // final变量
    static FinalExample obj;
    public FinalExample(){
        i = 1;   // 写普通域
        j = 2;   // 写final域
    }
    public static void writer(){   // A线程执行写操作
        obj = new FinalExample();
    }
    public static void reader(){   // B线程执行读操作
        FinalExample finalExample = obj;   // 读对象引用
        int a = finalExample.i;    // 读普通变量
        int b = finalExample.j;    // 读final变量
    }
}
```

## 写final域的重排序规则
写final域的重排序股则禁止把final域的写重排序到构造函数之外。
- Jmm禁止编译器把final域的写重排序到构造函数之外
- 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。
### 重排序规则的确保
写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。

## 读final域的重排序规则
### 规则
读final域的重排序规则，在一个线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作(这个规则仅仅是针对处理器)。编译器会在读final域操作的前面插入一个LoadLoad屏障。
### 规则说明
初次读对象引用与初次读该对象包含的final域，这两个操作之间存在间接依赖关系。由于编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。大多数处理器也会遵守间接依赖规则，也不会重排序这两个操作。但少有处理器允许对存在间接依赖关系的操作做重排序，这个规则就是专门用来针对这种处理器的。
### 读重排序规则的确保
读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。
## final域为引用类型
### 示例代码
```java
public class FinalReferenceExample{
    final int[] intArray;
    static FinalReferenceExample obj;
    public FinalReferenceExample(){
        intArray = new int[1];
        intArray[0] = 1;
    }
    
    public static void writeOne(){
        obj = new FinalReferenceExample();
    }
    public static void writeTwo(){
        obj.intArray[0] = 2;
    }
    public static void reader(){
        if(obj != null){
            int temp1 = obj.intArray[0];
        }
    }
}
```
### 对编译器和处理器增加的约束
对于引用类型，写final域的重排序规则对编译器和处理器增加了如下约束：在构造函数内部对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序

## 为什么final引用不能从构造函数内“溢出”
### 保证写final域重排序规则的确保的前提
要想得到这个确保还需要一个保证：在构造函数内部，不能让这个被构造对象的引用为其他线程所见，也就是对象引用不能在构造函数中溢出
### 示例代码
```java
public class FinalReferenceExample{
    final int i;
    static FinalReferenceExample obj;
    
    public FinalReferenceExample(){
        i = 1;          // 1写final域
        this = obj;     // 2 this引用在此溢出
    }

    public static void writer(){
        new FinalReferenceExample();
    }
    public static void reader(){
        if(obj != null){   // 3
            int temp = obj.i;   //4
        }
    }
}
```
### 代码说明
假设一个线程A执行writer()方法，另一个线程B执行reader()方法。这里的操作2使得对象还未完成构造前就为线程B可见。即是这里的操作2是构造函数的最后一步，且在程序中操作2排在操作1后面，执行reader()方法的线程任然可能无法看见final域被初始化后的值，因为这里的操作1和操作2之间可能重排序，此时的final域可能还没有被初始化。

## final语义在处理器中的实现
由于X86处理器不会对写-写操作做重排序，所以在X86处理器中，写final域需要的StoreStore屏障会被省略掉。同样，由于X86处理器不会对存在间接依赖关系的操作做重排序，所以在X86处理器中，读final域需要的LoadLoad屏障也会被省略掉。也就是说，在X86处理器中，final域的读/写不会插入任何内存屏障。

最后两个偷个懒，放原文连接了，最后两个是happens-before跟双重锁定，等我哪天看到了，再来写个总结，连续写了两天，感觉有点疲了。。。。所以随后放两个传送门。。。。


[happens-before](https://mp.weixin.qq.com/s?__biz=MzAxODcyNjEzNQ==&mid=2247485263&idx=1&sn=995ba88b794aa7c3a54c55b5a9c398b1&chksm=9bd0aad7aca723c139473c59176db58d66d622649ecc606173f90c185287544dc8604b4441ef&scene=27#wechat_redirect)


[双重锁定](https://www.infoq.cn/article/double-checked-locking-with-delay-initialization)