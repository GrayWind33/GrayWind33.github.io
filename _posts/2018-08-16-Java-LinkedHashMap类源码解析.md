---
layout:     post
title:      Java LinkedHashMap类源码解析
subtitle:   分析Map集合LinkedHashMap的实现原理
date:       2018-08-16
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

LinkedHashMap继承了HashMap，他在HashMap的基础上增加了一个双向链表的结构，链表默认维持key插入的顺序，重复的key值插入不会改变顺序，适用于使用者需要返回一个顺序相同的map对象的情况。还可以生成access-order顺序的版本，按照最近访问顺序来存储，刚被访问的结点处于链表的末尾，适合LRU，put get compute merge都算作一次访问，其中put key值相同的结点也算作一次访问，replace只有在换掉一个键值对的时候才算一次访问，putAll产生的访问顺序取决于原本map的迭代器实现。

在插入键值对时，可以通过对removeEldestEntry重写来实现新键值对插入时自动删除最旧的键值对

拥有HashMap提供的方法，迭代器因为是通过遍历双向链表，所以额外开销与size成正比与capacity无关，因此选择过大的初始大小对于遍历时间的增加没有HashMap严重，后者的遍历时间依赖与capacity。

同样是非线程安全方法，对于LinkedHashMap来说，修改结构的操作除了增加和删除键值对外，还有对于access-order时进行了access导致迭代器顺序改变，主要是get操作，对于插入顺序的来说，仅仅修改一个已有key值的value值不是一个修改结构的操作，但对于访问顺序，put和get已有的key值会改变顺序。迭代器也是fail-fast设计，但是fail-fast只是一个调试功能，一个设计良好的程序不应该出现这个错误

因为HashMap加入了TreeNode，所以现在LinkedHashMap也有这个功能

 以下描述中的链表，若无特别说明都是指LinkedHashMap的双向链表

------

先来看一下基本结构，每个键值对加入了前后指针，集合加入了头尾指针来形成双向链表，accessOrder代表链表是以访问顺序还是插入顺序存储

```Java
static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after;//增加了先后指针来形成双向链表
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }

    /**
     * The head (eldest) of the doubly linked list.头部
     */
    transient LinkedHashMap.Entry<K,V> head;

    /**
     * The tail (youngest) of the doubly linked list.尾部
     */
    transient LinkedHashMap.Entry<K,V> tail;

    //true访问顺序 false插入顺序
    final boolean accessOrder;
```

然后是几个内部方法。linkNodeLast将p连接到链表尾部

```java
    private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;//原本链表为空则p同时为头部
        else {
            p.before = last;
            last.after = p;
        }
    }
```

transferLinks用dst替换src

```java
    private void transferLinks(LinkedHashMap.Entry<K,V> src,
                               LinkedHashMap.Entry<K,V> dst) {
        LinkedHashMap.Entry<K,V> b = dst.before = src.before;
        LinkedHashMap.Entry<K,V> a = dst.after = src.after;
        if (b == null)
            head = dst;
        else
            b.after = dst;
        if (a == null)
            tail = dst;
        else
            a.before = dst;
    }
```

reinitialize在调用HashMap方法的基础上，将head和tail设为null

```java
    void reinitialize() {
        super.reinitialize();
        head = tail = null;
    }
```

newNode生成一个LinkedHashMap结点，next指向e，插入到LinkedHashMap链表末端

```java
    Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);//新建一个键值对，next指向e
        linkNodeLast(p);//p插入到LinkedHashMap链表末端
        return p;
    }
```

replacementNode根据原结点生成一个LinkedHashMap结点替换原结点

```java
    Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
        LinkedHashMap.Entry<K,V> t =
            new LinkedHashMap.Entry<K,V>(q.hash, q.key, q.value, next);//生成一个新的键值对next是给出的next参数
        transferLinks(q, t);//用t替换q
        return t;
    }
```

newTreeNode生成一个TreeNode结点，next指向next，插入到LinkedHashMap链表末端

```java
    TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
        TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);//生成一个TreeNode，next指向参数next
        linkNodeLast(p);//p插入到LinkedHashMap链表末端
        return p;
    }
```

replacementTreeNode根据结点p生成一个新的TreeNode，next设为给定的next，替换原本的p

```java
    TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        LinkedHashMap.Entry<K,V> q = (LinkedHashMap.Entry<K,V>)p;
        TreeNode<K,V> t = new TreeNode<K,V>(q.hash, q.key, q.value, next);
        transferLinks(q, t);//根据结点p生成一个新的TreeNode，next设为给定的next，替换原本的p
        return t;
    }
```

afterNodeRemoval从LinkedHashMap的链上移除结点e

```java
    void afterNodeRemoval(Node<K,V> e) { 
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```

afterNodeInsertion可能移除最旧的结点，需要evict为true同时链表不为空同时removeEldestEntry需要重写

```java
    void afterNodeInsertion(boolean evict) { 
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {//removeEldestEntry需要重写才从发挥作用，否则一定返回false
            K key = first.key;//移除链表头部的结点
            removeNode(hash(key), key, null, false, true);
        }
    }
```

afterNodeAccess在访问过后将结点e移动到链表尾部，需要Map是access-order，若移动成功则增加modCount

```java
    void afterNodeAccess(Node<K,V> e) { 
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {//Map是access-order同时e不是链表的尾部
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)//将结点e从链表中剪下
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;//结点e移动到链表尾部
            ++modCount;//因为有access-order下结点被移动，所以增加modCount
        }
    }
```



构造函数方面，accessOrder默认是false插入顺序，初始大小为16，负载因子为0.75，这里是同HashMap。复制构造也是调用了HashMap.putMapEntries方法

containsValue遍历链表寻找相等的value值，这个操作一定不会造成结构改变

```java
    public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {//检查同样是根据LinkedHashMap提供的链表顺序进行遍历
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
```

 get方法复用HashMap的getNode方法，若找到结点且Map是访问顺序时，要将访问的结点放到链表最后，若没找到则返回null。而getOrDefault仅有的区别是没找到时返回defaultValue

```java
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)//复用HashMap的getNode方法
            return null;
        if (accessOrder)
            afterNodeAccess(e);//access-order时将e放到队尾
        return e.value;
    }

    public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;//复用HashMap的getNode方法，若没有找到对应的结点则返回defaultValue
       if (accessOrder)
           afterNodeAccess(e);//access-order时将e放到队尾
       return e.value;
   }
```

clear方法在HashMap的基础上要把head和tail设为null

```java
    public void clear() {
        super.clear();
        head = tail = null;
    }
```

removeEldestEntry在put和putAll插入键值对时调用，原本是一定返回false的，如果要自动删除最旧的键值对要返回true，需要进行重写。比如下面这个例子，控制size不能超过100

```java
     private static final int MAX_ENTRIES = 100;

     protected boolean removeEldestEntry(Map.Entry eldest) {
        return size() > MAX_ENTRIES;
     }
```

下面两个方法和HashMap相似，返回key的Set和value的Collection还有返回键值对的Set，这个是直接引用，所以对它们的remove之类的修改会直接反馈到LinkedHashMap上

```java
    public Set<K> keySet() {
        Set<K> ks = keySet;
        if (ks == null) {
            ks = new LinkedKeySet();
            keySet = ks;
        }
        return ks;//返回key值的set
    }

    public Collection<V> values() {
        Collection<V> vs = values;
        if (vs == null) {
            vs = new LinkedValues();
            values = vs;
        }
        return vs;//返回一个包含所有value值的Collection
    }

    public Set<Map.Entry<K,V>> entrySet() {
        Set<Map.Entry<K,V>> es;
        return (es = entrySet) == null ? (entrySet = new LinkedEntrySet()) : es;//返回一个含有所有键值对的Set
    }
```

检查HashMap的putVal方法，我们可以看到在找到了相同key值并修改value值时会调用afterNodeAccess，对于access-order会改变结点顺序

```java
            if (e != null) { // 找到了相同的key则修改value值并返回旧的value
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
```