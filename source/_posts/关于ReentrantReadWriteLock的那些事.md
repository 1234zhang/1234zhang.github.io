---
layout: '[layout]'
title: 关于ReentrantReadWriteLock的那些事
date: 2020-06-22 15:13:10
tags:
---

## 关于ReentrantReadWriteLock的使用例子
下面是javadoc中对ReentrantReadWriteLock的使用例子
```java
class CacheData {
    Object data;
    volatile boolean cacheValid;
    //读写锁实例
    final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    void processCachedData() {
        // 获取读锁
        lock.readLock().lock();
        if (!cacheValid) { // 如果缓存过期了，或者为null
            // 释放读锁，加写锁（如果没有释放掉读锁就加写锁，会导致死锁的产生）
            lock.readLock().unlock();
            lock.writeLock().lock();

            try {
                if (!cacheCalid) { // 这里判断的主要目的是查看是否有其他线程执行过程中，对缓存空间进行了写操作
                    data = ...;
                    cacheValue = true;

                }
                //  获取读锁（持有写锁的情况下，允许获取读锁，这是是锁降级
                lock.writeLock().unlock();
            }
        }
        try {
            use(data);
        } finally {
            // 释放锁
            lock.readLock().unlock();
        }
    }
}
```