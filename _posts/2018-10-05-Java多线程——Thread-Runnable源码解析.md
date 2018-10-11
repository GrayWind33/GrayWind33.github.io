---
layout:     post
title:      Java多线程——Thread Runnable源码解析
subtitle:   分析多线程启动方式Thread和Runnable的实现原理
date:       2018-10-05
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 多线程
---

Java多线程的两种实现方法大家都应该知道了：继承Thread的子类实例化和实现Runnable接口用这个接口实现类去创建Thread实例。

Java的线程在Linux平台上使用的是NPTL机制，JVM线程跟内核轻量线程（LWP）一一对应。KLT是内核线程，它提供轻量进程给程序使用，调度由操作系统内核完成，所以Java程序无法在多个线程就绪状态下预测哪个线程会获得CPU调度。

![101352_BgMZ_1859679](https://yqfile.alicdn.com/5a2a926f5322f9f71f938e5f4641b61c45d57350.png)

在JVM的内存分配中线程私有的部分是栈（包括Java栈和本地方法栈）和程序计数器，线程可以访问共有的Java堆和常量池、方法区（这个视虚拟机实现而定，JVM1.8里永久代被移除）。

Java线程的状态有以下这些：

```java
    public enum State {
        NEW,//声明一个线程还没有开始
        RUNNABLE,//声明一个可运行的线程。一个线程在可运行状态下在Java虚拟机中执行，但它可能在等待操作系统中的其他资源比如处理器
        BLOCKED,//声明一个线程阻塞等待一个监视器锁。它需要等待监控器锁来进入一个同步的块/方法或者在调用Object.wait后重新进入一个同步的块/方法
        WAITING,//声明一个等待线程。一个线程在等待状态因为调用了以下方法：Object.wait没有超时时间；Thread.join没有超时时间；LockSupport.park。一个线程在等待状态是在等待其他线程执行一个特定的行为。比如，一个线程调用Object.wait()，是在等待另一个线程对它调用Object.notify或者Object.notifyAll()。一个线程调用Thread.join()是在等待这个线程终止。
        TIMED_WAITING,//线程处于限时等待状态，因为它调用了以下方法带有正数等待时间：Thread.sleep；Object.wait带有超时时间；Thread.join带有超时时间；LockSupport.parkNanos；LockSupport.parkUntil
        TERMINATED;//线程执行完毕，进入结束状态
    }
```

跟系统的三状态模型相比：就绪、运行、阻塞，这是可以对应上的

![Java_](https://yqfile.alicdn.com/537e1b4f79e8533bb1b97dfccd2fd20c67a7f464.jpeg)

## Runnable

Runnable是一个函数式接口@FunctionalInterface，它的实例可以通过lamda表达式，方法引用，构造器引用来创建。所谓函数式接口，该注解只能标记在"有且仅有一个抽象方法"的接口上。抽象方法是指接口中声明的方法，不包括加static关键字的静态方法和加default关键字的默认方法，也不包括重写java.lang.Object中的方法。该注解可以不加，但是如果加了编译器会对类进行检查，如果不符合要求会编译失败。

因为Runnable是一个接口，所以可以利用一个类可以实现多个接口的特点来更加灵活的构造。

Runnable接口应该被所有计划实例通过线程执行的类实现。类必须定义一个没有参数的 run方法。

此接口旨在为那些希望在它们活动时执行代码的对象提供一个通用的协议。例如，Runnable被Thread类实现。活动的意思是说一个线程已启动并且尚未停止。

此外，Runnable给不是Thread子类的类提供一个活动的手段。一个类可以通过实现Runnable，将自己传递给实例一个Thread对象的实例化来运行。在大多数情况下，Runnable接口应该是用于你只打算重写run()方法并没有重写其他Thread方法。这是重要的因为除非程序员打算修改或增强类的基本行为，否则类不应该作为子类。

——以上是关于Runnable的Javadoc说明内容

总结一下：在不需要对Thread方法进行修改或扩展的情况下，使用Runnable接口较好，并且可以利用实现多接口的方法增大灵活性，实现类在实例化之后传给Thread构造函数然后运行。

## Thread

Thread是在程序中执行的一个线程。java虚拟机允许应用程序可以有多个同时执行的线程。 
每一个线程都有一个优先权。具有更高优先级的线程在优先于较低优先级的线程执行。每一个线程可能会或可能不会被标记为一个守护进程。在一些线程中运行的代码创建了一个新的Thread对象时，新线程的优先级被设置为等于创建线程的优先级，新线程是守护线程的当且仅当创建线程是一个守护进程。

当一个java虚拟机启动时，通常有一个单一的非守护线程（通常调用方法为某指定的类中命名为main的）。java虚拟机持续执行线程，直到发生以下情况：

- Runtime类的exit方法被调用并且安全管理器允许退出操作发生。 
- 非守护线程的所有线程都已经死了，要么从调用到run方法返回或抛出一个异常传播到run方法。 

有两种方法来创建一个新的执行线程。一是声明一个类是Thread的子类。这个子类应重写类Thread的run方法。子类的一个实例可以被分配和启动。例如，一个线程计算大于规定值的素数可以写成如下：

```java
 class PrimeThread extends Thread {
     long minPrime;
     PrimeThread(long minPrime) {
         this.minPrime = minPrime;
     }

     public void run() {
         // compute primes larger than minPrime
          . . .
     }
 }
```

下面的代码将创建一个线程并开始运行它：

```java
 PrimeThread p = new PrimeThread(143);
 p.start();
```

创建一个线程的另一个方法是声明一个类实现Runnable接口。该类实现run方法。这个类的一个实例可以分配，创建Thread时作为一个参数传递，并开始。同样的例子在这个方式下如下：

```java
 class PrimeRun implements Runnable {
     long minPrime;
     PrimeRun(long minPrime) {
         this.minPrime = minPrime;
     }

     public void run() {
         // compute primes larger than minPrime
          . . .
     }
 }
```

下面的代码将创建一个线程并开始运行：

```java
 PrimeRun p = new PrimeRun(143);
 new Thread(p).start();
```

每一个线程都有一个用于识别的名称。可能有一个以上的线程有相同的名称。如果创建一个线程时没有指定名称，则为它生成一个新名称。

除非另有说明，通过null作为参数给类的构造函数或方法会导致一个NullPointerException被抛出。

——以上是对Thread的Javadoc说明内容

------

Thread这个类比较难以分析的地方在于绝大部分方法实际上是直接与操作系统内核间的交互，因为需要直接操作轻量级的内核线程，所以绝大部分方法实现都是基于native方法的。以下分析主要是基于使用角度上的分析，native方法分析鉴于目前拙劣的系统内核知识和c++水平暂时无法开展。

### 构造函数

构造函数最多有4个参数：

- group，线程组，参数为null的话，如果有安全管理器，组由SecurityManager.getThreadGroup()来决定。如果没有安全管理器或者前面这个方法返回是null，组设为当前线程的组。
- target，线程启动时哪个对象的run方法会被调用。如果该对象为null，调用这个线程的run方法。
- name，新线程的名字
- stackSize，为新线程请求的栈大小，如果是0的话说明这个参数可以被忽略

```java
    public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) {
        init(group, target, name, stackSize);
    }
```

根据使用参数的不同存在不同的重载，但是stackSize只有在4个参数全在时才可以使用，group不能单独使用。线程名如果不给出则自动生成一个，其它的参数默认为null，stackSize默认为0

```java
    public Thread() {
        init(null, null, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(Runnable target) {
        init(null, target, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(ThreadGroup group, Runnable target) {
        init(group, target, "Thread-" + nextThreadNum(), 0);
    }

    public Thread(String name) {
        init(null, null, name, 0);
    }

    public Thread(ThreadGroup group, String name) {
        init(group, null, name, 0);
    }

    public Thread(Runnable target, String name) {
        init(null, target, name, 0);
    }

    public Thread(ThreadGroup group, Runnable target, String name) {
        init(group, target, name, 0);
    }
```

还有一种构造函数式继承给出的AccessControlContext，这不是一个public构造函数

```java
    Thread(Runnable target, AccessControlContext acc) {
        init(null, target, "Thread-" + nextThreadNum(), 0, acc, false);
    }
```

从构造函数中可以看到调用了init方法来初始化线程。其实主要做的就是存储各参数到本地变量里，比较麻烦一点的是获取线程组，如果没有给定线程组的话需要从安全管理器中获得，还有从父线程继承上下文类加载器、优先级、是否是守护线程等变量。另外，线程ID和线程名是两个不同的变量。

```java
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
        init(g, target, name, stackSize, null, true);
    }

    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        if (name == null) {//线程名不能为null
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;//设置线程名，这里可以看出线程名指定的话可能重复

        Thread parent = currentThread();//返回当前线程
        SecurityManager security = System.getSecurityManager();
        if (g == null) {//没有指定线程组
            /* Determine if it's an applet or not确定是否是小型应用程序 */

            /* If there is a security manager, ask the security manager
               what to do.如果有安全管理器，询问安全管理器做什么 */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security doesn't have a strong opinion of the matter
               use the parent thread group.如果安全管理器对于问题没有强硬的意见，使用父线程的线程组 */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in.不管线程组是否是明确被传递进来，都需要检查访问权限 */
        g.checkAccess();

        /*
         * Do we have the required permissions?是否有要求的权限
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();//增加未开始线程的数量

        this.group = g;
        this.daemon = parent.isDaemon();//如果父线程是守护线程，子线程也是，否则不是
        this.priority = parent.getPriority();//优先级为父线程的优先级
        if (security == null || isCCLOverridden(parent.getClass()))//没有安全管理器或者父类没有重写安全敏感的方法
            this.contextClassLoader = parent.getContextClassLoader();//返回的还是父类的上下文类加载器
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);//调用底层方法修改线程的优先级
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);//可以从父类继承线程本地变量
        /* Stash the specified stack size in case the VM cares存储给定的栈大小以免虚拟机需要 */
        this.stackSize = stackSize;

        /* Set thread ID设置线程ID */
        tid = nextThreadID();
    }
```

### start

start方法会令一个新的线程进入就绪状态，我们无法知道这个线程到底什么时候会被执行，由java虚拟机调用这个线程的 run方法。这个方法的结果是，两个线程同时运行：当前线程（从调用start方法返回）和其他的线程（执行run方法）。启动线程超过一次永远是不合法的。特别的，一个线程一旦已经完成执行，它不能再被重新启动。

```java
    public synchronized void start() {
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* 通知线程组这个线程已经准备好被启动，这样它可以被添加到组的线程列表中并且组的未启动计数器会缩减 */
        group.add(this);

        boolean started = false;
        try {
            start0();//调用底层方法启动线程
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);//通知线程组尝试启动线程失败了
                }
            } catch (Throwable ignore) {
                /* 什么都不做，这样start0抛出的Throwable会被传递到调用栈 */
            }
        }
    }
```

### run

如果这个线程使用一个单独的Runnable对象来构造运行对象，那么这个Runnable的run方法会被调用，否则这个方法什么也不做世界返回。如果是Thread的子类需要重写这个方法。不同于start是启动一个线程来执行run方法，直接调用run是由当前线程来进行同步调用run，不会创建新的线程。

```java
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```

### yield

yield是另这个线程给调度器一个提示，当前线程愿意放弃当前对处理器的使用。调度器可以直接忽视这个提示。yield是一个启发式的尝试以改善线程之间的相对进展，否则将过度使用一个CPU。它的使用应结合详细的分析和确定基准，以确保它实际上有所需的效果。使用这种方法很少是恰当的。它可能是有调试或测试的目的，它可能有助于重现由于竞争环境的错误。在设计并发控制结构如在java.util.concurrent.locks包中的时候可能是有用的。**yield会使线程让出当前时间片进入就绪状态，但是由于还在RUNNABLE状态，我们不能预知这个线程会不会由于调度立刻再次开始运行**。

```java
    public static native void yield();
```

### sleep

引起当前正在执行的线程休眠（暂停执行）为指定的毫秒数，根据系统定时器精度和调度的准确性。线程不失去任何监视器的所有权。也就是当前线程会进入限时等待状态但所持有的锁对象不会被释放。

```java
    public static native void sleep(long millis) throws InterruptedException;

    public static void sleep(long millis, int nanos)
    throws InterruptedException {
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
            millis++;
        }

        sleep(millis);
    }
```

### wait和notify

尽管Thread并没有重写Object中的这个方法，但依然是线程的常用方法，它的作用是让线程进入等待状态，是否是限时等待取决于时间参数，和sleep相比，wait会释放线程持有的监视器锁。

notify()可以让一个等待的线程结束等待状态转入Runnable状态，如果有多个线程的话由系统内核任意选一个线程唤醒。notifyAll()则唤醒所有的等待线程。这里的唤醒代码如果退出之前在一个同步块或执行一个同步方法，需要检查当前能否获得锁。

### interrupt

中断这个线程。如果当前线程中断本身，这总是允许的。否则要被中断线程的checkAccess方法被调用，这可能会导致SecurityException被抛出。如果线程在调用Object类的wait()，wait(long)，或wait(long, int)方法阻塞，或者调用这个类的join()，join(long)，join(long, int)，sleep(long)，或sleep(long, int)方法导致阻塞，那么它的中断状态将被清除，它会收到InterruptedException。如果该线程在I/O操作被阻塞是因为一个InterruptibleChannel，则通道将被关闭，该线程的中断状态将被设置，并且线程将获得ClosedByInterruptException。如果该线程是在一个Selector被阻塞，然后该线程的中断状态将被设置，它会立即从选择操作返回一个可能是非零的值，就像选择器的wakeup方法调用。如果前面的条件都没有成立，那么这个线程的中断状态将被设置。中断一个不再活动的线程没有任何效果。**阻塞状态的进程调用interrupt会抛出ClosedByInterruptException，然后线程终止**。

```java
    public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // 只是设置一个中断标记
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
```

下面两个方法检查当前线程是否被中断，区别在于是否会把这个结果重置为false。一个线程中断因为当时线程已经不再活动而被无视的情况会通过这个方法返回false来反应

```java
    public boolean isInterrupted() {
        return isInterrupted(false);
    }

    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
```

### join

假设a是一个线程，如果一个线程b调用a.join()，那么线程b会进入等待状态，直到监视到a线程终止了才会继续进行下去，不输入等待时间或者0就会在线程活动时一直等待下去。因为调用的是wait，所以会释放当前进程的监控器锁。这个方法经常用于一个线程统计多个线程计算结果这样子的情况，需要等待计算进程全部结束。

```java
    public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);//只要线程还活着就wait下去
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);//wait有限的时间
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

### getState

getState方法获取线程的状态，设计是用于系统状态监控而不是用于同步保证。上面那个情形，如果使用这个方法，可以达到循环检测哪个线程已经结束，就先统计它的结果这样的异步操作。但这样会存在一个问题是，检测的线程一直占着时间片，如果长时间没有计算完的线程，会浪费处理器时间。

```java
    public State getState() {
        // get current thread state
        return sun.misc.VM.toThreadState(threadStatus);
    }
```