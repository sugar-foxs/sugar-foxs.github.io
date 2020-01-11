---
layout:     post
title:      周期调度线程池ScheduledThreadPoolExecutor
subtitle:   
date:       2020-01-01 20:00:00
author:     sugar-foxs
catalog: 	true
tags:
    - java
---

> java中实现周期性任务调度的方式有多种，比如Timer，@Schedule注解，还有就是本篇将要介绍的周期调度线程池ScheduledThreadPoolExecutor，主要看看是如何进行周期调度的。
<!-- more -->
- 我们从他的周期调度方法scheduleAtFixedRate和scheduleWithFixedDelay着手：
- 写在前面，scheduleAtFixedRate和scheduleWithFixedDelay的区别是：scheduleWithFixedDelay是在任务执行完之后延迟一段时间再执行下一次任务；scheduleAtFixedRate是每隔一段时间就执行一次，但如果任务执行时间比周期时间长，下一次任务会在上次任务执行完立即执行，所以周期时间设置一般需要比任务执行时间长。

# scheduleAtFixedRate
```java
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit){
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    // 生成周期任务ScheduledFutureTask
    ScheduledFutureTask<Void> sft = new ScheduledFutureTask<Void>(
        command,
        null,
        // triggerTime是计算出执行的时间
        triggerTime(initialDelay, unit),
        // 这里和scheduleWithFixedDelay不一样，另一个是unit.toNanos(-delay)
        unit.toNanos(period));
    // 这个方法默认只是原样返回了sft
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    // 核心方法在这
    delayedExecute(t);
    return t;
}
```

- 我们进一步看下delayedExecute方法，scheduleWithFixedDelay方法最终也是执行的该方法。
```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        reject(task);
    else {
        // 任务加入到队列中
        super.getQueue().add(task);
        if (isShutdown() &&
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            // 预启动线程，运行的线程数小于核心线程数，启动一个线程；运行的线程数==0，启动一个非核心线程数。保证至少有一个运行的线程
            ensurePrestart();
    }
}
```

- 上面的代码逻辑是线程池正常，会把任务放进队列中，从构造函数能够看到是DelayedWorkQueue。但是没看到任务是何时怎么被执行的。所以我们得找到从队列取出任务执行的代码。
```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
        new DelayedWorkQueue());
}
```

- 现在需要知道的是任务何时从队列中取出，和如何进行周期执行任务。
- 任务被取出执行是在父类ThreadPoolExecutor中执行的，是线程从队列中任务进行执行，对于ScheduledThreadPoolExecutor线程池来说，即从DelayedWorkQueue队列中取出ScheduledFutureTask任务来执行。这时候我们先去看下DelayedWorkQueue的取出逻辑。
```java
public RunnableScheduledFuture<?> take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
            // 取出第一个任务
            RunnableScheduledFuture<?> first = queue[0];
            if (first == null)
                available.await();
            else {
                // 第一个任务的delay延时时间
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0)
                    // 如果delay<0,会被立即取出，剩余任务根据delay时间重新将最小的delay放到第一个。
                    // 其实就是一个最小树
                    return finishPoll(first);
                first = null; // delay>0,即需要等待，将引用置null
                if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        // 等待delay时间，进入下一次循环，取出第一个任务
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

// delay = time-now
public long getDelay(TimeUnit unit) {
        return unit.convert(time - now(), NANOSECONDS);
    }
```
- 从上面可以看出从DelayedWorkQueue队列中取出ScheduledFutureTask任务，是获取第一个delay最小的任务，如果delay<0,会被立即执行，否知等待delay时间再取出执行，下面看下任务执行逻辑以及如何周期执行的。
- 下面是ScheduledFutureTask的run方法，是ScheduledThreadPoolExecutor的内部类。
```java
public void run() {
    // 首先判断是否是周期性任务(period!=0)
    boolean periodic = isPeriodic();
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    else if (!periodic)
        ScheduledFutureTask.super.run();
    else if (ScheduledFutureTask.super.runAndReset()) {
        // runAndReset是任务执行完成
        // 主要看周期任务的逻辑，首先设置下一次任务时间
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
}
```

- 周期性任务首先设置下一次任务时间，执行reExecutePeriodic方法，下面分别看下setNextRunTime和reExecutePeriodic方法

```java
private void setNextRunTime() {
    long p = period;
    if (p > 0)
        // 对应scheduleAtFixedRate的period，算出每次执行的时间
        time += p;
    else
        // 对应scheduleAtFixedDelay的period，执行时间是当前时间+period
        time = triggerTime(-p);
}

long triggerTime(long delay) {
    return now() +
        ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}
```

- 这里就看出了scheduleAtFixedRate和scheduleAtFixedDelay的不同了，他们计算下一次任务执行时间的方式不一样。scheduleAtFixedRate的执行时间是确定的，每隔period执行一次，但是真正执行的时间不一定，因为是在上一个任务执行完后才会将下一个任务加到队列中，所以下一个任务取决于上一个任务的执行结束时间和周期时间，如果超过了本来算好应该执行的时间，即这时delay=time-now<0,任务被取出时会立即执行；scheduleAtFixedDelay的执行时间是上一个任务执行完，当前时间加period。

- 下面看下reExecutePeriodic方法：

```java
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
    if (canRunInCurrentRunState(true)) {
        // 将重新设置时间的任务放进DelayedWorkQueue
        super.getQueue().add(task);
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
```

# 总结
- ScheduleThreadPoolExecutor可以使用scheduleAtFixedRate和scheduleAtFixedDelay两种周期调度方式。
- scheduleAtFixedRate和scheduleAtFixedDelay的不同是：他们计算下一次任务执行时间的方式不一样。
    - scheduleAtFixedRate的执行时间是确定的，每隔period执行一次，但是真正执行的时间不一定，因为是在上一个任务执行完后才会将下一个任务加到队列中，所以下一个任务取决于上一个任务的执行结束时间和周期时间，如果超过了本来算好应该执行的时间，即这时delay=time-now<0,任务被取出时会立即执行。
    - scheduleAtFixedDelay的执行时间是上一个任务执行完，当前时间+period。

- scheduleAtFixedRate和scheduleAtFixedDelay的相同点是：都是将任务放入延迟队列中来是实现周期性调度的效果，任务执行完计算下一次执行的时间并放入队列。

- 后面还会研究下DelayedWorkQueue的实现。








