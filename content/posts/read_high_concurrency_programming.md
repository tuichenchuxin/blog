---
title: "读深入理解高并发编程后相关总结"
date: 2024-01-31T15:13:06+08:00
draft: false
---

# 线程池

## 线程池构造方法

Executors 类可以方便构造常用的线程池。

底层真正调用的就两个

ThreadPoolExecutor
```java
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
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

ForkJoinPool
```java
    public ForkJoinPool(int parallelism,
                        ForkJoinWorkerThreadFactory factory,
                        UncaughtExceptionHandler handler,
                        boolean asyncMode,
                        int corePoolSize,
                        int maximumPoolSize,
                        int minimumRunnable,
                        Predicate<? super ForkJoinPool> saturate,
                        long keepAliveTime,
                        TimeUnit unit) {
        checkPermission();
        int p = parallelism;
        if (p <= 0 || p > MAX_CAP || p > maximumPoolSize || keepAliveTime <= 0L)
            throw new IllegalArgumentException();
        if (factory == null || unit == null)
            throw new NullPointerException();
        this.factory = factory;
        this.ueh = handler;
        this.saturate = saturate;
        this.keepAlive = Math.max(unit.toMillis(keepAliveTime), TIMEOUT_SLOP);
        int size = 1 << (33 - Integer.numberOfLeadingZeros(p - 1));
        int corep = Math.min(Math.max(corePoolSize, p), MAX_CAP);
        int maxSpares = Math.min(maximumPoolSize, MAX_CAP) - p;
        int minAvail = Math.min(Math.max(minimumRunnable, 0), MAX_CAP);
        this.bounds = ((minAvail - p) & SMASK) | (maxSpares << SWIDTH);
        this.mode = p | (asyncMode ? FIFO : 0);
        this.ctl = ((((long)(-corep) << TC_SHIFT) & TC_MASK) |
                    (((long)(-p)     << RC_SHIFT) & RC_MASK));
        this.registrationLock = new ReentrantLock();
        this.queues = new WorkQueue[size];
        String pid = Integer.toString(getAndAddPoolIds(1) + 1);
        this.workerNamePrefix = "ForkJoinPool-" + pid + "-worker-";
    }
```

## 特殊的队列 SynchronousQuery 实现分析

关于 BlockingQueue<Runnable> workQueue

其中 fixedThreadPool 使用了 SynchronousQueue

该队列其实没有容量，有线程取，那么才能允许线程放入，相当于放入会阻塞，如果没有线程取的话。总之可以提高效率。内部使用了 TransferStack 和 transferQueue 两种。

```java
/**
 * @author 一灯架构
 * @apiNote SynchronousQueue示例
 **/
public class SynchronousQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        // 1. 创建SynchronousQueue队列
        SynchronousQueue<Integer> synchronousQueue = new SynchronousQueue<>();

        // 2. 启动一个线程，往队列中放1个元素
        new Thread(() -> {
            try {
                synchronousQueue.put(0);
                System.out.println(Thread.currentThread().getName() + " 入队列 0");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        // 3. 等待1000毫秒
        Thread.sleep(5000L);

        // 4. 启动一个线程，往队列中放1个元素
        new Thread(() -> {
            try {
                synchronousQueue.put(1);
                System.out.println(Thread.currentThread().getName() + " 入队列 1");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        // 5. 等待1000毫秒
        Thread.sleep(5000L);

        // 6. 再启动一个线程，从队列中取出1个元素
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + " 出队列 " + synchronousQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        // 7. 等待1000毫秒
        Thread.sleep(5000L);

        // 8. 再启动一个线程，从队列中取出1个元素
        new Thread(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + " 出队列 " + synchronousQueue.take());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }

}
```

内部队列实际的使用，实话比较难懂。大概就是线程挂起和唤醒，其它也没看出来。

```java
        /**
         * Puts or takes an item.
         */
        @SuppressWarnings("unchecked")
        E transfer(E e, boolean timed, long nanos) {
            /*
             * Basic algorithm is to loop trying one of three actions:
             *
             * 1. If apparently empty or already containing nodes of same
             *    mode, try to push node on stack and wait for a match,
             *    returning it, or null if cancelled.
             *
             * 2. If apparently containing node of complementary mode,
             *    try to push a fulfilling node on to stack, match
             *    with corresponding waiting node, pop both from
             *    stack, and return matched item. The matching or
             *    unlinking might not actually be necessary because of
             *    other threads performing action 3:
             *
             * 3. If top of stack already holds another fulfilling node,
             *    help it out by doing its match and/or pop
             *    operations, and then continue. The code for helping
             *    is essentially the same as for fulfilling, except
             *    that it doesn't return the item.
             */

            SNode s = null; // constructed/reused as needed
            int mode = (e == null) ? REQUEST : DATA;

            for (;;) {
                SNode h = head;
                if (h == null || h.mode == mode) {  // empty or same-mode
                    if (timed && nanos <= 0L) {     // can't wait
                        if (h != null && h.isCancelled())
                            casHead(h, h.next);     // pop cancelled node
                        else
                            return null;
                    } else if (casHead(h, s = snode(s, e, h, mode))) {
                        long deadline = timed ? System.nanoTime() + nanos : 0L;
                        Thread w = Thread.currentThread();
                        int stat = -1; // -1: may yield, +1: park, else 0
                        SNode m;                    // await fulfill or cancel
                        while ((m = s.match) == null) {
                            if ((timed &&
                                 (nanos = deadline - System.nanoTime()) <= 0) ||
                                w.isInterrupted()) {
                                if (s.tryCancel()) {
                                    clean(s);       // wait cancelled
                                    return null;
                                }
                            } else if ((m = s.match) != null) {
                                break;              // recheck
                            } else if (stat <= 0) {
                                if (stat < 0 && h == null && head == s) {
                                    stat = 0;       // yield once if was empty
                                    Thread.yield();
                                } else {
                                    stat = 1;
                                    s.waiter = w;   // enable signal
                                }
                            } else if (!timed) {
                                LockSupport.setCurrentBlocker(this);
                                try {
                                    ForkJoinPool.managedBlock(s);
                                } catch (InterruptedException cannotHappen) { }
                                LockSupport.setCurrentBlocker(null);
                            } else if (nanos > SPIN_FOR_TIMEOUT_THRESHOLD)
                                LockSupport.parkNanos(this, nanos);
                        }
                        if (stat == 1)
                            s.forgetWaiter();
                        Object result = (mode == REQUEST) ? m.item : s.item;
                        if (h != null && h.next == s)
                            casHead(h, s.next);     // help fulfiller
                        return (E) result;
                    }
                } else if (!isFulfilling(h.mode)) { // try to fulfill
                    if (h.isCancelled())            // already cancelled
                        casHead(h, h.next);         // pop and retry
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {
                        for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       // m is s's match
                            if (m == null) {        // all waiters are gone
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
                            SNode mn = m.next;
                            if (m.tryMatch(s)) {
                                casHead(s, mn);     // pop both s and m
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                            } else                  // lost match
                                s.casNext(m, mn);   // help unlink
                        }
                    }
                } else {                            // help a fulfiller
                    SNode m = h.next;               // m is h's match
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else {
                        SNode mn = m.next;
                        if (m.tryMatch(h))          // help match
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }
```

## 为什么将任务添加到队列后，能够持续读取任务并执行

```java
// ctl 中既存了worker 数量，又存了线程池状态
int c = ctl.get();
// 小于核心线程数
if (workerCountOf(c) < corePoolSize) {
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
// 添加到队列中
if (isRunning(c) && workQueue.offer(command)) {
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
// 尝试增加工作线程
else if (!addWorker(command, false))
    reject(command);
```

```java
    /**
     * Checks if a new worker can be added with respect to current
     * pool state and the given bound (either core or maximum). If so,
     * the worker count is adjusted accordingly, and, if possible, a
     * new worker is created and started, running firstTask as its
     * first task. This method returns false if the pool is stopped or
     * eligible to shut down. It also returns false if the thread
     * factory fails to create a thread when asked.  If the thread
     * creation fails, either due to the thread factory returning
     * null, or due to an exception (typically OutOfMemoryError in
     * Thread.start()), we roll back cleanly.
     *
     * @param firstTask the task the new thread should run first (or
     * null if none). Workers are created with an initial first task
     * (in method execute()) to bypass queuing when there are fewer
     * than corePoolSize threads (in which case we always start one),
     * or when the queue is full (in which case we must bypass queue).
     * Initially idle threads are usually created via
     * prestartCoreThread or to replace other dying workers.
     *
     * @param core if true use corePoolSize as bound, else
     * maximumPoolSize. (A boolean indicator is used here rather than a
     * value to ensure reads of fresh values after checking other pool
     * state).
     * @return true if successful
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (int c = ctl.get();;) {
            // Check if queue empty only if necessary.
            if (runStateAtLeast(c, SHUTDOWN)
                && (runStateAtLeast(c, STOP)
                    || firstTask != null
                    || workQueue.isEmpty()))
                return false;
            // CAS 来增加线程数
            for (;;) {
                if (workerCountOf(c)
                    >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateAtLeast(c, SHUTDOWN))
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
        // 添加线程
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int c = ctl.get();

                    if (isRunning(c) ||
                        (runStateLessThan(c, STOP) && firstTask == null)) {
                        if (t.getState() != Thread.State.NEW)
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        workerAdded = true;
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                    }
                } finally {
                    mainLock.unlock();
                }
                // 开启线程
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

这个方法解释了为什么只要添加到队列中，就会一直有线程处理任务，他是 loop 获取的，并且 die 之后会补充。
```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                    runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                try {
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

## 线程池是如何优雅退出的

提供了 shutdown 和 shutdownNow 两种方法

```java
public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }

```

```java
// 传入 false
 private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                // 获取锁的线程都调用 interrupt,如果 worker 正在运行，那么会占用该锁
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

```java
void interruptIfStarted() {
    Thread t;
    // worker 状态开始，未中断
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```

## ScheduledThreadPool 为什么能实现定时？
```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
            DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
            // 应该是它
            new DelayedWorkQueue());
}
```

内部使用了平衡二叉树，所以先了解下平衡二叉树
> 平衡二叉树是平衡因子绝对值 <=1 ，并且是排序树
> 关于平衡二叉树的调整，找新插入方向上，不平衡的树的根节点上相邻的三个点，重新排就行。

队列的任务则是ScheduledFutureTask，他们依据执行剩余延迟时间来构造二叉树。

```java
public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                        long delay,
                                        TimeUnit unit) {
    if (callable == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<V> t = decorateTask(callable,
        new ScheduledFutureTask<V>(callable,
                                    // 触发时间
                                    triggerTime(delay, unit),
                                    // 记录进入队列的顺序
                                    sequencer.getAndIncrement()));
    delayedExecute(t);
    return t;
}
```

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                    long initialDelay,
                                                    long delay,
                                                    TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0L)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,
                                        null,
                                        triggerTime(initialDelay, unit),
                                        // 增加了延迟，如果有 period 那么就是周期性任务
                                        -unit.toNanos(delay),
                                        sequencer.getAndIncrement());
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
```

```java
public void run() {
            if (!canRunInCurrentRunState(this))
                cancel(false);
            else if (!isPeriodic())
                super.run();
            else if (super.runAndReset()) {
                setNextRunTime();
                // 重复执行
                reExecutePeriodic(outerTask);
            }
        }
```

最终还是调用 addWorker 方法
```java
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    if (wc < corePoolSize)
        addWorker(null, true);
    else if (wc == 0)
        addWorker(null, false);
}
```

# AQS

AQS 是 AbstractQueuedSynchronizer，是个抽象类
ReentrantLock， Semaphore， SynchronousQueue 都是基于 AQS

可以从 ReentrantLock 出发，来分析下，加锁，释放是如何跟 AQS 来交互的。
AQS 内部是使用 CLH 双向队列。

ReetrantLock lock 后如果不能立刻 CAS 获取锁，那么会调用该方法，能增加到 CLH 队列中
代码比较复杂，看不明白。。
大概是在设置 CLH 队列的节点，wait 和唤醒之类的

```java
/**
     * Main acquire method, invoked by all exported acquire methods.
     *
     * @param node null unless a reacquiring Condition
     * @param arg the acquire argument
     * @param shared true if shared mode else exclusive
     * @param interruptible if abort and return negative on interrupt
     * @param timed if true use timed waits
     * @param time if timed, the System.nanoTime value to timeout
     * @return positive if acquired, 0 if timed out, negative if interrupted
     */
    final int acquire(Node node, int arg, boolean shared,
                      boolean interruptible, boolean timed, long time) {
        Thread current = Thread.currentThread();
        byte spins = 0, postSpins = 0;   // retries upon unpark of first thread
        boolean interrupted = false, first = false;
        Node pred = null;                // predecessor of node when enqueued

        /*
         * Repeatedly:
         *  Check if node now first
         *    if so, ensure head stable, else ensure valid predecessor
         *  if node is first or not yet enqueued, try acquiring
         *  else if node not yet created, create it
         *  else if not yet enqueued, try once to enqueue
         *  else if woken from park, retry (up to postSpins times)
         *  else if WAITING status not set, set and retry
         *  else park and clear WAITING status, and check cancellation
         */

        for (;;) {
            if (!first && (pred = (node == null) ? null : node.prev) != null &&
                !(first = (head == pred))) {
                if (pred.status < 0) {
                    cleanQueue();           // predecessor cancelled
                    continue;
                } else if (pred.prev == null) {
                    Thread.onSpinWait();    // ensure serialization
                    continue;
                }
            }
            if (first || pred == null) {
                boolean acquired;
                try {
                    if (shared)
                        acquired = (tryAcquireShared(arg) >= 0);
                    else
                        acquired = tryAcquire(arg);
                } catch (Throwable ex) {
                    cancelAcquire(node, interrupted, false);
                    throw ex;
                }
                if (acquired) {
                    if (first) {
                        node.prev = null;
                        head = node;
                        pred.next = null;
                        node.waiter = null;
                        if (shared)
                            signalNextIfShared(node);
                        if (interrupted)
                            current.interrupt();
                    }
                    return 1;
                }
            }
            if (node == null) {                 // allocate; retry before enqueue
                if (shared)
                    node = new SharedNode();
                else
                    node = new ExclusiveNode();
            } else if (pred == null) {          // try to enqueue
                node.waiter = current;
                Node t = tail;
                node.setPrevRelaxed(t);         // avoid unnecessary fence
                if (t == null)
                    tryInitializeHead();
                else if (!casTail(t, node))
                    node.setPrevRelaxed(null);  // back out
                else
                    t.next = node;
            } else if (first && spins != 0) {
                --spins;                        // reduce unfairness on rewaits
                Thread.onSpinWait();
            } else if (node.status == 0) {
                node.status = WAITING;          // enable signal and recheck
            } else {
                long nanos;
                spins = postSpins = (byte)((postSpins << 1) | 1);
                if (!timed)
                    LockSupport.park(this);
                else if ((nanos = time - System.nanoTime()) > 0L)
                    LockSupport.parkNanos(this, nanos);
                else
                    break;
                node.clearStatus();
                if ((interrupted |= Thread.interrupted()) && interruptible)
                    break;
            }
        }
        return cancelAcquire(node, interrupted, interruptible);
    }
```
