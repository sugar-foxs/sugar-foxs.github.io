---
layout:     post
title:      java并发-LongAdder
subtitle:   
date:       2020-04-05
author:     sugar-foxs
catalog: 	true
tags:
    - java
    - 并发编程
---

本文是关于LongAdder的原理学习。

> 由于AtomicLong在高并发下CAS会大概率的进行重试而导致性能不佳，Doug Lea大神在java1.8又增加了LongAdder类来处理这种情况，下面我们来看看LongAdder是如何来解决高并发下的性能问题的。
<!-- more -->

# LongAdder
- 先看下LongAdder的定义和基础属性
```java
public class LongAdder extends Striped64 implements Serializable {
    private static final long serialVersionUID = 7249069246863182397L;
    public LongAdder() {
    }
}
```
- 可以看到LongAdder继承自Striped64抽象类。所以需要看下Striped64类的基础属性。
```java
// cpu核数, 用来约束cells数组的大小
static final int NCPU = Runtime.getRuntime().availableProcessors();

// 用来存放cell,数组size是2的n次方
transient volatile Cell[] cells;

// 基础值，除了cells数组中的值，base值也是总和的一部分
transient volatile long base;

// 在初始化和扩容时作为锁使用
transient volatile int cellsBusy;
```
- 这里大概知道下LongAdder里也继承了这些属性。后面都会用到，下面看下LongAdder的add方法作为切入点。

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    // cells数组已初始化过或者对base属性进行CAS操作没有成功，如果对base设置成功了，则add成功了
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        boolean uncontended = true;
        if (as == null || 
            (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```
- add的逻辑是这样的：
    - cells没初始化过，对base字段尝试CAS，如果成功了直接返回，如果不成功没进入下一步。
    - 不执行longAccumulate方法的情况：cells初始过，且随机获取cells一个cell不为null,且对获取到的cell进行一次CAS尝试，如果成功了，便不用执行longAccumulate方法。
    - 其他情况都需要执行longAccumulate方法。
- 所以总结一下add的逻辑其实就是：在执行longAccumulate方法之前，进行一些小范围的尝试，如对base字段的cas操作，对cells数组中随机一个cell的cas操作。如果尝试都不成功，才执行longAccumulate方法。如果尝试成功了，add的速度变提高了。

- 下面再具体看下longAccumulate方法：
```java
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // 线程安全的获取随机值
        h = getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        if ((as = cells) != null && (n = as.length) > 0) {
            // 随机获取的cell为null
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {
                    Cell r = new Cell(x);   // 新建一个cell,获取锁之后，将新建的cell放入对应位置。
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {         
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;   // 这里说明插入不成功，当前位置已经被其他线程插入了cell,继续下个循环      
                    }
                }
                // 这里说明其他线程获取到了锁
                collide = false;
            }
            // 随机获取的cell不是null,但是cas已经失败，这里wasUncontended是参数里传进来的，即之前add方法cas尝试失败
            else if (!wasUncontended)    
                wasUncontended = true;      // 设置wasUncontended为true，下一次循环继续，即下一次循环会跳过这个if,持续的尝试cas操作
            else if (a.cas(v = a.value, ((fn == null) ? v + x : fn.applyAsLong(v, x))))
                break;
            // cas尝试不成功，进入下一步
            // cells数组大小大于NCPU或者cells被其他线程更改过
            else if (n >= NCPU || cells != as)
                collide = false;
            else if (!collide)
                collide = true;
            // collide=true,且cells数组大小小于NCPU同时cells没被其他线程更改过
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    // 进行扩容
                    if (cells == as) {
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;
            }
            // 重新换一个cell尝试
            h = advanceProbe(h);
        }
        // 这是是初始化cells数组的逻辑
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                         
                if (cells == as) {
                    Cell[] rs = new Cell[2]; // cells初始大小是2
                    rs[h & 1] = new Cell(x); // 随机放在下标0 或者 1 上
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        // 这里是兜底的方法，上面都不成功的话，通过对base变量进行CAS操作来完成
        else if (casBase(v = base, ((fn == null) ? v + x :fn.applyAsLong(v, x))))
            break;                          // Fall back on using base
    }
}
```

- 再看下LongAdder的值是如何取的
```java
public long longValue() {
    return sum();
}
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```
- 可以看到，获取LongAdder的值是使用sum方法获取的，sum方法通过将base值和所有cells数组中cell的值相加得到。由于sum方法并没有使用锁，所以可以知道这个方法得到的值其实并不精确，因为在求和时，可能会有其他线程在修改base或者cells。

# 总结
- 在并发量比较小的时候，通过对base进行cas操作，实现add功能。此时相当于AtomicLong。
- cells数组里的所有值加上base的值就是LongAdder的值。
- 当对base进行cas操作失败后，会尝试往cells数组里插值
    - 如果cells数组为空，会先拿到锁，进行初始化cells，初始容量为2，值随机放在下标0 或者 1 上。
    - 如果cells不为空，会随机获取一个cell
        - 如果cell为空，会拿到锁，新建一个cell,并插入到当前下标。如果插入成功则返回，如果插入失败，表示当前下标被其他线程插入了，进入下一个循环。
        - 如果cell不为空，会尝试进行cas操作，成功则返回，如果不成功，判断数组大小是否大于CPU核数或者cells被其他线程更改过。
            - 如果数组大小小于核数，且拿到了锁，便开始进行扩容，扩大至之前的2倍。之后再重新随机选择一个cell进行操作。
            - 如果数组大小大于核数或者cells被其他线程更改过，此时没必要扩容，直接再重新随机选择一个cell进行操作。
- 和AtomicLong比较，LongAdder通过在高并发时将对单一变量的CAS操作分散为对cells中多个元素的CAS操作，取值时进行求和，但是因为增加了数组，使用了更多的空间。并发较低时仅对base变量进行CAS操作，与AtomicLong类原理相同。