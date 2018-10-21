---
layout:     post
title:      Java多线程——线程池ThreadPoolExecutor源码解析
subtitle:   分析线程池ThreadPoolExecutor类的实现原理
date:       2018-10-21
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 多线程
---

对于线程的使用其实是需要严格控制的，CPU的一个核同一时间内只能有一个线程获得真正的运行时间，线程的创建和启动本身就存在不小的开销，并且它们会保存在内存中消耗空间，同时，过多的等待线程之间会存在竞争又消耗资源，并且线程间的上下文切换也有额外的消耗。因此，多线程原本是为了在CPU密集度低的任务存在时更好的压榨CPU资源，如果出现过多的等待线程反而降低性能，得不偿失。

为了解决这种每有一个新的任务就new Thread的做法，我们需要使用的是线程池ThreadPoolExecutor。对于这类本身资源有限的线程、连接等等东西都应该提高它们的复用率，一个线程执行完一个任务后应该接收执行下一个任务而不是直接销毁。线程池本身管理着一定数量的线程，现在向它提交一个任务，线程池检查有无空余线程，有则直接由空余线程执行任务。如果没有需要检查池的最大容量还能否创建新的线程，如果能则创建新的线程来执行这个任务；如果已经满了，则阻塞这个任务的提交线程加入等待队列，等待其他线程执行结束，线程归还给线程池再安排这个执行结束的线程去执行新的任务。根据参数设定，线程池可能还会管理活跃线程的数量，如果有超出这个数量的等待线程，则会销毁这些多余的等待线程。

接下来主要分析几个关键点：

- 线程池如何创建
- 如何将任务提交给线程池
- 线程池如何创建线程和运行任务
- 线程池如何关闭终止
- 线程池的同步问题

### 线程池如何创建

首先可以通过ThreadPoolExecutor的公共构造函数，关键的参数有

- corePoolSize：池内保持的空闲线程，除非设置了allowCoreThreadTimeOut
- maximumPoolSize：池内允许的最大线程数
- keepAliveTime：线程数量超过核心数时，在被终止前超过的空闲的线程将会等待新任何的最大时间
- unit：keepAliveTime参数的时间单位
- workQueue：在任务被执行前持有它们的队列，这个队列只持有由execute方法提交的Runnable任务，这个队列有多种选择，常用的有ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue，不同的阻塞队列选择会影响线程池的运行。
- threadFactory：执行创建一个新的线程时使用的工厂
- handler：执行因为线程范围和队列容量到达被阻塞时使用的处理器

threadFactory默认值是Executors.defaultThreadFactory()，handler默认值是defaultHandler

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             threadFactory, defaultHandler);
    }

    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), handler);
    }

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

第二类是借助Executors的静态方法来创建线程池，主要有四个方法，本质上还是基于ThreadPoolExecutor的构造函数，但是预置好了几种参数组合。它们都有一个可选的参数ThreadFactory，也可以不输入使用Executors.DefaultThreadFactory。

newFixedThreadPool创建一个定长线程池使用固定数量的线程来操作一个共享的无界队列。在任何时间，最多nThreads个线程是在活动处理任务。如果有更多的任务在所有线程都活动时被提交，它们会在队列中等到一个线程可用。如果任何线程在执行期间因为失败而在关闭前终止，如果需要的话一个新的线程会替代它执行任务。池内的线程会一直存在到它被明确关闭。使用的阻塞队列是LinkedBlockingQueue。**newFixedThreadPool的好处是简单且减少了创建和销毁线程的开销，缺点是空闲线程不会被释放导致消耗资源**。

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

newSingleThreadExecutor创建一个单线程线程池，如果这个线程因为错误而提前终止，如果还有后续任务会创建一个新的线程代替它。任务保证顺序执行，任何时间不会有一个以上的任务被执行。跟newFixedThreadPool(1)返回的线程池不同，这个方法返回的线程池保证不会被重构为使用额外线程的方法，因为外面套了一层FinalizableDelegatedExecutorService封装。**适合用一个单独线程去执行一类单一任务的情形**，比如单独线程负责数据落盘。

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

newCachedThreadPool创建一个没有常驻的空闲线程的不定长线程池，所有线程超过60秒空闲就会被终止，因此，一个线程池长期处于空闲状态不会消耗资源。并且它的最大线程上限是整数的最大值，但一般真创建那么多早内存不足了，所以可以认为没有上限。使用的阻塞队列是SynchronousQueue。**适合任务频率比较平缓细水长流或者任务非常少的场景，不适合同一时间爆发性并发的场景，因为会产生大量一次性线程**。

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

newScheduledThreadPool 创建一个不定长线程池，支持定时延迟开始执行任务及周期性任务执行。可以指定最大空闲线程数，默认是1。区别在于使用的阻塞队列是DelayedWorkQueue，具有延迟开始的功能。

```java
    public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory) {
        return new DelegatedScheduledExecutorService
            (new ScheduledThreadPoolExecutor(1, threadFactory));
    }

    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }

    public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
              new DelayedWorkQueue());
    }
```

### 向线程池提交任务

使用ThreadPoolExecutor.execute提交一个Runnable实例command，处理的步骤是：

1. 如果少于corePoolSize的线程正在运行，尝试创建一个新的线程以command作为它的第一个任务。调用addWorker原子性地检查runState和workerCount，避免虚假警告增加线程。
2. 如果一个任务能够成功入队，我们仍然需要双重检查是否需要增加线程，因为存在上一次检查之后有线程死了，否则条目进入后线程池关闭了。因此我们再次检查状态，并且如果有必要的话回滚停止的入队，如果没有线程了的话启动一个新的线程
3. 如果我们不能入队任务，那么尝试增加新的线程。如果失败，我们可以知道正在做关闭或者饱和了，所以拒绝任务。

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {//当前的worker数量是否小于corePoolSize
            if (addWorker(command, true))//尝试增加worker，command是线程的第一个任务
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {//线程池运行状态下尝试入队
            int recheck = ctl.get();//再次检查线程池状态
            if (! isRunning(recheck) && remove(command))
                reject(command);//如果线程池关闭从队列移除任务，拒绝任务
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);//当前已经没有工作线程，新增一个
        }
        else if (!addWorker(command, false))//不能入队尝试新建一个线程
            reject(command);//创建线程失败说明线程池关闭或者饱和，拒绝任务
    }
```

先来看一下ctl，主要线程控制状态ctl是一个原子整数包装了两个概念性字段：workerCount有效线程数，runState是运行还是关闭状态。为了将它们包装到一个int中，我们限制workerCount最大到(2^29)-1大约500万而不是(2^31)-1大约2亿。workerCount是工作者的数量，允许开始不允许停止。它的值可能有片刻和存活线程数不同，比如当一个ThreadFactory创建线程失败时，当存在线程终止前还在进行记账时。用户可见的池大小是工作者集的当前大小。runState提供了主要生命周期控制，包括以下状态：

- RUNNING:  接收新的任务，处理队列中的任务
- SHUTDOWN: 不接受新的任务，但还处理队列中的任务
- STOP: 不接受新的任务，也不处理队列中的任务，中断处理中的任务
- TIDYING:  所有任务都终止了，workerCount为0，线程变更到TIDYING状态会运行terminated()方法
- TERMINATED: terminated()完成了

可以看到ctl最高的3位是有符号数runState，初始值是-1。其余位置都是workerCount，初始值是0。通过与CAPACITY之间的位操作可以进行包装与拆装ctl。

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));//0xE0000000
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;//0x1FFFFFFF

    // runState存储在高位
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 包装和拆装ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

然后是addWorker，这个方法是关键。首先检查是否有一个新的worker可以在遵守当前线程池状态和给出的边界(core或maximum)被添加。如果可以，调整worker计数器并且创建启动一个新的worker，运行firstTask作为它的第一个任务。如果池被停止或者符合关闭条件，这个方法返回false。如果线程工厂创建线程失败，也返回false。如果线程因为线程工厂返回null或者抛出异常(特别是OutOfMemoryError)，需要回滚。从代码上看，线程创建是通过new Worker()完成的，增加worker失败时通过addWorkerFailed来回滚，所以主要需要分析这两个部分。

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;//STOP TIDYING TERMINATED状态或者firstTask不为null或者队列为空

            for (;;) {
                int wc = workerCountOf(c);//worker数量
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;//worker数量超过上限
                if (compareAndIncrementWorkerCount(c))//尝试将ctl加1
                    break retry;
                c = ctl.get();  // 重新读取ctl
                if (runStateOf(c) != rs)
                    continue retry;//如果runState变化需要跳出到外面的循环
                //否则CAS因为workerCount改变而失败，在循环内重试
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);//新建一个worker，firstTake为它的第一个任务
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                	//在持有锁状态下再次检查，如果ThreadFactory失败或者获得锁之前关闭则退出
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable预检查t是否可以启动
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;//到达最大过的线程数量
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();//增加worker成功时启动线程
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);//worker添加失败
        }
        return workerStarted;
    }
```

### 线程如何创建和运行任务

从上面可以看到excute要增加线程必须通过addWorker，而后者是通过new Worker新建线程，Worker本身实现了Runnable接口，而默认的线程构造工厂是Excutors.DefaultFactory。DefaultFactory新建线程还是通过new Thread，将Worker实例作为Runnable对象。

```java
        Worker(Runnable firstTask) {
            setState(-1); // 直到runWorker前禁止打断
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())//是否建立了守护线程
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
```

run方法调用了runWorker方法，后者负责任务的获取和执行。它的核心是一个循环，当还能够从队列中获得任务的时候，就会不断调用Runnable实例的run方法，如果队列为空或者超过了等待时间且线程池达到了最大空闲线程数，这个worker就会被判定为要死亡，结束这个循环，调用processWorkerExit处理worker死亡后要处理的信息。特别注意到，线程刚创建时参数给入的第一个任务不需要进入队列，直接作为第一个任务由新的线程来执行。线程池中的线程通过不断调用Runnable.run方法来执行任务。

```java
        public void run() {
            runWorker(this);
        }

    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts允许打断
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {//当前任务为空则尝试从队列获取任务，直到队列为空或者超时时退出循环
                w.lock();
                //如果线程池停止，确保线程被中断。如果没有，确保线程没有被中断。这要求一个重新检查第二种情况来解决清除中断时的shutdownNow竞争
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);//预留扩展接口
                    Throwable thrown = null;
                    try {
                        task.run();//Runnable.run
                    } catch (RuntimeException x) {//分别处理异常与错误
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);//预留扩展接口
                    }
                } finally {
                    task = null;//Runnable设为null
                    w.completedTasks++;//增加计数
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);//worker死亡时清除一些内容
        }
    }
```

getTask从队列中获取Runnable对象实例，因为阻塞队列提供了限时等待和不限时等待，所以worker线程在队列中暂时没有任务时会根据是否有允许存在的空闲线程选择是限时等待获取任务还是不限时等待获取任务。在等待期间，线程会阻塞，依靠这个方式线程池中可以存在空闲线程而不停止。

```java
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?从队列获取任务超时

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {//SHUTDOWN状态队列为空，或者STOP状态
                decrementWorkerCount();//将worker计数减到0
                return null;
            }

            int wc = workerCountOf(c);

            // workers是否需要裁剪
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;//线程池内空闲线程有没有等待时间上限

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();//从队列获取任务，take是无限时间的等待，poll有时间限制
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

processWorkerExit要从集合中移除已经死亡的worker，检查是否可以终止线程池，如果线程池依然要继续运行，则检查是否已经达到允许的空闲线程数量，如果没有达到，说明有线程因为抛出异常而提前死亡，需要新建一个线程来替换死亡的线程。

```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;//增加完成任务计数
            workers.remove(w);//从集合中移除worker
        } finally {
            mainLock.unlock();
        }

        tryTerminate();//尝试终止线程池

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);//如果因为抛出异常等原因导致线程死亡，但线程池还在运行且允许空闲线程存在，需要新建一个线程来替代死亡线程
        }
    }
```

### 线程池如何关闭终止

线程池的关闭方法有两种，第一个是调用shutdown，它的特点是之前提交的任务会继续执行完成，但线程池不会再接收新的任务。这个方法会先中断空闲的线程，然后调用tryTerminate来终止线程池。

```java
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();//检查对worker的操作权限
            advanceRunState(SHUTDOWN);//CAS自旋修改状态为SHUTDOWN
            interruptIdleWorkers();//中断空闲的线程
            onShutdown(); // 用于ScheduledThreadPoolExecutor的钩子
        } finally {
            mainLock.unlock();
        }
        tryTerminate();//终止线程池
    }
```

从tryTerminate中可以看到，因为状态已经被改为SHUTDOWN，所以如果队列为空则直接检查是否已经中断所有线程，然后修改线程池状态即可。如果还有任务未执行，则该方法退出，改由运行任务的线程发现已经无法从队列中获取任务时触发的tryTerminate来完成。

```java
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;//如果还在RUNNING状态，或者已经终止，或者在SHUTDOWN状态但队列中还有任务直接返回
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {//将状态修改为TIDYING
                    try {
                        terminated();//预留方法
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));//将状态修改为TERMINATED
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

shutdownNow会直接中断当前所有的工作线程，同时禁止提交新的任务，所以是立即将线程池转入终止状态，并且返回队列中还没有执行的任务。drainQueue这个方法很容易理解，就是把当前队列中的任务出队并加入到list中。

```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);//状态直接改为STOP
            interruptWorkers();//中断工作线程
            tasks = drainQueue();//队列中剩余未执行的任务
        } finally {
            mainLock.unlock();
        }
        tryTerminate();//终止线程池
        return tasks;
    }
```

### 线程池的同步问题

线程池主要存在多线程安全的几个情况有：任务提交，新的工作线程创建，线程池状态的修改，线程的中断。

任务提交，如果只是入队是通过阻塞队列封装好的同步入队方法来完成的。如果是新建worker的情况，需要通过ReentrantLock加锁来保证线程安全。ReentrantLock是显示加锁，同一个线程可以多次获取加锁，但多次加锁时需要由这个线程相同次数解锁才能释放。

线程池状态的修改是基于自旋CAS的策略来完成。

线程的中断只能通过shutdownNow来进行，而shutdownNow通过ReentrantLock加锁来保证同步。