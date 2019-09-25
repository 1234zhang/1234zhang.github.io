---
layout: '[layout]'
title: Java基本数据类型与操作符
date: 2019-09-20 21:18:48
tags:
    - java基础
categories:
    - Java
    - Java基础
---

Java数据类型分为内置类型和扩展类型两大类。内置类型就是基础类型，如short, int, long, float, double, boolean, byte, char. 扩展类型就是基本类型的扩展类。

## 内置类型

| 类型名称 | 类型定义 | 类型取值 |
| :----: | :----: | :----: |
| boolean | 布尔值，做二元判断 | true，false |
| byte | 八位有符号整数 | 最小值-128, 最大值 127 |
| short | 16位有符号整数 | 最小值-32768， 最大值32767 | 
| int | 32位有符号整数 | 最小值-2147483648，最大值2147483647|
| long | 64位有符号整数 | 最小值(-2^63), 最大值是 2^63-1 |
| float | 32位浮点数 | 1.4E-45~3.4028235E38|
| double | 64位浮点数 | 4.9E-324~1.7976931348623157E308 | 
| char | 16位Unicode字符 | |

### 基本数据类型的注意
- boolean 只有两个值：true、false，可以使用 1 bit 来存储，但是具体大小没有明确规定。JVM 会在编译时期将 boolean 类型的数据转换为 int，使用 1 来表示 true，0 表示 false。JVM 支持 boolean 数组，但是是通过读写 byte 数组来实现的。

- 浮点数在使用的时候难以确定。在实际应用中，在经历了一系列的运算之后，在逻辑上应该等于某个值，但是使用(\==)运算符的时候，可能会返回false,或者比较两个浮点数大小的时候，可能会出现相反的结果。所以我们在比较浮点数的时候，不能简单的使用(\==, <,>)运算符，应该规定一个精度，再比较。

## 包装类型
每个基本数据类型，都有一个包装类型。包装类型都是使用final关键字修饰的类，所以不能继承。

```java
    Integer x = 2; // 装箱，使用了Integer.valueOf(2)
    int y = x; //拆箱,使用了x.intValue();
```

### 缓冲池
new Integer(123) 和 Integer.valueOf(123)的区别是
- new Integer(123) 每次都会创建一个新的对象
- Integer.valueOf(123) 会首先使用缓冲池中的对象，如果没有才重新创建一个新的对象。

```java
    Integer x1 = new Integer(123);
    Integer x2 = new Integer(123);
    System.out.println(x1 == x2); // false
    Integer y1 = Integer.valueOf(123);
    Integer y2 = Integer.valueOf(123);
    System.out.println(y1 == y2); // true
```
valueOf方法的主要思想是，首先在缓冲池查找相关值是不是在缓冲池中，如果在就直接返回，没有再重新创建。
```java
    public static Integer valueOf(int i){
        if(i >= IntegerCache.low && i <= IntegerCache.high){
            return IntegerCache.cache[i + (-IntegerCache.low)];
        }
        return new Integer(i);
    }
```

在Java8中，缓冲池的大小扩展为-128~127
```java
// 创建缓冲池的代码。
    static final int low = -128;
    static final int high;
    static final Integer cache[];
    static{
        int h = 127;
        // 从属性文件中获取配置
        String integerCacheHighPropValue = 
            sum.misc.VM.getSgetSavedProperty("java.lang.Integer.IntegerCache.high");
        if(integerCacheHighPropValue != null){
            int i = praseInt(intgerCacheHighPropValue);
            i = Math.max(i, 127);
            // 数组最长的长度是Integer.MAX_VALUE;
            h = Math.min(i, Integer.MAX_VALUE - (-low) - 1);
        }catch( NumberFormatException nfe) {
            // If the property cannot be parsed into an int, ignore it.
        }
    }
    high = h;

    cache = new Integer[(high - low) - 1];
    int j = low;
    for(int k = 0; k < cache.length; k++)
        cache[k] = new Integer(j++);

    // range [-128, 127] must be interned (JLS7 5.1.7)
    assert IntegerCache.high >= 127;
```
在自动装箱的过程中，会调用valueOf(int i)方法。因此在多个相同的数值而且在缓冲池中，就会直接返回缓冲池中的对象。

基本数据类型的缓冲池大小：
- boolean values true and false
- all byte values
- short values between -128 and 127
- int values between -128 and 127
- char in the range \u0000 to \u007F
如果值在缓冲池中，就会直接返回缓冲池中的内容。

## String
跟其他包装类一样，String是不可变对象，不能被继承。
在java8中，String使用char[]存储数据
```java
    public final class String implements Serializable,
     Comparable<String>,
      CharSequence{
        private final char value[];
    }
```
在Java9之中，使用byte数组存储数据，并用coder来标识使用哪种编码
```java
    public final class String implements Serializable,
    Comparable<String>, CharSequence{
        private final byte value[];
        private final byte coder;
    }
```
为了保证String数组的不变性，使用final来修饰value数组，保证在数组初始化之后，不会被改变，并且没有提供修改value数组的方法，从而保证了String的不变性。

### 不变性的用处
#### 可以缓存hash值
因为String中的hash值经常被使用，例如使用String作为hash的key值，不可变特性可以使得hash值可以不变，只需要进行一次运算。
#### String pool的需要
如果一个String的值已经被创建过了，那么就会从String pool中获取到引用。
#### 安全性
因为String对象经常被用做参数传递，比较Request中的参数就是使用String来传输的。如果在网络较差的时候，String不能保证不可变性，在传输过程中就会被改变。也在网络连接中，如果String被改变，改变String的乙方以为连接他的是其他主机，但并不是。
#### 保证线程安全
在并发条件之下，其他线程不能改变String的值，这就保证了线程安全。

### String， StringBuffer， StringBuilder
#### 可变性
String 不可变
StringBuffer和StringBuilder是可变的
#### 线程安全
String 是线程安全的， StringBuffer是线程安全的，内部使用Synchronized
StringBuilder是线程不安全的。

### String Pool
首先查看一下jvm中的内存简略分布
![](https://brandonxcc.top/jvm.jpg)

String类型的常量池比较特殊，主要使用方法有两种
- 直接使用双引号声明出来的String对象会直接存储在常量池中
- 使用new关键字声明的String对象，可以使用intern方法，将String对象存放在常量池中。intern会首先判断String对象是否在常量池中，如果不存在就会将String对象存放在常量池中。

```java
    // 直接使用双引号声明String对象
    String a = "aaa";
    String b = "aaa";
    // 因为使用双引号声明的String对象会首先在String pool中查找，如果有相同的对象，则直接返回引用。
    System.out.println(a == b); // true
```
下例是首先使用new String(),创建两个不同的String对象。而s3，s4是s1.intern之后取得的一个引用。intern首先将s1存放在常量池中，然后返回一个引用，所以s3，s4是同一个引用。
```java
    String s1 = new String("bbb");
    String s2 = new String("bbb");
    System.out.println(s1 == s2)  // false;

    String s3 = s1.intern();
    String s4 = s1.intern();
    System.out.println(s3 == s4);  // true;
```
[深入解析String#intern](https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html)

#### new String("abd")
在使用new 关键字的时候，会创建两个对象(前提是String pool中没有"abd")
- "abd" 是字符串字面量，在编译期的时候会创建一个字符串对象，指向"abd"这个字符串字面量
- 在Java堆中创建一个字符串对象

```java
public class NewStringTest {
    public static void main(String[] args) {
        String s = new String("abc");
    }
}


使用javap -verbose进行反编译，获得

// ...
Constant pool:
// ...
   #2 = Class              #18            // java/lang/String
   #3 = String             #19            // abc
// ...
  #18 = Utf8               java/lang/String
  #19 = Utf8               abc
// ...

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=2, args_size=1
         0: new           #2                  // class java/lang/String
         3: dup
         4: ldc           #3                  // String abc
         6: invokespecial #4                  // Method java/lang/String."<init>":(Ljava/lang/String;)V
         9: astore_1
// ...

```

在 Constant Pool 中，#19 存储这字符串字面量 "abc"，#3 是 String Pool 的字符串对象，它指向 #19 这个字符串字面量。在 main 方法中，0: 行使用 new #2 在堆中创建一个字符串对象，并且使用 ldc #3 将 String Pool 中的字符串对象作为 String 构造函数的参数。

## 操作符
### 赋值操作符
"=" 是一个赋值操作符，表示将右边的值复制给左边。右边可以是常数、变量或者复杂表达式(只要能表示一个确切的数)。左边必须是一个明确的、已命名的变量。在对一个对象“赋值”的时候，准确的来说，是操作的对象的引用。实际是上
将“引用”从一个地方复制到另一个地方。

```java
public class Dog{
    private String name;
    
    public Dog(String name){
        this.name = name;
    }
    public String getName(){
        return this.name;
    }
    public static void main(){
        Dog dog = new Dog("abc");
        Dog dog1 = dog;
        System.out.println(dog1.getName); // abc
    }
}

```

#### 参数传递
Java方法中的参数的是以值传递的形式传入方法中，并非引用传递

以下代码中Dog dog的dog是一个指针，存储的是对象的地址。在将一个参数传入方法中的时候，本质上是将对象的地址的值传入到形参中。因此在方法中使指针引用其他对象，那么这两个指针指向的是两个不同的地址。在改变其中一方的所指对象内容的时候，另外一方不受影响

```java
public class Dog {

    String name;

    Dog(String name) {
        this.name = name;
    }

    String getName() {
        return this.name;
    }

    void setName(String name) {
        this.name = name;
    }

    String getObjectAddress() {
        return super.toString();
    }
}
```

```java
public class PassByValueExample {
    public static void main(String[] args) {
        Dog dog = new Dog("A");
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        func(dog);
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        System.out.println(dog.getName());          // A
    }

    private static void func(Dog dog) {
        System.out.println(dog.getObjectAddress()); // Dog@4554617c
        dog = new Dog("B");
        System.out.println(dog.getObjectAddress()); // Dog@74a14482
        System.out.println(dog.getName());          // B
    }
}
```

### float和double
Java不能隐式执行向下转型，因为这个会使得精度降低
```java
1.1字面量属于double，不能将1.1直接赋值给float型，这是向下转型
// float t = 1.1;
1.1f才是float类型
float t  = 1.1f;
```

