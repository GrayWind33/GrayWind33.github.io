---
layout:     post
title:      Java HashMap类源码解析
subtitle:   分析Map集合HashMap实现原理
date:       2018-08-11
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 集合
---

　　作为重要的常用集合，HashMap主要是提供键值对的存取，通过key值可以快速找到对应的value值。Hash表是通过提前设定好的规则计算一个元素的hash值来找到他在数组中的存储位置进行快速定位，假设有一个大小为10的数组，可以设定简单的计算规则为元素转为int后mod 10，由此元素的hash值一定会落在大小为10的数组内。由于不同元素可能会计算出相同的hash值，如例子中1和11都应该在下标为1的位置，这就是hash值的冲突。为了解决这个问题有几种常用的策略：

1. 链表法，先加入11存储在A[1]的位置，然后加入1，检查A[1]已经有数了，将1连接到11的后面形成链表。
2. 开放地址法，检查到冲突后根据一定规则去检查另外的位置是否有空可以存储新的元素。根据探测方法的不同，常见的有线性探测法，按照A[1], A[2]…的顺序检查，以及平方探测法，按照A[1], A[1+1^2], A[1+2^2]…
3. 再hash法，按照另外的计算公式重新计算hash值知道不再冲突。

　　由此引入一个hash表的属性——**负载因子**，负载因子=存储的元素个数/数组大小。很显然，链表法由于冲突位置链无限延长的特点，若不加以限制负载因子可以超过1，负载因子越大代表表中的数据越密集。

　　HashMap的key和value值都可以为null，get操作时若找不到对应的key值会返回null，具体见下方的例子：

```java
	public static void main(String args[]){
        Map<String, String> map = new HashMap<>();
        System.out.println(map.put(null, "123"));//null
        System.out.println(map.put("456", null));//null
        System.out.println(map.get("123"));//null
        System.out.println(map.get(null));//123
        System.out.println(map.get("456"));//null
        System.out.println(map.put(null, "345"));//123
        System.out.println(map.get(null));//345
    }
```

　　因为篇幅加难度的原因TreeNode部分的分析见**Java HashMap类源码解析(续)-TreeNode**

　　首先大致翻译下note的主要内容：通常是桶式hash表（链表解决冲突），但是桶过大达到TREEIFY_THRESHOLD值的时候会转为树状TreeNode使得密度过高时的操作可以变得更快。一般对象在树中是按hashcode排序，但是对于实现了Comparable<C>的对象是通过comapreTo来排序。由于TreeNode的大小接近普通node的两倍，当桶变小时会转回线性链表。TreeNode是JDK8引入的红黑树结构，树根通常是hash映射的第一个结点，除了Iterator.remove之外。链表在树化或是分裂时保证结点的遍历顺序是一致的。

```java
	static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            //key和value的hashCode亦或
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                //key == e.getKey()或者key.equals(e.getKey())
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
}
```

　　然后我们来看下Node的结构。一般的箱式结点实现了Map.Entry<K,V>接口，这是一个key-value键值对，内部有4个属性hash值、K、V以及指向下个结点的引用。hashCode方法是对key和value的hash值求异或，也重写了equals和toString方法。

```java
	static final int hash(Object key) {
         int h;
         //key的hashCode无符号右移16位的值与自己进行异或
         return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	}
```

对于HashMap自身的hash方法，这样做的目的是避免hash值高位因为表大小而永远不会被用于hash值的计算，使得分布可以更加均匀。

```java
	static final int tableSizeFor(int cap) {
        int n = cap - 1;//cap=0001 1000 0001 1111(6175) n = 0001 1000 0001 1110
        n |= n >>> 1;//n = 0001 1100 0001 1111
        n |= n >>> 2;//n = 0001 1111 0001 1111
        n |= n >>> 4;//n = 0001 1111 1111 1111
        n |= n >>> 8;//n = 0001 1111 1111 1111
        n |= n >>> 16;//n = 0001 1111 1111 1111(8191)
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

根据cap返回恰好大于等于该值的2的指数大小。以cap = 6175进行演示，可得n最终为8191即2^13-1，所以返回值为2^13

 HashMap内部属性如下：

```java
	//hash表的底层数组，大小永远是2的指数
    transient Node<K,V>[] table;

    //含有键值对的set高速缓存
    transient Set<Map.Entry<K,V>> entrySet;

    //键值对的数量
    transient int size;

    //HashMap被结构性修改的次数，包括改变键值对个数的操作和rehash等改变内部结构的操作，用于迭代器在线程不安全时快速抛错
    transient int modCount;

    //达到某个大小后就要改变数组大小，等于capacity * load factor
    int threshold;

    //负载因子
    final float loadFactor;
```

构造函数：

```java
	//参数缺省值为16和0.75
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        //大小最大不能超过1<<30
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        //计算恰好大于等于initialCapacity的2的指数作为容量大小
        this.threshold = tableSizeFor(initialCapacity);
    }
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

这里通过一个已有Map来构造时用到了putMapEntries这个方法，先来看下这个方法

```java
	final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
        int s = m.size();
        if (s > 0) {
            if (table == null) { //当前没有元素
                float ft = ((float)s / loadFactor) + 1.0F;//计算出m的容量
                int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                         (int)ft : MAXIMUM_CAPACITY);
                if (t > threshold)
                    threshold = tableSizeFor(t);
            }
            else if (s > threshold)
                resize();//若m的元素个数超过了threhold则需要扩展表
            for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
                K key = e.getKey();
                V value = e.getValue();
                putVal(hash(key), key, value, false, evict);//将键值对插入到表中
            }
        }
    }
```

可以看到其中调用了用于扩展表的resize和插入键值对的putVal。先看resize方法，这个方法在表大小超过threhold时就会被调用，作用就是扩展数组大小，并将元素复制到数组中，同时对于冲突链表不需要重新计算hash值而是会根据他们的hash值决定要不要复制到数组的高位去

```java
	final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;//当前表中元素个数
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // 当前表中元素个数大于等于16且小于上限的一半时，threshold加倍
        }
        else if (oldThr > 0) // 这个条件成立时说明构造时给了capacity参数，由此计算出了threhold
            newCap = oldThr;
        else {               //没有任何参数的初始化直接使用默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {//当前表中没有元素的情况
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        //e没有后续链表结点时，因为newCap是oldCap的2倍，相当于掩码多了一位，原本hash值的这个多出来的有效位是0或1会决定它在新数组中下标是否变化
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // 存在非树的链表时，保持先后顺序不变
                        Node<K,V> loHead = null, loTail = null;//低位链表
                        Node<K,V> hiHead = null, hiTail = null;//高位链表
                        Node<K,V> next;
                        do {
                            next = e.next;
                          //这里的运算相当于直接检查hash新增的高位是0还是1，因为oldCap是2的指数所以只有最高位是1其余都是0，旧hash的高位为1时要进行移动
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;//低位链表直接复制到原本所在的位置
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;//高位链表的移动规则是原本的下标+oldCap
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

putVal这个方法是把值存入表中，在多个put类方法中被调用

```java
	/**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value为true表示不改变已有值
     * @param evict if false, the table is in creation mode.为false表示是新建表
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;//当前表的table数组为空时需进行扩展
        if ((p = tab[i = (n - 1) & hash]) == null)//hash值截断到n-1对应的位数进行定位
            tab[i] = newNode(hash, key, value, null);//若该下标位置为空，则直接放入数组
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;//检查表上的根结点的hash与key值是否与新增的结点相等，若相等则将修改根结点的value
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);//已经是树调用树的遍历方法
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);//若在箱式链表中没有找到key相等的结点，则新建结点插入到链表末尾
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);//若增加该结点后，链表上的结点数超过了TREEIFY_THRESHOLD则转为树，该判断仅在遍历到链表末尾时执行
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))//找到了key和hash值相等的结点
                        break;
                    p = e;
                }
            }
            if (e != null) { // 找到了相同的key则修改value值并返回旧的value
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;//新增结点时增加modCount
        if (++size > threshold)
            resize();//大小超过threshold时要扩容
        afterNodeInsertion(evict);//这个方法是用于继承了HashMap的LinkedHashMap，用来移除最早放入的结点，保持插入的顺序，为false时代表是新建表不需要进行这个过程
        return null;
    }
```

treeifyBin将指定hash值对应的位置上的链表替换为树，除非整个表的大小太小时调用resize，(n - 1) & hash等效于hash mod n，只保留hash除以n的余数作为index的值

```java
	final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();//表长度小于MIN_TREEIFY_CAPACITY调用resize
        else if ((e = tab[index = (n - 1) & hash]) != null) {//该hash值对应的index位置上有元素
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);//将链表转换为树
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);//树放入index的位置
        }
    }
```

对于移除操作会调用removeNode，这个方法在多个移除方法中被使用。若移除指定key值成功的结点会返回value值，否则返回null

```java
	public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
    /**
     * 用于移除操作
     *
     * @param hash hash for key
     * @param key the key
     * @param value 仅matchValue为true时需要考虑，其他时间不起效
     * @param matchValue 为true时只移除value相等的
     * @param movable 为false时不移动其他结点
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {//表不为空且hash值对应的index位置存在元素
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))//根结点的key值相等
                node = p;
            else if ((e = p.next) != null) {//根结点key值不相等，存在后续结点
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);//调用树的遍历方法寻找结点
                else {
                    do {
                        if (e.hash == hash &&//为箱式链表时遍历链表寻找key相等的点
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {//matchValue为true还需要验证value是否相等，否则忽略
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);//移除树结点
                else if (node == p)
                    tab[index] = node.next;//为箱式链表根结点时，将第二个结点放到数组上
                else
                    p.next = node.next;//为箱式链表非结点时，修改上下结点间的指针
                ++modCount;//增加modCount
                --size;
                afterNodeRemoval(node);//预留给LinkedHashMap的方法
                return node;
            }
        }
        return null;
    }
```

clear方法不难理解，将表内所有元素设为null，size变为0，增加modCount

```java
	public void clear() {
        Node<K,V>[] tab;
        modCount++;
        if ((tab = table) != null && size > 0) {
            size = 0;
            for (int i = 0; i < tab.length; ++i)
                tab[i] = null;
        }
    }
```

寻找表内有无相等的value，遍历整个链表找到则返回true

```java
	public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {//遍历hash表
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {//遍历链表
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```

key是一个set集合，value是一个collection集合，调用keySet()和values()返回的集合是对HashMap中key和value的直接引用，所以操作会直接反应在HashMap上

```java
	public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new KeySet();
            keySet = ks;
        }
        return ks;
    }

    final class KeySet extends AbstractSet<K> {
        public final int size()                 { return size; }
        public final void clear()               { HashMap.this.clear(); }//调用的是HashMap.clear()，所以整个表会被清空
        public final Iterator<K> iterator()     { return new KeyIterator(); }
        public final boolean contains(Object o) { return containsKey(o); }
        public final boolean remove(Object key) {
            return removeNode(hash(key), key, null, false, true) != null;
        }
        public final Spliterator<K> spliterator() {
            return new KeySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super K> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e.key);
                }
                if (modCount != mc)//遍历迭代器要求不能被其他线程修改表内元素个数而引起modCount变化
                    throw new ConcurrentModificationException();
            }
        }
    }
```

根据上面对putVal的分析，该方法不会改变已有的key值，返回值为旧值或null

```java
     public V putIfAbsent(K key, V value) {
         return putVal(hash(key), key, value, true, true);
     }
```

然后来看一下两个replace方法，区别在于返回值和是否检查value值

```java
	@Override
    public boolean replace(K key, V oldValue, V newValue) {
        Node<K,V> e; V v;
        if ((e = getNode(hash(key), key)) != null &&
            ((v = e.value) == oldValue || (v != null && v.equals(oldValue)))) {//key和value要同时符合条件
            e.value = newValue;
            afterNodeAccess(e);//也是用于LinkedHashMap保持结点插入顺序用的
            return true;
        }
        return false;
    }

    @Override
    public V replace(K key, V value) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) != null) {//仅key符合条件
            V oldValue = e.value;
            e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
        return null;
    }
```

 computeIfAbsent这个方法的作用是若key值在map中已有非null的value值，则直接返回旧value值；若value值为null则根据mappingFunction计算出新的value值并修改map中存在的键值对，返回新value值；若不存在key值则新增一个键值对插入到key的hash值对应的table数组位置链表的头部，并返回新的value值。**注意，putVal方法插入的结点是在链表尾部，而该方法是在链表头部。**

```java
	public V computeIfAbsent(K key,
                             Function<? super K, ? extends V> mappingFunction) {
        if (mappingFunction == null)
            throw new NullPointerException();
        int hash = hash(key);
        Node<K,V>[] tab; Node<K,V> first; int n, i;
        int binCount = 0;
        TreeNode<K,V> t = null;
        Node<K,V> old = null;
        if (size > threshold || (tab = table) == null ||
            (n = tab.length) == 0)
            n = (tab = resize()).length;//table空间不足时扩展数组
        if ((first = tab[i = (n - 1) & hash]) != null) {//hash值对应的下标在table内不为空
            if (first instanceof TreeNode)
                old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);
            else {
                Node<K,V> e = first; K k;
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) {//对箱式链表搜索key相等的结点
                        old = e;
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
            V oldValue;
            if (old != null && (oldValue = old.value) != null) {//找到了key相等的结点且value不为null
                afterNodeAccess(old);//LinkedHashMap方法
                return oldValue;//返回旧value值
            }
        }
        V v = mappingFunction.apply(key);//根据function计算出新的v值
        if (v == null) {
            return null;//新的v值为null则直接返回
        } else if (old != null) {//找到了key相等的结点且value为null，赋予新v值后返回新v值
            old.value = v;
            afterNodeAccess(old);
            return v;
        }
        else if (t != null)
            t.putTreeVal(this, tab, hash, key, v);//树结点处理
        else {
            tab[i] = newNode(hash, key, v, first);//没有找到key相等的结点，新建一个结点并且插入到链表头部
            if (binCount >= TREEIFY_THRESHOLD - 1)
                treeifyBin(tab, hash);//若新增结点后链表长度达到了TREEIFY_THRESHOLD则转为树
        }
        ++modCount;//该部分仅新增结点时执行
        ++size;
        afterNodeInsertion(true);
        return v;
    }
```

然后是两个相近的方法：computeIfPresent存在key相等且value不为null的结点，计算新的value值，新value不为null则覆盖，新value为null则移除原本的结点。compute方法结合了前两者，存在key相等的结点时不考虑旧value值，新value为null则移除，不为null则覆盖value值；不存在key相等的结点时，新value值不为null则新增结点，对箱式链表插入到链表头部，插入后要检车是否需要转为树。

```java
	public V computeIfPresent(K key,
                              BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (remappingFunction == null)
            throw new NullPointerException();
        Node<K,V> e; V oldValue;
        int hash = hash(key);
        if ((e = getNode(hash, key)) != null &&
            (oldValue = e.value) != null) {//存在key相等且value不为null的结点
            V v = remappingFunction.apply(key, oldValue);
            if (v != null) {
                e.value = v;//新value值不为null则修改value值
                afterNodeAccess(e);
                return v;
            }
            else
                removeNode(hash, key, null, false, true);//新value值为null则移除这个结点
        }
        return null;
    }

    @Override
    public V compute(K key,
                     BiFunction<? super K, ? super V, ? extends V> remappingFunction) {
        if (remappingFunction == null)
            throw new NullPointerException();
        int hash = hash(key);
        Node<K,V>[] tab; Node<K,V> first; int n, i;
        int binCount = 0;
        TreeNode<K,V> t = null;
        Node<K,V> old = null;
        if (size > threshold || (tab = table) == null ||
            (n = tab.length) == 0)
            n = (tab = resize()).length;//table空间不足时调用resize
        if ((first = tab[i = (n - 1) & hash]) != null) {
            if (first instanceof TreeNode)
                old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);//树中寻找key相等的结点
            else {
                Node<K,V> e = first; K k;
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) {
                        old = e;//找到了key相等的结点
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
        }
        V oldValue = (old == null) ? null : old.value;
        V v = remappingFunction.apply(key, oldValue);
        if (old != null) {
            if (v != null) {
                old.value = v;//找到了key值相等的结点且新value不为null则旧结点的value设为新值
                afterNodeAccess(old);
            }
            else
                removeNode(hash, key, null, false, true);//找到了key值相等的结点且新value为null则移除旧结点
        }
        else if (v != null) {//没有找到key值相等的结点且新value值不为null
            if (t != null)
                t.putTreeVal(this, tab, hash, key, v);//树中插入新结点
            else {
                tab[i] = newNode(hash, key, v, first);//新建结点插入到链表头部
                if (binCount >= TREEIFY_THRESHOLD - 1)
                    treeifyBin(tab, hash);//新增后链表长度达到TREEIFY_THRESHOLD则转为树
            }
            ++modCount;//仅新增结点时执行
            ++size;
            afterNodeInsertion(true);
        }
        return v;
    }
```

merge这个方法和前面差不多，也是先寻找key值相同的结点，若存在则看该结点value是否为null，不为null根据function和参数中的value以及结点原本的value计算出新的value值，否则直接赋予参数中的value值。若没有找到结点，则按照参数中key和value值新建一个结点插入到树中或者箱式链表的头部。

```java
	public V merge(K key, V value,
                   BiFunction<? super V, ? super V, ? extends V> remappingFunction) {
        if (value == null)
            throw new NullPointerException();
        if (remappingFunction == null)
            throw new NullPointerException();
        int hash = hash(key);
        Node<K,V>[] tab; Node<K,V> first; int n, i;
        int binCount = 0;
        TreeNode<K,V> t = null;
        Node<K,V> old = null;
        if (size > threshold || (tab = table) == null ||
            (n = tab.length) == 0)
            n = (tab = resize()).length;//表空间不足时调用resize
        if ((first = tab[i = (n - 1) & hash]) != null) {
            if (first instanceof TreeNode)
                old = (t = (TreeNode<K,V>)first).getTreeNode(hash, key);//寻找树中key值相等的结点
            else {
                Node<K,V> e = first; K k;
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) {
                        old = e;//寻找箱式链表中key值相等的结点
                        break;
                    }
                    ++binCount;
                } while ((e = e.next) != null);
            }
        }
        if (old != null) {//寻找到key值相等的结点
            V v;
            if (old.value != null)//旧值不为null
                v = remappingFunction.apply(old.value, value);//根据remappingFunction和旧value和参数中的value计算出新的value值
            else
                v = value;//旧值为null则新value=参数中的value
            if (v != null) {
                old.value = v;//计算出的新value值不为null则覆盖寻找到结点的value值
                afterNodeAccess(old);
            }
            else
                removeNode(hash, key, null, false, true);//计算出的新value值为null则移除找到的结点
            return v;
        }
        if (value != null) {//没有寻找到key相等的结点
            if (t != null)
                t.putTreeVal(this, tab, hash, key, value);//树中新增结点
            else {
                tab[i] = newNode(hash, key, value, first);//新建结点插入到链表头部
                if (binCount >= TREEIFY_THRESHOLD - 1)
                    treeifyBin(tab, hash);//新增结点后链表长度达到TREEIFY_THRESHOLD则转为树
            }
            ++modCount;//新增结点后执行
            ++size;
            afterNodeInsertion(true);
        }
        return value;
    }
```

两个批量操作不难理解，同样要保证过程中没有其他线程修改了对象的元素个数

```java
	public void forEach(BiConsumer<? super K, ? super V> action) {
        Node<K,V>[] tab;
        if (action == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next)
                    action.accept(e.key, e.value);//对每个结点执行对应的操作
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();//有其他线程修改了HashMap中的元素个数时抛错
        }
    }
    public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function) {
        Node<K,V>[] tab;
        if (function == null)
            throw new NullPointerException();
        if (size > 0 && (tab = table) != null) {
            int mc = modCount;
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    e.value = function.apply(e.key, e.value);
                }
            }
            if (modCount != mc)
                throw new ConcurrentModificationException();
        }
    }
```

HashMap实现了clone方法，可以产生一个新的完全一样的HashMap

```java
	public Object clone() {
        HashMap<K,V> result;
        try {
            result = (HashMap<K,V>)super.clone();//产生一个复制HashMap
        } catch (CloneNotSupportedException e) {
            // 因为HashMap支持clone方法，应该不会抛出这个错误
            throw new InternalError(e);
        }
        result.reinitialize();//初始化参数值，所有集合数组都设为null
        result.putMapEntries(this, false);//将原本集合中的键值对都插入到新产生的Map中
        return result;
    }
```

capacity这个方法先看table是否为null，不为null直接返回table.length。然后看threshold是否为0，不为0返回threshold否则返回默认容量16

```java
     final int capacity() {
         return (table != null) ? table.length :
             (threshold > 0) ? threshold :
             DEFAULT_INITIAL_CAPACITY;
     }
```

HashMap的序列化方法同样利用的是ObjectOutputStream

```java
	private void writeObject(java.io.ObjectOutputStream s)
        throws IOException {
        int buckets = capacity();
        // 先写入数组大小和集合内元素的个数
        s.defaultWriteObject();
        s.writeInt(buckets);
        s.writeInt(size);
        internalWriteEntries(s);
    }
    // 只有writeObject会调用这个方法
    void internalWriteEntries(java.io.ObjectOutputStream s) throws IOException {
        Node<K,V>[] tab;
        if (size > 0 && (tab = table) != null) {
            for (int i = 0; i < tab.length; ++i) {//遍历整个数组，按照链表顺序写入key和value值
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    s.writeObject(e.key);
                    s.writeObject(e.value);
                }
            }
        }
    }
```

序列化输入是通过ObjectInputStreams，loadFactor一定会限制在0.25-4.0之间，而threshold是根据size/loadFactor + 1.0然后计算出大于等于该值的最小2的指数次幂。

```java
	private void readObject(java.io.ObjectInputStream s)
        throws IOException, ClassNotFoundException {
        // Read in the threshold (ignored), loadfactor, and any hidden stuff
        s.defaultReadObject();
        reinitialize();
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new InvalidObjectException("Illegal load factor: " +
                                             loadFactor);
        s.readInt();                // 读取数组大小
        int mappings = s.readInt(); // 读取元素个数
        if (mappings < 0)
            throw new InvalidObjectException("Illegal mappings count: " +
                                             mappings);
        else if (mappings > 0) { // (if zero, use defaults)
            // loadFactor一定在0.25-4.0之间
            float lf = Math.min(Math.max(0.25f, loadFactor), 4.0f);
            float fc = (float)mappings / lf + 1.0f;
            int cap = ((fc < DEFAULT_INITIAL_CAPACITY) ?
                       DEFAULT_INITIAL_CAPACITY :
                       (fc >= MAXIMUM_CAPACITY) ?
                       MAXIMUM_CAPACITY :
                       tableSizeFor((int)fc));//threshold的值在不超过范围的情况下设定为恰好大于等于size/loadFactor + 1的2的指数
            float ft = (float)cap * lf;
            threshold = ((cap < MAXIMUM_CAPACITY && ft < MAXIMUM_CAPACITY) ?
                         (int)ft : Integer.MAX_VALUE);
            @SuppressWarnings({"rawtypes","unchecked"})
                Node<K,V>[] tab = (Node<K,V>[])new Node[cap];
            table = tab;

            // Read the keys and values, and put the mappings in the HashMap
            for (int i = 0; i < mappings; i++) {
                @SuppressWarnings("unchecked")
                    K key = (K) s.readObject();
                @SuppressWarnings("unchecked")
                    V value = (V) s.readObject();
                putVal(hash(key), key, value, false, false);//从输入流中得去key和value值并通过putVal插入
            }
        }
    }
```

