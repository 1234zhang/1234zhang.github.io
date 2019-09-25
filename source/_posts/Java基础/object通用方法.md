---
layout: '[layout]'
title: object通用方法
date: 2019-09-22 09:35:52
tags:
    - java基础
categories:
    - Java
    - Java基础
---
```java
java Object是所有类的父类，提供11个基本方法
public final native Class<?> getClass()
public native int hashCode()
public boolean equals(Object obj)
protected native Object clone() throws CloneNotSupportedException
public String toString()
public final native void notify()
public final native void notifyAll()
public final native void wait(long timeout) throws InterruptedException
public final void wait(long timeout, int nanos) throws InterruptedException
public final void wait() throws InterruptedException
protected void finalize() throws Throwable { }
```

## getClass()方法
使用final修饰，不允许子类重写，并且也是一个native方法。

返回当前`运行时`的Class对象，比如下列代码中n是一个Number的实例，但是Java数值的默认类型是Integer类型，所以getClass方法返回的是java.lang.Integer
```java
"str".getClass // class java.long.String
Number n = 0;
Class<? extends Number> = n.getClass() // class java.lang.Integer.
```
## equals()
### 1.等价关系
- 自反性
```java
    x.equals(x); // true;
```

- 对称性
```java
    x.equals(y); // true;
    y.equals(x); // false;
```

- 传递性
```java
    if(x.equals(y) && y.equals(z)){
        x.equals(z);
    }
```

- 一致性
```java
 无论调用多少次，结果不变
    x.equals(z) == x.equals(z); // true;
```

- 与null比较
```java
对任意一个非null对象，调用equals都是false
x.equals(null); // false;
```
### 相等与等价
- 对于基本类型，== 判断两值是否相等，基本类型没有equals方法
- 对于引用类型，== 判断两个引用的是否为同一个对象，而equals()判断两个对象是否相等。
```java
Integer a = new Integer(1);
Integer b = new Integer(1);
System.out.println(a.equals(b)); // true;
System.out.println(a == b);  // false;
```
### 实现
- 检查是否为同一个对象引用，如果是直接返回true
- 检查是否为同一类对象，如果不是返回false；
- 将Object 对象进行转型
- 逐一检查每个关键域是否相等

```java
public class EqualsExample{
    private int x;
    private int y;
    private int z;

    public EqualsExample(int x, int y, int z){
        this.x = x;
        this.y = y;
        this.z = z;
    }

    @Ovrride
    public boolean equals(Object o){
        if(o == this) return true;
        if(o == null || getClass != o.getClass){
            return false;
        }
        EqualsExample that = (EqualsExample) o;
        if(this.x != that.x) return false;
        if(this.y != that.y) return false;
        return this.z == that.z;
    }
}
```
## hashCode()
hashCode()返回序列值，而equals()判断两个对象是否等价。两个等价对象的序列值一定相等，序列值相等的两个对象不一定等价。
在覆盖了对象的equals方法的时候，应该同时覆盖对象的hashCode方法，保证两个等价对象的序列值也相等。
下列代码中，两个等价的对象，并将它们添加到hashSet中。我们希望两个对象等价的时候，hashSet中只添加一个对象，因为EqualsExample没有覆盖hashCode方法，所以两个等价对象的序列值不同，这个时候hashSet中就添加了两个对象

```java
EqualExample e1 = new EqualExample(1, 1, 1);
EqualExample e2 = new EqualExample(1, 1, 1);
System.out.println(e1.equals(e2)); // true
HashSet<EqualExample> set = new HashSet<>();
set.add(e1);
set.add(e2);
System.out.println(set.size());   // 2
```
理想的散列函数应当具有均匀性，即不相等的对象应当均匀分布到所有可能的散列值上。这就要求了散列函数要把所有域的值都考虑进来。可以将每个域都当成 R 进制的某一位，然后组成一个 R 进制的整数。R 一般取 31，因为它是一个奇素数，如果是偶数的话，当出现乘法溢出，信息就会丢失，因为与 2 相乘相当于向左移一位。

一个数与 31 相乘可以转换成移位和减法：`31*x == (x<<5)-x`，编译器会自动进行这个优化。
```java
@Override
public int hashCode() {
    int result = 17;
    result = 31 * result + x;
    result = 31 * result + y;
    result = 31 * result + z;
    return result;
}
```
## toString()
默认返回 ToStringExample@4554617c 这种形式，其中 @ 后面的数值为散列码的无符号十六进制表示。

## clone()
clone()是Object的一个protected方法，一个类如果不重写clone方法，那么久不能显示的调用clone()
```java
    public class CloneTest{
        private int a;
        private int b;
    }

    CloneExample e1 = new CloneExample();
// CloneExample e2 = e1.clone(); // 'clone()' has protected access in 'java.lang.Object'
```

重写clone方法如下实现
```java
public class CloneExample {
    private int a;
    private int b;

    @Override
    public CloneExample clone() throws CloneNotSupportedException {
        return (CloneExample)super.clone();
    }
}
```
```java

CloneExample e1 = new CloneExample();
try {
    CloneExample e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
```
```java
java.lang.CloneNotSupportedException: CloneExample

如果重写clone方法的类，没有实现Cloneable接口，所以会抛出CloneNotSuportedException
```
应该注意的是，clone() 方法并不是 Cloneable 接口的方法，而是 Object 的一个 protected 方法。Cloneable 接口只是规定，如果一个类没有实现 Cloneable 接口又调用了 clone() 方法，就会抛出 CloneNotSupportedException。
### 浅拷贝
拷贝对象和原始对象的引用的同一个对象

```java
public class ShallowCloneExample implements Cloneable {

    private int[] arr;

    public ShallowCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected ShallowCloneExample clone() throws CloneNotSupportedException {
        return (ShallowCloneExample) super.clone();
    }
}
```

```java
ShallowCloneExample e1 = new ShallowCloneExample();
ShallowCloneExample e2 = null;
try {
    e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
e1.set(2, 222);
System.out.println(e2.get(2)); // 222
```
### 深拷贝
拷贝对象与原始对象引用的是不同对象
```java
public class DeepCloneExample implements Cloneable {

    private int[] arr;

    public DeepCloneExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }

    @Override
    protected DeepCloneExample clone() throws CloneNotSupportedException {
        DeepCloneExample result = (DeepCloneExample) super.clone();
        result.arr = new int[arr.length];
        for (int i = 0; i < arr.length; i++) {
            result.arr[i] = arr[i];
        }
        return result;
    }
}

```
```java
DeepCloneExample e1 = new DeepCloneExample();
DeepCloneExample e2 = null;
try {
    e2 = e1.clone();
} catch (CloneNotSupportedException e) {
    e.printStackTrace();
}
e1.set(2, 222);
System.out.println(e2.get(2)); // 2
```
### clone()替代方法
使用 clone() 方法来拷贝一个对象即复杂又有风险，它会抛出异常，并且还需要类型转换。Effective Java 书上讲到，最好不要去使用 clone()，可以使用拷贝构造函数或者拷贝工厂来拷贝一个对象。

```java
public class CloneConstructorExample {

    private int[] arr;

    public CloneConstructorExample() {
        arr = new int[10];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = i;
        }
    }

    public CloneConstructorExample(CloneConstructorExample original) {
        arr = new int[original.arr.length];
        for (int i = 0; i < original.arr.length; i++) {
            arr[i] = original.arr[i];
        }
    }

    public void set(int index, int value) {
        arr[index] = value;
    }

    public int get(int index) {
        return arr[index];
    }
}

```
```java
CloneConstructorExample e1 = new CloneConstructorExample();
CloneConstructorExample e2 = new CloneConstructorExample(e1);
e1.set(2, 222);
System.out.println(e2.get(2)); // 2
```
## wait()
wait也是一个native方法，被final关键字修饰，所以也不能被重写
wait方法会让当前线程等待直到另外一个线程调用对象的notify或者notifyAll方法，或者超过参数设定timeout时间。
跟notify()和notifyAll方法一样，只有当前线程是监视器对象的所有者，否则会抛出IllegalMonitorStateException异常。

要使得wait线程被唤醒并进入等待队列，只要发生下面事件中的某一件:
- 其他某个线程调用此对象的notify()方法，并T被选择为唤醒线程
- 其他某个线程调用此对象的notifyAll()方法。
- 某个线程调用Thread.interrupt()中断这个等待线程
- 时间到了timeout参数设置时间，线程被自动唤醒。如果timeout时间设置为0，则会一直等待
## notify()与notifyAll()
### notify
notify()是一个native方法，被final关键字修饰，所以是一个不能被重写的方法。
唤醒一个在监视器上等待的对象。如果所有线程都在这个对象上等待，那么就会随机选择一个线程。一个线程在对象监视器等待可以调用wait方法。
直到当前线程放弃对象上的锁以后，被唤醒线程才能继续处理。被唤醒的线程将以常规的方式与该对象上主动同步的其他线程进行竞争
notify方法只能被作为此对象监视器的所有者的线程来调用。一个线程想要称为对象监视器的所有者有下列三种方法：
- 执行对象的同步实列方法
- 使用synchronized内置锁
- 对于Class类型的对象，执行同步静态方法。
`因为notify只能在对象监视器所有者线程中调用，否则会抛出IllegalMonitorStateException异常`
### notifyAll
notifyAll方法与notify方法相同，只不过notifyAll方法是唤醒所有在监视器对象上等待的对象。

### 根据wait和notify方法实现的消费者/生产者模型
```java
public class WaitNotifyTest {

    public static void main(String[] args) {
        Factory factory = new Factory();
        new Thread(new Producer(factory, 5)).start();
        new Thread(new Producer(factory, 5)).start();
        new Thread(new Producer(factory, 20)).start();
        new Thread(new Producer(factory, 30)).start();
        new Thread(new Consumer(factory, 10)).start();
        new Thread(new Consumer(factory, 20)).start();
        new Thread(new Consumer(factory, 5)).start();
        new Thread(new Consumer(factory, 5)).start();
        new Thread(new Consumer(factory, 20)).start();
    }

}

class Factory {

    public static final Integer MAX_NUM = 50;

    private int currentNum = 0;

    public void consume(int num) throws InterruptedException {
        synchronized (this) {
            while(currentNum - num < 0) {
                this.wait();
            }
            currentNum -= num;
            System.out.println("consume " + num + ", left: " + currentNum);
            this.notifyAll();
        }
    }

    public void produce(int num) throws InterruptedException {
        synchronized (this) {
            while(currentNum + num > MAX_NUM) {
                this.wait();
            }
            currentNum += num;
            System.out.println("produce " + num + ", left: " + currentNum);
            this.notifyAll();
        }
    }

}

class Producer implements Runnable {
    private Factory factory;
    private int num;
    public Producer(Factory factory, int num) {
        this.factory = factory;
        this.num = num;
    }
    @Override
    public void run() {
        try {
            factory.produce(num);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}


class Consumer implements Runnable {
    private Factory factory;
    private int num;
    public Consumer(Factory factory, int num) {
        this.factory = factory;
        this.num = num;
    }
    @Override
    public void run() {
        try {
            factory.consume(num);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
注意的是Factory类的produce和consume方法都将Factory实例锁住了，锁住之后线程就成为了对象监视器的所有者，然后才能调用wait和notify方法。
```java
输出：
produce 5, left: 5
produce 20, left: 25
produce 5, left: 30
consume 10, left: 20
produce 30, left: 50
consume 20, left: 30
consume 5, left: 25
consume 5, left: 20
consume 20, left: 0
```

## finalized()
finalize方法是一个protected方法，Object类的默认实现是不进行任何操作。

该方法的作用是实例被垃圾回收器回收的时候触发的操作，就好比 “死前的最后一波挣扎”。


![](https://brandonxcc.top/天气之子.jpg)