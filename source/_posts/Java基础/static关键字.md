---
layout: '[layout]'
title: static、final关键字
date: 2019-08-10 16:33:20
tags:
    - java基础
categories:
    - Java
    - Java基础
---
## static 关键字
static变量也称作静态变量，静态变量和静态变量的区别是：静态变量被所有的对象所共享，在内存中只有一个副本，它当且仅当在类初始化加载时会被初始化。而非静态变量是对象所拥有的，在创建对象的时候被初始化，存在多个副本，各个对象拥有的副本互不影响。

### 静态变量

- 静态变量：又称类变量，也就是在这个变量属于类的，类所有的实例都共享静态你变量，可以直接通过类名来访问它。静态变量在内存中只存在一份
- 实例变量：每创建一个实例就会创建一个实例变量，他与实例共生死
```java
public class A {

    private int x;         // 实例变量
    private static int y;  // 静态变量

    public static void main(String[] args) {
        // int x = A.x;  // Non-static field 'x' cannot be referenced from a static context
        A a = new A();
        int x = a.x;
        int y = A.y;
    }
}
```
### 静态方法
静态方法在类加载的时候就存在了，它不依赖实例。所以静态方法必须有实现，也就是抽象方法不能是静态的
```java
public abstract class A {
    public static void func1(){
    }
    // public abstract static void func2();  // Illegal combination of modifiers: 'abstract' and 'static'
}
```
只能访问所属类的静态字段和静态方法，方法中不能有this和super关键字。
```
this关键字只能在方法内部调用，表示”调用方法的那个对象“的引用。
通常写this的时候表示“当前那个对象”或者“这个对象”
静态方法可以不用实例化对象直接使用，则违背了this关键字定义。所以静态方法中不能出现this关键字
```
```java
public class A {

    private static int x;
    private int y;

    public static void func1(){
        int a = x;
        // int b = y;  // Non-static field 'y' cannot be referenced from a static context
        // int b = this.y;     // 'A.this' cannot be referenced from a static context
    }
}
```

### 静态语块
静态语块在类初始化的时候运行一次
```java
public class A {
    static {
        System.out.println("123");
    }

    public static void main(String[] args) {
        A a1 = new A();
        A a2 = new A();
    }
}

output:
123
```

### 静态内部类
非静态内部类依赖于外部类的实例，而静态内部类不需要。
```java
public class OuterClass {

    class InnerClass {
    }

    static class StaticInnerClass {
    }

    public static void main(String[] args) {
        // InnerClass innerClass = new InnerClass(); // 'OuterClass.this' cannot be referenced from a static context
        OuterClass outerClass = new OuterClass();
        InnerClass innerClass = outerClass.new InnerClass();
        StaticInnerClass staticInnerClass = new StaticInnerClass();
    }
}

```

### 初始化顺序
```java
public static String staticField = "静态变量";

static{
    "静态语句块";
}

public String commonField = "普通变量";

{
    "普通语句块";
}

最后构造函数的初始化
public initialOrderTest(){
    System.out.println("最后构造函数的初始化");
}
```

存在继承的情况下，初始化顺序为：

- 父类（静态变量、静态语句块）
- 子类（静态变量、静态语句块）
- 父类（实例变量、普通语句块）
- 父类（构造函数）
- 子类（实例变量、普通语句块）
- 子类（构造函数）

[深入理解static关键字](https://www.cnblogs.com/dolphin0520/p/3799052.html)

## final关键字
### 数据
声明数据是常量，可以是编译时的常量，也可以时运行时被初始化之后不能被改变的量
- 对于基本类型：final使数值不变
- 对于引用类型，final使引用不变，不能引用其他对象，但是被引用对象本身可以更改
```java
final int x = 1;
// x = 2;  // cannot assign value to final variable 'x'
final A y = new A();
y.a = 1;

```
### 方法
声明方法不能被子类重写。

private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法

### 类
声明类不能被继承

![](https://brandonxcc.top/abc.jpg)