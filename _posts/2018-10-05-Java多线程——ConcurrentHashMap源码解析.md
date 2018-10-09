---
layout:     post
title:      Java多线程——ConcurrentHashMap源码解析
subtitle:   分析并发集合类ConcurrentHashMap的实现原理
date:       2018-10-05
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

在之前讨论HashMap与HashTable时提到过，HashMap没有任何关于线程安全的处理，所以它不适合线程不安全的场景，而HashTable所有的操作方法都是加锁的，所以它是线程安全的，但是由于HashTable的一些设计上的缺陷比如每一次put或者get操作都需要重新对hash值取模来计算它的位置所以效率低。我们可以在多线程环境下通过对调用HashMap的方法进行加锁来确保其安全性，但是这样子效率还是很低。以get来说，我们知道在多线程环境下get需要检查集合中有没有某个key值，它需要根据hash值计算出位置然后检查对应的列表中有无寻找的元素，由于不能确定当前有没有正在插入元素的线程，所以需要加锁来保证安全性。但是，我们知道HashMap本身是箱式hash表，在下标位置不同时，两个线程不会操作同一个链表，所以它们之前相互不影响，也就不存在冲突，可以不用抢占同一个锁。针对不同的箱有不同的锁，这就是ConcurrentHashMap的设计方式——分段锁。

ConcurrentMap也继承了AbstractMap，实现了ConcurrentMap接口，key和value都不能为null这点和HashMap是不同的，JDK1.8采用了Node+CAS+synchronized的设计取代了1.7中的Sgement+HashEntry的设计。并且沿用了HashMap的链表转红黑树的设计。如果会修改集合中内容的方法发现正在进行resize操作，则要帮助先完成resize操作再继续。

ConcurrentMap设计核心是在HashMap箱式链表的结构基础上，每个线程只对一个箱上的结点加锁，不同箱之间不存在线程冲突。对于初始化操作使用乐观锁策略，先新建对象，通过compareAndSwap策略修改值。CAS策略基于CPU指令的支持，比加锁开销要小很多，但是它在操作失败时会自旋而不会阻塞让出CPU时间片，导致如果设计合理线程间冲突小则性能很高，如果线程间冲突严重会显著降低性能。

打开文件一看，都超过6k行了，所以还是采取从常用的方法入口开始逐渐向里追溯的分析方法。主要分析构造函数以及get、put、remove、replace、size几个常用的方法。

### 主要内部变量

这里需要注意的主要是resize过程中，为了多线程帮助转移原有的链表，用nextTable作为过程中的中间表。然后sizeCtl这个值正负有不同的意义，不再是capacity那样只用来表示容量的。为了避免插入新结点时计数器之前的碰撞，采用了分段式的计数器counterCells。

```java
    /**
     * 箱数组，在第一次插入的时候懒汉式初始化。大小总是2的幂。迭代器可以直接访问。
     */
    transient volatile Node<K,V>[] table;

    /**
     * 下一个使用的表，仅在resize过程中是非null
     */
    private transient volatile Node<K,V>[] nextTable;

    /**
     * 基本计数器，主要用于没有争夺时，但也用在表初始化争夺中的返回，通过CAS更新
     */
    private transient volatile long baseCount;

    /**
     * 表初始化和resize控制。为负数时，这个表正在初始化或者resize：-1是初始化，否则是-(1+活跃的resize线程)。
     * 未非负数时，表为null时持有创建表时的初始化大小如果是0采用默认大小。初始化之后，持有下一个resize操作的计数界限。
     */
    private transient volatile int sizeCtl;

    /**
     * 在resize时下一个分裂处的下标
     */
    private transient volatile int transferIndex;

    /**
     * 通过CAS锁定的自旋锁，用于resize时或者创建CounterCells时
     */
    private transient volatile int cellsBusy;

    /**
     * 计数器存储格表，非null时大小为2的指数次
     */
    private transient volatile CounterCell[] counterCells;
```

### 构造函数

构造函数最多是3个参数，不输入就采用默认值：表初始大小initialCapacity默认为16；负载因子loadFactor默认为0.75；并发级别concurrencyLevel代表线程数的估计值，默认为1但除了初始大小不能小于这个值以外没有其他作用。这里表的初始大小也会取恰好大于等于参数中初始大小除以负载因子加1和的2的指数次幂作为大小。sizeCtl为下一次扩展集合的边界，也就是容量。

```java
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (initialCapacity < concurrencyLevel)   // Use at least as many bins
            initialCapacity = concurrencyLevel;   // as estimated threads
        long size = (long)(1.0 + (long)initialCapacity / loadFactor);
        int cap = (size >= (long)MAXIMUM_CAPACITY) ?
            MAXIMUM_CAPACITY : tableSizeFor((int)size);//tableSizeFor获取恰好大于等于size的2的指数次幂
        this.sizeCtl = cap;
    }
```

### 静态工具方法

spread将hash值的高16位与低16位进行异或，作用是利用hash值的高位减少hash冲突，之前讲过HashMap中直接通过截取与容量大小-1的二进制位长度来进行快速定位数组中的位置。>>>是无符号右移

```java
    static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }
```

tableSizeFor计算大于等于给定值的最小2的指数次幂。我们知道二进制下低位全是1的数再加1可以获得2的整数次幂，而这个全是1的数的获取方法是，将原数-1后的值的最高位扩展到所有的低位，最后返回的结果再加1。

```java
    private static final int tableSizeFor(int c) {
        int n = c - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

然后是三个CAS方法，都是基于Unsafe调用CPU支持的系统函数完成的

```java
    //获取tab中下标为i的元素
	static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
	//修改tab中下标为i的元素值为v，要求从内存中取出时的值为c
    static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                        Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
    }
	//修改tab中下标为i的元素的值为v
    static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
    }
```

### put操作

put方法有两种类型put和putIfAbsent，可以看到它们都调用了putVal这个方法，区别在于第三个参数

```java
    public V put(K key, V value) {
        return putVal(key, value, false);
    }

    public V putIfAbsent(K key, V value) {
        return putVal(key, value, true);
    }
```

putVal就是实际操作了，首先检查箱数组有没有初始化，没有的话先要初始化。然后根据hash值计算出箱的位置，如果箱的位置为空则CAS新增一个结点。如果已有结点，则先检查有没有在做resize操作，有的话协助转移Node，没有在进行resize时对这个箱的结点加锁，检查箱式链表中有没有key值相等的结点，有的话视onlyIfAbsent的参数情况决定要不要覆盖。如果插入了新的结点到链表末尾需要检查链表长度是否达到8需要转为树结构，然后检查是否达到要进行resize的大小。最后的返回值，如果key值已经存在则返回旧的value值，否则返回null。在第一行的检查中我们可以看到，**key和value都不能为null，否则抛出NullPointerException**。

```java
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();//插入的key和value都不能是null
        int hash = spread(key.hashCode());//高位也参与hash值计算
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {//tab为箱数组
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)
                tab = initTable();//在第一次插入时进行懒汉式初始化箱数组
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {//tabAt从内存中读取箱数组指定位置
                //箱数组指定位置为null
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))//期待值n为ull，更新后的值为新建的Node，替换成功时返回true
                    break;                   //增加到一个空箱时不加锁
            }
            //箱数组指定位置已有Node
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);//正在resize移动过程中，帮助转移Node
            else {//存在普通状态Node
                V oldVal = null;
                synchronized (f) {//锁箱数组中对应位置的结点
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {
                            binCount = 1;//统计链表长度
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {//存在key相等的Node
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;//put调用时更新这个结点的value值
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);//不存在key相等的Node，将新Node添加到链表末尾
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {//树状表
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;//存在key相等的结点时返回该结点的value值
                    break;
                }
            }
        }
        addCount(1L, binCount);//添加了新的结点时增加基础计数器，并检查是否需要扩大数组
        return null;//不存在key相等的结点时返回null
    }
```

initTable初始化表，大小为记录在sizeCtl中的值，只有在第一次向集合中添加键值对时才会调用。线程会尝试将堆中实例的sizeCtl值设为-1，如果成功的话代表该线程竞争到了初始化创建数组的权限，它将新建一个大小为sizeCtl原本值的Node数组，并修改实例中的table，这里的负载因子一定是按照0.75计算。最后将sizeCtl重置回原本的值。对于没有竞争到的线程会通过Thread.yield()会主动让出线程执行时间，线程转为就绪状态，但这不代表它一定不会立刻再进入运行状态。

```java
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // 在初始化竞争中失败，自旋
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {//尝试替换sizeCtl的值为-1，先替换成功的线程竞争成功，由它进行初始化
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);//负载因子是0.75，所以sizeCtl=n-n/4
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```

addCount在新增结点时被调用，增加count，如果表太小并且还没有resize，初始化转移。如果已经resize，帮助转移。重新检查转移后的占用来看是否需要另一个resize，因为resize比增加要延迟。如果check<0不检查resize，如果check<=1只在没有竞争时检查resize。

```java
    private final void addCount(long x, int check) {
        CounterCell[] as; long b, s;
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {//counterCells存在或者当前增加baseCount存在竞争
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {//计数器未初始化或者计数器增加失败
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();
        }
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {//需要进行resize时循环
                int rs = resizeStamp(n);//获取结束位
                if (sc < 0) {
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;//没有在进行resize
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))//已有nextTab
                        transfer(tab, nt);//转移tab中的内容到nextTable
                }
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);//新建nextTable
                s = sumCount();
            }
        }
    }
```

fullAddCount作用是在在计数格不存在或者baseCount增加失败时，尝试初始化计数格或者循环尝试增加baseCount。分析fullAddCount可以发现，计数器数组为空时增加计数器数组大小为2，而传递给addCount的check参数大小是链表长度。

```java
    private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        if ((h = ThreadLocalRandom.getProbe()) == 0) {//检查线程本地随机数有没有初始化
            ThreadLocalRandom.localInit();      // force initialization强制初始化
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty上一个槽非空时为true
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
            if ((as = counterCells) != null && (n = as.length) > 0) {//counterCells不为空
                if ((a = as[(n - 1) & h]) == null) {//当前线程对应的计数器槽为空
                    if (cellsBusy == 0) {            // Try to attach new Cell尝试关联到一个新的计数格
                        CounterCell r = new CounterCell(x); // Optimistic create乐观创建
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {//CAS算法尝试替换cellsBusy值为1
                            boolean created = false;
                            try {               // Recheck under lock在锁下重新检查
                                CounterCell[] rs; int m, j;
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;//赋值新建的计数格
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;//重置cellsBusy为0
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty槽现在是非空
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail CAS已经知道失败
                    wasUncontended = true;      // Continue after rehash再重新hash之后继续
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;//计数器增加成功则跳出
                else if (counterCells != as || n >= NCPU)
                    collide = false;            // At max size or stale最大大小或者已经过时
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {//计数格数组已经存在但不够大，竞争扩展它的权限
                    try {
                        if (counterCells == as) {// Expand table unless stale除非已经过时否则扩展表
                            CounterCell[] rs = new CounterCell[n << 1];//新建一个大小为2倍的数组
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];//逐个赋值复制计数格
                            counterCells = rs;//修改数组
                        }
                    } finally {
                        cellsBusy = 0;//重置cellsBusy
                    }
                    collide = false;
                    continue;                   // Retry with expanded table重新检查扩展后的表
                }
                h = ThreadLocalRandom.advanceProbe(h);//伪随机前进并记录给定的探针值
            }
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {//counterCells为空，竞争到初始化的权利
                boolean init = false;
                try {                           // Initialize table初始化表
                    if (counterCells == as) {
                        CounterCell[] rs = new CounterCell[2];//计数格表初始化大小为2
                        rs[h & 1] = new CounterCell(x);//新建计数格
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;//初始化成功
            }
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base使用base回退
        }
    }
```

transfer移动或者复制每个箱中的结点到新的表中，代码很长，大概可以分为几个部分：1如果nextTab为空新建一个；2每个线程分配到一个下标；3检查这个下标是否有存在的结点并且没有其他线程在修改它，有的话对它加锁然后搬运。比较复杂的地方主要是分配下标，考虑到多线程进行操作时，每个箱同时只能由一个线程来处理，如果所有线程都采用从0到n的遍历方式，冲突次数会大幅度上升，这里采取的策略是通过transferIndex来记录下一个线程进入时开始的下标位置进行分段，该值的变化间隔为16或者n/8/CPU值的较大值。对一个箱的搬运工作参照HashMap的工作方式通过高位hash值将链表拆分到数组高位。

```java
    private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)//n/8/CPU个数<16
            stride = MIN_TRANSFER_STRIDE; // subdivide range细分范围
        if (nextTab == null) {            // initiating初始化
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];//新建一个当前表大小2倍的数组给nextTable
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab在提交nextTab前确保扫除完成
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {//减小transferIndex的值
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {//该线程负责转移部分已经完成
                int sc;
                if (finishing) {//resize完成
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);//n*0.75
                    return;
                }
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {//增加一个活跃线程帮助resize
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit提交前重新检查
                }
            }
            else if ((f = tabAt(tab, i)) == null)//表中不存在这个结点
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed已经在移动
            else {//表中有这个结点并且没有在移动
                synchronized (f) {//每个箱只能由一个线程在处理
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```

在putVal时，如果发现箱正在被转移说明resize操作正在进行，调用helpTransfer帮助转移

```java
    final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {//通过CAS将sizeCtl+1来竞争操作权
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

### get

相比之下，get操作就比较简单了，从堆中找到table中下标位置的结点，然后顺着链表寻找key值相等的结点。find方法是预留给Node的子类重写hash值小于0的方法

```java
    public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {//从内存中获取table指定下标位置的结点
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;//找到了指定的key值
            }
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;//留给子类的重写方法，否则还是从这个链表中遍历寻找
            while ((e = e.next) != null) {//遍历链表寻找hash值
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```

### size

size返回键值对的数量，最大不能超过整数上限。

```java
    public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }
```

其中的关键是sumCount方法，我们前面提到有baseCount和counterCells数组两个和计数有关的变量，分析addCount时我们可以看到优先增加baseCount的值，如果失败的话找到线程对应的计数格增加它的计数器，所以键值对的个数为两类变量的总和。

```java
    final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
```

### remove和replace

remove是移除指定的结点，replace是将指定结点替换为指定值。虽然有不同的重载版本，区别是要不要指定结点的value值，但除了一些参数上的判断之外，它们都是基于replaceNode方法来完成的。

```java
    public V remove(Object key) {
        return replaceNode(key, null, null);
    }

    public boolean remove(Object key, Object value) {
        if (key == null)
            throw new NullPointerException();
        return value != null && replaceNode(key, null, value) != null;
    }

    public boolean replace(K key, V oldValue, V newValue) {
        if (key == null || oldValue == null || newValue == null)
            throw new NullPointerException();
        return replaceNode(key, newValue, oldValue) != null;
    }

    public V replace(K key, V value) {
        if (key == null || value == null)
            throw new NullPointerException();
        return replaceNode(key, value, null);
    }
```

replaceNode实现4个remove/replace操作：替换结点值为v，如果cv不是null则它原本的value需要等于cv，如果结果value值是null则删除这个结点。

```java
    final V replaceNode(Object key, V value, Object cv) {
        int hash = spread(key.hashCode());
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0 ||
                (f = tabAt(tab, i = (n - 1) & hash)) == null)
                break;//表不存在或者表为空或者表中找不到hash值对应的结点，直接返回
            else if ((fh = f.hash) == MOVED)
                tab = helpTransfer(tab, f);//正在resize先帮助转移
            else {//表中存在该位置的结点
                V oldVal = null;
                boolean validated = false;
                synchronized (f) {
                    if (tabAt(tab, i) == f) {//双重确认
                        if (fh >= 0) {//hash值大于0说明是链表结点
                            validated = true;
                            for (Node<K,V> e = f, pred = null;;) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    V ev = e.val;
                                    if (cv == null || cv == ev ||
                                        (ev != null && cv.equals(ev))) {
                                        oldVal = ev;
                                        if (value != null)
                                            e.val = value;//value不为null替换值
                                        else if (pred != null)//需要删除结点
                                            pred.next = e.next;//该结点不是链表头
                                        else
                                            setTabAt(tab, i, e.next);//该结点时链表头
                                    }
                                    break;
                                }
                                pred = e;
                                if ((e = e.next) == null)
                                    break;
                            }
                        }
                        else if (f instanceof TreeBin) {//树状链表
                            validated = true;
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> r, p;
                            if ((r = t.root) != null &&
                                (p = r.findTreeNode(hash, key, null)) != null) {
                                V pv = p.val;
                                if (cv == null || cv == pv ||
                                    (pv != null && cv.equals(pv))) {
                                    oldVal = pv;
                                    if (value != null)
                                        p.val = value;
                                    else if (t.removeTreeNode(p))
                                        setTabAt(tab, i, untreeify(t.first));
                                }
                            }
                        }
                    }
                }
                if (validated) {
                    if (oldVal != null) {
                        if (value == null)
                            addCount(-1L, -1);//减少计数器，因为第二个参数是-1所以肯定不会检查resize
                        return oldVal;
                    }
                    break;
                }
            }
        }
        return null;
    }
```