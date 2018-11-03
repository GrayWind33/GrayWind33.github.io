---
layout:     post
title:      Java多线程——LinkedBlockingQueue ArrayBlockingQueue源码解析
subtitle:   分析多线程有界阻塞队列LinkedBlockingQueue和ArrayBlockingQueue的实现原理
date:       2018-11-02
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 多线程
    - 集合
---

LinkedBlockingQueue和ArrayBlockingQueue都是有界阻塞队列，符合先进先出的原则。当达到队列上限时，入队根据方法会被阻塞或者直接失败。LinkedBlockingQueue底层是链表，一定是非公平锁，阻塞的线程是随机竞争。ArrayBlockingQueue底层是在构造时建立的固定数组，锁根据构造时的参数可以是公平锁也可以是非公平锁，默认是非公平的。

## LinkedBlockingQueue

链表结构的**单向有界**阻塞队列，从last处入队，从head处出队，head是头部空结点，而last是最后一个入队的结点。被阻塞的线程是非公平竞争的，也就是说并没有先来后到的顺序，完全随机分配

### 内部变量

从内部变量上即可看出LinkedBlockingQueue是有界队列，它的元素个数通过原子整型来存储，存储了头部和尾部的指针，并且有头部空指针。入队和出队有不同的锁，和它们各自的等待队列。

```java
    /**  容量上限，不设置的话是整数最大值*/
    private final int capacity;

    /** 当前元素个数*/
    private final AtomicInteger count = new AtomicInteger();

    /**
     * 链表头部。head.item == null不变
     */
    transient Node<E> head;

    /**
     * 链表尾部。last.next == null不变
     */
    private transient Node<E> last;

    /** 出队锁*/
    private final ReentrantLock takeLock = new ReentrantLock();

    /** 等待获取元素的队列*/
    private final Condition notEmpty = takeLock.newCondition();

    /** 入队锁*/
    private final ReentrantLock putLock = new ReentrantLock();

    /** 等待放入元素的队列*/
    private final Condition notFull = putLock.newCondition();
```

### 构造函数

构造函数中可以看出，默认的容量大小是整数的最大值，构造函数会默认增加头尾为同一个值为null的空结点

```java
    public LinkedBlockingQueue() {
        this(Integer.MAX_VALUE);
    }

    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
```

结点结构非常简单，包含一个值和一个指向下个结点的指针，所以这是一个单向链表

```java
    static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) { item = x; }
    }
```

复制构造函数有加锁，虽然构造函数没有线程间的竞争，但是这里还是加锁的目的是让实例能够进入堆内存，保证其可见性。

```java
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // 没有竞争，但是为了可见性要加锁
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

### 入队

put操作插入指定元素到队列尾部，如果需要的话等待有足够的空间，并且这里的等待是**不限时等待**，只要不抛出异常，一定会入队成功

```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * 尽管count没有被锁保护，但它却能用来作为等待的监视，因为count只在这里或者其他put操作中减少。并且我们会被通知它改变了。其他使用count作为等待监视也是同样的。
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();//增加计数
            if (c + 1 < capacity)
                notFull.signal();//有新的空余空间，唤醒等待的线程
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
```

offer(E e)不做等待，如果队列满了会直接入读失败

```java
    public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
        if (count.get() == capacity)
            return false;//满了直接返回false
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            if (count.get() < capacity) {
                enqueue(node);
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return c >= 0;
    }
```

offer(E e, long timeout, TimeUnit unit)能够设置等待的时间，等待超时会返回false

```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        if (e == null) throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        int c = -1;
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            while (count.get() == capacity) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);//等待指定的时间
            }
            enqueue(new Node<E>(e));
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
        return true;
    }
```

观察enqueue方法，插入的结点是在链表尾部，同时last始终指向最后一个入队的结点

```java
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }
```

我们注意到，所有入队方法在入队前容量为0时都会调用signalNotEmpty，这个方法的作用是唤醒等待线程，只有入队方法会调用它。

```java
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();//出队加锁
        try {
            notEmpty.signal();//唤醒等待获取元素队列中的线程
        } finally {
            takeLock.unlock();
        }
    }
```

### 出队

take方法出队是不限时等待，返回出队的元素

```java
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();//队列为空时不限时等待
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

poll()不做等待，而poll(long timeout, TimeUnit unit)等待指定的时间，这点和上面入队相对应

```java
    public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;//队列为空时，不等待直接返回null
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

出队时，是头部的元素先被弹出，head始终指向一个item为null的结点

```java
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
```

同样的，出队也有在容量满时去唤醒等待的入队线程的方法

```java
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();//入队加锁
        try {
            notFull.signal();//唤醒等待放入元素队列中等待的线程
        } finally {
            putLock.unlock();
        }
    }
```

peek方法虽然是只获取头部元素并不删除元素，但是也需要获取出队锁

```java
    public E peek() {
        if (count.get() == 0)
            return null;//不做等待
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();//需要获取出队锁
        try {
            Node<E> first = head.next;//获取头部元素
            if (first == null)
                return null;
            else
                return first.item;
        } finally {
            takeLock.unlock();
        }
    }
```

remove方法，检查队列中有没有指定的对象，有的话删除并返回true，否则返回false，因为需要遍历，需要对两个锁都加锁

```java
    public boolean remove(Object o) {
        if (o == null) return false;
        fullyLock();//入队和出队都加锁
        try {
            for (Node<E> trail = head, p = trail.next;
                 p != null;
                 trail = p, p = p.next) {
                if (o.equals(p.item)) {
                    unlink(p, trail);
                    return true;
                }
            }
            return false;
        } finally {
            fullyUnlock();
        }
    }
```

## ArrayBlockingQueue

底层是**数组的有界**阻塞队列，只有一把锁。锁根据构造时的参数可以是不公平的，也可以是公平的。

### 内部变量

很明显底层是数组，只有一把锁。因为数组大小指定后不能改变，所以不需要额外记录容量。因为count只在入队出队时有锁保护下变化，所以不需要额外保护

```java
    /** 队列元素 */
    final Object[] items;

    /** 下一个出队的元素 */
    int takeIndex;

    /** 下一个入队的位置 */
    int putIndex;

    /** 队列中的元素 */
    int count;

    /*
     * Concurrency control uses the classic two-condition algorithm
     * found in any textbook.
     */

    /** Main lock guarding all access */
    final ReentrantLock lock;

    /** Condition for waiting takes */
    private final Condition notEmpty;

    /** Condition for waiting puts */
    private final Condition notFull;
```

### 构造函数

必须指定数组大小，数组大小仅在构造时确定。不同之处在于，ArrayBlockingQueue可以指定锁是否是公平的，公平锁是等待线程先进先出。

```java
    public ArrayBlockingQueue(int capacity) {
        this(capacity, false);
    }

    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

### 入队

add方法本质上就是直接调用offer方法

```java
    public boolean add(E e) {
        return super.add(e);//本质上是直接调用offer
    }
```

offer不加参数时为不等待，并且它对加锁是等待获取锁

```java
    public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();//非中断加锁
        try {
            if (count == items.length)
                return false;//队列满了直接返回false
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
```

offer有时间参数时是限时等待，加锁方式变为中断式加锁

```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {

        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();//抢占式加锁
        try {
            while (count == items.length) {
                if (nanos <= 0)
                    return false;//限时等待结束
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(e);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

put方法是中断式加锁加上不限时等待

```java
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();//抢占式锁
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

enqueue，因为是循环数组，所以下标到达数组边界时要重新回到头部

```java
    private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;//添加到putIndex指向的位置
        if (++putIndex == items.length)
            putIndex = 0;//循环数组
        count++;
        notEmpty.signal();//唤醒等待获取元素阻塞的线程
    }
```

### 出队

弹出元素的三个方法

poll()非中断加锁，等待锁期间不会相应中断，获取锁后不做等待

poll(long timeout, TimeUnit unit)中断加锁，获取锁后限时等待

take()中断加锁，获取锁后不限时等待

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();//非中断加锁
        try {
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();//中断加锁
        try {
            while (count == 0)
                notEmpty.await();//不限时等待
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();//中断加锁
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);//限时等待
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

dequeue方法，操作循环数组的下标外，还需要通知迭代器有元素出队

```java
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;//将takeIndex指向的元素缓存后设为null
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();//如果有存在的迭代器，需要通知它们
        notFull.signal();//唤醒入队阻塞的线程
        return x;
    }
```

返回takeIndex指向的元素

```java
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();//等待加锁
        try {
            return itemAt(takeIndex); // 队列为空时返回null
        } finally {
            lock.unlock();
        }
    }
```

