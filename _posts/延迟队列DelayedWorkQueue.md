---
layout:     post
title:      延迟队列DelayedWorkQueue
subtitle:   
date:       2020-01-12
author:     sugar-foxs
catalog: 	true
tags:
    - java
---

本文主要介绍ScheduledThreadPoolExecutor的内部类DelayedWorkQueue，它是周期线程池专用的延迟队列。
<!-- more -->

# DelayedWorkQueue
- 先看下它的fields
```java
static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {
    // DelayedWorkQueue是基于堆实现的。每个ScheduledFutureTask还将其索引记录到堆数组中。这消除了在取消时查找任务的需要，极大地加快了删除速度（从O（n）降到O（log n））。
    // 所有堆操作都必须记录索引更改-主要在siftUp和siftDown中。删除后，任务的heapIndex设置为-1。
    private static final int INITIAL_CAPACITY = 16;
    // 堆的底层数据结构是数组
    private RunnableScheduledFuture<?>[] queue =
        new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
    private final ReentrantLock lock = new ReentrantLock();
    private int size = 0;

    // leader线程用于在队列头等待任务。类似于主从模式，用于最大程度地减少不必要的定时等待。当一个线程成为leader时，它仅等待下一个延迟过去，而其他线程将无限期地等待。 leader线程必须在从take或poll返回之前通知其他线程，除非其他线程成为过渡期间的leader。每当队列头被过期时间更早的任务替换时，leader字段都会被重置为null来使之无效，并且会通知一些等待线程（但不一定是当前领导者）。 因此，等待线程必须准备好在等待时获得并失去领导能力。
    private Thread leader = null;

    // 当较新的任务在队列的开头可用或新的线程可能需要成为领导者时，会使用这个condition通知。
    private final Condition available = lock.newCondition();
```

- 从这可以知道DelayedWorkQueue基于最小堆（数组实现）实现，但也有一些优化。下面具体看看如何操作这个延迟队列的。通过队列的结构变化了解延迟队列的实现。

## setIndex
```java
// 如果任务是ScheduledFutureTask，记录它在堆中的下标，这是前面所说的为了更快的删除，具体为什么更快，后面会看到。
private void setIndex(RunnableScheduledFuture<?> f, int idx) {
    if (f instanceof ScheduledFutureTask)
        ((ScheduledFutureTask)f).heapIndex = idx;
}
```

## offer 
- 向队列增加任务
```java
public boolean offer(Runnable x) {
    if (x == null) throw new NullPointerException();
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    // 加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        int i = size;
        if (i >= queue.length)
            grow();//数组满了之后扩容
        size = i + 1;
        if (i == 0) {
            // 增加的是堆里的第一个任务
            queue[0] = e;
            setIndex(e, 0);
        } else {
            // 不是第一个,调用siftUp，下面看下具体逻辑，大概就是排序，将最小delay的任务放到第一个
            siftUp(i, e);
        }
        // 当前任务是第一个，leader置null,通知其他线程任务可用
        if (queue[0] == e) {
            leader = null;
            available.signal();
        }
    } finally {
        lock.unlock();
    }
    return true;
}
```

## grow 
```java
private void grow() {
    int oldCapacity = queue.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
    if (newCapacity < 0) // overflow
        newCapacity = Integer.MAX_VALUE;
    queue = Arrays.copyOf(queue, newCapacity);
}
```
- 扩容比较简单，新数组容量增加50%，复制到新数组。

## siftUp
```java
private void siftUp(int k, RunnableScheduledFuture<?> key) {
    while (k > 0) {
        // 跟它的父节点比较，如果比父节点小，和父节点交换，直到到达堆顶，否则退出。
        int parent = (k - 1) >>> 1;
        RunnableScheduledFuture<?> e = queue[parent];
        if (key.compareTo(e) >= 0)
            break;
        queue[k] = e;
        setIndex(e, k);
        k = parent;
    }
    queue[k] = key;
    setIndex(key, k);
}
```
- 和之前的猜想一样，通过比较delay值从下往上将新的任务移动到合适的位置，保证delay值比父节点的大，即最小delay的任务必须在第一个。

## poll
- 从队列取任务，没有合适的直接返回null
```java
public RunnableScheduledFuture<?> poll() {
    // 加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        RunnableScheduledFuture<?> first = queue[0];
        // 如果第一个为null或者第一个delay值大于0，直接返回null。否则执行finishPoll方法。
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return finishPoll(first);
    } finally {
        lock.unlock();
    }
}

private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
    // 把最后一个元素拿出来，执行siftDown方法，返回第一个元素。
    int s = --size;
    RunnableScheduledFuture<?> x = queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x); // 大概也是排序，保证第一个元素是最小的。
    setIndex(f, -1); // 第一个任务被取出，heapIndex置为-1.
    return f;
}
```
- 从代码中可以看到，如果第一个任务为null或者第一个任务的delay值大于0，直接返回null。否则返回第一个任务，并执行siftDown方法，从上往下调整。

## siftDown
```java
private void siftDown(int k, RunnableScheduledFuture<?> key) {
    // 概念上key是放在k的位置上， 然后从上往下比较
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        RunnableScheduledFuture<?> c = queue[child];
        int right = child + 1;
        // 选出key的两个子节点中小的那个，然后和最小的那个比较，如果key最小，退出，否则和小的那个互换，继续下一轮循环。
        if (right < size && c.compareTo(queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo(c) <= 0)
            break;
        queue[k] = c;
        setIndex(c, k);
        k = child;
    }
    queue[k] = key;
    setIndex(key, k);
}
```
- 和siftUp不同的是：方向不一样，该方法是从上往下进行调整，目标一致，都是为了保证节点比它的子节点的delay值小。即维持最小堆的性质。

## take
- 取任务，必须等到有可以执行的任务才退出。
```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    // 加锁
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        // 死循环，直到拿到可以执行的任务
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    return finishPoll(first);
                first = null; // 等待的时候引用置null
                // 主线程等待delay时间
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```
- 第一个任务的delay大于0的时候：
    - leader不为null, 释放锁，等待唤醒，因为leader在等待任务
    - leader为null,设置当前线程为leader线程，等待delay时间，释放锁，因为leader是当前线程，其他线程只能等待，当过了delay时间之后，当前线程会拿到第一个任务返回。返回之前如果堆顶任务不为null,唤醒其他任意一个线程去取任务。

## remove
```java
public boolean remove(Object x) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 拿到任务在堆中的下标，即数组的下标
        int i = indexOf(x);
        if (i < 0)
            return false;
        setIndex(queue[i], -1);// 删除这个任务
        int s = --size;
        RunnableScheduledFuture<?> replacement = queue[s];
        queue[s] = null;
        // 把最后一个任务放到当前位置，进行调整
        if (s != i) {
            // 先尝试往下调整
            siftDown(i, replacement);
            // 如果siftDown没动过位置，当前就是最小的，便往上调整。
            if (queue[i] == replacement)
                siftUp(i, replacement);
        }
        return true;
    } finally {
        lock.unlock();
    }
}

private int indexOf(Object x) {
    if (x != null) {
        // 这里可以看到前面记录index的好处了，当删除ScheduledFutureTask任务时是获取下标，可以快速拿到
        if (x instanceof ScheduledFutureTask) {
            int i = ((ScheduledFutureTask) x).heapIndex;
            if (i >= 0 && i < size && queue[i] == x)
                return i;
        } else {
            // 遍历拿到下标
            for (int i = 0; i < size; i++)
                if (x.equals(queue[i]))
                    return i;
        }
    }
    return -1;
}
```
- 删除时，利用增加ScheduledFutureTask任务时记录了其下标，提高了部分任务的删除速度。

## drainTo
```java
public int drainTo(Collection<? super Runnable> c) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        RunnableScheduledFuture<?> first;
        int n = 0;
        // 从队列头把所有可以执行的任务取出放到c中
        while ((first = peekExpired()) != null) {
            c.add(first);   // In this order, in case add() throws.
            finishPoll(first);
            ++n;
        }
        return n;
    } finally {
        lock.unlock();
    }
}
```
- drainTo方法是一次性将多个可以执行的任务取出。
- 这是具有不同参数的drainTo方法，限制了取出的最大任务数，其他的和drain没区别。
```java
public int drainTo(Collection<? super Runnable> c, int maxElements) {
    if (c == null)
        throw new NullPointerException();
    if (c == this)
        throw new IllegalArgumentException();
    if (maxElements <= 0)
        return 0;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        RunnableScheduledFuture<?> first;
        int n = 0;
        while (n < maxElements && (first = peekExpired()) != null) {
            c.add(first);
            finishPoll(first);
            ++n;
        }
        return n;
    } finally {
        lock.unlock();
    }
}
```

## peekExpired
- 取出第一个超时的任务，但和poll不一样的是这里不会从堆中删除。否则返回null
```java
private RunnableScheduledFuture<?> peekExpired() {
    RunnableScheduledFuture<?> first = queue[0];
    return (first == null || first.getDelay(NANOSECONDS) > 0) ?
        null : first;
}
```

# 总结
- DelayedWorkQueue是阻塞队列，使用最小堆（数组）和ReentrantLock实现。比较的是任务的delay值，delay值最小的永远在堆顶。
- 线程获取锁才可以去拿任务，任务delay小于0，直接取出。释放锁。但是任务delay大于0的时候就需要leader线程了。
    - leader不为null, 释放锁，等待唤醒，因为leader在等待任务。
    - leader为null,设置当前线程为leader线程，等待delay时间之后释放锁，因为leader是当前线程，其他线程只能等待，当过了delay时间之后，当前线程会拿到第一个任务返回。返回之前如果堆顶任务不为null,唤醒其他任意一个线程去取任务。
- 加入ScheduledFutureTask任务时，会记录其下标，所以删除时可以立即拿到下标，不需要线性搜索，提高了部分任务的删除速度。
- 其他的操作跟一般的堆类似。

