---
layout: '[layout]'
title: Java泛型
date: 2019-09-26 08:37:10
tags:
    - java基础
categories:
    - Java
    - Java基础
---
Java泛型是Java5引入的重要特性之一，无论是语法还是还是运用的环境中，Java泛型都与C++模板类似。但是这种相似仅仅只是停留在表面，Java语言的泛型完全通过编译器实现，由编译器执行类型检查和类型推断，然后生成普通的非泛型字节码(class文件)。这种实现技术称之为类型擦除(编译器使用泛型信息保证类型安全，然后生成字节码将其擦除)

## Java泛型
### 泛型的声明
#### 泛型类
泛型类和普通类的声明一样，只是在类名后面加上了类型表示。泛型类可以有一个或者多个类型表示，不同的类型表示之间使用逗号隔开
```java
public class Box<T> {

  private T t;

  public void add(T t) {
    this.t = t;
  }

  public T get() {
    return t;
  }

  public static void main(String[] args) {
     Box<Integer> integerBox = new Box<Integer>();
     Box<String> stringBox = new Box<String>();

     integerBox.add(new Integer(10));
     stringBox.add(new String("Hello World"));

     System.out.printf("Integer Value :%d\n\n", integerBox.get());
     System.out.printf("String Value :%s\n", stringBox.get());
  }
}
```

输出
```java
Integer Value :10
String Value :Hello World
```
#### 泛型接口
泛型接口在声明接口的时候指定，类在继承接口的时候要补充泛型类型
定义泛型接口
```java
public interface Info<T> {
    public T getInfo();
}
```

定义一个类实现这个接口
```java
public InfoImp implements Info<String> {
    public String getInfo() {
        return "Hello World!";
    }
}
```
也可在实现这个泛型接口到时候不补充泛型类型
```java
public InfoImp<T> implements Info<T> {
    public T getInfo() {
        return null;
    }
}
```

#### 泛型方法
在普通方法声明上加入泛型
```java
public static < E > void printArray( E[] inputArray )
{
    for ( E element : inputArray ){
        System.out.printf( "%s ", element );
    }
    System.out.println();
}


实现
// Create arrays of Integer, Double and Character
Integer[] intArray = { 1, 2, 3, 4, 5 };
Double[] doubleArray = { 1.1, 2.2, 3.3, 4.4 };
Character[] charArray = { 'H', 'E', 'L', 'L', 'O' };

System.out.println( "Array integerArray contains:" );
printArray( intArray  ); // pass an Integer array

System.out.println( "\nArray doubleArray contains:" );
printArray( doubleArray ); // pass a Double array

System.out.println( "\nArray characterArray contains:" );
printArray( charArray ); // pass a Character array
```

泛型方法的声明格式
```java
[权限] [修饰符] [泛型] [返回值] [方法名]  (    [参数列表]   ) {}
public static  < E >   void   printArray( E[] inputArray ) {}
```
泛型的声明，必须在方法的修饰符（public,static,final,abstract等）之后，返回值声明之前。可以声明多个泛型，用逗号隔开。泛型的声明要用`<>`包裹。

### 泛型标识
- E - Element (在集合中使用，因为集合中存放的是元素)
- T - Type（Java 类）
- K - Key（键）
- V - Value（值）
- N - Number（数值类型）
- ? - 表示不确定的java类型
- S、U、V - 2nd、3rd、4th types

### 通配符
泛型中`?`是通配符，表示未知类型。通配符可以用于参数、属性、局部变量或者返回值的类型，但是不能用于泛型方法调用或者创建泛型类实例的类型参数
#### 无界通配符
单独使用`?`表示无界通配符，例如List<?>,表示未知类型list，下面两个场景适合使用无界通配符：
- 如果编写一个方法，只使用Object类中的功能，使用List<?> 代替List<Object>;
- 当使用泛型类型的方法不依赖参数类型时，例如只使用到List.size或者List.clear方法。事实上Class<?> 方法也经常出现，这是因为Class<T>大多数方法不依赖T。
<?> 无界操作符只能用于接收，不能用作定义
正确例子
```java
Box<?> box = new Box<String>();  // 可以表示为将后面的类型转换到前面类型
```
错误例子
```java
class Box<?> {} //错误的泛型类定义
public static <?> void sayHello(? helloString) {} //错误的泛型方法定义
interface Box<?> {} //错误的泛型接口定义
```
#### 上界通配符
上界通配符可以放宽对变量的限制。语法为` ? extends superType `, superType可以是类或者接口。如果想写一个对List<Integer>, List<Double>, List<Number>都适用的方法，就可以适用List<? extends Number>。
```java
public static double sumOfList(List<? extends Number> list) {
    double s = 0.0;
    for (Number n : list)
        s += n.doubleValue();
    return s;
}
```
#### 下届通配符
下届通配符可以限制为特定的类型或该类型的父类。语法为` ? super superType `想写一个方法添加Integer到对象list中，可以是List<Integer>, List<Double>, List<Object>。都可以使用List<? super Integer>。
```java
public static void addNumbers(List<? super Integer> list) {
    for (int i = 1; i <= 10; i++) {
        list.add(i);
    }
}
```

### 泛型不是协变的
虽然将泛型看作数组的抽象对泛型的理解有帮助，但是泛型还具备了数组所不具备的特殊性质。Java中数组是可以协变(covariant)。也就是说，当Integer扩展了Number，那么不仅Integer 扩展了Number，而且Integer[] 也拓展了 Number[ ]。正式的说，Number是Integer的基类，那么Number[] 也是Integer[] 的基类。但是这个并不适用与泛型，这就是因为泛型的类型安全机制。如果泛型是协变的能够将List<Integer>赋值给List<Number>,那么下列代码就允许将非Integer放入List<Integer>中
```java
List<Integer> li = new ArrayList<Integer>(); 
List<Number> ln = li; // illegal 
ln.add(new Float(3.1415));
```
因为ln是List<Number>类型的，所以向里面添加float是完全合法的。但是如果 ln是 li的别名，那么这就破坏了蕴含在 li定义中的类型安全承诺 —— 它是一个整数列表，这就是泛型类型不能协变的原因

数组能够协变，而泛型不能协变的另一个后果是，不能实例化泛型类型的数组。例如`new List<String>[size].`, 这是不合法的，因为Java的泛型是用擦除实现的，在运行时就会把类型擦掉。那么运行时就会被看作List[]，这个违背了Java的`类型安全`的原则。
如果我们作死允许声明泛型数组，会发生什么事情么
```java
List<String>[] lsa = new List<String>[10]; // illegal 
Object[] oa = lsa;  // OK because List<String> is a subtype of Object 
List<Integer> li = new ArrayList<Integer>(); 
li.add(new Integer(3)); 
oa[0] = li; 
String s = lsa[0].get(0);
```
在第一行定义的时候，编译器就会报错。最后一行将抛出 ClassCastException，因为这样将把 List<Integer>填入本应是 List<String>的位置。因为数组协变会破坏泛型的类型安全，所以不允许实例化泛型类型的数组（除非类型参数是未绑定的通配符，比如 List<?>）。

### 泛型的擦除
如果没有了解过泛型擦除的人，会将List<String>和List<Integer>的Class认作不一致。
```java
List<String> strList = new ArrayList<>();
List<Integer> intList = new ArrayList<>();
System.out.println(strList.getClass().getName());   // java.util.ArrayList
System.out.println(intList.getClass().getName());   // java.util.ArrayList
System.out.println(strList.getClass() == intList.getClass());   // true
```
在编译时期，List<String> 和List<Integer> 是不一样的，但是在运行时期，两个又一样了。背后的原因就是类型擦除。

Java泛型添加是为了提供编译时期的类型检查和支持泛型编程，并没有运行时期的支持。所以Java编译器会用类型擦除来删除所有泛型类型检查代码，并在必要时插入强制类型转换。类型擦除确保不为参数类型创建新类，所以`ArrayList<E>`的Class类型还是`java.uitl.ArrayList`，相应的不会增加运行时的开销。Java编译器在应用泛型类型擦除的时候有以下行为：
- 将泛型中所有参数化类型替换为泛型边界，如果参数化类型是无界的，则替换为 Object 类型。字节码中没有任何泛型的相关信息。
- 为了类型安全，在必要时插入类型转换代码。
- 生成桥接方法来保持泛型类型的多态性。

#### 参数类型替换
对于无界参数类型，类型擦除时候会替换为Object。
下面单列表节点类：
```java
public class Node<T> {
    private T data;
    private Node<T> next;
    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }
    public T getData() { return data; }
    // ...
}
```
经过类型擦除之后
```java
public class Node {
    private Object data;
    private Node next;
    public Node(Object data, Node next) {
        this.data = data;
        this.next = next;
    }
    public Object getData() { return data; }
    // ...
}
```
对于有界参数类型来说，类型擦除之后会替换为第一个边界

如果节点类使用有界参数类型
```java
public class Node<T extends Comparable<T>> {
    private T data;
    private Node<T> next;
    public Node(T data, Node<T> next) {
        this.data = data;
        this.next = next;
    }
    public T getData() { return data; }
    // ...
}
```
经过类型擦除之后
```java
public class Node {
    private Comparable data;
    private Node next;
    public Node(Comparable data, Node next) {
        this.data = data;
        this.next = next;
    }
    public Comparable getData() { return data; }
    // ...
}
```

#### 类型转换
经过参数化类型替换后，在使用泛型相关内容时，通常需要添加类型转换代码，看下面代码
```java
Node<String> node = new Node<>("Hello", null);
String data = node.getData();   // 实际上 node.getData() 返回的是 Object 类型
```

所以编译器还会插入类型转换代码，编译后如下：
```java

Node node = new Node("Hello", null);
String data = (String) node.getData();
```
#### 桥接方法
当编译一个类继承泛型类或泛型接口，在类型擦除的过程中编译器会生成一个合成方法，也称为桥接方法。

看下面代码：
```java
interface Comparable <A> { 
    public int compareTo( A that); 
} 
final class NumericValue implements Comparable <NumericValue> { 
    priva te byte value;  
    public  NumericValue (byte value) { this.value = value; } 
    public  byte getValue() { return value; }  
    public  int compareTo( NumericValue t hat) { return this.value - that.value; } 
}
```
经过参数类型化之后，Comparable接口的compareTo方法的参数类型为Object，而NumericValue也需要实现compareTo(Object) 方法，经过类型擦除之后
```java

interface Comparable{
    public int compareTo(Object that);
}
final class NumericVlaue implements Comparable{
    private byte value;
    public NumericVlaue(byte vlaue){this.value = value;}
    public byte getValue(){return this.value;}
    public int compareTo(NumericValue that){return this.byte - that.byte;}
    // 新和成的桥接方法
    public int compareTo(Object that){return this.compareTo(NumericValue(that));}
}
```
类型擦除之后`NumericValue.compareTo(NumericValue that)`不再是接口实现的方法，这是擦除类型的一个副作用：两个方法在接口的实现类中，在类型擦除之前具有相同的签名，而在类型擦除之后具有不同的签名。
为了让NumericValue依然正确地实现实现（Comparable）接口，编译器添加了一个桥接方法，和接口的方法签名相同，桥接方法委托给原始类中的原始方法。
虽然存在桥接方法，但在一般情况之下，编译器不允许我们调用桥接方法
```java
NumericValue value = new NumericValue((byte) 0);
value.compareTo(value); // OK
value.compareTo("abc"); // error
```
但是还有两个方法可以调用桥接方法，使用原始类型(Raw Type)或者反射，但是原始类型中的桥接方法有强制类型转换，所以传其他类型会有运行报错。下面是使用桥接方法的例子
```java
Comparable comparable = new NumericValue((byte) 0);
comparable.compareTo(comparable); // OK
comparable.compareTo("abc");    // OK at compile time, throws ClassCastException at run time
```
下面是参照的三篇博客：
- [泛型声明](https://sumygg.com/2015/12/15/java-generic-type-one-two-and-three/)
- [泛型不是协变的](https://www.ibm.com/developerworks/cn/java/j-jtp01255.html)
- [泛型的类型擦除](https://johnnyshieh.me/posts/java-generics/)


![](https://brandonxcc.top/哈哈哈哈哈哈哈.jpg)