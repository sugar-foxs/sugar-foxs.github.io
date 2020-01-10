---
layout:     post
title:      spring的@schedule注解调度任务原理
subtitle:   
date:       2020-01-06
author:     sugar-foxs
catalog: 	true
tags:
    - java
    - spring
---

本篇文章主要阅读源码看看@schdule注解是如何工作的。问题写在前面：
- schedule注解的任务是何时实例化成任务的
- 调度任务如何调度的
<!-- more -->

# schedule注解

- 首先看下@schedule注解：
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

- 通过查看这个注解里的方法的调用方，可以看到是在ScheduledAnnotationBeanPostProcessor里调用的。

# ScheduledAnnotationBeanPostProcessor

- ScheduledAnnotationBeanPostProcessor也被初始配置成了bean。

```java
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class SchedulingConfiguration {
	@Bean(name = TaskManagementConfigUtils.SCHEDULED_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
		return new ScheduledAnnotationBeanPostProcessor();
	}
}
```

## processScheduled
- 解析schedule注解里参数并生成调度任务的方法
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
        if (this.embeddedValueResolver != null) {
            initialDelayString = this.embeddedValueResolver.resolveStringValue(initialDelayString);
        }
        if (StringUtils.hasLength(initialDelayString)) {
            try {
                initialDelay = parseDelayAsLong(initialDelayString);
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
            // 任务注册到registrar中
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
        // 任务注册到registrar中
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
            // 任务注册到registrar中
            tasks.add(this.registrar.scheduleFixedDelayTask(new FixedDelayTask(runnable, fixedDelay, initialDelay)));
        }
    }

    // 解析fixed rate,生成FixedRateTask任务
    long fixedRate = scheduled.fixedRate();
    if (fixedRate >= 0) {
        Assert.isTrue(!processedSchedule, errorMessage);
        processedSchedule = true;
        // 任务注册到registrar中
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
            // 任务注册到registrar中
            tasks.add(this.registrar.scheduleFixedRateTask(new FixedRateTask(runnable, fixedRate, initialDelay)));
        }
    }
    synchronized (this.scheduledTasks) {
        Set<ScheduledTask> registeredTasks = this.scheduledTasks.get(bean);
        if (registeredTasks == null) {
            registeredTasks = new LinkedHashSet<>(4);
            this.scheduledTasks.put(bean, registeredTasks);
        }
        registeredTasks.addAll(tasks);
    }
}
```

- processScheduled方法负责解析schedule注解的参数，生成相应的调度任务，存入ScheduledTaskRegistrar实例中。
- ScheduledAnnotationBeanPostProcessor实例中包含了ScheduledTaskRegistrar实例，这个实例中有线程池TaskScheduler，后面会讲到线程池是如何设置的。
- 解析生成的任务都放入了线程池中执行，执行任务的逻辑在下面的scheduleTasks方法里。

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
- 使用TaskScheduler调度各种调度任务，并将返回结果future封装在ScheduleTask中，可以看到最底层的线程池其实就是ScheduledThreadPoolExecutor。见另一篇{% post_link ScheduledThreadPoolExecutor周期调度%}

- 此方法是在afterPropertiesSet方法中调用的，从下面的代码可以看到afterPropertiesSet是在ScheduledAnnotationBeanPostProcessor类的onApplicationEvent方法中调用的，而onApplicationEvent则是重写了ApplicationListener接口的方法。

```java
public void onApplicationEvent(ContextRefreshedEvent event) {
    if (event.getApplicationContext() == this.applicationContext) {
        finishRegistration();
    }
}
private void finishRegistration() {
    if (this.scheduler != null) {
        this.registrar.setScheduler(this.scheduler);
    }

    if (this.beanFactory instanceof ListableBeanFactory) {
        Map<String, SchedulingConfigurer> beans =
                ((ListableBeanFactory) this.beanFactory).getBeansOfType(SchedulingConfigurer.class);
        List<SchedulingConfigurer> configurers = new ArrayList<>(beans.values());
        AnnotationAwareOrderComparator.sort(configurers);
        for (SchedulingConfigurer configurer : configurers) {
            configurer.configureTasks(this.registrar);
        }
    }
    if (this.registrar.hasTasks() && this.registrar.getScheduler() == null) {
        try {
            this.registrar.setTaskScheduler(resolveSchedulerBean(beanFactory, TaskScheduler.class, false));
        }
        catch (NoUniqueBeanDefinitionException ex) {
            try {
                this.registrar.setTaskScheduler(resolveSchedulerBean(beanFactory, TaskScheduler.class, true));
            }
        }
        catch (NoSuchBeanDefinitionException ex) {
            try {
                this.registrar.setScheduler(resolveSchedulerBean(beanFactory, ScheduledExecutorService.class, false));
            }
            catch (NoUniqueBeanDefinitionException ex2) {
                try {
                    this.registrar.setScheduler(resolveSchedulerBean(beanFactory, ScheduledExecutorService.class, true));
                }
            }
        }
    }
    this.registrar.afterPropertiesSet();
}
```

- 上面的finishRegistration方法主要负责通过by-type/by-name方式找到可用的线程池bean,并设置给ScheduledTaskRegistrar实例，用于调度任务。

- 但是还有个疑问onApplicationEvent是什么时候调用的。这时候就需要研究下ApplicationListener这个类。看
onApplicationEvent方法的参数是ContextRefreshedEvent，可以知道监听的是ContextRefreshedEvent事件。只要知道事件在何时发布就可以知道任务在什么时候会被调度了。

```java
protected void finishRefresh() {
    。。。
    // 发布事件
    publishEvent(new ContextRefreshedEvent(this));
    。。。
}
```

- 我们可以看到事件是在finishRefresh方法中发布的，往上还可以追溯到SpringApplication的run方法，这里就不贴代码了，所以是在web启动时，refresh最后阶段，广播了ContextRefreshedEvent事件。web启动的过程以后再总结。

# 总结
- schedule注解何时实例化任务的：有schedule注解的方法在其所在的类实例化成bean的时候已经被解析成Scheduled，解析Scheduled里的参数生成调度任务是在ScheduledAnnotationBeanPostProcessor初始化之后调用的。
- 任务如何调度的：应用启动的最后阶段会发布ContextRefreshedEvent事件，作为监听了这个事件的bean(ScheduledAnnotationBeanPostProcessor)当接收到事件时，会获取可用线程池，设置在ScheduledTaskRegistrar实例中，ScheduledTaskRegistrar负责调度任务。


