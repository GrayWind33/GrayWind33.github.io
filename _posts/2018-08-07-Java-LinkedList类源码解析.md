---
layout:     post
title:      Java LinkedList类源码解析
subtitle:   分析List集合LinkedList实现原理
date:       2018-08-07
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

LinkedList底层为双向链表同样继承了AbstractSequentialList<E>，跟ArrayList的数组相比读取效率低，不支持随机读取，碎片化空间利用率高，平均随机插入效率相对高。同时可以用来实现queue。属性有:

**transient** **int** size = 0;list大小

**transient** Node<E> first;头指针

**transient** Node<E> last;尾指针

**private** **void** linkFirst(E e)

**void** linkLast(E e)

将e添加到链表的头部和尾部，size与modCount加一

**void** linkBefore(E e, Node<E> succ)将e插入到succ结点之前

**private** E unlinkFirst(Node<E> f)

**private** E unlinkLast(Node<E> l)

移除头部或尾部结点，size减1，modCount加1，将移除结点的next prev item值都设为null以触发gc

E unlink(Node<E> x)移除结点x，需要再判断有无前驱和后驱结点，若没有则要改变头尾指针，同样将移除结点的next prev item值都设为null以触发gc

**public** E getFirst()

**public** E getLast()

返回first或last指向的结点，链表为空时抛错

**public** E removeFirst()

**public** E removeLast()

调用unlinkLast移除并返回头部或尾部的结点，链表为空时抛错

**public** **void** addFirst(E e)调用linkFirst(e)，将元素插入到头部

**public** **void** addLast(E e)

**public** **boolean** add(E e)

两个方法都是调用linkLast(e)，除了返回值外是等价的

**public** **boolean** remove(Object o)o==null时，通过unlink方法移除所有x==null的元素，否则移除o.equals(x)的元素，每有一个符合的元素就调用一次unlink所以modCount的增加值为移除元素的个数

**public** **boolean** addAll(**int** index, Collection<? **extends** E> c)将c中的集合插入到index位置。首先检查index是否符合链表长度范围，若c中没有元素则直接返回false，否则遍历c中的元素产生新的结点并链接到index指向位置，检查是否需要修改first和last的位置，最后修改size和modCount++

**public** **void** clear()遍历所有结点，将last prev item全部设为null，size为0，modCount++

**public** E set(**int** index, E element)检查index范围后设置为item=element，不会改变modCount

**public** **void** add(**int** index, E element)检查index范围，若index==size即插入再末尾，调用linkLast(element)，否则调用linkBefore(element, node(index))因此会造成modCount++

**public** E remove(**int** index) 检查index范围，index >= 0 && index < size则调用unlink(node(index))移除元素，modCount++

Node<E> node(**int** index)返回index下标的结点，若index超过size的一半则从last开始向头寻找，否则从first开始向后寻找

**public** **int** indexOf(Object o)寻找与o相等的下标最小的链表元素，若没有则返回-1，比较逻辑依然根据o是否是null来区分

**public** **int** lastIndexOf(Object o)从尾部开始搜索第一个符合条件的元素下标，和上面一个方法类似

**public** E peek()返回first指向结点的item，若为空则返回null

**public** E element()也是返回first.item，区别是为空会抛错

**public** E poll()在peek()的基础上，若不为null会删除第一个元素

**public** **boolean** offer(E e)同add(e)

**public** **boolean** offerFirst(E e)同addFirst

**public** **boolean** offerLast(E e)同addLast

**public** E peekFirst()同peek()

**public** E peekLast()返回尾部元素，为空则返回null

**public** E pollFirst()同poll()

**public** E pollLast()返回尾部元素，为空则返回null，不为空移除尾部元素

**public** **void** push(E e)将e添加到头部

**public** E pop()移除头部元素

**public** **boolean** removeFirstOccurrence(Object o)同remove(o)

**public** **boolean** removeLastOccurrence(Object o)移除最后一个与o相等的元素

**public** Object[] toArray()新建一个数组，遍历链表将元素复制到数组中

**private** **void** writeObject(java.io.ObjectOutputStream s)

**private** **void** readObject(java.io.ObjectInputStream s)

序列化的方式和ArrayList相同，是通过对象输入输出流来完成，输入时调用linkLast将读取到的元素加入链表末尾