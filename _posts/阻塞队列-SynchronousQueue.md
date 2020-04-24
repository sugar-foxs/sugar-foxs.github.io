---
layout:     post
title:      阻塞队列-SynchronousQueue
subtitle:   
date:       2020-04-06
author:     sugar-foxs
catalog: 	true
tags:
    - java
    - 并发编程
---

本文主要介绍阻塞队列其中一种实现：SynchronousQueue。

<!-- more -->

> 先大概了解一下这个阻塞队列的基本情况：当生产线程生产，如果当前没有线程消费产品，此生产线程必须阻塞，等待一个消费线程调用take操作，take操作将会唤醒该生产线程，同时消费线程会获取生产线程的产品。相反，消费线程当消费线程时，如果当前没有线程生产产品，此消费线程必须阻塞，等待一个生产线程调用put操作，put操作将会唤醒消费线程获取生产线程的产品。
- 下面通过源码的解读来看看是如何做到的。

# SynchronousQueue

## 构造函数和属性
- 先看下SynchronousQueue的构造函数：
```java
// cpu核数
static final int NCPUS = Runtime.getRuntime().availableProcessors();

// 设置超时的最大自旋次数
static final int maxTimedSpins = (NCPUS < 2) ? 0 : 32;

// 没设置超时的最大自旋次数
static final int maxUntimedSpins = maxTimedSpins * 16;

// 
static final long spinForTimeoutThreshold = 1000L;

private transient volatile Transferer<E> transferer;

// 默认是不公平队列
public SynchronousQueue() {
    this(false);
}

// TransferQueue是公平队列，TransferStack是不公平队列，它们都继承自Transferer
public SynchronousQueue(boolean fair) {
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}

abstract static class Transferer<E> {
    // 此方法既负责put,也负责take,下面看如何做到的。TransferQueue和TransferStack分别实现此方法。
    abstract E transfer(E e, boolean timed, long nanos);
}

```

## put,take,offer,poll
- 上面的解释只是给一个大概了解，具体如何实现的还得看put和take等方法。

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, false, 0) == null) {
        // put失败，会抛出中断异常
        Thread.interrupted();
        throw new InterruptedException();
    }
}

public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);
    if (e != null) // 消费成功
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
// offer和poll方法可设置超时时间，也可以被中断；不带参数的offer和poll方法：超时时间为0，生产不成功，立即返回
public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}

public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = transferer.transfer(null, true, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}
public E poll() {
    return transferer.transfer(null, true, 0);
}
```

- 生产和消费数据的方法最终都是用的transfer方法，只是参数不一样，通过参数识别操作是生产还是消费。
- 下面看具体子类的transfer方法，我们以公平模式的TransferQueue为例看下，可以先到后面看图更好理解一点：
```java
TransferQueue() {
    QNode h = new QNode(null, false); // 初始化一个dummy node，头尾指针都指向它。
    head = h;
    tail = h;
}

E transfer(E e, boolean timed, long nanos) {

    QNode s = null; // constructed/reused as needed
    boolean isData = (e != null); // isData=true表示put，否则表示take

    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         //自旋等待初始化
            continue;                       

        if (h == t || t.isData == isData) { // 队列为空或者尾节点和当前操作模式相同
            QNode tn = t.next;
            if (t != tail)                  // 防止其他线程并发操作导致的不一致读
                continue;
            if (tn != null) {               // lagging tail
                advanceTail(t, tn);         // cas设置tn为新的尾节点
                continue;                  
            }
            if (timed && nanos <= 0)        
                return null;
            if (s == null)
                s = new QNode(e, isData);
            if (!t.casNext(null, s))        // cas 设置next
                continue;                   // cas没成功，自旋

            advanceTail(t, s);              // cas成功的话，即插入成功，cas设置尾节点
            Object x = awaitFulfill(s, e, timed, nanos);// 看下下面此方法的具体注释
            if (x == s) {                   // 等待被中断
                clean(t, s);
                return null;
            }

            if (!s.isOffList()) {           // not already unlinked
                advanceHead(t, s);          // unlink if head
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
            return (x != null) ? (E)x : e;

        } else {                            // 互补模式，当前操作和尾节点的模式不同
            QNode m = h.next;               // node to fulfill
            if (t != tail || m == null || h != head)
                continue;                   // 自旋等待达到此状态：h==head,t=tail,m!=null

            Object x = m.item;              // 消费对头的下一个节点
            if (isData == (x != null) ||    
                x == m ||                   
                !m.casItem(x, e)) {         // cas item 
                advanceHead(h, m);          // dequeue and retry
                continue;                   // 上面的条件满足一个就继续循环
            }

            advanceHead(h, m);              // cas item 成功之后，设置m为新的head
            LockSupport.unpark(m.waiter);   // 唤醒m的waiter线程
            return (x != null) ? (E)x : e;
        }
    }
}


Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
    /* Same idea as TransferStack.awaitFulfill */
    final long deadline = timed ? System.nanoTime() + nanos : 0L; // 等待的截止时间
    Thread w = Thread.currentThread();
    int spins = ((head.next == s) ?
                    (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        if (w.isInterrupted())
            s.tryCancel(e);  // 被中断
        Object x = s.item;
        if (x != e)
            return x; // 返回的是s
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) { // 阻塞超时
                s.tryCancel(e);
                continue;
            }
        }
        if (spins > 0) // 自旋次数
            --spins;
        else if (s.waiter == null)
            s.waiter = w; // 设置当前线程为s的waiter，表示当前线程等待另一种角色出现，即消费线程被生产线程唤醒，生产线程被消费线程唤醒
        else if (!timed)
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanos);
    }
}
```

- 光看代码还是不能清晰的看出具体的执行规则，我画个图解释下吧。

## 图解
1. 初始状态,false代表是消费线程，true代表是生成线程
![1.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1gdpvmmann9j20fa07o0to.jpg)
2. 消费线程a来消费数据，但是现在没有生产线程在阻塞，所以线程a阻塞
![2.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1gdpvn0ky7lj20lg0ck0v3.jpg)
3. 消费线程b来消费数据，也进入阻塞状态
![3.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1gdpvnbalc4j20wu0dawi7.jpg)
4. 现在来了一个生产线程c,唤醒第一个阻塞的消费线程，生产的元素被线程a获取
![4.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1gdpvnkjr32j20uk0e2jvt.jpg)
5. 生产线程又生产了一个元素，线程b被唤醒，获取到生产的元素
![5.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1gdpvnrp44dj20uq08iwgq.jpg)
6. 生产线程c又生产数据的话，如果目前没有消费线程阻塞，生产线程c会阻塞
![6.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1gdpwr3a6mij20m80eaq5y.jpg)

- 阻塞也可以指定超时时间，阻塞超时的话相当于操作失败，会从队列中去除。
