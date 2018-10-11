---
layout:     post
title:      Java ArrayList类源码解析
subtitle:   分析List集合ArrayList实现原理
date:       2018-08-05
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 集合
---

ArrayList是最常用的集合类，底层是由数组实现的

首先可以看到，有两个static final对象数组，也就是被线程间共享的，EMPTY_ELEMENTDATA是非default大小的空集合，原因是要辨别第一次添加元素时应该扩展的大小。

private static final Object[] EMPTY_ELEMENTDATA = {};

private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

transient Object[] elementData;这里是存储实际元素的对象数组，transient关键字表示该部分内容不能被序列化，ArrayList实际上序列化调用的是writeObject和readObject方法，而不是直接由elementData进行序列化，原因是elementData里面有空元素，这部分不需要进行序列化，可以节约内存和时间。

private int size;是实际元素的个数

protected transient int modCount = 0;modCount的作用的保证线程安全，ArrayList本身是非同步的集合类，**出现遍历操作都要检查modCount值是否改变。除了自身的set方法外，其他所有存在对elementData写入的操作都会增加modCount值。**

**public** ArrayList(**int** initialCapacity)给定参数为0时elementData直接赋值为EMPTY_ELEMENTDATA，否则新建一个指定大小的数组

**public** ArrayList()直接赋值DEFAULTCAPACITY_EMPTY_ELEMENTDATA

**public** ArrayList(Collection<? **extends** E> c)若集合c不为空，elementData引用c.toArray产生的数组，若产生的不是Object[]需要再调用Arrays.copyOf进行复制。集合为空则赋值为EMPTY_ELEMENTDATA

**public** **void** trimToSize()通过Arrays.copyOf将elementData中的空元素去掉，若size==0则赋值为EMPTY_ELEMENTDATA

**public** **void** ensureCapacity(**int** minCapacity)参数给定所需的最小大小，检查该大小是否大于最小扩展大小（DEFAULTCAPACITY_EMPTY_ELEMENTDATA时该值为10否则为0），大于时调用ensureExplicitCapacity(minCapacity)对集合进行扩展，可以手动调用

**private** **void** ensureCapacityInternal(**int** minCapacity)和上一个相似，add元素时检查大小是否足够，仍未DEFAULTCAPACITY_EMPTY_ELEMENTDATA时扩展10和minCapacity的较大值，否则直接扩展minCapacity

**private** **void** ensureExplicitCapacity(**int** minCapacity) minCapacity大于elementData的元素个数时，调用grow(minCapacity)，无论是否增长了大小，该方法都会增加modCount

**private** **void** grow(**int** minCapacity)增长大小为minCapacity和elementData.length*1.5中的较大值，最大不能超过Integer.max(2的31次)-8，通过Arrays.copyOf复制元素

**public** **int** size()返回size值

**public** **boolean** isEmpty()返回size是否为0

**public** **int** indexOf(Object o)o为null时检查elementData中是否有元素==o，否则检查是否有o.equals(x)，存在则返回**第一个符合**的下标，不存在返回-1

**public** **int** lastIndexOf(Object o)与上一个方法相比区别在于从尾部开始遍历，返回的是下标最大的一个

**public** **boolean** contains(Object o)调用indexOf，返回不为-1时为true

**public** Object clone()新建一个ArrayList，elementData复制过去，其他内部变量也相同

**public** Object[] toArray()返回Arrays.copyOf(elementData, size)，所以不含后面的空数组

**public** <T> T[] toArray(T[] a)若a.length<size，则新建一个包含elementData中实际对象的数组返回。否则将elementData中的元素复制给a并设a[size] = null，返回a

**public** E get(**int** index)获取指定下标元素内容

**public** E set(**int** index, E element)将指定下标元素改为element并返回旧值

**public** **boolean** add(E e)检查数组大小是否需要扩展后，添加元素，增加size，一定会增加modCount

**public** **void** add(**int** index, E element)先检查index是否符合范围，然后检查数组大小是否需要扩展，将index后的元素向后复制以为，然后elementData[index]=element，一定会增加modCount

**public** E remove(**int** index) 先检查index是否符合范围，然后modCount增加，index后的元素往前复制一位，elementData[size]的值为null便于触发gc，size减一，返回旧元素的值

**public** **boolean** remove(Object o)首先按照indexOf的逻辑寻找第一个相等的元素，如果存在则将后面的元素往复制一位最后一个元素设为null并增加modCount返回true，否则返回false。因为index肯定是符合返回的，所以调用不检查index是否符合范围**private** **void** fastRemove(**int** index)

**public** **void** clear()modCount增加，将size个元素都设为null，size改为0

**public** **boolean** addAll(Collection<? **extends** E> c)通过System.arraycopy将c的元素复制到集合的后面，modCount增加，size增加c的元素个数，若新size不为0则返回true

**public** **boolean** addAll(**int** index, Collection<? **extends** E> c)和上面相比区别在于要将c复制到从index开始的位置而不是末尾，同样会增加modCount

**protected** **void** removeRange(**int** fromIndex, **int** toIndex)增加modCount，将toIndex后面的元素复制到fromIndex开始的位置，多余的设为null便于触发GC，修改size值

**public** **boolean** removeAll(Collection<?> c)移除和c中相同的元素，若原本集合中元素个数变动增加modCount

**public** **boolean** retainAll(Collection<?> c)保留和c中相同的元素，若原本集合中元素个数变动增加modCount

**private** **void** writeObject(java.io.ObjectOutputStream s) 序列化函数，通过s.writeObject将对象写入到输出流中，通过检查modCount的值是否改变来确保线程安全

**private** **void** readObject(java.io.ObjectInputStream s)序列化函数，通过s.readObject()将输入流中的对象读取到elementData中

**public** Iterator<E> iterator()返回一个实现的迭代器，只能从头向尾移动，移动时元素个数不能被其他线程改变。可以移除和forEachRemaining (Consumer<? super E> consumer)进行批量操作，批量操作时modCount不能被其他线程改变

**public** ListIterator<E> listIterator(**int** index)返回一个从指定下标开始的列表迭代器是Iterator<E>的继承类，若不输入参数默认从0开始。可以前后移动，移动时检查元素个数不能被其他线程修改。额外提供了set和add方法。

**public** List<E> subList(**int** fromIndex, **int** toIndex)返回一个子集合，对子集合的add remove set操作会改变parent对应的元素，并改变两者的modCount值。subList为抽象类，本身并没有elementData区域，对他元素的操作会直接操作在parent的elementData上

**public** **boolean** removeIf(Predicate<? **super** E> filter)根据过滤器filter移除符合的元素，整个过程要求modCount不能被其他线程改变，先用一个BitSet存储要删除的序号，然后通过removeSet.nextClearBit(i)进行复制移动，最后modCount++

**public** **void** replaceAll(UnaryOperator<E> operator)根据规则替换所有符合的元素，会改变modCount

**public** **void** sort(Comparator<? **super** E> c)排序，会改变modCount