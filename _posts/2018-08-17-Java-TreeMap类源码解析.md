---
layout:     post
title:      Java TreeMap类源码解析
subtitle:   分析Map集合TreeMap的实现原理
date:       2018-08-17
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 集合
---

TreeMap实现的是**基于红黑树的有序键值对集合**，底层完全是树状链表不含有数组，key不能为null，value可以为null。本身含有comparator，若comparator不为null则所有关于key的比较都是通过comparator完成，否则直接根据key本身的class实现来比较，若此时key不是可比较类则会抛出错误。遍历的顺序是中序遍历，也就是说key是从小到大排列的。所有涉及遍历的操作都是fast-fail机制，这个在我集合解析系列中提过多次了，只有在put或remove操作中新增、删除结点造成树结构变更时会增加modCount值，本身是非线性安全类所有方法都没有synchronized修饰。

因为红黑树的插入和删除时维持红黑树性质的操作在TreeNode中分析过一次了，这里就不再重复介绍了（摸了），详细了解移步：[Java HashMap类源码解析(续)-TreeNode](https://yq.aliyun.com/articles/625213?spm=a2c4e.11155435.0.0.11763312Fy8hYe)

------

### 定义与构造函数部分

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```

	首先我们可以看到他继承的是AbstractMap<K,V>这个和HashMap相同，但是实现的第一个接口不一样，NavigableMap<K,V>是一个有序排列的键值对接口，不能想到TreeMap是通过自身二叉搜索树的性质来维持有序排列。

```java
    private final Comparator<? super K> comparator;

    private transient Entry<K,V> root;

    private transient int size = 0;

    private transient int modCount = 0;
```

从内部属性中我们可以看出，增加了用于保持树排序的comparator，如果使用key值的自然排序则comparator为null。增加了根结点root。

```java
    public TreeMap() {
        comparator = null;
    }

    public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }

    public TreeMap(Map<? extends K, ? extends V> m) {
        comparator = null;
        putAll(m);
    }

    public TreeMap(SortedMap<K, ? extends V> m) {
        comparator = m.comparator();
        try {
            buildFromSorted(m.size(), m.entrySet().iterator(), null, null);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
    }
```

从构造函数可以看出，如果不在参数中给定comparator，那么TreeMap会默认使用key值的自然排序。使用Map和使用SortedMap的复制构造是不同的，前者调用putAll方法根据m中是否有相同的comparator来进行不同的构造，后者会使用m中包含的comparator。这里调用了根据排序好的数据构造树的方法buildFromSorted，其中又调用了computeRedLevel来计算红结点的深度，我们可以看到在实际的递归构造过程中，redLevel是不变的，也就是说只有在一个深度上回出现红色

```java
    public void putAll(Map<? extends K, ? extends V> map) {
        int mapSize = map.size();
        if (size==0 && mapSize!=0 && map instanceof SortedMap) {//map是SortedMap
            Comparator<?> c = ((SortedMap<?,?>)map).comparator();
            if (c == comparator || (c != null && c.equals(comparator))) {//两者的comparator相等
                ++modCount;
                try {
                    buildFromSorted(mapSize, map.entrySet().iterator(),
                                    null, null);//和使用SortedMap构造时相同的方法
                } catch (java.io.IOException cannotHappen) {
                } catch (ClassNotFoundException cannotHappen) {
                }
                return;
            }
        }
        super.putAll(map);
    }

	/**
     * 根据排序好的数据在线性时间构建树。可以从迭代器或者流中接受key value值。这导致了更多的参数，不过看起来比参数可选要好一些，可以从以下4类对象接收数据
     *
     *    1) An iterator of Map.Entries.  (it != null, defaultVal == null).
     *    2) An iterator of keys.         (it != null, defaultVal != null).
     *    3) A stream of alternating serialized keys and values.
     *                                   (it == null, defaultVal == null).
     *    4) A stream of serialized keys. (it == null, defaultVal != null).
     *
     * 假设了在这个方法调用之前已经实现了比较器
     *
     * @param size 数据个数
     * @param it 迭代器
     * @param str 流，it和str至少一个应该是非null的，优先从it读取
     * @param defaultVal 默认值非null时，map中的所有value都是该值，为null则从迭代器或流中读取value值
     * @throws java.io.IOException 流读取时抛出，str为null时不会有这个错误
     * @throws ClassNotFoundException 流readObject时抛出，str为null时不会有这个错误
     */
    private void buildFromSorted(int size, Iterator<?> it,
                                 java.io.ObjectInputStream str,
                                 V defaultVal)
        throws  java.io.IOException, ClassNotFoundException {
        this.size = size;
        root = buildFromSorted(0, 0, size-1, computeRedLevel(size),
                               it, str, defaultVal);
    }

    /**
     * 寻找在某个level以下所有结点都是黑色的，剩下的结点是红色的。计算通过寻找到达0结点的分裂次数
     */
    private static int computeRedLevel(int sz) {
        int level = 0;
        for (int m = sz - 1; m >= 0; m = m / 2 - 1)
            level++;
        return level;//计算lg(size)
    }

    /**
     * 上面那个方法的实际工作方法，假设了comparator和size已经提前设置好了
     *
     * @param level 树的level初始为0
     * @param lo 子树第一个点的下标，初始为0
     * @param hi 子树最后一个点的下标，初始为size-1
     * @param redLevel 红结点的level应该与computeRedLevel(size)相等
     */
    @SuppressWarnings("unchecked")
    private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
                                             int redLevel,
                                             Iterator<?> it,
                                             java.io.ObjectInputStream str,
                                             V defaultVal)
        throws  java.io.IOException, ClassNotFoundException {
        /*
         * 根是最中间的点，我们首先递归的构建整个左子树，然后处理右子树
         *
         * lo和hi是当前子树从迭代器或者流中拉取出的最小和最大的下标。他们不是完全索引，我们只是用来确保处理是有序的
         */

        if (hi < lo) return null;

        int mid = (lo + hi) >>> 1;//mid=(lo+hi)/2

        Entry<K,V> left  = null;
        if (lo < mid)
            left = buildFromSorted(level+1, lo, mid - 1, redLevel,
                                   it, str, defaultVal);//递归构建左子树

        //lo==hi时开始进入此段，从迭代器或流中读取，构造一个结点
        K key;
        V value;
        if (it != null) {//迭代器不为null
            if (defaultVal==null) {//defaultVal不为null则value=defaultVal，否则value从迭代器中读取
                Map.Entry<?,?> entry = (Map.Entry<?,?>)it.next();
                key = (K)entry.getKey();
                value = (V)entry.getValue();
            } else {
                key = (K)it.next();
                value = defaultVal;
            }
        } else { //迭代器为null，流不为null
            key = (K) str.readObject();
            value = (defaultVal != null ? defaultVal : (V) str.readObject());//defaultVal不为null则value=defaultVal，否则value从流中读取
        }

        Entry<K,V> middle =  new Entry<>(key, value, null);//根据key value构造一个新结点

        // 将非满的最底层结点染红，递归过程中redLevel没变也就是说只有一个level会出现红色
        if (level == redLevel)
            middle.color = RED;

        if (left != null) {
            middle.left = left;
            left.parent = middle;
        }

        if (mid < hi) {
            Entry<K,V> right = buildFromSorted(level+1, mid+1, hi, redLevel,
                                               it, str, defaultVal);//递归构建右子树
            middle.right = right;
            right.parent = middle;
        }

        return middle;
    }
```

------

### 查询函数部分

首先两个conatains方法，containsKey根据是否有comparator决定是否调用getEntryUsingComparator，两个判断比较除了比较器不同外都是一样的。在没有comparator时，key为null时会抛NullPointerException，key是不可比较类时会抛ClassCastException

```java
    public boolean containsKey(Object key) {
        return getEntry(key) != null;
    }

    final Entry<K,V> getEntry(Object key) {
        // Offload comparator-based version for sake of performance
        if (comparator != null)
            return getEntryUsingComparator(key);//有comparator时调用getEntryUsingComparator来寻找key值对应的entry
        if (key == null)
            throw new NullPointerException();//key为null时抛错，说明TreeMap也不支持null为key
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;//没有comparator时直接根据key本身来比较，key是不可比较类的话会抛错
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = k.compareTo(p.key);//根据二叉搜索树的性质寻找key值compareTo相等的结点
            if (cmp < 0)
                p = p.left;
            else if (cmp > 0)
                p = p.right;
            else
                return p;
        }
        return null;
    }

    final Entry<K,V> getEntryUsingComparator(Object key) {
        @SuppressWarnings("unchecked")
            K k = (K) key;
        Comparator<? super K> cpr = comparator;
        if (cpr != null) {
            Entry<K,V> p = root;
            while (p != null) {
                int cmp = cpr.compare(k, p.key);
                if (cmp < 0)
                    p = p.left;
                else if (cmp > 0)
                    p = p.right;
                else
                    return p;
            }
        }
        return null;
    }
```

containsValue这个方法的遍历顺序是根据树的中序遍历进行的，getFirstEntry从根结点开始寻找最左的叶结点，successor返回t的后继，也就是在剩余结点中key比t.key大的最小结点，valEquals比较o1==null o2==null或者o1.equals(o2)

```java
    public boolean containsValue(Object value) {
        for (Entry<K,V> e = getFirstEntry(); e != null; e = successor(e))
            if (valEquals(value, e.value))
                return true;
        return false;
    }
    
    final Entry<K,V> getFirstEntry() {
        Entry<K,V> p = root;
        if (p != null)
            while (p.left != null)
                p = p.left;
        return p;
    }

    static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
        if (t == null)
            return null;
        else if (t.right != null) {
            Entry<K,V> p = t.right;//t的右子树不为null时，是右子树最左的叶结点，也就是右子树中key最小的结点
            while (p.left != null)
                p = p.left;
            return p;
        } else {
            Entry<K,V> p = t.parent;//向上寻找父结点中恰好大于t的那个
            Entry<K,V> ch = t;
            while (p != null && ch == p.right) {
                ch = p;
                p = p.parent;
            }
            return p;
        }
    }

    static final boolean valEquals(Object o1, Object o2) {
        return (o1==null ? o2==null : o1.equals(o2));
    }
```

get方法主要就是调用了getEntry，这个前面已经讲过了

```java
    public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }
```

getCeilingEntry获取key值对应的键值对，如果没有则返回key大于他的键值对里key最小的那个，还没有就返回null，从compare可以看出这里key的比较优先根据comparator，若不存在则直接比较key

```java
    final Entry<K,V> getCeilingEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            if (cmp < 0) {//key<p.key
                if (p.left != null)
                    p = p.left;//从左子树中寻找key值小于等于key的结点
                else
                    return p;//已经找到子树中最左的叶结点
            } else if (cmp > 0) {//key>p.key
                if (p.right != null) {
                    p = p.right;//从右子树中寻找key值大于等于key的结点
                } else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    while (parent != null && ch == parent.right) {//向上寻找直到当前结点为右儿子，此时父结点是恰大于key的最小结点
                        ch = parent;
                        parent = parent.parent;
                    }
                    return parent;
                }
            } else
                return p;
        }
        return null;
    }

    final int compare(Object k1, Object k2) {
        return comparator==null ? ((Comparable<? super K>)k1).compareTo((K)k2)
            : comparator.compare((K)k1, (K)k2);//comparator为null只用key直接比较若key不能比较则抛错，否则使用comparator比较
    }
```

下面3个方法和上面的那个非常类似：getFloorEntry返回key值对应的键值对，如果没有就返回小于的结点里key值最大的那个，再没有就返回null；getHigherEntry返回key大于参数key的结点中最小的，没有则返回null，相比getFloorEntry只是少了等于的判断，代码略；getLowerEntry返回key小于参数key的结点中最大的一个，没有则返回null

```java
    final Entry<K,V> getFloorEntry(K key) {
        Entry<K,V> p = root;
        while (p != null) {
            int cmp = compare(key, p.key);
            if (cmp > 0) {//key>p.key
                if (p.right != null)
                    p = p.right;//从右子树中寻找对应的结点
                else
                    return p;//找到了左子树中最右的叶结点
            } else if (cmp < 0) {//key<p.key
                if (p.left != null) {
                    p = p.left;//从左子树中寻找对应的结点
                } else {
                    Entry<K,V> parent = p.parent;
                    Entry<K,V> ch = p;
                    while (parent != null && ch == parent.left) {//p没有左子树时，向上寻找当前结点是左儿子的点，为key恰好小于p中最大的
                        ch = parent;
                        parent = parent.parent;
                    }
                    return parent;
                }
            } else
                return p;

        }
        return null;
    }
```

下面几个方法返回的都是Map.Entry区别在于poll会移除返回的结点，因为删除这个操作涉及到红黑树的修改，所以后面再分析

```java
    public Map.Entry<K,V> firstEntry() {
        return exportEntry(getFirstEntry());//根据最小的键值对新生成一个Map.Entry
    }
    
    public Map.Entry<K,V> lastEntry() {
        return exportEntry(getLastEntry());//根据最大的键值对新生成一个Map.Entry
    }

    public Map.Entry<K,V> pollFirstEntry() {
        Entry<K,V> p = getFirstEntry();//获取最小的键值对
        Map.Entry<K,V> result = exportEntry(p);//新生成一个Map.Entry
        if (p != null)
            deleteEntry(p);//存在时，删除键值对
        return result;
    }

    public Map.Entry<K,V> pollLastEntry() {
        Entry<K,V> p = getLastEntry();//获取最大的键值对
        Map.Entry<K,V> result = exportEntry(p);//新生成一个Map.Entry
        if (p != null)
            deleteEntry(p);//存在时，删除键值对
        return result;
    }

    static <K,V> Map.Entry<K,V> exportEntry(TreeMap.Entry<K,V> e) {
        return (e == null) ? null :
            new AbstractMap.SimpleImmutableEntry<>(e);
    }
```

------

### 非结构性修改函数

replace根据是否给出oldValue决定是否在修改时要对比value值是否相等

```java
    public boolean replace(K key, V oldValue, V newValue) {
        Entry<K,V> p = getEntry(key);
        if (p!=null && Objects.equals(oldValue, p.value)) {
            p.value = newValue;
            return true;
        }
        return false;
    }

    public V replace(K key, V value) {
        Entry<K,V> p = getEntry(key);
        if (p!=null) {
            V oldValue = p.value;
            p.value = value;
            return oldValue;
        }
        return null;
    }
```

对于遍历操作，可以看到他们是按照中序遍历，所以顺序是按照key从小到大进行的，遍历为fast-fail机制

```java
    public void forEach(BiConsumer<? super K, ? super V> action) {
        Objects.requireNonNull(action);
        int expectedModCount = modCount;
        for (Entry<K, V> e = getFirstEntry(); e != null; e = successor(e)) {
            action.accept(e.key, e.value);

            if (expectedModCount != modCount) {
                throw new ConcurrentModificationException();
            }
        }
    }

    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        Objects.requireNonNull(function);
        int expectedModCount = modCount;

        for (Entry<K, V> e = getFirstEntry(); e != null; e = successor(e)) {
            e.value = function.apply(e.key, e.value);

            if (expectedModCount != modCount) {
                throw new ConcurrentModificationException();
            }
        }
    }
```

TreeMap可以支持返回一些subMap，这些Map可以是升序或者降序的，他们全部都是基于NavigableSubMap<K,V>的继承类，添加了一些指针和迭代器以及分割器，**还是基于原本的树链表只是移动方向不同并不会复制数据**，所以所有对于subMap的修改操作都会直接作用到原本的TreeMap上面