---
layout: '[layout]'
title: 匿名内部类以及lambda表达式
date: 2019-09-25 14:46:57
tags:
    - java基础
categories:
    - Java
    - Java基础
---
## 匿名内部类
### 内部类
可以将一个类的定义放在另一个类的定义内部，这就是内部类。内部类允许把一些逻辑相关的类组织放在一起。
```java
内部类的创建。
public class OutClass{
    class InnerClass{
        private int i = 1;
        public int value(){return i;}
    }

    class Inner2Class{
        private String label;
        Inner2Class(String whereTo){
            label = whereTo;
        }
        public String readLabel(){return label;}
    }

    public void ship(){
        InnerClass a = new InnerClass();
        Inner2Class b = new Inner2Class();
        System.out.println(a.value() + b.readLabel());
    }
}
```
如果想从外部类的非静态方法之外的任意位置创建某个内部类的对象，就要具体指明这个对象的类型：OuterClassName.InnerClassName

### 匿名内部类
```java
定义一个接口类
public interface Test{
    int value();
}


public class InnerTest{
    public Test test(){
        return new Test(){
            private int value = 1;
            public int value(){
                return value;
            }
        };
    }
    public static void main(String[] args) {
        InnerTest t = new InnerTest();
        System.out.println(t.test().value());
    }
}
```

test()创建一个继承自Test的匿名类对象，通过new表达式返回引用被自动向上转型为对Test的引用，看起来好像是要创建一个Test的对象。上述匿名内部类的用法与下述表达形式相同
```java
public class InnerTest{
    class MyTest implements Test{
        private int val = 1;
        public int value(){
            return value;
        }
    }
    public Test test(){
        return new MyTest();
    }
    public static void main(String[] args){
        InnerTest t = new InnerTest();
        System.out.println(t.test().value());
    }
}
```
### 使用内部类的原因
每个内部类都可以独立的继承某个接口或者非接口类，无论外部类是否继承了接口或者非接口类。也就是说，因为内部类，Java实现了多重继承。
```java
public class Dog extends Animal{
    class Car extends Toy{
        public void play(){
            System.out.println("dog play car");
        }
    }
    public Car cars(){
        return new Car();
    }
    public void shout(){
        System.out.println("dog shout !!!!");
    }
    public static void main(String[] args) {
        Dog dog = new Dog();
        dog.shout();
        dog.cars().play();
    }
}
```
使用内部类还能获得一些其它的特性：
- 内部类可以有多个实例，每个实例都有自己的状态信息，并且与其外围类的对象的信息相互独立。
- 在单个外围类中，可以让多个内部类以不同的方式实现同一个接口，或继承同一个类
- 创建内部类对象的时刻并不依赖于外围类对象的创建
## lambda表达式 
在Java中处处是对象，但是在java定义中不能将函数或者方法独立，更不能将函数或者方法看作参数传递也不能作为函数的放回值。
我们总是通过匿名类的方式给方法传递函数功能。下面是swing的事件监听代码。
```java
someObject.addMouseListener(new MouseAdapter() {
        public void mouseClicked(MouseEvent e) {

            //Event listener implementation goes here...

        }
    });
```
在给addMouListener添加自定义代码时，我们创建了MouseAdapter对象，通过这种方式我们将自定函数传递给监听事件。

### lambda表达式简介
Lambda表达式是一种匿名函数，没有名字，没有修饰符，没有返回值声明。
lambda表达式的格式为：(argument) -> (body)书写方法。
```java
(arg1, arg2)->(body);
(type1 arg1, type2 arg2, .....) -> (body);

eg:
() -> (System.out.println("hello world"));
(String s) -> (System.out.println(s));
```

### lambda表达式的结构
- 一个lambda表达式可以有0个或者多个参数
- 参数类型可以声明，也可以根据上下文判断
- 空圆括号表示参数为空 `()`
- 当只有`一个`参数的时候，而且参数类型可以通过上下文判断时，圆括号可以省略, 例如`a -> System.out.println(a)`
- lambda表达式可以包含零条或者多条语句
- 当主体只有一条语句时`{}`可以省略，返回值与主体语句一致
- 当主体有多条语句时，{}不能省略，返回值与主体块的类型一致

### 函数式接口
在Java中，marker(标记型)接口是一种没有声明和方法的接口。例如Serialization接口，简单来说marker类型接口是空接口。函数式接口是一种只有一种方法声明的接口 , `java.lang.Runnable`接口就是函数式接口，只有一个方法`void run` 
lambda可以通过隐式的赋值给函数式接口
```java
我们可以通过如下方式为Runnable接口赋值
Runnable r = () -> {System.out.println("hello world")};

当不指明函数式接口时，他会自动初始化
new Thread(
    ()->System.out.println("hello world")
).start();
```
上面代码中，编译器会自动识别：根据构造函数的签名`public Thread(Runnable r){}` 将lambda语句赋值给Runnable接口。

java 8 提供了一个新的注释@FunctionalInterface, 用来定义一个函数式接口
```java
@FunctionalInterface
public interface FunInter{
    void test(int arg1, int arg2);
}
```
函数式接口的使用
```java
public class FunTest{
    
    public void test(FunInter funInter, int a, int b){ 
        funInter.test(a, b);
    }
    public static void main(String[] args) {
        FunTest funTest = new FunTest();
        // (a, b) 相当于(int a, int b)所以不需要其地方声明a,b
        funTest.test((a, b) -> System.out.println(a + b), 11 , 12);
    }
}
```

### lambda表达式的案例
在lambda表达式中，全新定义了冒号操作符( :: ) 将常规方法转换为lambda表达式 :)
```java
// old way 
List<Integer> nums = Arrays.asList(1,2,3,4,5);
for(Integer a : nums){
    System.out.println(a);
}

// new way
nums.forEach(b ->{
    System.out.println(b);
})

// use the '::' operator
nums.forEach(System.out::println);
```

在Java 8中添加了流API, 在java.util.stream.Stream接口中包含了许多有用的方法。例如计算列表中所有值的平方，使用map()方法，该方法会作用于流中所有元素
```java

//Old way:
List<Integer> list = Arrays.asList(1,2,3,4,5,6,7);
for(Integer n : list) {
    int x = n * n;
    System.out.println(x);
}

//New way:
List<Integer> list = Arrays.asList(1,2,3,4,5,6,7);
list.stream().map((x) -> x*x).forEach(System.out::println);
```
[java 8流的使用详情](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/index.html)

### lambda表达式与匿名类的区别
使用匿名类与 Lambda 表达式的一大区别在于关键词的使用。对于匿名类，关键词 this 解读为匿名类，而对于 Lambda 表达式，关键词 this 解读为写就 Lambda 的外部类。

Lambda 表达式与匿名类的另一不同在于两者的编译方法。Java 编译器编译 Lambda 表达式并将他们转化为类里面的私有函数，它使用 Java 7 中新加的 invokedynamic 指令动态绑定该方法，关于 Java 如何将 Lambda 表达式编译为字节码。

![](https://brandonxcc.top/fate.jpg)