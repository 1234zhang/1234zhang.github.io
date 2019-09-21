---
layout: '[layout]'
title: static关键字
date: 2019-08-10 16:33:20
tags:
---
## 简介
static变量也称作静态变量，静态变量和静态变量的区别是：静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初始化加载时会被初始化。而非静态变量是对象所拥有的，在创建对象的时候被初始化，存在多个副本，各个对象拥有的副本互不影响。

<!-- more -->

## static用法
```
在《Java编程思想》中，这么阐述了static的用法
“可以在没有创建任何对象的前提下，仅仅通过类本身来调用static方法”
```
### 代码示例
```java
public class TestStatic{
    public static void main(String[] args){
        sayHello();
    }
    public static void sayHello(){
        System.out.println("hello world");
    }
}
```
我们不用创建TestStatic的对象，就可以直接调用sayHello这个静态方法。总之就是在没有创建对象的情况下对方法/变量的直接调用。

### 类初始化时，什么时候加载static变量。
- 静态存储区域的初始化
```java
class Bowl{
    public Bowl(int bowl){
        System.out.println("Bowl(" + bowl + ")");
    }
    public void f1(int mark){
        System.out.println("Bowl`s mark is " + mark);
    }
}

public Table{
    static Bowl bowl1 = new Bowl(1);
    public Table(int market){
        System.out.println("Table(" + market + ")");
        bowl1.f1(10);
    }
    public void f2(int mark){
        System.out.println("the table mark is " + mark):
    }
    static Bowl bowl2 = new Bowl(2);
}

public Cuppend{
    Bowl bowl3 = new Bowl(3);
    static Bowl bowl4 = new Bowl(4);
    public Cuppend(int market){
        System.out.println("Cuppend(" + market + ")");
        bowl3.f1(20);
    }

    public void f3(int mark){
        System.out.prinln("cuppend`s mark is " + mark);
    }
}

public class StaticTest{
    public static void main(String[] args){
        System.out.println("create new table in main function");
        new Table();
    }
    static Table table = new Table();
    static Cuppend cuppend = new Cuppend();
}
```
1.没有显式地使用static关键字，构造器也式静态方法。当首次创建X(某个类型为X的对象)对象时，或者首次访问X对象的静态变量或者静态方法，Java解释器必须查找类路径。以定位X.class文件。
2. 
