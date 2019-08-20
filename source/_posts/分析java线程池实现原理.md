---
title: 分析java线程池实现原理
date: 2018-01-09 14:46:21
categories: 学习笔记
tags:
    - java
	- concurrent
---
## 背景
线程是稀缺资源，如果无限制创建会消耗大量系统资源，同时也不利于管理。线程池的出现就是为了解决这类问题。

<!-- more -->

## 代码解析

### 构造参数解析

通常我们使用Executors这个工厂类来快速初始化一个符合要求的线程池，其实本质都是通过不同的参数实例化ThreadPoolExecutor，下面我们通过jdk文档来了解各个参数的意思，源码如下：

```JAVA
 /**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

#### corePoolSize
核心线程数，如果线程池中创建了核心线程，那么他们会一直存活(即使空闲)。如果你将allowCoreThreadTimeOut设置为true的话，那么空闲的核心线程才会被终止。如果你执行了prestartAllCoreThreads()，会提前启动所有核心线程。

#### maximumPoolSize
线程池中能持有的最大线程数

#### keepAliveTime
超过核心线程数的那部分线程最大空闲时间

#### unit
keepAliveTime的时间单位

#### workQueue
保存用户被执行任务的队列，现有的几种阻塞队列如下：
1. ArrayBlockingQueue
2. LinkedBlockingQueue
3. SynchronousQuene
4. PriorityBlockingQueue

#### threadFactory
创建线程的工厂类

#### handler
当线程执行阻塞或者容量已经到达限度时，需要执行的策略处理方式。现有的处理方式如下：
1. AbortPolicy：直接抛出异常(默认策略)
2. CallerRunsPolicy：用调用者所在的线程来执行任务
3. DiscardOldestPolicy：去掉队列中最老的任务，再次调用ThreadPoolExecutor.execute执行该任务
4. DiscardPolicy：直接抛弃该任务，不做任何操作

### 执行任务
提交我们需要执行的任务通常使用ThreadPoolExecutor.execute(Runnable r)，源码如下
```JAVA
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```
上述ctl这个变量的作用在于获取当前线程池运行状态以及当前线程数(稍后详细说明)
通过上面可以将执行的流程分成三部分：
1. 如果当前工作线程少于corePoolSize，则添加新的核心线程来执行任务，addworker方法会自动检查线程池运行状态以及工作线程数目并且通过返回值来通知是否成功创建线程。
2. 线程数大于corePoolSize的时候，我们会优先将任务送入workQueue。同时我们仍然需要对线程池状态以及线程数进行二次检查，避免进入该方法时该线程池已经停止接受任务，这时需要将任务从队列中移除并且执行reject操作。如果此时线程池中无可用线程，会创建一个空的非核心线程。
3. 如果不能往队列中送入任务，那么会让线程池创建一个非核心的线程来执行任务。如果没有创建成功，则执行reject操作。

具体流程图如下：
![execute执行流程](/assets/blogImg/threadpoolexecutor.png)

后续更新...