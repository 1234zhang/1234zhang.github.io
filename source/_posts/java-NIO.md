---
layout: '[layout]'
title: java NIO的组件
date: 2019-10-17 16:39:55
tags:
    - java提高
categories:
    - Java
    - Java流操作
    - nio
---

# NIO三大组件Buffer、Channel、Selector
## buffer
一个buffer实质上是内存中的一块，我们可以将数据写入这块内存之中，然后从这个内存中读取数据。下面是java.nio包中定义的几个buffer
![](https://brandonxcc.top/buffer.png)
前面是byteBuffer的修饰，我们主要使用的还是byteBuffer。下面是有关buffer的相关操作，以及关于buffer的重要属性和方法。

### position、limit、capacity
![](https://brandonxcc.top/position.png)

**capacity**：表示这个缓存区的容量，一旦设定就不能更改。一旦缓存区所含数据大小达到容量大小，就不能再存放数据，必须清空缓存区才能继续添加数据。
**position**：初始值是0，每添加一个数据，position就+1。读操作类似。
![](https://brandonxcc.top/buffe读写操作.png)
### **flip()方法**
flip方法将buffer从写模式转换为读模式，而且只能这么干。调用flip()方法会使得position值变为0，并将limit设置为原来position的值。也就是position开始用于表示读的位置，limit表示总共有多少个数据。
```java
public final Buffer flip() {
    limit = position; // 将 limit 设置为实际写入的数据数量
    position = 0; // 重置 position 为 0
**limit**：写操作模式下，limit代表的是最大能写入的数据，这个时候limit与capacity相同。当读操作模式下，limit的值与position值相同，表示buffer所含有的数据实际大小。
    mark = -1; // mark 之后再说
    return this;
}
```

### **初始化buffer**
每个buffer都提供了一个allocate(int capacity)静态方法，用于初始化buffer(eg: ByteBuffer byteBuffer = ByteBuffer.allocate(1024))
### **填充buffer**
每个buffer类都提供了put方法将数据填充进buffer
```java
// 填充一个byte进buffer
public abstract ByteBuffer put(byte b);
// 在一个指定的位置加入byte
public abstract ByteBuffer put(int index, byte b);
//将一个数组填充进去
public final ByteBuffer put(byte[] src){....}
public ByteBuffer put(byte[] src, int offset, int length){...}
```
最常用的应该是从channel中读取数据，填充到buffer中。
```java
int num = channel.read(bytebuffer);
将会返回buffer读取数据的数值。
```
### **读取buffer**
如果要读取buffer中的值，需要将buffer从写入操作转换为读出操作。使用flip方法可以达到操作的目的。我们通常说的从channel中读取数据，其实是将数据写入buffer中。
buffer也提供了相应的get方法
```java
// 根据 position 来获取数据
public abstract byte get();
// 获取指定位置的数据
public abstract byte get(int index);
// 将 Buffer 中的数据写入到数组中
public ByteBuffer get(byte[] dst)
```
除了从buffer中取出数据来用，我们更加常用的方法是将buffer中的数据读取出来写入channel中。
```java
int num = channel.write(buffer);
```
### **mark()**
mark()用于临时保存position的值，每次调用mark()方法都会将mark设值为当前position，便于后续需要时候调用。
```java
public final Buffer mark() {
    mark = position;
    return this;
}
```
mark()方法可以配合着reset()方法一起使用，如果我们在读取到position为5时候，调用mark()方法，然后继续往下面读，如果我们又想返回到position为5的位置上时，就可以调用reset方法，使得position为5
```java
public final Buffer reset() {
    int m = mark;
    if (m < 0)
        throw new InvalidMarkException();
    position = m;
    return this;
}
``` 

### **rewind()&clear()&&compact()**
**rewind()**: 会将position的值设置为0，通常用于从头读写buffer
```java
public final Buffer rewind() {
    position = 0;
    mark = -1;
    return this;
}
```
**clear()**：可以理解为重置buffer，调用这个方法之后，我们就可以往buffer中重新填充数据
```java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```
**compact()**： 和clear有点相似的是，都是在准备往buffer中重新填充数据之前调用。
前面说的 clear() 方法会重置几个属性，但是我们要看到，clear() 方法并不会将 Buffer 中的数据清空，只不过后续的写入会覆盖掉原来的数据，也就相当于清空了数据了。

而 compact() 方法有点不一样，调用这个方法以后，会先处理还没有读取的数据，也就是 position 到 limit 之间的数据（还没有读过的数据），先将这些数据移到左边，然后在这个基础上再开始写入。很明显，此时 limit 还是等于 capacity，position 指向原来数据的右边。

## Channel
nio所有数据始于通道，通道是数据的来源或者写入的目的地。我们主要关注java.nio包中如下几个channel
![](https://brandonxcc.top/channel.png)

1. fileChannel: 文件通道，用于文件的读和写
2. DatagramChannel: 用于UDP的链接和接收
3. SocketChannel: 可以理解为TCP的客户端
4. ServerSocketChannel: 用于监听TCP某个端口的请求。

channel经常和buffer打交道，读操作的时候将Channel中的数据填充到buffer中。
![](https://brandonxcc.top/bufferReader.png)
![](https://brandonxcc.top/bufferWrite.png)


### socketChannel
可以将socketChannel看作是tcp的客户端，socketChannel一般是配合着serverSocketChannel使用。
打开一个tcp链接
```java
SocketChannel socketChannel = SocketChannel.open(new inetSocketAddress("http://localhost", 80))
```
上面的代码也等价于下面的代码
```java
// 打开一个通道
SocketChannel socketChannel = SocketChannel.open();
// 发起连接
socketChannel.connect(new InetSocketAddress("http://localhost", 80));
```
### serverSocketChannel
上面所说的socketChannel对应着TCP的客户端，那么serverSocketChannel就对应着TCP的服务端。
serverSocketChannel用于监听端口，管理从这个端口来的TCP连接。
```java
// 实例化
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
// 监听 8080 端口
serverSocketChannel.socket().bind(new InetSocketAddress(8080));

while (true) {
    // 一旦有一个 TCP 连接进来，就对应创建一个 SocketChannel 进行处理
    SocketChannel socketChannel = serverSocketChannel.accept();
}
```
上面展示了socketChannel第二种实例化的方法。
所以我们单纯将socketChannel看作是TCP客户端的理解有点狭隘，它不仅仅代表着tcp客户端，也是一个网络通道，可读可写。
ServerSocketChannel不与buffer打交道，它并不处理数据，它一旦接收到请求之后，就会实例化一个socketChannel，之后在这个连接通道上传递的数据它就不管了，因为他要继续监听端口，接收下一个请求。

### DatagramChannel
DatagramChannel一个类处理了客户端和服务端。
监听端口：
```java
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(9090));

ByteBuffer buf = ByteBuffer.allocate(48);

channel.receive(buf);
```

发送数据
```java
String newData = "New String to write to file..."
                    + System.currentTimeMillis();

ByteBuffer buf = ByteBuffer.allocate(48);
buf.put(newData.getBytes());
buf.flip();

int bytesSent = channel.send(buf, new InetSocketAddress("jenkov.com", 80));
```

## Selector
多路复用的主要实现方式，用于实现一个线程管理多个channel。
selector的接口基本操作 
1. 开启一个选择器
```java
Selector selector = Selector.open();
```
2. 将channel注册到Selector上。selector建立在非阻塞模式之上，所以注册到Selector的channel必须支持非阻塞模式，fileChannel不支持非阻塞模式
```java
channel.configuraBlocking(false);// 设置channel为非阻塞模式。
SelectorKey key = channel.register(selector, SelectorKey.OP_READ);
```
register第二个参数是int型的(使用二进制表示),用于表示对监听哪些事件感兴趣。共下面四个事件
- SelectionKey.OP_READ 
```
对应 00000001，通道中有数据可以进行读取
```
- SelectionKey.OP_WRITE
```
对应 00000100，可以往通道中写入数据
```
- SelectionKey.OP_CONNECT
```
对应 00001000，成功建立 TCP 连接
```
- SelectionKey.OP_ACCEPT
```
对应 00010000，接受 TCP 连接
```
我们可以监听一个channel中的多个事件，比如我们要监听 ACCEPT 和 READ 事件，那么指定参数为二进制的 00010001 即十进制数值 17 即可。
注册方法返回值是 SelectionKey 实例，它包含了 Channel 和 Selector 信息，也包括了一个叫做 Interest Set 的信息，即我们设置的我们感兴趣的正在监听的事件集合。

3. 调用select()方法获取通道中的信息。用于判断我们感兴趣的事情是不是发生了。

对于seletor中的几个方法，我们应该熟悉下面几个方法
- select()
调用这个方法，会将上次select之后准备好的channel对应的selectionKey复制到selected set中。如果通道中没有任何方法准备好，那么就会阻塞在这个方法里，直到监听到某个感兴趣的事件发生。
- selectNow()
与select方法功能相同，但是区别的是如果没有监听到任何事件，这个方法会立即返回。
- select(long timeout)
如果通道没有准备好，会等待一会儿，然后再返回。不会一直阻塞在这个方法中。
- wakeup()
这个方法是用来唤醒等待在select() 和select(long timeout)方法上的线程。如果wakeup被提前调用，此时没有线程在select上阻塞。那么之后的select或者select(long timeout)就会立即返回，而不会阻塞，这个只生效一次。

![](https://brandonxcc.top/zwx.jpg)