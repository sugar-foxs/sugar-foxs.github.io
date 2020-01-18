---
layout:     post
title:      阻塞队列-LinkedBlockingQueue
subtitle:   
date:       2020-01-13
author:     sugar-foxs
catalog: 	true
tags:
    - java
    - 并发编程
---

本文主要介绍阻塞队列其中一种实现：LinkedBlockingQueue。

<!-- more -->

> LinkedBlockingQueue是一个基于链表的、范围任意的阻塞队列。按FIFO排序元素，队列的头部是在队列中时间最长的元素。队列的尾部是在队列中时间最短的元素。新元素插入到队列的尾部，并且队列检索操作会获得位于队列头部的元素。链接队列的吞吐量通常要高于基于数组的队列，但是在大多数并发应用程序中，其可预知的性能要低。
- LinkedBlockingQueue多用于任务队列（单线程发布任务，任务满了就停止等待阻塞，当任务被完成消费少了又开始发布任务)如果不指定容量默认为Integer.MAX_VALUE。通过putLock和takeLock两个锁进行同步，两个锁分别实例化notFull和notEmpty两个Condtion，用来协调多线程的存取动作。

# offer
- 入队时使用putLock锁
```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```

# enqueue
```java
private void enqueue(Node<E> node) { //入队
    last = last.next = node;
}
```

# poll 
- 出队时使用takeLock锁,队列头结点是一个空节点。
```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout);
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

# dequeue
```java
private E dequeue() {   //出队
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; //循环引用，虚拟机的可达性分析算法帮助回收
    head = first;
    E x = first.item;
    first.item = null;   //因为头结点是一个空节点
    return x;
}
```
- 其中某些方法(如remove,toArray,toString,clear等)的同步需要同时获得这两个锁，并且总是先putLock.lock紧接着takeLock.lock(在同一方法fullyLock中)，这样的顺序是为了避免可能出现的死锁情况。

# remove
```java
public boolean remove(Object o) {
    if (o == null) return false;
    fullyLock();
    try {
        for (Node<E> trail = head, p = trail.next;
             p != null;
             trail = p, p = p.next) {
            if (o.equals(p.item)) {
                unlink(p, trail);
                return true;
            }
        }
        return false;
    } finally {
        fullyUnlock();
    }
}
void fullyLock() {
    putLock.lock();
    takeLock.lock();
}
```