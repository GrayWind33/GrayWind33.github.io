---
layout:     post
title:      Java多线程——PriorityBlockingQueue DelayQueue源码解析
subtitle:   分析多线程无界优先级阻塞队列PriorityBlockingQueue和DelayQueue的实现原理
date:       2018-11-03
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 多线程
    - 集合
---

PriorityBlockingQueue和DelayQueue的底层实现其实很相似，都是基于堆排序的优先队列，堆中元素最小的最先出队。它们都只有一把非公平锁来控制入队和出队等public操作，并且都是无界阻塞队列，也就是说入队一定不会因为容量大小而阻塞，但是可能因为数组大小超过内存而OOM。后者的区别在于，出队获取元素前一定会经过设定的等待时间，而前者没有，所以可以用于定期过期的缓存等使用场景。

## PriorityBlockingQueue

PriorityBlockingQueue是无界阻塞队列，底层是基于堆排序的优先队列数组，每次出队的是当前队列中优先度最高的元素，而不是最先入队的元素。只有一把锁来控制入队、出队操作，并且是非公平锁。

### 内部变量

默认的初始数组大小是11，最大是Integer.MAX_VALUE - 8。底层是由数组实现的堆，只有一把锁来保护所有public方法。comparator是比较器，如果为null则依赖于元素本身的排序，注意comparator仅可在构造时中设定。

```java
    /**
     * 默认大小
     */
    private static final int DEFAULT_INITIAL_CAPACITY = 11;

    /**
     * 分配数组大小的最大上限
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 优先队列通过comparator排序，或者当comparator为null时为元素的自然排序：堆中结点大于它下方的结点，最小的是queue[0]
     */
    private transient Object[] queue;

    /**
     * 优先队列中的元素个数
     */
    private transient int size;

    /**
     * 比较器，为null时使用元素的自然排序
     */
    private transient Comparator<? super E> comparator;

    /**
     * 用于所有public方法的锁
     */
    private final ReentrantLock lock;

    /**
     * 队列为空时的阻塞条件
     */
    private final Condition notEmpty;

    /**
     * 通过CAS分配和获取时的自旋锁
     */
    private transient volatile int allocationSpinLock;

    /**
     * 只用于序列化的扁平优先队列
     */
    private PriorityQueue<E> q;
```

### 构造函数

构造函数默认初始数组大小是11，默认没有比较器，如果要设定的话需要在构造函数中给出

```java
    public PriorityBlockingQueue() {
        this(DEFAULT_INITIAL_CAPACITY, null);
    }

    public PriorityBlockingQueue(int initialCapacity) {
        this(initialCapacity, null);
    }

    public PriorityBlockingQueue(int initialCapacity,
                                 Comparator<? super E> comparator) {
        if (initialCapacity < 1)
            throw new IllegalArgumentException();
        this.lock = new ReentrantLock();//非公平锁
        this.notEmpty = lock.newCondition();
        this.comparator = comparator;
        this.queue = new Object[initialCapacity];
    }
```

对于复制构造，如果输入的是排序堆，则构造的优先队列会获取它的比较器，否则没有比较器。输入的是非排序堆时，需要再做堆排序的初始化。

```java
    public PriorityBlockingQueue(Collection<? extends E> c) {
        this.lock = new ReentrantLock();
        this.notEmpty = lock.newCondition();
        boolean heapify = true; // true if not known to be in heap order不知道是否是堆排序时为true
        boolean screen = true;  // true if must screen for nulls必须筛选null时为true
        if (c instanceof SortedSet<?>) {
            SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
            this.comparator = (Comparator<? super E>) ss.comparator();
            heapify = false;
        }
        else if (c instanceof PriorityBlockingQueue<?>) {
            PriorityBlockingQueue<? extends E> pq =
                (PriorityBlockingQueue<? extends E>) c;
            this.comparator = (Comparator<? super E>) pq.comparator();
            screen = false;
            if (pq.getClass() == PriorityBlockingQueue.class) // exact match
                heapify = false;
        }
        Object[] a = c.toArray();//集合c转为数组
        int n = a.length;
        // If c.toArray incorrectly doesn't return Object[], copy it.
        if (a.getClass() != Object[].class)
            a = Arrays.copyOf(a, n, Object[].class);
        if (screen && (n == 1 || this.comparator != null)) {
            for (int i = 0; i < n; ++i)
                if (a[i] == null)
                    throw new NullPointerException();
        }
        this.queue = a;
        this.size = n;
        if (heapify)
            heapify();//如果c不是堆排序，则需要对它进行排序
    }
```

heapify是堆排序初始化，只有复制构造时会调用。我们知道，堆排序需要从底部第二层最左边开始向右再向上进行遍历，检查是否每个结点都小于它的子结点，或者本身已经是叶结点，如果不符的话需要下降该结点。

```java
    private void heapify() {
        Object[] array = queue;
        int n = size;
        int half = (n >>> 1) - 1;//堆排序初始化从底部第二次最左边的结点开始
        Comparator<? super E> cmp = comparator;
        if (cmp == null) {
            for (int i = half; i >= 0; i--)
                siftDownComparable(i, (E) array[i], array, n);//无比较器排序
        }
        else {
            for (int i = half; i >= 0; i--)
                siftDownUsingComparator(i, (E) array[i], array, n, cmp);//有比较器排序
        }
    }
```

siftDownComparable和siftDownUsingComparator分别是使用元素本身和提供的比较器的两个将结点下降的方法，插入的结点会被下降直到它小于它的两个儿子或者它成为叶结点。

```java
    private static <T> void siftDownComparable(int k, T x, Object[] array,
                                               int n) {
        if (n > 0) {
            Comparable<? super T> key = (Comparable<? super T>)x;
            int half = n >>> 1;           // loop while a non-leaf在非叶结点时循环
            while (k < half) {
                int child = (k << 1) + 1; // assume left child is least假设左儿子最小
                Object c = array[child];
                int right = child + 1;
                if (right < n &&
                    ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                    c = array[child = right];//因为右儿子小于左儿子，child指向右儿子
                if (key.compareTo((T) c) <= 0)
                    break;//插入结点小于它较小的儿子则不需要调整
                array[k] = c;//插入结点的值太大，向下降
                k = child;
            }
            array[k] = key;
        }
    }

    private static <T> void siftDownUsingComparator(int k, T x, Object[] array,
                                                    int n,
                                                    Comparator<? super T> cmp) {
        if (n > 0) {
            int half = n >>> 1;
            while (k < half) {
                int child = (k << 1) + 1;
                Object c = array[child];
                int right = child + 1;
                if (right < n && cmp.compare((T) c, (T) array[right]) > 0)
                    c = array[child = right];
                if (cmp.compare(x, (T) c) <= 0)
                    break;
                array[k] = c;
                k = child;
            }
            array[k] = x;
        }
    }
```

### 入队

因为PriorityBlockingQueue是无界队列，所以**入队不会存在阻塞**，因此add、put和offer的延时方法没有实际作用，都等于调用offer(E e)。offer(E e)不能入队为null的元素，首先需要获取锁，非中断式加锁，所以不会响应中断异常。入队时，先检查是否需要扩容，在容量足够后先将元素放到数组的末尾，然后检查是否需要上升来保持堆的性质。

```java
    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();//队列中不能放入null
        final ReentrantLock lock = this.lock;
        lock.lock();//非中断式加锁
        int n, cap;
        Object[] array;
        while ((n = size) >= (cap = (array = queue).length))
            tryGrow(array, cap);//容量满了，尝试扩容
        try {
            Comparator<? super E> cmp = comparator;
            if (cmp == null)
                siftUpComparable(n, e, array);
            else
                siftUpUsingComparator(n, e, array, cmp);
            size = n + 1;
            notEmpty.signal();//唤醒队列为空时阻塞的出队线程
        } finally {
            lock.unlock();
        }
        return true;
    }
```

结点上移的方法同样根据比较器的不同分为两个版本。新结点从插入位置开始，不断判断是否大于它的父结点，如果小于的话需要交换自己和父结点的位置。

```java
    private static <T> void siftUpComparable(int k, T x, Object[] array) {
        Comparable<? super T> key = (Comparable<? super T>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;//k的父结点
            Object e = array[parent];
            if (key.compareTo((T) e) >= 0)
                break;//插入的结点大于父结点则符合堆性质退出方法
            array[k] = e;//父结点下移动到插入结点的位置
            k = parent;//插入结点上移到父结点位置
        }
        array[k] = key;
    }

    private static <T> void siftUpUsingComparator(int k, T x, Object[] array,
                                       Comparator<? super T> cmp) {
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = array[parent];
            if (cmp.compare(x, (T) e) >= 0)
                break;
            array[k] = e;
            k = parent;
        }
        array[k] = x;
    }
```

扩容tryGrow，首先需要竞争自旋锁来决定哪个线程来分配新的数组，此时不需要lock，在分配完之后才需要获取lock，然后复制数组元素到新的数组中。

```java
    private void tryGrow(Object[] array, int oldCap) {
        lock.unlock(); // must release and then re-acquire main lock
        Object[] newArray = null;
        if (allocationSpinLock == 0 &&
            UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,
                                     0, 1)) {//CAS竞争自旋锁，自旋循环在调用方法中完成
            try {
                int newCap = oldCap + ((oldCap < 64) ?
                                       (oldCap + 2) : // grow faster if small
                                       (oldCap >> 1));//原本大小小于64时乘以2否则是加2
                if (newCap - MAX_ARRAY_SIZE > 0) {    // possible overflow
                    int minCap = oldCap + 1;
                    if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                        throw new OutOfMemoryError();//可能会超过内存上限
                    newCap = MAX_ARRAY_SIZE;
                }
                if (newCap > oldCap && queue == array)
                    newArray = new Object[newCap];
            } finally {
                allocationSpinLock = 0;
            }
        }
        if (newArray == null) // back off if another thread is allocating如果其他线程分配时后退
            Thread.yield();
        lock.lock();
        if (newArray != null && queue == array) {
            queue = newArray;//修改数组
            System.arraycopy(array, 0, newArray, 0, oldCap);//复制
        }
    }
```

### 出队

出队方法在当前队列为空时的处理方法不同，可能会阻塞等待，也可能直接返回

poll()非中断式的获取锁，然后不做等待，直接尝试出队，如果当前队列为空返回null

poll(long timeout, TimeUnit unit)在等待获取锁时响应中断，获取锁后如果队列为空会进行限时等待

take()在等待获取锁时响应中断，获取锁后如果队列为空会进行不限时等待

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return dequeue();//直接进行出队操作
        } finally {
            lock.unlock();
        }
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        E result;
        try {
            while ( (result = dequeue()) == null)
                notEmpty.await();//不限时等待
        } finally {
            lock.unlock();
        }
        return result;
    }

    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        E result;
        try {
            while ( (result = dequeue()) == null && nanos > 0)
                nanos = notEmpty.awaitNanos(nanos);//限时等待
        } finally {
            lock.unlock();
        }
        return result;
    }
```

分析dequeue()方法，首先出队的是对顶也就是堆中最小的元素，然后将数组末尾元素放到堆顶，然后对它进行循环降级操作

```java
    private E dequeue() {
        int n = size - 1;
        if (n < 0)
            return null;
        else {
            Object[] array = queue;
            E result = (E) array[0];//出队的是堆顶也就是最小的元素
            E x = (E) array[n];//取出数组末尾元素
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            if (cmp == null)//对新放到堆顶的元素进行降级
                siftDownComparable(0, x, array, n);
            else
                siftDownUsingComparator(0, x, array, n, cmp);
            size = n;
            return result;
        }
    }
```

peek()方法是返回堆顶元素但不出队

```java
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return (size == 0) ? null : (E) queue[0];//返回堆顶元素，为空时返回null
        } finally {
            lock.unlock();
        }
    }
```

remove(Object o)查找队列中有无指定对象，有则移除返回true，没有则返回false。移除时，如果移除的是数组末尾元素则不需要调整，如果不是，则将数组末尾元素移动到移除元素的位置，然后尝试对它进行降级或升级。

```java
    public boolean remove(Object o) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            int i = indexOf(o);//遍历数组，寻找与o相等的元素
            if (i == -1)
                return false;//没找到返回false
            removeAt(i);
            return true;
        } finally {
            lock.unlock();
        }
    }

    private void removeAt(int i) {
        Object[] array = queue;
        int n = size - 1;
        if (n == i) // 移除最后一个元素
            array[i] = null;
        else {
            E moved = (E) array[n];//将数组末尾元素先放到移除元素的位置
            array[n] = null;
            Comparator<? super E> cmp = comparator;
            if (cmp == null)//对末尾元素降级
                siftDownComparable(i, moved, array, n);
            else
                siftDownUsingComparator(i, moved, array, n, cmp);
            if (array[i] == moved) {
                if (cmp == null)//如果末尾元素不需要降级则尝试对它升级
                    siftUpComparable(i, moved, array);
                else
                    siftUpUsingComparator(i, moved, array, cmp);
            }
        }
        size = n;
    }
```

## DelayQueue

DelayQueue底层是一个PriorityQueue来存储队列中的元素，所以它也是无界的优先队列，只有一把非公平锁来保证入队和出队的线程安全。所以，总体和PriorityBlockingQueue类似，区别在于出队获取元素时只要获取到元素就一定会等待元素本身设定的延迟时间。使用场景：具有过期时间的缓存

### 内部变量

初始化时会构造一个优先队列和一把锁，leader是指定等待元素的队列头部的线程

```java
    private final transient ReentrantLock lock = new ReentrantLock();
    private final PriorityQueue<E> q = new PriorityQueue<E>();
	private Thread leader = null;
```

### 入队

add(E e)、put(E e)和offer(E e, long timeout, TimeUnit unit)都是调用offer(E e)。offer(E e)方法在获取锁之后，会调用优先队列的入队方法，同样也是先将元素放到数组末尾再尝试上升。同样，入队不存在容量上限造成阻塞。

```java
    public boolean offer(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            q.offer(e);//队列入队e
            if (q.peek() == e) {
                leader = null;
                available.signal();
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;//先将元素放到数组末尾
        if (i >= queue.length)
            grow(i + 1);
        size = i + 1;
        if (i == 0)//队列为空时不需要调整
            queue[0] = e;
        else
            siftUp(i, e);//队列不为空时需要元素上升
        return true;
    }
```

### 出队

poll()获取锁后不做等待，直接尝试出队

```java
    public E poll() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            E first = q.peek();
            if (first == null || first.getDelay(NANOSECONDS) > 0)
                return null;//队列头部元素不存在且延迟已经超时返回null
            else
                return q.poll();//队列为空时依然返回null
        } finally {
            lock.unlock();
        }
    }
```

take()获取锁后，根据堆中元素的延迟时间，在队列中有元素时一定会经过这段延迟等待时间才会返回元素。队列为空时会不限时等待。

```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null)
                    available.await();//队列为空时不限时等待
                else {
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();//延迟超时，直接进行出队
                    first = null; // don't retain ref while waiting
                    if (leader != null)
                        available.await();
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            available.awaitNanos(delay);//等待延迟时间
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```

而poll(long timeout, TimeUnit unit)在队列为空时限时等待，队列不为空成功弹出元素时一定要等待弹出的元素设定的延迟时间。

```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            for (;;) {
                E first = q.peek();
                if (first == null) {//队列为空时限时等待
                    if (nanos <= 0)
                        return null;
                    else
                        nanos = available.awaitNanos(nanos);
                } else {//队列不为空时需要等待元素的延迟时间
                    long delay = first.getDelay(NANOSECONDS);
                    if (delay <= 0)
                        return q.poll();
                    if (nanos <= 0)
                        return null;
                    first = null; // don't retain ref while waiting
                    if (nanos < delay || leader != null)
                        nanos = available.awaitNanos(nanos);
                    else {
                        Thread thisThread = Thread.currentThread();
                        leader = thisThread;
                        try {
                            long timeLeft = available.awaitNanos(delay);
                            nanos -= delay - timeLeft;
                        } finally {
                            if (leader == thisThread)
                                leader = null;
                        }
                    }
                }
            }
        } finally {
            if (leader == null && q.peek() != null)
                available.signal();
            lock.unlock();
        }
    }
```

peek()和remove(Object o)不需要等待

```java
    public E peek() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return q.peek();
        } finally {
            lock.unlock();
        }
    }

    public boolean remove(Object o) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return q.remove(o);
        } finally {
            lock.unlock();
        }
    }
```

