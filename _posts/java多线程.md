---
layout:     post
title:      java多线程
subtitle:   
date:       2017-06-16
author:     sugar-foxs
catalog: 	true
tags:
    - java
---

> 首先先弄清两个概念:并发和并行.
并行指两个及多个事件实际意义的同时进行,而并发是宏观上的并行,其实一个cpu上还是顺序执行,只是cpu通过调度算法快速的切换不同事件执行,达到看似多个事件同时进行的效果.说白了并发是为了提高效率接近并行的执行效果.而实现并发的手段是多进程或多线程.
<!-- more -->
本文主要从线程的产生，状态，同步，通信，安全等方面介绍java中的多线程:

# 线程产生

## 继承Thread类

```java
public class Main{
   public static void main(String args[]){
       MyThread t1 =new MyThread();
       t2.start();
   }
}
class MyThread extends Thread {
     public void run() {
            //TODO
     }
}
```

## 实现Runnable接口,也是使用Thread来执行这个任务

```java
public class Main{
   public static void main(String args[]){
        MyRunnable r = new MyRunnable();
        Thread t1 = new Thread(r); t1.start(); 
    }
}

class MyRunnable implements Runnable { 
    public void run() { 
        //TODO 
    }
}
```
## 实现Callable接口 ,任务完成后可返回结果,但前两种方法不行

```java
public class Main{
    MyCallable c = new MyCallable();
    FutureTask<Integer> result = new FutureTask<Integer>(c); //FutureTask类既实现了Runnable接口,也实现了Future<V>接口
    new Thread(result).start();  //启动线程
    try {
        Integer sum = result.get();
        System.out.println(sum);
    } catch (InterruptedException | ExecutionException e) {
        e.printStackTrace();
    }
}
class MyCallable implements Callable<Integer>{
      public Integer call() throws Exception{
              return 1;
      }  
} 
```

之后会详细介绍Callable,Future,FutureTask

# 线程状态
1. 新状态：线程对象已经创建，还没有在其上调用start()方法。

2. 可运行状态：当线程有资格运行，但调度程序还没有把它选定为运行线程时线程所处的状态。当start()方法调用时，线程首先进入可运行状态。在线程运行之后或者从阻塞、等待或睡眠状态回来后，也返回到可运行状态。

3. 运行状态：线程调度程序从可运行池中选择一个线程作为当前线程时线程所处的状态。这也是线程进入运行状态的唯一一种方式。

4. 等待/阻塞/睡眠状态：这是线程有资格运行时它所处的状态。实际上这个三状态组合为一种，其共同点是：线程仍旧是活的，但是当前没有条件运行。换句话说，它是可运行的，但是如果某件事件出现，他可能返回到可运行状态。

5. 死亡态：当线程的run()方法完成时就认为它死去。这个线程对象也许是活的，但是，它已经不是一个单独执行的线程。线程一旦死亡，就不能复生。 如果在一个死去的线程上调用start()方法，会抛出java.lang.IllegalThreadStateException异常。

# 线程同步

## 互斥同步
- 1.使用synchronized修饰方法或语句块

synchronized是一个内置锁的加锁机制，当某个方法加上synchronized关键字后，就表明要获得该内置锁才能执行，但并不能阻止其他线程访问不需要获得该内置锁的方法。

synchronized可以加在方法上，也可以直接加在对象上，从而保证一段代码只能有一个线程在运行，保证线程的同步。

- synchronized加在不同方法上有什么不同？

    - 如果synchronized加在一个类的普通方法上，那么相当于synchronized(this)。是一个对象锁。

    - 如果synchronized加载一个类的静态方法上，那么相当于synchronized(Class对象)。是一个类锁，一个类只有一个类锁。

- 2.使用显式锁ReentrantLock

    - 这里是[synchronized和ReentrantLock的比较](https://blog.csdn.net/u013014724/article/details/76098517)

## 非阻塞同步

- 使用CAS算法

# 线程通信

1. wait和notify机制

2. Condition的等待/通知机制:await,signal,signalAll

# 线程安全

> 为了并发执行程序，提高执行效率，使用多线程编程，而多线程编程就会涉及到安全问题，比如多个线程访问共享的可变数据的问题。
如果你的代码所在的进程中有多个线程在同时运行，而这些线程可能会同时运行这段代码。如果每次运行结果和单线程运行的结果是一样的，而且其他的变量的值也和预期的是一样的，就是线程安全的。

- 但是如果不安全，如何实现线程安全呢？

1. 通过上面的线程同步的方法

2. 使用本地线程存储ThreadLocal
