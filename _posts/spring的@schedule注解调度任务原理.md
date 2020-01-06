---
layout:     post
title:      spring的@schedule注解调度任务原理
subtitle:   
date:       2019-12-24
author:     sugar-foxs
header-img: img/home-bg-o.jpg
catalog: 	true
tags:
    - java
    - spring
---

本篇文章主要阅读源码看看@schdule注解是如何工作的。
<!-- more -->

首先看下@schedule注解：
```java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
// Schedules注解可以包含多个Scheduled，即多个调度任务
@Repeatable(Schedules.class)
public @interface Scheduled {
	// corn 表达式
	String cron() default "";
	//时区，默认服务器本地时区
	String zone() default "";

	// 一次任务调用结束和下一次调用开始之间的间隔
	long fixedDelay() default -1;
	// 上一个的string格式
	String fixedDelayString() default "";

	// 两次任务执行开始之间的间隔
	long fixedRate() default -1;
	// 上一个的string格式
	String fixedRateString() default "";

	// 初始延迟
	long initialDelay() default -1;
	// 上一个的string格式
	String initialDelayString() default "";
}
```

- 下面看这个注解在什么地方被调用的，通过查看这个注解里的方法的调用方，可以看到是被ScheduledAnnotationBeanPostProcessor调用的。从中可以看到processScheduled方法是解析注解参数生成具体任务的方法：

```java
protected void processScheduled(Scheduled scheduled, Method method, Object bean) {
    Method invocableMethod = AopUtils.selectInvocableMethod(method, bean.getClass());
    Runnable runnable = new ScheduledMethodRunnable(bean, invocableMethod);
    boolean processedSchedule = false;
    String errorMessage =
            "Exactly one of the 'cron', 'fixedDelay(String)', or 'fixedRate(String)' attributes is required";

    Set<ScheduledTask> tasks = new LinkedHashSet<>(4);

    // 取出初始延迟参数
    long initialDelay = scheduled.initialDelay();
    String initialDelayString = scheduled.initialDelayString();
    if (StringUtils.hasText(initialDelayString)) {
        Assert.isTrue(initialDelay < 0, "Specify 'initialDelay' or 'initialDelayString', not both");
        if (this.embeddedValueResolver != null) {
            initialDelayString = this.embeddedValueResolver.resolveStringValue(initialDelayString);
        }
        if (StringUtils.hasLength(initialDelayString)) {
            try {
                initialDelay = parseDelayAsLong(initialDelayString);
            }
            catch (RuntimeException ex) {
                throw new IllegalArgumentException(
                        "Invalid initialDelayString value \"" + initialDelayString + "\" - cannot parse into long");
            }
        }
    }

    // 解析cron表达式，生成CronTask任务
    String cron = scheduled.cron();
    if (StringUtils.hasText(cron)) {
        String zone = scheduled.zone();
        if (this.embeddedValueResolver != null) {
            cron = this.embeddedValueResolver.resolveStringValue(cron);
            zone = this.embeddedValueResolver.resolveStringValue(zone);
        }
        if (StringUtils.hasLength(cron)) {
            Assert.isTrue(initialDelay == -1, "'initialDelay' not supported for cron triggers");
            processedSchedule = true;
            TimeZone timeZone;
            if (StringUtils.hasText(zone)) {
                timeZone = StringUtils.parseTimeZoneString(zone);
            }
            else {
                timeZone = TimeZone.getDefault();
            }
            tasks.add(this.registrar.scheduleCronTask(new CronTask(runnable, new CronTrigger(cron, timeZone))));
        }
    }

    // 初始延迟<0的都设置为0
    if (initialDelay < 0) {
        initialDelay = 0;
    }

    // 解析fixed delay，生成FixedDelayTask任务
    long fixedDelay = scheduled.fixedDelay();
    if (fixedDelay >= 0) {
        Assert.isTrue(!processedSchedule, errorMessage);
        processedSchedule = true;
        tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
    }
    String fixedDelayString = scheduled.fixedDelayString();
    if (StringUtils.hasText(fixedDelayString)) {
        if (this.embeddedValueResolver != null) {
            fixedDelayString = this.embeddedValueResolver.resolveStringValue(fixedDelayString);
        }
        if (StringUtils.hasLength(fixedDelayString)) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            try {
                fixedDelay = parseDelayAsLong(fixedDelayString);
            }
            catch (RuntimeException ex) {
                throw new IllegalArgumentException(
                        "Invalid fixedDelayString value \"" + fixedDelayString + "\" - cannot parse into long");
            }
            tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
        }
    }

    // 解析fixed rate,生成FixedRateTask任务
    long fixedRate = scheduled.fixedRate();
    if (fixedRate >= 0) {
        Assert.isTrue(!processedSchedule, errorMessage);
        processedSchedule = true;
        tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
    }
    String fixedRateString = scheduled.fixedRateString();
    if (StringUtils.hasText(fixedRateString)) {
        if (this.embeddedValueResolver != null) {
            fixedRateString = this.embeddedValueResolver.resolveStringValue(fixedRateString);
        }
        if (StringUtils.hasLength(fixedRateString)) {
            Assert.isTrue(!processedSchedule, errorMessage);
            processedSchedule = true;
            try {
                fixedRate = parseDelayAsLong(fixedRateString);
            }
            catch (RuntimeException ex) {
                throw new IllegalArgumentException(
                        "Invalid fixedRateString value \"" + fixedRateString + "\" - cannot parse into long");
            }
            tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
        }
    }

    // Check whether we had any attribute set
    Assert.isTrue(processedSchedule, errorMessage);

    // 注册调度任务，存入以bean为key的map中。
    synchronized (this.scheduledTasks) {
        Set<ScheduledTask> registeredTasks = this.scheduledTasks.get(bean);
        if (registeredTasks == null) {
            registeredTasks = new LinkedHashSet<>(4);
            this.scheduledTasks.put(bean, registeredTasks);
        }
        registeredTasks.addAll(tasks);
    }
}
}
```
- 上面的方法的逻辑是解析schedule注解的参数，生成相应的调度任务，存入以Bean为key的map中。
- 下面看下这些任务是如何存入线程池中的，可以看到任务都被注册到ScheduledTaskRegistrar类中，该类中有TaskScheduler，执行任务的逻辑在scheduleTasks方法里。

```java
protected void scheduleTasks() {
    if (this.taskScheduler == null) {
        this.localExecutor = Executors.newSingleThreadScheduledExecutor();
        this.taskScheduler = new ConcurrentTaskScheduler(this.localExecutor);
    }
    if (this.triggerTasks != null) {
        for (TriggerTask task : this.triggerTasks) {
            // 调度triggerTask
            addScheduledTask(scheduleTriggerTask(task));
        }
    }
    if (this.cronTasks != null) {
        for (CronTask task : this.cronTasks) {
            // 调度CronTask
            addScheduledTask(scheduleCronTask(task));
        }
    }
    if (this.fixedRateTasks != null) {
        for (IntervalTask task : this.fixedRateTasks) {
            // 调度FixedRateTask
            addScheduledTask(scheduleFixedRateTask(task));
        }
    }
    if (this.fixedDelayTasks != null) {
        for (IntervalTask task : this.fixedDelayTasks) {
            // 调度FixedDelayTask
            addScheduledTask(scheduleFixedDelayTask(task));
        }
    }
}
```
- 此方法是在afterPropertiesSet方法中调用的，在bean初始化后。
- 使用TaskScheduler调度各种调度任务，并将返回结果future封装在ScheduleTask中，可以看到最底层的线程池其实就是ScheduledThreadPoolExecutor。见另一篇{% post_link ScheduledThreadPoolExecutor周期调度%}
