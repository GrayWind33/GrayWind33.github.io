---
layout:     post
title:      Java多线程 ThreadLocal源码解析
subtitle:   分析ThreadLocal和其内部类ThreadLocalMap的实现原理
date:       2018-10-08
author:     GrayWind
header-img: img/post-bg-mma-0.png
catalog: true
tags:
    - JDK源码解析
    - 多线程
---

ThreadLocal是作为key值存储在ThreadLocalMap里面的，而ThreadLocalMap是一个典型的hash表，它的实例存储在了Thread.threadLocals，并且由于并非所有线程实例都需要用到threadLocals，它是懒汉式的初始化，在第一次插入时才会初始化创建ThreadLocalMap实例。输入的value是一个泛型对象，它可以是Integer、Double等基本装箱类型，也可以是自定义的bean类，可以认为Thread实例t拥有一个ThreadLocalMap实例threadLocals，它是一个hash表，它以ThreadLocal的实例作为key值，以具体内容泛型变量value作为value值，每个线程在存活期间有自己的ThreadLocalMap实例，所以各线程间互不干涉。示例如下：

```java
public class ThreadLocalTest {
	ThreadLocal<Long> longLocal = new ThreadLocal<Long>();
	ThreadLocal<String> stringLocal = new ThreadLocal<String>();

	public void set() {
		longLocal.set(Thread.currentThread().getId());
		stringLocal.set(Thread.currentThread().getName());
	}

	public long getLong() {
		return longLocal.get();
	}

	public String getString() {
		return stringLocal.get();
	}

	public static void main(String[] args) throws InterruptedException {
		final ThreadLocalTest test = new ThreadLocalTest();

		test.set();
		System.out.println(test.getLong());
		System.out.println(test.getString());

		Thread thread1 = new Thread() {
			public void run() {
				test.set();
				System.out.println(test.getLong());
				System.out.println(test.getString());
			};
		};
		thread1.start();
		thread1.join();

		System.out.println(test.getLong());
		System.out.println(test.getString());
		/*1
        main
        11
        Thread-0
        1
        main*/
	}
}
```

### ThreadLocal

这个类提供线程局部变量。这些变量在每一个线程中的正常副本都不相同，每一个线程访问一个副本（通过其 get或 set法），副本有自己独立的变量初始化复制。ThreadLocal实例通常是类中私有的静态字段希望关联状态和线程（例如，一个用户ID或交易ID）。

例如，下面的类为每个线程生成唯一的标识符。一个线程的ID在第一次调用ThreadId.get()时分配并且在后续调用中保持不变。

```java
 public class ThreadId {
     // Atomic integer containing the next thread ID to be assigned
     private static final AtomicInteger nextId = new AtomicInteger(0);

     // Thread local variable containing each thread's ID
     private static final ThreadLocal<Integer> threadId =
         new ThreadLocal<Integer>() {
             @Override protected Integer initialValue() {
                 return nextId.getAndIncrement();
         }
     };

     // Returns the current thread's unique ID, assigning it if necessary
     public static int get() {
         return threadId.get();
     }
 }
```

每一个线程在活着的时候就持有一个暗示对它线程本地变量拷贝的引用并且ThreadLocal实例可以访问；线程死去后它的线程本地实例拷贝受垃圾回收管辖（除非存在其他对这些副本引用）。

——以上是对ThreadLocal注释的翻译

### 构造函数

因为ThreadLocal的作用是提供实例作为key值，它需要提供的内部变量就是hashcode，而hashcode是由别的方法生成，所以构造函数除了初始化实例外什么也不做

```java
    public ThreadLocal() {
    }
```

### set

set方法设置当前线程的线程本地变量副本为指定的值。大部分子类不需要重写这个方法，只依靠initialValue来设置线程本地变量值。先获取当前线程中的ThreadLocalMap实例，然后检查是否为null。还没有创建的话需要先新创建一个，将这个ThreadLoca和value值放入map中。

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    /**
     * 获取关联ThreadLocal的map。在InheritableThreadLocal中重写。
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    /**
     * 创建一个关联ThreadLocal的map。在InheritableThreadLocal中重写。
     */
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

### get

get返回这个线程本地变量在当前线程中的副本值。如果这个变量在当前线程中没有值，第一次通过调用initialValue初始化这个值并返回。

```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();//map不存在或map中没有这个对象时调用setInitialValue
    }
```

setInitialValue方法是set方法的变体，用于创建初始化值。如果用户已经重写了set方法，可以用作set方法的替代。

```java
    private T setInitialValue() {
        T value = initialValue();//未重写直接返回null
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

initialValue返回此线程局部变量的“初始值”。该方法将在一个线程第一次用get方法访问变量时被调用，除非线程之前调用了 set方法，在这种情况下， initialValue方法不会被这个线程调用。通常情况下，这种方法是每个线程调用一次，但它可能在后续调用get随后调用remove而再次调用。这种实现简单的返回null；如果程序员渴望线程局部变量有一个初始值而不是null，ThreadLocal必须有子类，并重写这个方法。通常情况下，将使用一个匿名内部类。**简单来说，如果未set这个ThreadLocal的value值而直接调用get会导致在map中添加一个初始化的value值，如果没有重写ThreadLocal中的这个方法，那么初始值是null**。

```java
    protected T initialValue() {
        return null;
    }
```

### remove

remove方法移除当前线程的线程本地变量值。如果这个线程本地变量随后被当前线程用get方法读取，它的值会被initialValue重新初始化，除非这之间当前线程调用了set方法。这可能会导致当前线程中多次调用initialValue方法。

```java
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)//map未初始化时什么都不做
             m.remove(this);
     }
```

## ThreadLocalMap

ThreadLocalMap是ThreadLocal的内部类，是一个典型的hash表，只适合存储线程本地变量。没有操作暴露到ThreadLocal类外部。这个类是包私有的，允许在Thread中声明字段。为了帮助解决非常大并且长期存活的使用，hash表条目对key使用WeakReference。然而，因为没有使用引用队列，旧的条目只在表用完空间时才保证移除。

### Entry

ThreadLocalMap的条目也是自己实现的。这个hash表的条目扩展了WeakReference，使用它的只要引用字段作为k(总是ThreadLocal对象)。注意null的key值(比如entry.get() == null)意味着key不再被引用，因此条目可以从表删除。这样的条目作为“旧条目”在下方代码中被引用。ThreadLocal是弱引用，而value是强引用，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** 关联这个ThreadLocal的值 */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);//将ThreadLocal作为key用于构造WeakReference
                value = v;
            }
        }
```

### 内部变量和构造函数

table是存储条目的数组，初始大小一定是16

```java
        /**
         * 初始容量，必需是2的指数次
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * 表，有必要的话需要resize。table.length必须总是2的指数次
         */
        private Entry[] table;

        /**
         * 表中条目的数量
         */
        private int size = 0;

        /**
         * 到达后要进行resize的大小
         */
        private int threshold; // Default to 0
```

构造函数前面在createMap(Thread, T)中已经提到过，创建一个新的map初始化包含firstKey, firstValue)。ThreadLocalMap是懒汉式构建，因此我们只有当有至少一个条目要放入时再创建一个。负载因子至少是2/3。

```java
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }

        /**
         * 设置resize的门槛保证最差有2/3的负载因子
         */
        private void setThreshold(int len) {
            threshold = len * 2 / 3;
        }
```

还有一个版本是创建一个新的map包含所有来自父map的可继承的ThreadLocal。只会由createInheritedMap调用。因为较少使用就不说了。

### getEntry

getEntry获取和key关联的条目。这个方法本身只处理快速通道：已有key直接命中，否则会转移到getEntryAfterMiss。这种设计是为了最大化直接命中的效率，部分通过使这个方法容易非线性读取来实现。

```java
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);//根据hash值计算出直接命中的位置
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;//命中
            else
                return getEntryAfterMiss(key, i, e);//未命中
        }
```

如果直接目标未命中，需要getEntryAfterMiss来进行线性探测。由于散列的hash值碰撞情况很小，所以比起直接用循环查找的方法效率更高。在线性探测过程中，移动就是最简单的移动到尾部循环至头部，如果发现ThreadLocal为null说明已经被移除，需要删除这个结点的引用使得可以进行回收。

```java
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);//不再有引用了，需要删除
                else
                    i = nextIndex(i, len);//向后移动一位
                e = tab[i];
            }
            return null;
        }

        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
```

### hash

刚才提到了hash和表中对象的关系，那么要研究下ThreadLocalMap的hash值是怎么设计的。首先，要明确的一点是ThreadLocalMap使用的是开放地址法线性探测解决hash碰撞的问题，而hash值是在ThreadLocal中的。我们可以看到ThreadLocal中跟hash值计算有关的部分，用一个AtomicInteger来计算，是static也就是说有一个静态的值每新建一个ThreadLocal实例就会增加，但增加的间隔并不是1，而是0x61c88647，这个值得选取是为了减少碰撞的发生，具体原理和斐波那契散列法以及黄金分割有关。 所以，对于同样的ThreadLocal变量，各线程的ThreadLocalMap中它们都处于同一个数组位置。因此，还是建议每个线程只存一个变量，这样的话所有的线程存放到map中的Key都是相同的ThreadLocal，如果一个线程要保存多个变量，就需要创建多个ThreadLocal，多个ThreadLocal放入Map中时会极大的增加hash冲突的可能。如果必须使用多个变量，0x61c88647可以尽可能减少冲突的发生。因为表的大小永远是2的指数次，所以和len-1进行位与操作等价为直接取模。

```java
    /**
     * ThreadLocal依赖每个线程线性探测hash表关联到每个线程(Thread.threadLocals和inheritableThreadLocals)。
     * ThreadLocal对象作为key，通过threadLocalHashCode来查找。
     * 这是一个典型的hash值(只在ThreadLocalMaps内有用)消除当同样的线程连续创建ThreadLocal实例常见状况下的冲突，同时不太常见的状况也是良性的。
     */
    private final int threadLocalHashCode = nextHashCode();

    /**
     * 下一个要给出的hash值，自动更新，从0开始。
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * 连续产生的hash值之间的差值-使得连续的线程本地ID变得隐式连续，接近最佳地扩展到大小是2的指数次表的乘法倍数的hash值上。
     */
    private static final int HASH_INCREMENT = 0x61c88647;//‭0110 0001 1100 1000 1000 0110 0100 0111‬

    /**
     * 返回下一个hash值
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```

### set

set没有像get那样使用快速路径是因为使用set创建一个新的条目至少和替换一个已有的是一样频率，这样快速路径会经常失败。set会从hash对应的位置开始线性查找指定的key值，如果找到entry存在但key为null的位置说明已经被remove，使用replaceStaleEntry替换过时的值。如果找到了这个key值则替换已有值。如果找到了entry为null的位置说明还未被使用过，直接新建一个entry放到这个位置上。

```java
        private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;//替换已有值
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);//替换过时的值
                    return;
                }
            }

            tab[i] = new Entry(key, value);//hash值对应的位置没有插入过元素
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

replaceStaleEntry用一个有指定key的条目替换在set操作中遇到的过时条目。value参数传递的值存储在条目中，无论有这个key的条目是否已经存在。作为一个副作用，这个方法擦除了一趟中所有过时的条目(一趟指两个entry为null位置间的序列)。新的条目放在staleSlot位置上，而从这个位置开始向前和向后倒两个entry为null的位置之间所有key为null的条目都会被清除。

```java
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            //返回检查当前趟中之前的过期条目。我们一次清除整个趟避免因为垃圾回收释放串中的引用而频繁增加的rehash
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))//从staleSlot开始向前直到entry为null的位置为止最靠前的引用为null的条目
                if (e.get() == null)
                    slotToExpunge = i;

            //寻找key或者趟里后面null位置两者中先发生的
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {//循环从staleSlot向后倒entry为null的位置
                ThreadLocal<?> k = e.get();

                //如果找到key，我们需要交换它和过期条目来保持hash表顺序。新的过期位置或者在它之前任何其他过期位置，
                //可以被发送给expungeStaleEntry来移除或者rehash趟中的其他所有条目
                if (k == key) {//找到了set操作中要插入的key
                    e.value = value;

                    tab[i] = tab[staleSlot];//将key相同的结点与staleSlot位置的结点交换
                    tab[staleSlot] = e;

                    // 如果有的话开始擦除先前的过期结点
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);//第一个参数是从slotToExpunge到下一个null的位置，第二个参数是table长度
                    return;
                }

                //向前查找没有找到过期条目，在查找key时第一个发现的过期条目还是趟中的第一个
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // 如果没有找到key，将新的条目放在过期的位置
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // 如果这一趟中还有其他过期条目，擦除它们
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```

expungeStaleEntry通过rehash任何在staleSlot与下一个null位置之间可能冲突条目来擦除过期条目。这也擦除了在随后的null之前遇到的所有其他过期条目。返回staleSlot之后下一个null的位置(在staleSlot和这个位置之间的都被检查是否要擦除)。

```java
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // 擦除在staleSlot的条目
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // 直到遇到null之前rehash
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {//key为null说明已经被remove，删除其他数据
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {//key存在，移动到接近hash直接定位的地方
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```

cleanSomeSlots检查log2(n)个位置，除非找到了一个过期条目，额外检查log2(table.length)-1个位置。插入时调用这个参数是元素个数，replaceStaleEntry调用时这个参数是table的大小。

```java
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {//找到过期的条目需要清除
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);//n=n/2
            return removed;
        }
```

set在新增一个条目到没有使用过的位置时，导致size增加，同时会触发rehash。会清空所有过期的条目，并根据size大小判断是否需要扩大数组，如果要扩大数组则数组大小*2。

```java
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();//size达到了resize的大小，扩大数组
        }

		/**
         * 两倍扩大表
         */
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }

        /**
         * 清除表中稳定所有过期条目
         */
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
    }
```

### remove

移除key对应的条目，直接通过线性探测法找到对应的key值，条目调用clear()方法后key会变为null但条目仍然存在，通过expungeStaleEntry(int)将table中这个位置置为null，并且gc可以回收这个条目。**当线程终止时，它的ThreadLocal对象引用会变为null，那么在后续使用中table中不再有引用的条目会被垃圾回收，但是如果线程长时间存活但ThreadLocal对象不再被使用，需要显示的调用remove方法，避免内存泄漏。**

```java
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {//线性查找hash值相等的条目
                if (e.get() == key) {
                    e.clear();//清除引用
                    expungeStaleEntry(i);//删掉过期条目
                    return;
                }
            }
        }
```

