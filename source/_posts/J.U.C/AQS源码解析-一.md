---
layout: '[layout]'
title: AQS源码解析(一)
date: 2020-04-19 16:26:56
tags:
---
# 前言
开始拜读Doug Lea大神的并发源码了，作为一个小菜鸡，可能理解不了太多这里面的一些编程思想，只希望能了解一个大概过程，在平时使用或者解决问题的时候有一个清晰的思路去分析问题的原因然后去解决这个问题。

AbstractQueueSynchronizer(简称AQS)这个抽象类，是实现ReentrantLock、CountDownlatch、Semaphore、FutureTask等类的基础。

站在使用者的角度，AQS的功能主要分为两大类：独占锁和共享锁。在他的所有子类中，要么实现并使用了它独占锁的API，要么使用了共享锁的API，而不会同时使用两种API。即便是ReentrantReadWriteLock，也是通过两个内部类，读锁和写锁，分别实现了两套API来实现这两个功能。

# AQS结构
AQS的属性
```java
// 头节点，当前持有锁的线程
private transient volatile Node head;

// 阻塞尾节点，每个新的节点进来，都插入到最后，就形成了一个链表
private transient volatile Node tail;

// 代表当前锁的状态，0表示没有被占用，大于0代表有线程持有当前锁
// 这个值可以大于1，因为锁可以重入，每次重入都要加上1
private volatile int state;

// 代表当前持有独占锁的线程，使用例子(因为锁是可以重入的)
// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有的锁
// if(currentThreed == getExclusiveOwnerThread()) {state++;}
private transient Thread exclusiveOwnerThread; // 继承自AbstractOwnableSynchronizer
// 同时这个变量没有被volatile关键字修饰，因为这个变量只要自己能看到就可以了。
```

## Node结构(双向链表)
```java
static final class Node{
    // 表示当前节点是在共享模式下
    static final Node SHARED = new Node();
    // 表示当前节点是在独占模式下
    static final Node EXCLUSIVE = null;

    // 以下的常量是给waitState使用状态
    // 表示线程取消了等待
    static final int CANCELLED = 1; 

    // 表示下一个节点是通过park阻塞的，需要通过unpark唤醒
    static final int SIGNAL = -1; 

    // 表示线程在等待条件变量(先获取锁，加入到条件等待队列，然后释放锁，等待条件变量满足条件；只有重新获得锁之后才能返回)
    static final int CONDITION = -2; 

    // 表示后续节点会传播唤醒操作，共享模式下起作用。
    static final int PROPAGATE = -3; 

    // 等待状态，对于condition节点，初始化为CONDITION；其他默认为0，通过cas设置
    //
    volatile int waitStatus;

    volatile Node prev; // 前驱节点
    volatile Node next; // 后继节点
    volatile Thread thread; // 节点
}
```
现在有了node结构的概念之后，我们再回头看看AQS的等待队列示意图
![aqs](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer/aqs-3.png)

## AQS的模板方法以及介绍
1. isHeldExclusively(): 判断该线程是否正在独占线程。只有用到condition才需要去实现他
2. tryAcquire(int): 独占的方式。尝试获取资源，成功则返回true，失败则放回false
3. tryRelease(int): 独占方式。尝试释放资源，成功返回true，失败返回false
4. tryAcquireShared(int): 共享方式。尝试获取资源，负数表示失败。0 表示成功，但是没有资源可以被申请；正数表示成功，而且有资源能够被获取。
5. tryReleaseShared(int): 共享的方式。尝试释放资源，成功返回true，失败返回false。

以上就是AQS给子类提供的模板方法，给子类实现自定义的同步器。

## 争锁操作
通过ReentrantLock来看看AQS是如何实现的。ReentrantLock内部使用了一个内部类Sync来管理锁，所以真正的获取锁和释放锁是由Sync来实现的。
```java
abstract static class Sync extends AbstractQueuedSynchronizer

// 同时还有两个继承类，用来分别实现了公平锁与非公平锁、
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}

static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;
        final void lock() { // 争锁
            acquire(1);
        }
        //下面是父类AQS中的Acquire实现
        /*
            如果tryAcquire(arg) 返回true，则获取资源成功，便使用锁资源进行下一步操作
            如果返回false，acquireQueue方法将线程压到队列中
        */
         public final void acquire(int arg) { // 此时的arg == 1
             // 首先使用tryAcquire(1)，可能直接成功，就不需要进入阻塞队列
             
        if (!tryAcquire(arg) &&
            // 如果获取失败，就会将当前线程挂起，然后进入阻塞队列。
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        @ReservedStackAccess
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            // 获取当前资源锁的状态，如果为0 此时此刻没有线程持有该锁
            int c = getState();
            // 如果没有上锁，则尝试获取锁
            if (c == 0) {
                // 此时该锁是可以用的，但是公平锁就是讲究一个公平，他要查看阻塞队列中没有老哥一直等着的，如果有，他也会老老实实去排队
                if (!hasQueuedPredecessors() &&
                    // 如果没有人排队，就使用cas试一下，如果能成功
                    // 就直接获取该锁资源
                    // 如果不成功，说明有锁在跟我同时在获取这个资源。
                    compareAndSetState(0, acquires)) {
                    // 获取到锁资源之后，要进行标记，告诉大家我获得这个锁了
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 判断已经加锁的线程是本线程，则重入锁。
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            // 到这里，说明没有获取到锁，则要进入到阻塞队列中排队。
            return false;
        }
    }
```
加入到阻塞队列中addWaiter
```java
// 发现jdk9与jdk8有一些改变，但思想是没变的。
// jdk9改成使用VarHandle来新建节点，以及修改tail节点。
    private Node addWaiter(Node mode) {
        Node node = new Node(mode);
        for (;;) {
            // 获取老的尾节点
            Node oldTail = tail;
            // 如果tail不是空的话，新建node节点的前驱指针指向oldTail，反之初始化阻塞队列
            if (oldTail != null) {
                // 设置 新建节点的前驱指针指向tail
                node.setPrevRelaxed(oldTail);
                // 使用cas把自己设置为队尾，如果成功，tail == node，这个新建节点成为新尾巴
                if (compareAndSetTail(oldTail, node)) {
                    // 上面已经有了node.setPrevRelaxed(oldTail);，
                    // 加上下面这句，就实现了和之前的尾节点的挂钩的双向连接了。
                    oldTail.next = node;
                    // 新建节点入队列，然后返回这个节点。
                    return node;
                }
            } else {
                // 如果原来队列没有元素，就要初始化这个队列。
                initializeSyncQueue();
            }
        }
    }
    // 下面进入初始化队列
    // 只有队列为空的时候，才会运行这个方法
    private final void initializeSyncQueue() {
        Node h;
        if (HEAD.compareAndSet(this, null, (h = new Node())))
            tail = h;
    }
```
现在再来回顾一下，当线程没有抢到锁资源的时候发生了什么
    // if (!tryAcquire(arg) 
    //        && acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
    //     selfInterrupt();

```java
    // 接下来就要接着谈谈acqireQueue 这个方法
    // 下面这个方法很重要，在经过addWaiter(Node.EXCLUSIVE)，此时线程已经进入阻塞队列中
    // 如果acquireQueued返回true的话，意味着将要执行selfInterrupt()这个方法。
    // 所以acquireQueued这个方法很重要，线程的挂起以及被唤醒之后去获取锁，都在这个方法中。
    final boolean acquireQueued(final Node node, int arg) {
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                /*
                    p == head 说明当前节点虽然进到了阻塞队列中，但是是阻塞队列中的第一个，因为他的前驱节点是head。
                    注意， 阻塞队列不包含head节点，head一般是指的占有锁的进程，head后面的才是阻塞队列
                    首先 head是队列头，其次head有可能是刚刚初始化的node。
                    head是延时初始化的，而且new Node()的时候没有设置任何线程
                    也就是说，当前的head可能不属于任何线程，所以作为阻塞队列的队头，可以尝试去获取锁资源
                */
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }
                // 如果到这里，说明if分支没有成功，有两个原因
                // 1. 可能node本来就不是队头
                // 2. tryAcquire没有抢赢别人。
                // 下面两个判断如果都为true了，该线程就要被挂起
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } catch (Throwable t) {
            // 在tryAcquire抛出异常的时候，这个线程就会放弃争夺锁资源。
            cancelAcquire(node);
            throw t;
        }
    }
    // 正如前文，会运行这个方法。
    // 该方法表达的是"当前线程没有抢到锁，是否要挂起当前线程"
    // 第一个参数是前驱节点，第二个参数才是当前节点。
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;// 获取前驱节点的等待状态
        // 如果前驱节点是-1，则说明节点正常，可以挂起当前线程，返回true
        if (ws == Node.SIGNAL) 
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        // 前驱节点waitStatus大于0，说明前驱节点取消了排队
        // 在这里要注意的是，进入阻塞队列的线程都会被挂起，只能依靠前驱节点唤醒。
        // 所以下面的循环就是将当前节点的prev指向一个waitStatus<=0的节点
        // 这里要找到前面的waitStatus<=0的节点。才能保证能够唤醒当前节点。
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            /*
             * 走到这里说明前驱节点不为-1和1，只能是0或者-2，-3
             * 一路走来都没有看到设置waitStatus的地方。
             * 原本的tail的waitStatus是0，突然新来了一个节点之后，还没改变
             * 自己的waitStatus。在下面使用cas将prev的waitStatus设置为-1
             * 
             */
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
        // 这个方法返回false，就会再走一遍上面的for循环，
        // 再进来该方法，然后返回true。
        return false;
    }
```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node);

这个方法的返回值进行分析：
如果这个方法返回的是true，那么就说明当前线程会被挂起，等待前驱节点获取到锁之后来唤醒你。

再来看看调用方法的这个地方`if(shouldParkAfterAcquire(p, node) && parkAndCheckInterrupt()) interupted = true;`
1. 如果shouldParkAfterAcquire返回了true，那么就会进入下一个方法parkAndCheckInterrupt()，这个方法会挂起线程。
    ```java
    private final boolean parkAndCheckInterrupt() {
        // 这里使用了LockSupport.park(this);
        // 用来挂起线程，等待被唤醒。
            LockSupport.park(this);
            return Thread.interrupted();
        }
    ```
2. 如果返回了false，会进行一次for循环。其实在第一次进入shouldParkAfterAcquire这个方法的时候，一定会返回false，具体原因如上。这里还有提供了一个获取锁的机会，万一第二次循环的时候获取到锁了，那么就不用被挂起了。直接进行下一步的处理。

## 解锁操作
正常情况下，如果线程没有获取到锁，就会被`LockSupport.park(this);`挂起，等待被唤醒。
```java
    // 当前线程开始请求释放锁
    public void unlock() {
        sync.release(1);
    }

    public final boolean release(int arg) {
        // 判断是否可以释放锁了
        if (tryRelease(arg)) {
            // head是当前持有锁的节点
            Node h = head;
            // 如果head为空了，说明队列里面没有阻塞节点，也不用去唤醒了。
            // waitState == 0，根据加锁的代码可以知道，这个是后续节点设置的。
            // 如果waitSate == 0，则后面也没有需要唤醒的节点。
            // 如果都不满足，说明队列中还有节点需要被唤醒。
            if (h != null && h.waitStatus != 0)
                // 唤醒后续节点
                unparkSuccessor(h);
            // 锁释放成功。
            return true;
        }
        return false;
    }
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        // 判断当前线程是否是锁的拥有线程
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        // 是否完全释放锁
        boolean free = false;
        // 这里其实是一个重入锁的概念，判断还有没有嵌套锁，如果没有了，就释放完全了
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
    
    // 唤醒后续节点
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        // 记住，这里的node 是head
        int ws = node.waitStatus;
        if (ws < 0)
            // 首先设置head的waitState为0
            node.compareAndSetWaitStatus(ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
         // 然后找到head的后继节点 B
        Node s = node.next;
        // 如果B节点为空，或者取消了等待
        if (s == null || s.waitStatus > 0) {
            s = null;
            // 下面就要从该队列的尾巴部分，开始往前面找，
            // 直到找到最前面的符合条件的节点
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                // 在循环里面没有break，说明循环要一直执行下去。
                    s = p;
        }
        if (s != null)
            // 唤醒这个节点，开始争夺锁
            LockSupport.unpark(s.thread);
    }
```
至于为什么要从尾部往前面找符合条件的节点,要关注一下下面的代码
```java
 private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) {
            //1. 维护 node 前驱节点
            node.prev = pred;
            //2. 使用 CAS 将节点设置为队列尾节点
            if (compareAndSetTail(pred, node)) {
                //3. 维护 pred 的后继节点
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
如上面代码所说的，首先设置了前驱节点`node.pre = pred`,然后使用CAS入队，注意这里可能还没有入队，则就没有办法执行`pred.next = node`，这个时候如果是前面往后面开始找的话，就有可能发生找不到节点的情况。

```java
// 唤醒线程之后，被唤醒的线程将从下面的代码继续往前走
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 刚刚线程被挂起在这里了
    return Thread.interrupted();
}
// 又回到这个方法了：acquireQueued(final Node node, int arg)，这个时候，node的前驱是head了
```

## 总结
在并发的条件下，需要以下三个部件的协调
1. 锁状态。我们要知道锁是不是被别的线程占有了，这个就是 state 的作用，它为 0 的时候代表没有线程占有锁，可以去争抢这个锁，用 CAS 将 state 设为 1，如果 CAS 成功，说明抢到了锁，这样其他线程就抢不到了，如果锁重入的话，state进行 +1 就可以，解锁就是减 1，直到 state 又变为 0，代表释放锁，所以 lock() 和 unlock() 必须要配对啊。然后唤醒等待队列中的第一个线程，让其来占有锁。
2. 线程的阻塞和解除阻塞。AQS 中采用了 LockSupport.park(thread) 来挂起线程，用 unpark 来唤醒线程。
3. 阻塞队列。因为争抢锁的线程可能很多，但是只能有一个线程拿到锁，其他的线程都必须等待，这个时候就需要一个 queue 来管理这些线程，AQS 用的是一个 FIFO 的队列，就是一个链表，每个 node 都持有后继节点的引用。AQS采用了CLH锁的变体来实现。

![avatar](https://brandonxcc.top/20200422.jpg)
## 参考文章
[AQS功能分析](https://blog.csdn.net/pfnie/article/details/53191892)

[ReentrantLock公平锁的角度去分析AQS](https://www.javadoop.com/post/AbstractQueuedSynchronizer)

[AQS源码分析](https://www.jianshu.com/p/c244abd588a8)