---
layout:     post
title:      Java StringBuffer StringBuilder类源码解析
subtitle:   分析动态字符串StringBuffer和StringBuilder的实现原理
date:       2018-08-20
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - 字符串
---

## StringBuffer

StringBuffer是线程安全的字符动态序列，像String但是可以修改，在任何时点他都含有字符的特定序列，但是序列的长度和内容可以通过调用某些方法来修改。

StringBuffer对于多线程是安全的，在必要的方法上都加了synchronized。核心方法是append和insert，他们通过重载可以接受任何类型的数据。将数据转换为String然后扩展或者插入到StringBuffer中。append将字符添加到末尾，insert是添加到某个指定的位置。举个例子，z是一个StringBuffer，当前内容为"start"，此时调用z.append("le")则内容变为"startle"，若调用的是z.insert(4, "le")则内容变为"starlet"。sb是一个StringBuffer，sb.append(x)和sb.insert(sb.length(), x)是等效的。

当有一个包含源序列的操作发生时，只有StringBuffer同步操作，不会发生在源上。

由于StringBuffer被设计为线程安全类，所以在通过一个被多个线程共享的源序列构造和append insert操作时，调用的程序必须确保在这些操作期间源序列没有发生变化。这个可以通过调用者在操作期间加锁来保证，或者通过使用一个不可变的源序列，或者不使用线程共享的源序列。

除非另外说明，对构建或者其他方法传入一个null参数会引起抛出NullPointerException错误。

JDK5中，补充了StringBuffer的单线程版本StringBuilder，StringBuilder应该优先使用，他有同样的操作但是没有synchronized所以速度更快。

### 内部变量与构造函数

从类的定义中可以看出StringBuffer继承了AbstractStringBuilder，下面会介绍到复用了AbstractStringBuilder的内部变量与函数

```java
 public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
```

StringBuffer自身有一个内部变量toStringCache，这是上一个toString返回值的高速缓存，一旦StringBuffer被修改就会清空，作用是在调用toString的时如果没有变更可以快速返回结果不用重新构造字符串

```java
    private transient char[] toStringCache;
```

观察StringBuffer的构造函数，可以看到他们都是基于super(capacity)这个方法来展开的，也就是AbstractStringBuilder的构造函数

```java
    //构造一个初始大小为16的StringBuffer
	public StringBuffer() {
        super(16);
    }
	//构造指定初始容量大小
    public StringBuffer(int capacity) {
        super(capacity);
    }
	//构造一个StringBuffer，初始内容为str，初始大小为16+str的长度
    public StringBuffer(String str) {
        super(str.length() + 16);
        append(str);
    }
	//构造一个StringBuffer内容和CharSequence一致，初始容量为16+CharSequence.length，如果CharSequence的长度为0，则返回一个空的buffer容量为16
    public StringBuffer(CharSequence seq) {
        this(seq.length() + 16);
        append(seq);
    }
```

下面的内部变量和构造函数来自AbstractStringBuilder，可以看到他的构造方法主要是新分配了一个给定大小的数组

```java
    char[] value;//存储字符

    int count;//字符个数

    AbstractStringBuilder(int capacity) {
        value = new char[capacity];//分配一个大小为capacity的字符数组给value
    }
```

下面两个方法是对容量和字符长度的查询，只做查询而不会做出修改

```java
    public synchronized int length() {
        return count;//返回字符个数
    }

    public synchronized int capacity() {
        return value.length;//返回容量大小也就是数组大小
    }
```

而ensureCapacity是会修改数组大小的，他会确保value数组的大小不小于minimumCapacity，如果容量小于该大小，会分配一个新的数组并将原本的字符复制到新数组中，新数组大小是当前容量*2+2和minimumCapacity中的较大值，minimumCapacity有大小限制，超过一定的值会内存溢出

```java
    public synchronized void ensureCapacity(int minimumCapacity) {
        super.ensureCapacity(minimumCapacity);//确保value数组的大小不小于minimumCapacity
    }
//下面的代码来自父类
	//确保容量不小于最小值，如果当前容量小于参数值，分配一个新的更大的内部数组，他的大小是minimumCapacity和旧容量旧容量*2+2中的较大值。如果minimumCapacity是负数，什么也不做直接返回。
    public void ensureCapacity(int minimumCapacity) {
        if (minimumCapacity > 0)
            ensureCapacityInternal(minimumCapacity);
    }

    private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        if (minimumCapacity - value.length > 0) {
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));//根据minimumCapacity分配一个新的数组，并将原来的字符复制到新的数组中
        }
    }
	//返回不小于minCapacity的大小，如果当前大小*2+2足够的话就取该值。不会返回超过MAX_ARRAY_SIZE的大小，除非minCapacity超过该值
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int newCapacity = (value.length << 1) + 2;
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }//新的大小为旧大小*2+2与minCapacity中的较大值
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
	//如果minCapacity在MAX_ARRAY_SIZE到Integer.MAX_VALUE之间的话返回minCapacity，超过Integer.MAX_VALUE抛出OutOfMemoryError
    private int hugeCapacity(int minCapacity) {
        if (Integer.MAX_VALUE - minCapacity < 0) { // 内存溢出
            throw new OutOfMemoryError();
        }
        return (minCapacity > MAX_ARRAY_SIZE)
            ? minCapacity : MAX_ARRAY_SIZE;
    }
```

trimToSize在value中存在没有存储的空间时，会重新分配一个大小和字符个数相等的数组将字符复制过去，提高空间利用率，会改变capacity()的值

```java
    public synchronized void trimToSize() {
        super.trimToSize();//新分配一个数组仅保留与字符个数相等的大小，将字符复制过去
    }
	//尝试减少用于存储字符串的空间。如果缓冲区比保存当前字符串所需的空间更大，会变更大小提高空间利用率。这个方法可能会改变capacity()的返回值
    public void trimToSize() {
        if (count < value.length) {
            value = Arrays.copyOf(value, count);//新分配一个大小为字符个数的数组，将现有的字符复制过去
        }
    }
```

setLength在newLength小于等于当前数组大小时直接返回，大于时新分配一个大小为newLength和当前容量*2+2的较大值的新数组，并复制字符，然后将数组中的剩余位置填充上'\0'，count设为newLength

```java
    public synchronized void setLength(int newLength) {
        toStringCache = null;//清空上一次toString的缓存
        super.setLength(newLength);
    }

    public void setLength(int newLength) {
        if (newLength < 0)
            throw new StringIndexOutOfBoundsException(newLength);
        ensureCapacityInternal(newLength);//newLength小于等于当前数组大小的话直接返回，否则分配一个大小为newLength和当前容量*2+2的较大值的新数组，并复制字符

        if (count < newLength) {
            Arrays.fill(value, count, newLength, '\0');//字符个数小于newLength时，用'\0'填充剩余的位置
        }

        count = newLength;//count设为newLength
    }
```

charAt返回指定位置的字符，会检查index返回是否大于等于0且小于count

```java
    public synchronized char charAt(int index) {
        if ((index < 0) || (index >= count))
            throw new StringIndexOutOfBoundsException(index);
        return value[index];
    }   

	public char charAt(int index) {
        if ((index < 0) || (index >= count))
            throw new StringIndexOutOfBoundsException(index);
        return value[index];
    }
```

codePointAt是返回index位置的代码点，代码点这个东西之前在String里讲过，这里再贴一次：字符数据类型是一个采用UTF-16编码表示Unicode代码点的代码单元。大多数的常用Unicode字符使用一个代码单元就可以表示，而辅助字符需要一对代码单元表示。而length返回的是UTF-16下的代码单元的数量，而codePointCount返回的是代码点的数量。对于大部分人工输入的字符，这两者是相等的，会出现length比codePointCount长的通常是某些数学或者机器符号，需要两个代码单元来表示一个代码点 。codePointBefore返回index前一个位置的代码点，codePointCount则是统计指定序列段中的代码点数量

```java
    public synchronized int codePointAt(int index) {
        return super.codePointAt(index);
    }

	public int codePointAt(int index) {
        if ((index < 0) || (index >= count)) {
            throw new StringIndexOutOfBoundsException(index);
        }
        return Character.codePointAtImpl(value, index, count);
    }

	public synchronized int codePointBefore(int index) {
        return super.codePointBefore(index);
    }

    public int codePointBefore(int index) {
        int i = index - 1;
        if ((i < 0) || (i >= count)) {
            throw new StringIndexOutOfBoundsException(index);
        }
        return Character.codePointBeforeImpl(value, index, 0);
    }

    public synchronized int codePointCount(int beginIndex, int endIndex) {
        return super.codePointCount(beginIndex, endIndex);//统计从beginIndex到endIndex之间的代码点数量
    }

    public int codePointCount(int beginIndex, int endIndex) {
        if (beginIndex < 0 || endIndex > count || beginIndex > endIndex) {
            throw new IndexOutOfBoundsException();
        }
        return Character.codePointCountImpl(value, beginIndex, endIndex-beginIndex);
    }
```

offsetByCodePoints这个方法单看注释翻译比较难理解：返回从index到codePointOffset的代码点偏移index，每个不成对的代理(两个代码单元表示一个代码点时称为两个代理)在范围内被记为一个代码点。实际上可以理解为，如果不存在两个代码单元表示一个代码点的情况，返回的结果就是index+codePointOffset；如果存在那种特殊代码点，则index的变化量会偏移特殊代码点的个数，例如有3个特殊代码点，则返回值为index+codePointOffset+3(codePointOffset>0)或者index+codePointOffset-3(codePointOffset<0)

```java
    public synchronized int offsetByCodePoints(int index, int codePointOffset) {
        return super.offsetByCodePoints(index, codePointOffset);
    }

    public int offsetByCodePoints(int index, int codePointOffset) {
        if (index < 0 || index > count) {
            throw new IndexOutOfBoundsException();
        }
        return Character.offsetByCodePointsImpl(value, 0, count,
                                                index, codePointOffset);
    }
```

getChars会再检查参数范围后，复制指定位置的字符串到指定的位置

```java
    public synchronized void getChars(int srcBegin, int srcEnd, char[] dst,
                                      int dstBegin)
    {
        super.getChars(srcBegin, srcEnd, dst, dstBegin);//复制value从srcBegin到srcEnd的内容到dst从dstBegin开始的位置
    }
```

setCharAt修改指定位置的字符

```java
    public synchronized void setCharAt(int index, char ch) {
        if ((index < 0) || (index >= count))
            throw new StringIndexOutOfBoundsException(index);
        toStringCache = null;//清空toString缓存
        value[index] = ch;//修改对应位置的字符
    }
```

核心函数之一的append有众多的重载，篇幅原因就不全贴了。append需要注意一点，直接在参数里输入null是会报错的，但是以对象赋值null的方式传入是可行的，相当于添加"null"。对于传入的非字符串对象，统一调用toString方法转换为字符串；数值对象的话通过包装类的方法转为字符串。

```java
    public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }

    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);//确保数组容量足够大
        str.getChars(0, len, value, count);//将str从头到尾复制到value中从count开始的位置，实现拼接
        count += len;//增加字符数量
        return this;
    }

    private AbstractStringBuilder appendNull() {
        int c = count;
        ensureCapacityInternal(c + 4);
        final char[] value = this.value;
        value[c++] = 'n';//null当做"null"来进行扩展
        value[c++] = 'u';
        value[c++] = 'l';
        value[c++] = 'l';
        count = c;
        return this;
    }
```

delete删除包括start在内到end之前的字符，end开始部分保留，通过复制保留部分到start的位置来实现

```java
    public synchronized StringBuffer delete(int start, int end) {
        toStringCache = null;//清除toString缓存
        super.delete(start, end);//删除从start到end-1位置的元素
        return this;
    }

    public AbstractStringBuilder delete(int start, int end) {
        if (start < 0)
            throw new StringIndexOutOfBoundsException(start);
        if (end > count)
            end = count;//end最大为count
        if (start > end)
            throw new StringIndexOutOfBoundsException();
        int len = end - start;
        if (len > 0) {
            System.arraycopy(value, start+len, value, start, count-end);//将start+len开始的长度为count-end的部分，复制到start开始的位置
            count -= len;//修改count值
        }
        return this;
    }
```

deleteCharAt只删除单个字符，也是通过复制来实现

```java
    public synchronized StringBuffer deleteCharAt(int index) {
        toStringCache = null;
        super.deleteCharAt(index);//将index后一位开始的内容复制到index的位置
        return this;
    }
```

replace操作会移除start到end-1的内容，将str插入到start开始的位置，实现的话会先把value中的后面那段复制到他最终所处的位置，中间留出一段空间供str复制进去

```java
    public synchronized StringBuffer replace(int start, int end, String str) {
        toStringCache = null;
        super.replace(start, end, str);//移除start到end-1的内容，将str插入到start开始的位置
        return this;
    }
```

substring和subSequence方法截取子串，substring可以不输入end参数截取到末尾，方法都是基于父类的同一个函数来返回一个新的String

```java
    public String substring(int start, int end) {
        if (start < 0)
            throw new StringIndexOutOfBoundsException(start);
        if (end > count)
            throw new StringIndexOutOfBoundsException(end);
        if (start > end)
            throw new StringIndexOutOfBoundsException(end - start);
        return new String(value, start, end - start);
    }
```

insert方法同样是重载众多，但是主要参数只有在value中插入的位置、插入的对象、插入对象从哪里开始截取、截取长度是多少，后两个可以不输入那么就是整个对象进行插入。会清空toStringCache

```java
    public synchronized StringBuffer insert(int index, char[] str, int offset,
                                            int len)
    {
        toStringCache = null;
        super.insert(index, str, offset, len);
        return this;
    }

    public AbstractStringBuilder insert(int index, char[] str, int offset,
                                        int len)
    {
        if ((index < 0) || (index > length()))
            throw new StringIndexOutOfBoundsException(index);
        if ((offset < 0) || (len < 0) || (offset > str.length - len))
            throw new StringIndexOutOfBoundsException(
                "offset " + offset + ", len " + len + ", str.length "
                + str.length);
        ensureCapacityInternal(count + len);//确保空间足够，不足时扩展为当前容量*2+2和count+len的较大值
        System.arraycopy(value, index, value, index + len, count - index);//将index开始的内容复制到index+len的位置，空出留给str的空间
        System.arraycopy(str, offset, value, index, len);//str复制到留出的空间中
        count += len;//count增加str的长度
        return this;
    }
```

indexOf和lastIndexOf两个方法分别是从头开始向后寻找第一个完全相等的字符串和从尾部开始从头寻找第一个，可以指定开始寻找的位置，直接调用了String的同名方法

```java
    public synchronized int indexOf(String str, int fromIndex) {
        return super.indexOf(str, fromIndex);//调用了String.indexOf
    }

    public synchronized int lastIndexOf(String str, int fromIndex) {
        return super.lastIndexOf(str, fromIndex);//调用了String.lastIndexOf
    }
```

reverse这个方法会逆序字符串内容，从中心开始做轴对称的交换

```java
    public synchronized StringBuffer reverse() {
        toStringCache = null;
        super.reverse();//以中心为轴，从中间点开始做轴对称位置的字符复制交换
        return this;
    }
```

toString有缓存直接返回，否则新建一个数组复制value里的有效字符。所有会导致value中内容变化的方法都会清空缓存，还有setLength无论是否导致长度变化并填充了'\0'都会清空

```java
    public synchronized String toString() {
        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);//缓存无效时，创建一个新的数组将value中的有效字符复制进去
        }
        return new String(toStringCache, true);//缓存有效时直接返回，缓存中的字符串是被共享的
    }
```

## StringBuilder

JDK1.5加入，同样继承了AbstractStringBuilder，实现了java.io.Serializable, CharSequence接口。

StringBuilder是没有toStringCache的，所以他的toString函数必定是复制产生一个新的String，猜测是出于StringBuilder默认是用于单线程环境，不需要进行共享操作，所以也就没有了cache

```java
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
```

StringBuilder在单线程情况下由于没有了同步锁性能更好，推荐优先使用。他的实现和StringBuffer除了上面提到的cache和同步的问题外几乎没有区别，另外一个有区别的地方是序列化部分。

先看StringBuilder的序列化函数，非常简单，除了缺省对象外只有count和value的读写

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        s.defaultWriteObject();
        s.writeInt(count);
        s.writeObject(value);
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        count = s.readInt();
        value = (char[]) s.readObject();
    }
```

而StringBuffer就不同了，用了ObjectStreamField来声明序列化的字段，至于这两个序列化的方式到底有什么区别，以后能更新到IO流的时候再说吧

```java
    private static final java.io.ObjectStreamField[] serialPersistentFields =
    {
        new java.io.ObjectStreamField("value", char[].class),
        new java.io.ObjectStreamField("count", Integer.TYPE),
        new java.io.ObjectStreamField("shared", Boolean.TYPE),
    };

    private synchronized void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        java.io.ObjectOutputStream.PutField fields = s.putFields();
        fields.put("value", value);
        fields.put("count", count);
        fields.put("shared", false);
        s.writeFields();
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        java.io.ObjectInputStream.GetField fields = s.readFields();
        value = (char[])fields.get("value", null);
        count = fields.get("count", 0);
    }
```

