---
title: Springcloud环境下spring context的启动情况
date: 2020-01-09 19:29:09
categories: 学习笔记
tags:
    - java
    - spring
---

## 前言
本来想在Spring容器中用ApplicationListener处理一下ContextRefreshedEvent事件，原先以为直接写一个ApplicationListener的即可，没想到执行的时候发现这个listener执行了两次事件更新，印象中一个容器只会发送一次类似的映像事件，怎么会执行三次的？
接下来调试一下源码看看。

<!-- more -->

## 代码分析
自己写了一个如下的listener, 监听ContextRefreshedEvent
```java
public class DemoListener implements ApplicationListener<ContextRefreshedEvent>{

    private ApplicationContext context;

    public static final AtomicInteger NUM = new AtomicInteger(0);

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        log.info("listener has execute {} times!!!!", NUM.incrementAndGet());
    }
}
```

### 第一次触发时机
堆栈图如下：
![事件第一次触发](/assets/blogImg/EventFirstTrigger.jpg)
具体分析一下代码执行路径：
1. 自己启动的容器（也就是我们SpringApplication.run出来的那个容器) 发送事件ApplicationEnvironmentPreparedEvent
2. 事件被唯一的SpringApplicationRunListener监听到，也就是EventPublishingRunListener，然后这个listener委托一个SimpleApplicationEventMulticaster把这个事件广播出去，其实就是一个监听器模式，让所有注册了的listener响应这个事件
3. 这里可以看到BootstrapApplicationListener响应了这个事件，执行了bootstrapServiceContext新开了一个id为bootstrap的上下文
4. 新启动的bootstrap application context开始run
5. bootstrap application context 触发ContextRefreshedEvent事件，被自定义的DemoListener监听到，然后执行对应代码

上面分析可以看出，其实整个流程是 applicaion.run() -> environmentPrepared -> BootstrapApplicationListener -> 
bootstrapServiceContext -> bootstrap application.run() -> contextRefresh() -> Demolistener
意味着其实我们是启动了另外一个bootstrap context，然后这个context本身refresh的时候发送事件被Demolistener监听到了。
之后我们具体看一下BootstrapApplicationListener.bootstrapServiceContext()代码：
```java
builder.sources(BootstrapImportSelectorConfiguration.class);
final ConfigurableApplicationContext context = builder.run();
// gh-214 using spring.application.name=bootstrap to set the context id via
// `ContextIdApplicationContextInitializer` prevents apps from getting the actual
// spring.application.name
// during the bootstrap phase.
context.setId("bootstrap");
// Make the bootstrap context a parent of the app context
// 这里加了一个Initializer给当前启动的application, 目的就是为了让他的context的parent变成我们这次新建的bootstrap context
addAncestorInitializer(application, context);
// It only has properties in it now that we don't want in the parent so remove
// it (and it will be added back later)
bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
return context;
```
这里可以看出，这个context只会扫描*BootstrapImportSelectorConfiguration*这个class，看这个*ImportSelector*的执行情况，也就知道他只会扫描几个BootstrapConfiguration类以及配置项*spring.cloud.bootstrap.sources*配置的类，也就是说当前真正项目需要实例化的所有bean还是在我们启动的那个应用的上下文内实例化。
因此，这次触发的Demolistener其实是bootstrap context里面实例化的listener。然后我们全局搜索一下*org.springframework.cloud.bootstrap.BootstrapApplicationListener*，发现其实这个类被声明在*spring-cloud-context*的*spring.factories*文件里面（实际上因为引入spring-cloud-starter带来的引入）。那么以为着，如果我们使用springboot的时候引入spring-cloud-starter之后，会相对应的启动一个bootstrap context，用来作为当前启动的容器的父容器。

### 第二次触发时机
那么第二次的堆栈就显而易见了
![事件第二次触发](/assets/blogImg/EventSecondTrigger.jpg)
就是当前我们启动的application context 所触发的事件，其实也就是我们需求上最想要的那一次事件

### 第三次触发时机
从前面了解到，是多启动了一个容器，而引发了事件多执行一次，但是分属于不同的listener,一个是*bootstrap context*里面的listener执行的，一个是*application context*执行的，那第三次的触发是为什么呢？难道又启动了一个容器？我们可以看下第三次的执行堆栈
![事件第三次触发](/assets/blogImg/EventThirdTrigger.jpg)
可以看到其实是两个不同的context触发了两次publishEvent，代码如下：
```java
// Multicast right now if possible - or lazily once the multicaster is initialized
if (this.earlyApplicationEvents != null) {
    this.earlyApplicationEvents.add(applicationEvent);
}
else {
    // 调用multicaster发送容器事件
    getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
}

// Publish event via parent context as well...
// 如果父容器存在，则叫父容器触发一次对应的容器事件
if (this.parent != null) {
    if (this.parent instanceof AbstractApplicationContext) {
        ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
    }
    else {
        this.parent.publishEvent(event);
    }
}
```
这里我们可以看出其实是我们的application context触发事件之后，先发送一遍给自己的listener，然后看下自己有没有父容器，有的话再发送一遍这个事件给父容器，当然咯，事件的source肯定是触发的容器咯，也就是application context。
同时，我们可以看到这里是通过context 调用multicaster 把事件发出去的，所以这里会有一个判断是否有父容器的逻辑。第一次触发的时候实际上是通过EventPublishingRunListener 调用multicaster把事件发出去的。所以说，如果是我们的listener监听的其他事件，不一定是说只执行三次，要看具体情况分析。例如ContextRefreshedEvent这类容器的事件，spring里面肯定都是通过context去发送，那么这种情况下肯定会有奇数次的（因为父容器会执行两遍）。

## 总结
因此，如果写一个只在application context 执行一次的listener就很重要，注意以下几点：
1. 只把listener注册在父容器或子容器
2. 判断event里面过来的context是不是我们想要的context。例如可以根据contextId == "bootstrap" 来判断是不是自己想要的容器，其实 context.getParent() == null 也可以，因为我们知道这个父容器是引入springcloud而带来的，但是如果我们没有引入，那么这个代码就不一定有用了，因为此时的application context也是没有parent的
3. 判断是不是已经执行过一遍listener里面的逻辑了。这个主要是防止通过context发送事件之后又传播到父容器再执行一遍（第三次触发）。
后续我还监听了ApplicationPreparedEvent，发现了执行了五次，里面是*org.springframework.cloud.context.restart.RestartListener* 在 ContextRefresh阶段发送了一次，应该是为了context 手工refresh的时候再通知一遍监听ApplicationPreparedEvent的listener来保证正确执行