---
layout:     post
title:      Java HashSet LinkedHashSet TreeSet类源码解析
subtitle:   分析Set集合HashSet LinkedHashSet TreeSet的实现原理
date:       2018-08-19
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

Set集合中不含有重复的元素，插入重复的元素会失败。常用的有HashSet LinkedHashSet TreeSet。HashSet是无序的集合，LinkedHashSet中的排序和插入成功的顺序一致重复插入，TreeSet中元素是有序排列的，排序的依据是自身的comparator如果为null则根据key从小到大排序。HashSet和LinkedHashSet都是支持插入null的，TreeSet是否支持视comparator的情况，如果comparator不支持比较null的大小或者没有自定义的comparator则不支持插入null

------

## HashSet

HashSet是基于HashMap实现的Set集合，所以**支持插入null，但是不保持任何顺序**。add成功时返回true，add已有元素时返回false。remove已有元素成功时返回true，remove不存在的元素时失败返回false。和HashMap一样是非线程安全类

```java
	public static void main(String args[]){
		Set<String> set = new HashSet<>();
		System.out.println(set.add("123"));//true
		System.out.println(set.add("123"));//false
		System.out.println(set.add(null));//true
		System.out.println(set.remove(null));//true
		System.out.println(set.remove(null));//false
	}
```

------

### 定义与构造函数

首先从定义上来看，HashSet实现的是Set接口，Set是一种没有重复元素的集合，比如HashMap的KeySet就是一种Set集合

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
```

从属性中可以看出，他是基于HashMap来实现的，PRESENT是Map跟底层Map相关联的虚拟值，实际上所有对HashSet的操作底层都可以理解为value=PRESENT的对HashMap的操作，后面几个方法中会用到

```java
    private transient HashMap<E,Object> map;

    private static final Object PRESENT = new Object();
```

然后来看构造函数，无参构造默认构建一个大小为16，负载因子为0.75的HashMap；复制构造时取c/0.75+1和16中的较大值作为大小，负载因子0.75；可以单独设定初始大小，负载因子取默认的0.75，也可以同时指定负载因子；最后一种HashSet(int initialCapacity, float loadFactor, boolean dummy)是内部方法只有LinkedHashSet使用，构建一个链表式HashSet，底层是LinkedHashMap。

```java
    public HashSet() {
        map = new HashMap<>();
    }

    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

------

### 对集合的操作函数

上面提到了，HashSet底层是基于HashMap的KeySet，所有value都是PRESENT，所以只需要增加参数后复用HashMap的操作即可完成大部分对HashSet的操作

```java
	//返回迭代器
	public Iterator<E> iterator() {
        return map.keySet().iterator();
    }
	//返回元素个数
    public int size() {
        return map.size();
    }
	//返回集合是否为空
    public boolean isEmpty() {
        return map.isEmpty();
    }
	//返回Set中是否含有某个对象，底层通过HashMap.containKey实现
    public boolean contains(Object o) {
        return map.containsKey(o);
    }
	//添加元素e，对于底层来说相等于添加键值对(e,PRESENT)，因为map.put会返回旧元素值，而set中不含有重复值，所以若直接map中不存在e则返回null此时函数返回true；而已有e时返回的是e，函数返回值为false
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
	//移除某个对象，map.remove方法移除了已有的键值对返回的是value的值，对于Set来说也就是PRESENT，所以返回PRESENT相当于移除成功返回true
    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
	//直接清除map中的所有对象
    public void clear() {
        map.clear();
    }
```

clone方法就是先复制一个空的HashSet，然后通过map.clone复制map，但是**元素本身没有复制，也就是说引用是相同的对象**

```java
    public Object clone() {
        try {
            HashSet<E> newSet = (HashSet<E>) super.clone();
            newSet.map = (HashMap<E, Object>) map.clone();
            return newSet;
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);//因为HashSet和HashMap都实现了clone接口，这个错误理论上不会出现
        }
    }
```

spliterator基于HashMap.KeySpliterator生成一个初始大小为空的分裂器

```java
    public Spliterator<E> spliterator() {
        return new HashMap.KeySpliterator<E,Object>(map, 0, -1, 0, 0);
    }
```

------

### 序列化函数

writeObject基本上就是HashMap的基础上去掉了value值的写入。readObject先读取基本参数，并调整大小保证集合至少有0.25的负载率，最后新建底层的Map并读取元素插入集合。

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // 写入隐藏的序列化对象
        s.defaultWriteObject();

        // 写入HashMap的大小和负载因子
        s.writeInt(map.capacity());
        s.writeFloat(map.loadFactor());

        // 写入元素个数
        s.writeInt(map.size());

        // 依次写入所有元素
        for (E e : map.keySet())
            s.writeObject(e);
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // 读取隐藏的序列化对象
        s.defaultReadObject();

        // 读取大小并检验不是负值
        int capacity = s.readInt();
        if (capacity < 0) {
            throw new InvalidObjectException("Illegal capacity: " +
                                             capacity);
        }

        // 读取负载因子，并检验是正数且不是NaN
        float loadFactor = s.readFloat();
        if (loadFactor <= 0 || Float.isNaN(loadFactor)) {
            throw new InvalidObjectException("Illegal load factor: " +
                                             loadFactor);
        }

        // 读取元素个数并检验是非负数
        int size = s.readInt();
        if (size < 0) {
            throw new InvalidObjectException("Illegal size: " +
                                             size);
        }

        // 重设大小，HashMap的大小需要至少有25%的负载率
        capacity = (int) Math.min(size * Math.min(1 / loadFactor, 4.0f),
                HashMap.MAXIMUM_CAPACITY);

        // 创建底层HashMap，若本身是LinkedHashSet则创建LinkedHashSet
        map = (((HashSet<?>)this) instanceof LinkedHashSet ?
               new LinkedHashMap<E,Object>(capacity, loadFactor) :
               new HashMap<E,Object>(capacity, loadFactor));

        // 从流中顺序读取所有元素，并以map.put(e, PRESENT)的方式加入到map中
        for (int i=0; i<size; i++) {
            @SuppressWarnings("unchecked")
                E e = (E) s.readObject();
            map.put(e, PRESENT);
        }
    }
```

## LinkedHashSet

LinkedHashSet是基于LinkedHashMap实现的，所以也**支持插入null并且可以保持元素的顺序一定和插入顺序是一致的，同时插入同一个元素而失败时不会影响排序**。整个集合完全是基于其他类实现的，所以代码非常少。先看一下关键的构造函数部分，基本上如前面HashSet中提到，这个true的参数除了鉴别重载类型外没有实际意义，而LinkedHashMap在新建时没有给出accessOrder参数所以默认是插入顺序。复制构造这里有些不同，初始大小是c的元素个数2倍和11中的较大值，而不是一般的16

```java
    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    public LinkedHashSet() {
        super(16, .75f, true);
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```

分裂器是重写过的，Spliterator.DISTINCT | Spliterator.ORDERED代表这个分裂器的集合是不重复的且有序的

```java
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
    }
```

## TreeSet

TreeSet是一个排序Set，并且他在没有重写过支持null的comparator的情况下是不支持插入null的

```java
	public static void main(String args[]){
		Set<String> set = new TreeSet<>();
		System.out.println(set.add("123"));//true
		System.out.println(set.add("123"));//false
		//System.out.println(set.add(null));报错
		//System.out.println(set.remove(null));报错
		System.out.println(set.remove("123"));//true
		System.out.println(set.remove("123"));//false
	}
```

### 定义与构造函数

首先我们可以看到TreeSet实现的接口就和HashSet不同，是NavigableSet<E>说明这是一个有序排列的Set，底层基于TreeMap实现的NavigableMap接口，同时也是使用一个内存中共享的PRESENT作为虚假的value值。

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
    
    private transient NavigableMap<E,Object> m;

    private static final Object PRESENT = new Object();
```

之前在TreeMap解析中提到过比较器的问题，如果不在构造时给出新的构造器，则默认根据key的大小来排序，此时若key是不能比较类或抛出错误。构造函数可以看出除非在构造时直接给出comparator或者根据SortedSet进行复制构造，否则都是按key自身来进行排序

```java
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }    

	public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
```

------

### 集合操作函数

TreeSet除了可以返回升序的迭代器外，还可以返回降序的迭代器和降序的Set，在TreeMap中分析过，这里的遍历是对红黑树的中序遍历，而降序迭代器只是改变了指针移动的顺序，**对于descendingSet来说因为TreeMap.descendingMap()是返回的自身而新建的TreeSet中只是将这个返回结果设为m，所以对descendingSet的操作会直接反映到原本的TreeSet上面**。

```java
    public Iterator<E> iterator() {
        return m.navigableKeySet().iterator();
    }

    public Iterator<E> descendingIterator() {
        return m.descendingKeySet().iterator();
    }

    public NavigableSet<E> descendingSet() {
        return new TreeSet<>(m.descendingMap());
    }
```

add和remove操作和HashSet的设计一样以PRESENT作为虚假的value值进行操作，由于TreeMap在使用key默认排序时不支持以null作为key，所以对于没有设置comparator时插入null会抛出java.lang.NullPointerException错误

```java
    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }

    public boolean remove(Object o) {
        return m.remove(o)==PRESENT;
    }
```

然后是addAll方法，在自身的底层集合为空，c不为空，c是SortedMap，自身的底层集合是TreeMap时可以通过addAllForTreeSet调用buildFromSorted来完成线性时间的构建，否则只能一个个插入树中，一次插入时间复杂度为O(lgn)，n是size大小

```java
    public  boolean addAll(Collection<? extends E> c) {
        // Use linear-time version if applicable
        if (m.size()==0 && c.size() > 0 &&
            c instanceof SortedSet &&
            m instanceof TreeMap) {//自身的底层集合为空，c不为空，c是SortedMap，自身的底层集合是TreeMap
            SortedSet<? extends E> set = (SortedSet<? extends E>) c;
            TreeMap<E,Object> map = (TreeMap<E, Object>) m;
            Comparator<?> cc = set.comparator();
            Comparator<? super E> mc = map.comparator();
            if (cc==mc || (cc != null && cc.equals(mc))) {//双方的comparator相等
                map.addAllForTreeSet(set, PRESENT);
                return true;
            }
        }
        return super.addAll(c);
    }
	//仅供TreeSet使用
    void addAllForTreeSet(SortedSet<? extends K> set, V defaultVal) {
        try {
            buildFromSorted(set.size(), set.iterator(), null, defaultVal);
        } catch (java.io.IOException cannotHappen) {
        } catch (ClassNotFoundException cannotHappen) {
        }
    }
```

下边几个子集函数，参数中int的XXElement都是下标类参数，Inclusive代表这个下标元素本身是否包含在截取范围内，因为TreeMap中所有子集操作都是基于自身而没有复制元素，所以**对于以下这些子集的修改都会反映到TreeSet上面**

```java
    //截取fromElement到toElement间，Inclusive为true时为下标自身包含在范围内
	public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                                  E toElement,   boolean toInclusive) {
        return new TreeSet<>(m.subMap(fromElement, fromInclusive,
                                       toElement,   toInclusive));//fromElement、toElement是截取的下标范围，Inclusive为true时本身也包括在内
    }
	//截取小于（inclusive为true时为小于等于）toElement的子集
    public NavigableSet<E> headSet(E toElement, boolean inclusive) {
        return new TreeSet<>(m.headMap(toElement, inclusive));
    }
	//截取大于（inclusive为true时为小于等于）fromElement的子集
    public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
        return new TreeSet<>(m.tailMap(fromElement, inclusive));
    }
	//截取包含从fromElement开始到不含toElement结束的子集
    public SortedSet<E> subSet(E fromElement, E toElement) {
        return subSet(fromElement, true, toElement, false);
    }
	//截取小于toElement的子集
    public SortedSet<E> headSet(E toElement) {
        return headSet(toElement, false);
    }
	//截取大于等于fromElement的子集
    public SortedSet<E> tailSet(E fromElement) {
        return tailSet(fromElement, true);
    }
```

下面几个都是查询方法，直接通过TreeMap对应的函数来寻找指定的值

```java
    public E first() {
        return m.firstKey();//返回最小的元素
    }

    public E last() {
        return m.lastKey();//返回最大的元素
    }

    public E lower(E e) {
        return m.lowerKey(e);//返回大于e的元素中最小的
    }

    public E floor(E e) {
        return m.floorKey(e);//返回小于e的元素中最大的
    }

    public E ceiling(E e) {
        return m.ceilingKey(e);//返回大于等于e的元素中最小的
    }

    public E higher(E e) {
        return m.higherKey(e);//返回小于等于e的元素中最大的
    }
```

poll在查询的基础上如果查找成功会删除对应的元素，只能从最小和最大的开始删除

```java
    public E pollFirst() {
        Map.Entry<E,?> e = m.pollFirstEntry();//删除最小的结点
        return (e == null) ? null : e.getKey();
    }

    public E pollLast() {
        Map.Entry<E,?> e = m.pollLastEntry();//删除最大的结点
        return (e == null) ? null : e.getKey();
    }
```

TreeMap的clone方法复制了底层的集合但没有复制元素本身，也就是说对树的操作相互之间不影响，但是对相同元素的操作是相互影响的。比如说Set中的元素是一个bean，将原TreeSet中第一个元素的某个属性修改了，clone出的TreeSet中的第一个元素也会改变，这点前面两个Set类也是同样的。

```java
    public Object clone() {
        TreeSet<E> clone;
        try {
            clone = (TreeSet<E>) super.clone();
        } catch (CloneNotSupportedException e) {
            throw new InternalError(e);
        }

        clone.m = new TreeMap<>(m);//复制出一个新的TreeMap
        return clone;
    }
```

------

### 序列化

writeObject比较容易看懂，就是依次写入比较器、元素个数和每个元素，元素的顺序是树的中序遍历顺序

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // 写入任何隐藏的部分
        s.defaultWriteObject();

        // 写入比较器
        s.writeObject(m.comparator());

        // 写入元素个数
        s.writeInt(m.size());

        // 根据迭代器的顺序也就是key从小到大写入所有元素
        for (E e : m.keySet())
            s.writeObject(e);
    }
```

readObject先读取比较器和大小，然后通过TreeMap.readTreeSet从流中读取构建一个TreeMap

```java
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // 读入隐藏部分
        s.defaultReadObject();

        // 读入比较器
        @SuppressWarnings("unchecked")
            Comparator<? super E> c = (Comparator<? super E>) s.readObject();

        // 创建一个TreeMap设为m
        TreeMap<E,Object> tm = new TreeMap<>(c);
        m = tm;

        // 读入元素个数
        int size = s.readInt();

        tm.readTreeSet(size, s, PRESENT);
    }
	//只有TreeSet.readObject调用
    void readTreeSet(int size, java.io.ObjectInputStream s, V defaultVal)
        throws java.io.IOException, ClassNotFoundException {
        buildFromSorted(size, null, s, defaultVal);//从流中读取构建一个TreeMap
    }
```