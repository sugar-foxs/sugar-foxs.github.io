---
layout:     post
title:      阻塞队列-ArrayBlockingQueue
subtitle:   
date:       2020-01-12
author:     sugar-foxs
catalog: 	true
tags:
    - java
    - 并发编程
---

本文主要介绍阻塞队列其中一种实现：ArrayBlockingQueue。
<!-- more -->

> ArrayBlockingQueue是用数组实现的线程安全的有界的阻塞队列。ArrayBlockingQueue实现了BlockingQueue接口，继承了AbstractQueue，内部使用Object数组存储元素，容量capacity在构造器中给出，之后不会自动扩容。使用ReentrantLock和两个Condition变量，实现互斥访问。构造器中可以指定锁是公平锁还是非公平锁。使用两个变量putIndex，takeIndex表示入队和出队的位置。

# 源码解析
## offer
```java
public boolean offer(E e) {
    checkNotNull(e);
    // 加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count == items.length)
            // 队列已满,返回false
            return false;
        else {
            // 队列没满，可以入队
            enqueue(e);
            return true;
        }
    } finally {
        lock.unlock();
    }
}
```
- 队列满了，返回false；不满，入队，返回true。

## enqueue
- 入队
```java
private void enqueue(E x) {
    final Object[] items = this.items;
    items[putIndex] = x;
    if (++putIndex == items.length)
        //因为队列由数组实现,当队列末尾没空间时,从数组开端入队
        putIndex = 0;
    count++;
    notEmpty.signal();
}
```
- 队列的实现是一个循环数组，当数组最后一个有元素时，设置下一个入队的位置为0。

## put
- put和offer都是入队，但是不一样的地方是：put方法会一直阻塞到有空间加入，当收到notFull.signal()时，便可以入队，且入队的操作可能会被中断。
```java
public void put(E e) throws InterruptedException {
    checkNotNull(e);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 锁可中断
    try {
        //这里是和offer不一样的地方,一直阻塞到有空间加入,当收到notFull.signal()时,便可以入队
        while (count == items.length)
            notFull.await();
        enqueue(e);
    } finally {
        lock.unlock();
    }
}
```

## poll
- 从队列中取出元素。
- 第一个方法，队列中没有元素，直接返回null，否则返回第一个。
```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        return (count == 0) ? null : dequeue();
    } finally {
        lock.unlock();
    }
}
- 第二个方法比第一个方法多了一个超时时间参数，超时之后还没元素可取，便返回null。
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0) {
            if (nanos <= 0)
                return null;
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```

## dequeue
- 出队
```java
private E dequeue() {
    final Object[] items = this.items;
    E x = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length)
        takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();// 有元素出队，队列现在不满了，通知其他线程
    return x;
}
```

## take
- 一直阻塞，直到有队列中有元素。
```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        while (count == 0)
            notEmpty.await();
        return dequeue();
    } finally {
        lock.unlock();
    }
}
```


- offer和poll对应,put和take对应。

## remove
- 删除元素
```java
public boolean remove(Object o) {
    if (o == null) return false;
    final Object[] items = this.items;
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        if (count > 0) {
            final int putIndex = this.putIndex;
            int i = takeIndex;
            // 遍历队列，从对头到对尾，找到元素删除。
            do {
                if (o.equals(items[i])) {
                    removeAt(i);
                    return true;
                }
                if (++i == items.length)
                    i = 0;
            } while (i != putIndex);
        }
        return false;
    } finally {
        lock.unlock();
    }
}
```

## removeAt
```java
void removeAt(final int removeIndex) {
    final Object[] items = this.items;
    if (removeIndex == takeIndex) {
        //如果正好等于出队下标,正常出队
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {
        //将要删除的元素后面的元素往前移
        final int putIndex = this.putIndex;
        for (int i = removeIndex;;) {
            int next = i + 1;
            if (next == items.length)
                next = 0;
            if (next != putIndex) {
                items[i] = items[next];
                i = next;
            } else {
                items[i] = null;
                this.putIndex = i;
                break;
            }
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    notFull.signal();
}
```

# ArrayBlockingQueue使用示例

## 生产者Producer
```java
public class Producer implements Runnable{
    //容器
    private final ArrayBlockingQueue<Bread> queue;
    Producer(ArrayBlockingQueue<Bread> queue){
        this.queue = queue;
    }
    
    @Override
    public void run() {
        while(true){
            produce();
        }
    }

    private void produce(){
        /**
         * put()方法是如果容器满了的话就会把当前线程挂起
         * offer()方法是容器如果满的话就会返回false。
         */
        try {
            Bread bread = new Bread();
            queue.put(bread);
            System.out.println("Producer:"+bread);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 消费者Consumer
```java
public class Consumer implements Runnable{
    //容器
    private final ArrayBlockingQueue<Bread> queue;

    Consumer(ArrayBlockingQueue<Bread> queue){
        this.queue = queue;
    }
    
    @Override
    public void run() {
        while (true) {
            consume();
        }
    }
    
    private void consume(){
        /**
         * take()方法和put()方法是对应的，从中拿一个数据，如果拿不到线程挂起
         * poll()方法和offer()方法是对应的，从中拿一个数据，如果没有直接返回null
         */
        try {
            Bread bread = queue.take();
            System.out.println("consumer:"+bread);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 生产的元素
```java
@Data
public class Bread {
    private int size;
}
```

## 测试Test
```java
public class Test {
    public static void main(String[] args) {
        int capacity = 10;
        ArrayBlockingQueue<Bread> queue = new ArrayBlockingQueue<>(capacity);
        new Thread(new Producer(queue)).start();
        new Thread(new Producer(queue)).start();
        new Thread(new Consumer(queue)).start();
        new Thread(new Consumer(queue)).start();
        new Thread(new Consumer(queue)).start();
    }
}
```

- producer和consumer会一直在生产和消费，输出就不贴了。

# 总结
- ArrayBlockingQueue是用数组实现的线程安全的有界的阻塞队列。使用ReentrantLock和两个Condition变量实现生产和消费的阻塞。