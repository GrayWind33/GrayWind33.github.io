---
layout:     post
title:      Java多线程——FutureTask源码解析
subtitle:   分析多线程返回结果类Callable基于FutureTask类执行的实现原理
date:       2018-10-13
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 多线程
---

一个很常见的多线程案例是，我们安排主线程作为分配任务和汇总的一方，然后将计算工作切分为多个子任务，安排多个线程去计算，最后所有的计算结果由主线程进行汇总。比如，归并排序，字符频率的统计等等。

我们知道Runnable是不返回计算结果的，如果想利用多线程的话，只能存储到一个实例的内部变量里面进行交互，但存在一个问题，如何判断是否已经计算完成了。用Thread.join是一个方案，但是我们只能依次等待一个线程结束后处理一个线程，如果线程1恰好特别慢，则后续已经完成的线程不能被及时处理。我们希望能够获知线程的执行状态，发现哪个线程处理完就先统计它的计算结果。可以考虑使用Callable和FutureTask来完成。

先说Callable它是一个功能接口，它只有一个方法V call()，计算一个结果，失败的话抛出一个异常。和Runnable不同的是，它不能直接交给Thread来执行，所以需要一个别的类来封装它与Runnable，这个类就是FutureTask。FutureTask是一个类，继承了RunnableFuture，而RunnableFuture是一个多继承接口，它继承了Runnable和 Future，所以FutureTask是可以作为实现了Runnable的实例交给Thread执行。

从内部变量来看，含有一个下层Callable实例，一个状态表示，一个返回结果，以及对运行线程的记录

```java
    /**
     * 任务的运行状态，最初是NEW。运行状态只在set, setException和cancel方法中过度到最终状态。
     * 在完成过程中，状态可能发生转移到COMPLETING(在设置结果时)或者INTERRUPTING(仅当中断运行来满足cancel(true)时)。
     * 从这些中间状态转移到最终状态使用成本更低有序/懒惰写入，因为值是唯一的且之后不能再修改。
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    /** 下层的callable，运行后为null */
    private Callable<V> callable;
    /** get()操作返回的结果或者抛出的异常*/
    private Object outcome; // 不是volatile，由reads/writes状态来保护
    /** 运行callable的线程，在run()通过CAS修改*/
    private volatile Thread runner;
    /** 等待线程的Treiber堆栈 */
    private volatile WaitNode waiters;
```

### 构造函数

构造函数总共有两种重载，第一种直接给出Callable实例

```java
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // 确保callable的可见性
    }
```

第二种，给出Runnable实例和期望的返回结果，如果Runnable实例运行成功则返回的是result

```java
    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);//创建一个callable
        this.state = NEW;       // 确保callable的可见性
    }
```

### run

run方法是Thread运行FutureTask内任务的接口。首先，根据最上方的进入条件可以看出，只有成功竞争到修改runnerOffset成功的线程才能执行后续方法，而搜索整个类文件，可以发现只有在run结束后才会重置为null，所以**同一时间只能有一个线程执行run方法成功**。然后要检查state和callable的状态，因为run会将它们修改。调用callable.call方法获取返回结果，成功的话设置结果，失败的话设置返回结果为异常。无论是否执行成功，runner会被重置为null。

```java
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;//状态必须是NEW且修改执行线程成功，否则直接返回，避免被多个线程同时执行
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();//调用callable.call方法获取返回结果
                    ran = true;//执行成功
                } catch (Throwable ex) {
                    result = null;
                    ran = false;//执行失败
                    setException(ex);//设置返回异常并唤醒等待线程解除阻塞
                }
                if (ran)
                    set(result);//设置结果
            }
        } finally {
        	//runner直到状态设置完成不能为null来避免并发调用run()
            runner = null;
            //在将runner设置为null后需要重新读取state避免漏掉中断
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```

set方法先修改state为COMPLETING，然后将outcome设置为刚才计算出来的结果，最后设置state为NORMAL，并调用finishCompletion。这个方法移除并通知所有等待的线程解除阻塞，调用done()，并将callable设为null。done方法默认是什么也不做。

```java
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // 最终状态
            finishCompletion();//唤醒等待线程，将callable设为null
        }
    }

    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);//如果线程被park阻塞，解除阻塞
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // 取消连接帮助gc
                    q = next;
                }
                break;
            }
        }

        done();//未重写时什么也不做

        callable = null;        // to reduce footprint减少覆盖区
    }
```

setException跟set逻辑上基本一样，除了设置返回结果是Throwable对象

```java
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {//修改状态
            outcome = t;//结果为Throwable
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // 最终状态
            finishCompletion();
        }
    }
```

### get

get方法如果FutureTask已经执行完成则返回结果，否则会等待并阻止线程调度。等待时长可以输入，单位为纳秒，不输入为不限时等待，限时等待超时仍然没有完成会抛出异常。

```java
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);//等待完成
        return report(s);//检查时返回结果还是抛出异常
    }

    public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (unit == null)
            throw new NullPointerException();
        int s = state;
        if (s <= COMPLETING &&
            (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)//限时等待完成
            throw new TimeoutException();
        return report(s);
    }
```

awaitDone这个方法会阻塞当前线程(get方法的调用线程)的调度并增加等待结点，阻塞时长根据输入的时间长度决定。如果执行Callable任务的线程完成了运行或者被中断，则会解除栈中等待结点对应线程的阻塞。然后会根据执行结果决定是否要抛出异常还是返回执行完成的结果。

```java
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;//是否完成入栈
        for (;;) {
            if (Thread.interrupted()) {//检查线程是否已经被中断
                removeWaiter(q);//移除被中断的等待结点
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {//已经完成
                if (q != null)
                    q.thread = null;//移除等待
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet还没有超时
                Thread.yield();//已经在赋值，所以只需让出时间片等待赋值完成
            //下方都是还在没有完成call方法的情况
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);//q加入到栈的最前方
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {//超时了
                    removeWaiter(q);//移除超时的等待结点
                    return state;
                }
                LockSupport.parkNanos(this, nanos);//阻塞当前线程nanos纳秒
            }
            else
                LockSupport.park(this);//阻塞当前线程
        }
    }
```

### cancel

cancel输入的参数表示如果当前还在运行中是否要中断执行线程，如果输入参数是false则只有线程已经执行完成或者抛出异常或者已经被中断时可以把状态修改为CANCELLED，如果是true则会中断线程并将状态改为INTERRUPTED。所以，cancel在该任务已经结束或者已被取消，或者竞争修改状态失败时都会失败。如果中断成功，会释放所有被阻塞的等待线程。

```java
    public boolean cancel(boolean mayInterruptIfRunning) {
        if (!(state == NEW &&
              UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
                  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
            return false;//已经完成或者被取消或者竞争取消失败返回false
        try {    // in case call to interrupt throws exception
            if (mayInterruptIfRunning) {
                try {
                    Thread t = runner;
                    if (t != null)
                        t.interrupt();
                } finally { // final state
                    UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
                }
            }
        } finally {
            finishCompletion();
        }
        return true;
    }
```

### 状态检查

非常简单的两个方法。因为CANCELLED是state中最大的，所以只有cancel方法成功才会是这种状态。而isDone只要不是还在运行或者还没有被执行就是返回true。

```java
    public boolean isCancelled() {
        return state >= CANCELLED;
    }

    public boolean isDone() {
        return state != NEW;
    }
```

### 简单的使用示例

```java
public class CallableTest implements Callable<Integer>{
	private int start;
	
	public CallableTest(int start) {
		this.start = start;
	}
	
	@Override
	public Integer call() throws Exception {
		Thread.sleep(500);
		return start + 1;
	}
	
	public static void main(String args[]) throws InterruptedException, ExecutionException{
		long start = System.currentTimeMillis();
		FutureTask<Integer> task1 = new FutureTask<>(new CallableTest(2));
		new Thread(task1).start();
		FutureTask<Integer> task2 = new FutureTask<>(new CallableTest(4));
		new Thread(task2).start();
		System.out.println(task1.get() + task2.get());//8
		long end = System.currentTimeMillis();
		System.out.println(end - start);//506
	}
}
```

