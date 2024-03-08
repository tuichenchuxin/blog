---
title: "java.lang.OutOfMemoryError: unable to create native thread"
date: 2024-03-07T18:20:11+08:00
draft: false
---

# 背景
用户反馈系统运行一段时间后就出现了如下错误。

```
java.lang.RuntimeException: java.lang.OutOfMemoryError: unable to create native thread: possibly out of memory or process/resource limits reached
```
用户执行了几次查询后，就发现线程疯狂飙升，而且jvm 内存释放后，线程却没释放，内存还是不够。

这说明，jvm 虽然内存不高，但是由于线程太多，java 线程和操作系统线程有对应关系，相当于操作系统线程占了pod 内的内存，造成内存不够。

开始我以为是监控不准，后来进入到 pod 后，发现 jvm 内存确实不高。那么就是线程泄漏。

这是用户的 stack 信息，发现 ss-0,ss-1,ss-2 线程个有 400 +

```
"ss-0" #721 daemon prio=5 os_prio=0 cpu=57.90ms elapsed=155563.07s tid=0x00007fc79a22dbd0 nid=0x316 waiting on condition  [0x00007fc5329c8000]
   java.lang.Thread.State: WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@17.0.2/Native Method)
	- parking to wait for  <0x000000040492a820> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(java.base@17.0.2/LockSupport.java:341)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionNode.block(java.base@17.0.2/AbstractQueuedSynchronizer.java:506)
	at java.util.concurrent.ForkJoinPool.unmanagedBlock(java.base@17.0.2/ForkJoinPool.java:3463)
	at java.util.concurrent.ForkJoinPool.managedBlock(java.base@17.0.2/ForkJoinPool.java:3434)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(java.base@17.0.2/AbstractQueuedSynchronizer.java:1623)
	at java.util.concurrent.LinkedBlockingQueue.take(java.base@17.0.2/LinkedBlockingQueue.java:435)
	at java.util.concurrent.ThreadPoolExecutor.getTask(java.base@17.0.2/ThreadPoolExecutor.java:1062)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@17.0.2/ThreadPoolExecutor.java:1122)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@17.0.2/ThreadPoolExecutor.java:635)
	at java.lang.Thread.run(java.base@17.0.2/Thread.java:833)

   Locked ownable synchronizers:
	- None

"ss-1" #722 daemon prio=5 os_prio=0 cpu=42.84ms elapsed=155563.07s tid=0x00007fc79a22ff30 nid=0x317 waiting on condition  [0x00007fc5328c7000]
   java.lang.Thread.State: WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@17.0.2/Native Method)
	- parking to wait for  <0x000000040492a820> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(java.base@17.0.2/LockSupport.java:341)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionNode.block(java.base@17.0.2/AbstractQueuedSynchronizer.java:506)
	at java.util.concurrent.ForkJoinPool.unmanagedBlock(java.base@17.0.2/ForkJoinPool.java:3463)
	at java.util.concurrent.ForkJoinPool.managedBlock(java.base@17.0.2/ForkJoinPool.java:3434)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(java.base@17.0.2/AbstractQueuedSynchronizer.java:1623)
	at java.util.concurrent.LinkedBlockingQueue.take(java.base@17.0.2/LinkedBlockingQueue.java:435)
	at java.util.concurrent.ThreadPoolExecutor.getTask(java.base@17.0.2/ThreadPoolExecutor.java:1062)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@17.0.2/ThreadPoolExecutor.java:1122)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@17.0.2/ThreadPoolExecutor.java:635)
	at java.lang.Thread.run(java.base@17.0.2/Thread.java:833)

   Locked ownable synchronizers:
	- None

"ss-2" #723 daemon prio=5 os_prio=0 cpu=47.49ms elapsed=155563.07s tid=0x00007fc79a230ea0 nid=0x318 waiting on condition  [0x00007fc5327c6000]
   java.lang.Thread.State: WAITING (parking)
	at jdk.internal.misc.Unsafe.park(java.base@17.0.2/Native Method)
	- parking to wait for  <0x000000040492a820> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(java.base@17.0.2/LockSupport.java:341)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionNode.block(java.base@17.0.2/AbstractQueuedSynchronizer.java:506)
	at java.util.concurrent.ForkJoinPool.unmanagedBlock(java.base@17.0.2/ForkJoinPool.java:3463)
	at java.util.concurrent.ForkJoinPool.managedBlock(java.base@17.0.2/ForkJoinPool.java:3434)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(java.base@17.0.2/AbstractQueuedSynchronizer.java:1623)
	at java.util.concurrent.LinkedBlockingQueue.take(java.base@17.0.2/LinkedBlockingQueue.java:435)
	at java.util.concurrent.ThreadPoolExecutor.getTask(java.base@17.0.2/ThreadPoolExecutor.java:1062)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@17.0.2/ThreadPoolExecutor.java:1122)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@17.0.2/ThreadPoolExecutor.java:635)
	at java.lang.Thread.run(java.base@17.0.2/Thread.java:833)

   Locked ownable synchronizers:
	- None

```

其实这里就可以推测出来是创建了 400 + 个线程池，但是没有回收。

所以经过一番排查后，终于找到了元凶，就是线程池创建后，没有关闭。。并且由于线程池是局部变量，所以被不停的调用，但一直不释放。

这里开始是想着会不会是程序创建了很多线程池后，没有办法创建回收的线程池的线程了。 结果后面惊喜发现有个局部变量的线程池，没回收。哈哈。