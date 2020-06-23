---
layout: '[layout]'
title: AQS源码解析三
date: 2020-05-13 23:11:44
tags:
    - java并发
categories:
    - Java
    - JUC
---

# 前言
在前两篇中，分析了ReentrantLock内部的实现，以及Condition的实现使用。这篇将要分析countDownLatch有关的共享模式，和CyclicBarrier、Semaphore的实现，其实在有了前两篇的经验之后，会发现这里面的套路其实有很多相似的地方。

# CountDownLatch
```
CountDownLatch的实现是比较经典的AQS共享模式的体现。
而且这个也是一个高频使用的类
```
## 使用案例
这个是Doug Lea在Java doc中的例子：

假设我们有N(N > 0)个任务，那么我们会用N来初始化一个CountDownlatch，然后将这个latch的引用传递到各个线程中，在每个线程完成了任务之后，调用latch.countDown()代表完成了一个任务， 调用`latch.await()`的方法的线程将会阻塞，直到所有的任务完成。

```java
class Driver2 {
    public static void main(String[] args) throw InterruptedException{
        CountDownLatch doneSignal = new CountDownLatch(10);
        Executor e = Executors.newFixedThreadPool(9);

        // 创建N个任务，交给线程池来完成
        for (int i = 0; i < 10; i++) {
            e.execute(new WorkerRunnable(doneSignal, i));
        }
        // 主线程调用await方法，直到所有的任务完成，这个方法才会返回。
        doneSignal.await();
    }
}

class WorkRunnable implements Runnable {
    private final CountDownLatch doneSignal;
    private final int i;

    public WorkRunnable(CountDownLatch doneSignal, int i) {
        this.doneSignal = doneSignal;
        this.i = i;
    }

    public void run() {
        try {
            doWork(i);
            doneSignal.coutDown();
        } catch (InterruptedException e) {
            // 捕获异常
        }
    }

    public void doWork(int num) {

    }
}
```

这里使用CountDownLatch将几个大的任务拆分成几个小的任务，然后开启多线程开始执行，等所有线程都执行完了之后，再往下执行其他操作，这个例子中，只有main线程调用了await方法。
下面还有一个很经典的例子，这里使用了两个CountDownLatch。
```java
class Driver {
    void main() throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1);
        CountDownLatch doneSignal = new CountDownLatch(N);

        for (int i = 0; i < N; ++i) {
            new Thread(new Work(startSignal, doneSignal)).start();
        }
        // 这里调用其他方法是为了让所有的线程都启动起来。
        doSomethingElse();
        // 因为这里的数量只有一，所以只要调用一次那么所有的await方法都可以通过
        startSignal.countDown();
        doSomethingElse();
        // 等待所有任务结束
        doneSignal.await();
    }
}

class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;

    Work(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }

    public void run() {
        try {
            // 为了让所有的线程同时开始任务，所以让所有的线程都再这里阻塞
            // 等所有的线程都准备好了，就可以打开这个栅栏
            startSignal.await();
            doWork();
            doneSignal.countDown();
        }catch (InterruptedException ex) {
            // return;
        }
    }
    
    void doWork() {...}
}
```
![avatar](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer-3/5.png)
如果始终只有一个线程调用await方法等待任务完成，那么CountDownLatch就会简单很多。对于多个线程做任务，那么就要构造这么一个场景：有m个线程是做任务的，有n个线程在某个栅栏上等待这m个线程做完任务，直到所有m个任务完成后，n个线程同时通过栅栏。

## 源码分析
构造方法，需要传入一个不小于0的整数：
```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

// 跟以前一样 内部封装一个Sync类继承AQS
private static final class Sync extends AbstractQueueSynchornizer {
    Sync(int count) {
        // 在这里state == count了
        setState(count);
    }
    ...
}
```
这里我们可以根据以前的经验先分析一遍：我们知道AQS内部有一个state，是一个整数值，这边用一个int count参数其实就是初始化了state这个值，所有调用了await方法的等待线程会挂起，然后有其他一些线程会做state = state - 1操作，当state减到0的同时，那个state减为0的线程会负责唤醒所有调用了await方法的线程。

下面有一个例子来帮助我们阅读代码
```java
public class CountDownLatchDemo {
    public static void main(String[] args) {
        CountDownLatch latch = new CountDownLatch(2);
        Thread t1 = new Thread(new Runnable(){
            @Override
            public void run() {
                try{
                    Thread.sleep(5000);
                } catch (InterruptedException ignore) {

                }
                // 这里使每个线程休息5秒，模拟每个线程工作了5秒
                latch.countDown();
            }
        }, "t1");

        Thread t2  = new Thread(new Runnable(){
            @Override
            public void run() {
                try {
                    Thread.sleep(10000);
                } catch(InterruptedException ignore) {
                    // return 
                }
                latch.coutDown();
            }
        }, "t2");
        t1.start();
        t2.start();

        Thread t3 = new Thread(new Runnable(){
            @Override
            public void run() {
                try {
                    latch.await();
                    System.out.println("线程t3从await中返回了");
                } catch (InterruptedException ignore) {
                    System.out.println("线程t3被中断了");
                    Thread.currentThread().interrupt();
                }
            }
        }, "t3");
        
        Thread t4 = new Thread(new Runnable(){
            @Override
            public void run() {
                try {
                    latch.await();
                    System.out.println("线程t4从await中返回了");
                } catch (InterruptedException ignore) {
                    System.out.println("线程t4被中断了");
                    Thread.currentThread().interrupt();
                }
            }
        }, "t4");
        t3.start();
        t4.start();
    }
}


// 运行上面的程序之后，大概十秒钟之后会出现下面的语句
// 下面的语句顺序不是绝对的。
线程t3从await中返回了
线程t4从await中返回了
```
根据流程：先await等待，然后被唤醒，await方法返回
### 等待过程
首先，我们可以关注await方法，它代表线程被阻塞，等待state减值到0的时候
```java
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        // 中断状态的抛出，在上一篇中有提到过
        if (Thread.interrupted())
            throw new InterruptedException();
        // t3和t4调用await的时候，state都大于0，此时state为2
        // 
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
    public int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
```
doAcquireSharedInterruptibly这个从名字上可以看出是获取到共享锁，并且这个方法还是可中断的

中断的时候抛出InterruptedException退出这个方法

```java
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        // 入队列
        final Node node = addWaiter(Node.SHARED);
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    // tryAcquireShared只要state不等于0，那么就返回-1
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
```

在线程t3入队以后我们可以得到下面的情况
![avatar](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer-3/2.png)

由于tryAcquireShared这个方法会返回-1，所以if (r >= 0) 这个分支不会进去。到shouldParkAfterFailedAcquire的时候，t3将head的waitStatus值设置为-1

 ![avatar](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer-3/3.png)
 
 然后进入到parkAndCheckInterrupt的时候，t3挂起。然后再分析t4入队，t4会将前驱节点t3所在的节点的waitStatus设置为-1
 ![avatar](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer-3/4.png)

 然后t3跟t4就等待被唤醒
### countDown减少到0的过程

这里将state数量添加到10，是为了更丰富示意图
![avatar](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer-3/1.png)
当state等于0的时候，就会开始唤醒阻塞线程
(注意我们其实没有十个线程，只有两个t1和t2两个线程，这里的十个线程只是为了示意图好看，并没有其他的含义)

接下来开始看代码
```java
public void countDown() {
    sync.releaseShared(1);
}
    
public final boolean releaseShared(int arg) {

    // tryReleaseShared只有在state == 0的时候才会返回true
    // 否则只是简单的state -= 1
    if (tryReleaseShared(arg)) {
        // 唤醒阻塞队列中的元素
        doReleaseShared();
        return true;
    }
    return false;
}

protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        // 使用自旋的方式对state进行减1操作
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c - 1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
```

countDown方法的作用很简单，就是对state进行减1操作，直到state减到0的时候，就对调用了await方法的线程进行唤醒

### state == 0的时候，开始唤醒线程
```java
// 调用这个方法的时候，state已经减少到0了
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) { // Node.SIGNAL == -1
                // 将头部的waitState设置为0
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 这里设置成功了，就唤醒head后继的第一个节点，也就是唤醒阻塞队列中的第一个节点
                // 这里是唤醒t3
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                        !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```
t3被唤醒之后，我们继续回到await方法中，parkAndCheckInterrupt返回，我们不考虑中断的情况
```java
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    // 这个时候state = 0，那么tryAcquireShared就会返回1
                    int r = tryAcquireShared(arg); 
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    // 检查被挂起的时候，是否发生过中断
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
```
那么当t3被唤醒之后，就会循环，然后进入到setHeadAndPropagate这个方法中，使得t3成为头部，然后唤醒后继节点
```java
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        // 下面说的是，唤醒node的后继节点
        // 即是t3已经醒了，要唤醒t4，依此类推
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                // 再一次运行这个方法，只不过现在的节点换成了t3
                doReleaseShared();
        }
    }
```
现在我们回过头来再看看doReleaseShared()这个方法
```java
    private void doReleaseShared() {
        for (;;) {
            // 此时的头部是t3
            Node h = head;
            // h == null 说明阻塞队列已经空了，就不必继续唤醒了
            // h == tail 说明头节点是刚刚初始化的头节点
            //           或者是普通线程节点，但该节点已经是头节点了，说明已经被唤醒了，阻塞队列没有其他的节点了
            // 所以这两种情况满足一种了，都不必再进行后继节点的唤醒
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // t4将头节点(此时是t3)的waitStatus设置为了Node.SIGNAL(-1)了
                if (ws == Node.SIGNAL) {
                    // 这里的cas失败，要结合下面一起看
                    if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    // 这里唤醒head的后继节点，也就是阻塞队列中的第一个节点
                    // 这里是唤醒t4
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&
                //  这里CAS失败的场景是：执行到这里的时候，刚好有一个节点入队，入队会将这个ws设置为-1
                         !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            // 到这里的时候，刚刚唤醒的线程已经占领了head，那么再循环
            // 如果head没有改变，则退出循环
            // 退出循环不意味着阻塞队列中的其他节点不唤醒，唤醒的线程之后还是会调用这个方法唤醒后继节点。
            if (h == head)                   // loop if head changed
                break;
        }
    }
```

这里来分析最后一个if语句，然后才能知道第一个CAS失败的原因
- head == h：说明头节点还没有被刚刚用unparkSuccessor唤醒的线程占用，此时break退出循环
- head != h：头节点被刚刚唤醒的线程占用，那么这里重新进入下一轮的循环，唤醒下一个节点。我们知道等到t4被唤醒之后，会主动去唤醒后继节点的，这里为什么要进入下一轮循环来唤醒后继节点t5呢？这里应该是对吞吐量的考虑。

在满足了上面2的场景之后，我们能知道为什么第一个CAS会失败。
因为当前进行for循环的线程到了这里之后，可能刚刚唤醒的t4线程也刚刚到这里，两个只能成功一个，就有可能触发失败

for循环第一轮的时候会唤醒t4，t4醒了之后会将自己设置为头节点。如果t4设置为头节点之后，for循环才跑到if (h == head)，那么此时就会返回一个false，for循环就会进入下一轮。t4唤醒之后也会进入这个方法，那么for循环的第二轮(此时的head是t4)和t4就有可能在这个CAS相遇，那么就只会成功一个。
# CyclicBarrier
CyclicBarrier字面意思就是"可重复使用的栅栏"或"周期性栅栏"，总之不是用了一次就不能用的，CyclicBarrier相比CountDownLatch来说，要简单很多，没有什么高深的地方，他是ReentrantLock和Condition的组合使用。

![avatar](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer-3/cyclicbarrier-2.png)

CyclicBarrier的源码实现大相径庭，CountDownLatch基于AQS，而CyclicBarrier基于Condition来实现。

用下面这张图来描述CyclicBarrier里面的一些概念，和一些基本使用流程
![avatar](https://www.javadoop.com/blogimages/AbstractQueuedSynchronizer-3/cyclicbarrier-3.png)

## 源码解读

```java
public class CyclicBarrier{

    // CyclicBarrier 是可以重复使用的，每次从开始使用到穿过栅栏当作一代，或者一个周期
    private static class Generation {
        boolean broken = false;
    }
    private final ReentrantLock lock = new ReentrantLock();

    // CyclicBarrier 是基于Condition的
    // Condition 是条件的意思，CyclicBarrier的等待线程通过barrier的条件是大家都到达了栅栏上
    private final Condition trip = lock.newCondition();

    // 参与的线程数
    private int parties;

    // 如果设置了这个，代表越过栅栏之前，要执行相应的操作
    private final Runnable barrierCommond;

    // 当前所处的代
    private Generation generation = new Generation();
    
    // 还没有到栅栏的线程数，这个值初始值是parties，然后递减
    // 还没有到栅栏的线程数 = parties - 已经到了栅栏的数量
    private int count;

    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllergalArgumentException();

        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction; 
    }
    
    public CyclicBarrier(int parties) {
        this(parties, null);
    }
}
```
怎么开启新的一代
```java
private void nextGeneration() {
    // 首先唤醒所有在栅栏上的线程
    trip.signalAll();
    // 初始化还没有到栅栏处的线程数
    count = parties;
    // 重新生成一代
    generation = new Generation();
}
```
如何打破一个栅栏
```java
private void breakBarrier() {
    // 设置broken状态为 true
    generation.broken = true;
    // 重置count为初始值
    count = parties;
    // 唤醒所有已经在等待的线程
    trip.signalAll();
}
```
下面开始分析通知在栅栏处等待的await方法
```java
    public int await() throws InterruptedException, BrokenBarrierException{
        try {
            return doawait(false, 0L);
        } catch(TimeoutException toe) {
            throw new Error(toe);
        }
    }
    
    // 带超时机制，如果超时抛出TimeoutException异常
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }
```
下面分析dowait方法
```java
// dowait
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {
        final ReentrantLock lock = this.lock;
        // 首先要获取锁
        // 在condition中，被await的时候，会释放锁，然后在signal()被唤醒的时候重新获取锁
        lock.lock();
        try {
            final Generation g = generation;
            // 如果栅栏被打破，抛出BrokenBarrierException异常
            if (g.broken)
                throw new BrokenBarrierException();
            // 检查中断状态，如果中断了，要抛出InterruptedException异常
            if (Thread.interrupted()) {
                // 要打破栅栏，避免其他线程一直等待
                breakBarrier();
                throw new InterruptedException();
            }
            // index是dowait方法的返回值
            // index是count递减的值
            int index = --count;
            // 这里判断是否还有没到达栅栏处的线程
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    // 在初始化的时候，指定了通过栅栏前需要操作的操作，在这里会得到执行
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    // 如果ranAction为true，说明在执行command.run()的时候没有报错
                    ranAction = true;
                    // 唤醒等待线程，并开启新一代。
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                    // 到这里说明执行command的时候发生了异常，会打破栅栏
                    // 打破意味着唤醒所有在栅栏处等待的线程。
                        breakBarrier();
                }
            }

            // loop until tripped, broken, interrupted, or timed out
            // 如果是最后一个线程调用await，那么上面就直接返回了
            // 下面是非最后一个线程调用await的执行过程。
            for (;;) {
                try {
                    // 如果有超时机制，调用超时的Condition的await方法等待，直到最后一个线程调用await
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    // 如果到这个代码块中，则说明等待的线程在条件队列中被中断了
                    if (g == generation && ! g.broken) {
                        // 打破栅栏
                        breakBarrier();
                        // 打破栅栏之后，将InterruptException抛出给外层处理
                        throw ie;
                    } else {
                        // 到这里说明g != generation 说明新的一代已经产生，即最后一个线程await执行完成
                        // 此时没有必要再抛出InterruptedException异常，记录下这个中断信息即可
                        // 或者栅栏已经被打破了，也不会抛出InterruptedException异常，
                        // 而是之后抛出BrokenBarrierException异常
                        Thread.currentThread().interrupt();
                    }
                }
                // 唤醒之后，检查栅栏是否是破的
                if (g.broken)
                    throw new BrokenBarrierException();
                // 这个for循环除了异常，就是要从这里退出了
                // 最后一个线程在执行完指定任务(如果有)，就会开启一个新的周期
                // 然后释放锁，其他线程从Condition的await方法中得到锁并返回，然后到这里的时候，就会满足g != generation
                // 那么什么时候会不满足
                // barrierCommand执行过程中抛出了异常，那么会执行打破栅栏的操作
                // 设置broken为true，然后唤醒这些线程，这些线程会从上面的if (g.broken)这个分支抛出BrokenBarrierException异常返回
                // 还有最后一个可能，就是await超时了，这种情况不会抛出异常，而是执行后面的代码
                if (g != generation)
                    return index;
                // 如果醒来了发现超时了，那么就要打破栅栏，并抛出异常
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }
```
如何知道还有多少线程到了栅栏上，处于等待状态
```java
public int getNumberWaiting() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return parties - count;
    } finally {
        lock.unlock();
    }
}
```
判断一个栅栏是否被打破了，直接查看broken状态即可
```java
public boolean isBroken() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return generation.broken;
    } finally {
        lock.unlock();
    }
}
```
什么时候栅栏会被打破
1. 中断：如果某个线程发生了中断，那么会打破栅栏，同时抛出InterruptedException异常
2. 超时：打破栅栏，同时抛出TimeoutException异常
3. 指定执行的操作抛出了异常
# Semaphore
Semaphore类似一个资源池，每个线程需要调用acquire()方法获取资源，然后才能执行，执行完之后要release资源，让其他线程使用

semaphore其实就是AQS的共享锁的使用，所有线程共用一个资源池

创建Semaphore的时候，需要一个参数permits，这个基本上可以确定是设置给AQS的state的，然后每个线程调用acquire的时候，执行state = state - 1，release的时候执行state = state + 1, acquire 的时候state = 0说明没有资源，需要等待其他线程release

构造方法
```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
acquire方法
```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
public void acquireUninterruptibly() {
    sync.acquireShared(1);
}
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
public void acquireUninterruptibly(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireShared(permits);
}
```
对比Semaphore公平策略和非公平策略，对比tryAcquireShared方法
```java
// 公平策略：
protected int tryAcquireShared(int acquires) {
    for (;;) {
        // 区别就在于是不是会先判断是否有线程在排队，然后才进行 CAS 减操作
        if (hasQueuedPredecessors())
            return -1;
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
// 非公平策略：
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```
然后再回到acquireShared方法
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```
由于tryAcquireShared(arg)返回小于0的时候，说明state已经小于0了(没资源了)，此时acquire不能立马拿到资源，需要进入到阻塞队列等待
```java
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
release方法
```java
// 任务介绍，释放一个资源
public void release() {
    sync.releaseShared(1);
}
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        // 溢出，当然，我们一般也不会用这么大的数
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```

tryReleaseShared 方法总是会返回 true，然后是 doReleaseShared，这个也是我们熟悉的方法了.
```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

![a](https://brandonxcc.top/20200516.jpg)