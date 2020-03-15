---
layout:     post
title:      spring监听器（观察者模式的应用）
subtitle:   
date:       2020-03-15
author:     sugar-foxs
catalog: 	true
tags:
    - java
    - spring
---

本文介绍spring中观察者模式的具体实践：spring监听器。
<!-- more -->

# spring监听器
- 熟悉观察者模式的应该都知道实现观察者模式需要的条件：
    - 事件，在spring中通过继承ApplicationEvent来定义一个事件。
    - 观察者（事件监听器），在spring中需要实现ApplicationListener接口定义观察者。
    - 发布者（事件发布器），在spring中需要实现ApplicationEventPublisherAware或者ApplicationContext去发布事件。

## 示例
- 事件
```java
/**
 * @author guchunhui
 **/
public class LogEvent extends ApplicationEvent {
    private String msg;
    public LogEvent(Object source) {
        super(source);
    }

    public LogEvent(Object source, String msg) {
        super(source);
        this.msg = msg;
    }

    public void print() {
        System.out.println("监听到日志事件,msg:"+msg);
    }
}
```

- 监听者
```java
/**
 * @author guchunhui
 **/
@Component
public class LogListener implements ApplicationListener<LogEvent> {
    @Override
    public void onApplicationEvent(LogEvent logEvent) {
        logEvent.print();
    }
}
```

- 发布者
```java
/**
 * @author guchunhui
 **/
@Component
public class LogPublisher implements ApplicationContextAware {
    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    public void publish(LogEvent logEvent) {
        applicationContext.publishEvent(logEvent);
    }
}
```

- 测试：
```java
@SpringBootTest
class DemoApplicationTests {
    @Autowired
    private LogPublisher logPublisher;

    @Test
    public void publishLogEvent() {
        logPublisher.publish(new LogEvent(new Object(), "打印日志"));
    }
}
```
- 输出：监听到日志事件,msg:打印日志
