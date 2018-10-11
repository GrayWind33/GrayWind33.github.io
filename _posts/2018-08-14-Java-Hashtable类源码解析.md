---
layout:     post
title:      Java Hashtable类源码解析
subtitle:   分析Map集合Hashtable的实现原理，与HashMap进行比较
date:       2018-08-14
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 集合
---

老生常谈的问题——Hashtable和HashMap有什么区别

大家一般都能说出几条，比如Hashtable是线程安全的，不支持null作为key和value值等等。那么，要仔细了解这个问题还是直接从Hashtable的源码入手。

先列一下我找到的区别：

1. 继承类不同，Hashtable继承的是Dictionary这是一个废弃类，而HashMap继承的是AbstractMap
2. 产生时间不同，Hashtable自JDK1.0版本就有了，而HashMap是JDK1.2才加入的，同时Hashtable可能因为历史原因并不是我们习惯的驼峰法命名的
3. Hashtable比HashMap多提供了elments()方法用于返回此Hashtable中的value的枚举
4. Hashtable既不支持null key也不支持null value
5. Hashtable的默认大小是11，扩大的逻辑是*2+1，对于给定大小不会做扩展。而HashMap是16，扩大时*2，初始大小会转换成恰好大于等于的2的指数次幂
6. Hashtable中的遍历操作是从高位开始的，而HashMap是从低位开始
7. Hashtable处理冲突元素时插入到链表头部，而HashMap是插入到链表尾部
8. Hashtable的hashcode方法计算所有entry的hashcode总和，HashMap没有这样的方法，同时HashMap在计算hash值时会用高位右移16位与低位异或来打散散列值，避免位与操作造成冲突过多
9. Hashtable每一次定位都要做一次完整的除法取余数，而HashMap使用的是与数组大小-1的位与计算，效率高很多
10. Hashtable的方法都加上了synchronized是线程安全的方法，而HashMap不是，所以单线程时前者额外开销很大。JDK8以后Hashtable也用了modCount来保证在遍历过程中其他线程修改对象的fast-fail机制。但是，即使是多线程环境下，依然应该优先选择对HashMap进行一些特殊处理而不是用Hashtable，因为所有方法都加上synchronized的程序并发性很差。实际上就我个人经验而言，在一些特定的具体情况下，比如大规模写入key值连续数据（出自今年的第四届阿里中间件性能挑战赛复赛题），链表法解决冲突性能可能不如开放地址法，即使加上了红黑树。所以说对于一些对极致压榨性能的情况下，适当的可以抛弃一些通用的集合而尝试自由发挥造轮子。

------

 首先从最上方的注释中可以看到Hashtable自JDK1.0版本就有了，而HashMap是JDK1.2才加入的。观察一下类的声明，我们可以看到他们继承的类也是不同的，Hashtable继承的Dictionary,**Dictionary这个类从注释上写着已经是obsolete被废弃了，所以连带Hashtable也基本不用了**。Hashtable也有元素个数，数组大小，负载因子这些属性，不用元素个数用的是count不是size。也是使用链表法来解决冲突。 

```java
public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
```

构造函数可以看出**默认大小是****11，同时初始大小给定多少初始数组就多大，不会做扩展到2的指数次幂这样的操作。**threshold=initialCapacity*loadFactor这点和HashMap相同。

```java
    public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }

    public Hashtable() {
        this(11, 0.75f);
    }
```

contains这个方法是**从表尾开始向前搜索的，同时也没有使用==来比较**

```java
    public synchronized boolean contains(Object value) {
        if (value == null) {
            throw new NullPointerException();
        }

        Entry<?,?> tab[] = table;
        for (int i = tab.length ; i-- > 0 ;) {
            for (Entry<?,?> e = tab[i] ; e != null ; e = e.next) {
                if (e.value.equals(value)) {
                    return true;
                }
            }
        }
        return false;
    }
```

从containsKey可以看出，**Hashtable的index计算逻辑是使用key.hashCode()的后31位然后除以tab.length****取余数**。HashMap的那种按位与的操作仅当操作数低位全是1时才等价为取余操作，也就是2的指数次幂-1才可成立，这样做计算速度比除法快很多，不过冲突数量会增加，所以加入了一些打散的设计比如hashCode高位与低位异或。

```java
    public synchronized boolean containsKey(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return true;
            }
        }
        return false;
    }
```

 扩展方法rehash的**扩大方式是旧数组大小\*2+1**，而HashMap是*2，要重新计算每一个的index所以效率低，同时冲突时将**后面的元素插入到前面元素的前一位**，所以会改变顺序 

```java
    protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;//新大小=旧大小*2+1
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];//创建一个新的数组

        modCount++;
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;//重新计算每一个元素的index
                e.next = (Entry<K,V>)newMap[index];//前后元素有冲突时，后面的元素插入到前面元素的前面
                newMap[index] = e;
            }
        }
    }
```

对于插入结点同样要先检查是否存在key值相同的点，存在则不插入，然后检查是否需要扩展数组，插入时如果发生冲突，也是将要**插入的元素放在链表的首位**，而putVal方法是放入尾部的。同时，可以看到Hashtable是**不支持null作为key或value值的**

```java
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {//value为null直接报错
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();//若key为null这里会报错
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
    private void addEntry(int hash, K key, V value, int index) {
        modCount++;

        Entry<?,?> tab[] = table;
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
    }
```

Hashtable的**hashcode方法计算所有entry的hash值总和**

```java
    public synchronized int hashCode() {
        int h = 0;
        if (count == 0 || loadFactor < 0)
            return h;  // Returns zero

        loadFactor = -loadFactor;  // Mark hashCode computation in progress
        Entry<?,?>[] tab = table;
        for (Entry<?,?> entry : tab) {
            while (entry != null) {
                h += entry.hashCode();
                entry = entry.next;
            }
        }

        loadFactor = -loadFactor;  // Mark hashCode computation complete

        return h;
    }
```

elements这个方法是Hashtable多出来的，**返回所有value值的枚举**

```java
    public synchronized Enumeration<V> elements() {
        return this.<V>getEnumeration(VALUES);
    }
```

我们可以注意到，Hashtable的**方法都加上了synchronized**，他们是线程安全的，但是对于本身是线程安全的情况就会大幅度影响性能，JDK8开始引入modCount来作为fast-fail机制，防止其他线程的非synchronzied方法对Hashtable进行修改。