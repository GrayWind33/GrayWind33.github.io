---
layout:     post
title:      Java ByteArrayInputStream ByteArrayOutputStream类源码解析
subtitle:   分析内存字节流ByteArrayInputStream和ByteArrayOutputStream的实现原理
date:       2018-08-22
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

## ByteArrayOutputStream

ByteArrayOutputStream继承了抽象类OutputStream，本质是一个存在于堆内存中的可扩展byte数组，因为所有操作都在内存中所以flush()也什么都没做。

该类实现了一个输出流，数据写入一个byte数组，缓冲区随着写入的数据自动增大。数据可以通过toByteArray()和toString()重新取回。

关闭ByteArrayOutputStream没有影响，类中的方法可以在流被关闭后调用而不抛出IOException，因为close()实际上什么也没做。

从下面的代码中可以看出，所有涉及到输出流内容读写的操作，包括write、reset、toByteArray、size和toString都加了synchronized关键字，因为grow操作只改变数组大小不改变有效内容所以不需要加synchronized，因为要避免输出内容产生错乱。

从内部变量中可以看到，底层是一个byte数组，一个统计比特数的计数器

```java
    //存储数据的缓冲区
    protected byte buf[];

	//缓冲区内的有效byte
    protected int count;
```

无参构造时数组大小初始设为32，也可以指定size大小，size为负数时抛出IllegalArgumentException

```java
    //创建一个新的ByteArrayOutputStream，初始缓冲区大小为32
	public ByteArrayOutputStream() {
        this(32);
    }
	//创建一个新的ByteArrayOutputStream，初始缓冲区大小为size
    public ByteArrayOutputStream(int size) {
        if (size < 0) {
            throw new IllegalArgumentException("Negative initial size: "
                                               + size);
        }
        buf = new byte[size];
    }
```

输入前要检查空间大小是否足够，下面一系列函数都是用于检查大小和进行扩大，数组大小存在上限，太大时会出现OOM错误。每次进行扩展时大小变为当前容量*2和需要的最小容量参数中的较大值，除非minCapacity>MAX_ARRAY_SIZE，否则不会突破这个大小，MAX_ARRAY_SIZE<minCapacity<Integer.MAX_VALUE时数组大小为Integer.MAX_VALUE

```java
	//在必要时增加容量来确保足够持有minCapacity表明的最小数目的元素
    private void ensureCapacity(int minCapacity) {
        // overflow-conscious code
        if (minCapacity - buf.length > 0)
            grow(minCapacity);
    }
	//数组的最大大小，一些虚拟机保留了数组头部的一些位置。试图分配更大的数组会导致OutOfMemoryError：请求的数组大小超过了虚拟机的限制
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = buf.length;//当前数组大小
        int newCapacity = oldCapacity << 1;//新容量=当前容量*2
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;//新容量为当前容量*2和minCapacity中的较大值
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);//当前容量*2>MAX_ARRAY_SIZE是要做处理
        buf = Arrays.copyOf(buf, newCapacity);//新分配一个数组把数据复制过去
    }

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // 负数代表minCapacity超过正数所能表示的范围了，所以内存溢出
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE ://minCapacity在Integer.MAX_VALUE到Integer.MAX_VALUE-8范围内，大小为Integer.MAX_VALUE
            MAX_ARRAY_SIZE;//小于MAX_ARRAY_SIZE时为MAX_ARRAY_SIZE
    }
```

每次的写入操作需要先确保数组大小足够，然后将内容复制到数组中

```java
	//将特定的byte写入到这个输出流中
    public synchronized void write(int b) {
        ensureCapacity(count + 1);//确保数组还能再多存储一个
        buf[count] = (byte) b;//int转为byte保存到数组中
        count += 1;//计数器+1
    }
	//将从off开始长度为len的bytes写入到输出流中
    public synchronized void write(byte b[], int off, int len) {
        if ((off < 0) || (off > b.length) || (len < 0) ||
            ((off + len) - b.length > 0)) {
            throw new IndexOutOfBoundsException();
        }
        ensureCapacity(count + len);
        System.arraycopy(b, off, buf, count, len);//复制内容到数组中
        count += len;
    }
```

writeTo是将当前输出流中的内容全部写入到参数中的输出流中去，相当于调用out.write(buf, 0, count)

```java
    public synchronized void writeTo(OutputStream out) throws IOException {
        out.write(buf, 0, count);
    }
```

reset将count设为0，当前累计的输出都被丢弃了，这个输出流可以被重新使用，再次利用已经分配好的缓冲区空间，因为没有可以直接更改count的方法，所以相当于旧的内容无法再被访问了

```java
    public synchronized void reset() {
        count = 0;
    }
```

toByteArray创建一个新的byte数组，大小为当前输出流中的byte大小，将其中的byte内容复制到新的数组中

```java
    public synchronized byte toByteArray()[] {
        return Arrays.copyOf(buf, count);
    }
```

size返回当前流中的byte大小

```java
    public synchronized int size() {
        return count;
    }
```

toString将输出流中的内容重新解码为String，可以指定字符集也可以直接默认使用平台字符集

```java
	//将缓冲区中的内容通过平台的默认字符集转换成string解码。新的String长度是字符集的函数，所以可能不等于buffer的大小,这个方法总是用平台的默认字符集中的默认替代字符串来替代畸形输入和非图形化表示字符序列
	public synchronized String toString() {
        return new String(buf, 0, count);
    }
	//跟上面方法相比指定了字符集
    public synchronized String toString(String charsetName)
        throws UnsupportedEncodingException
    {
        return new String(buf, 0, count, charsetName);
    }
```

close可以看到什么都没做，所以一切照常，原因是该类操作都在堆内存中，不需要关闭一些句柄、套接字之类的东西所以什么都不用做

```java
    public void close() throws IOException {
    }
```

## ByteArrayInputStream

ByteArrayInputStream是和ByteArrayOutputStream对应的输入流，继承了抽象类InputStream，本质上也是一个存储在堆内存中的byte数组，这个**数组必须在构造时传入并直接被使用而不是复制**，之后大小无法改变，可以通过直接对数组中的内容进行修改改变流中的内容。可以标记一个位置，通过reset重置回去来进行重复读取。

```java
	public static void main(String args[]) throws IOException{
		ByteArrayOutputStream out = new ByteArrayOutputStream();
		out.write("1234567890".getBytes());
		byte buf[] = out.toByteArray();
		ByteArrayInputStream in = new ByteArrayInputStream(buf);
		byte res[] = new byte[10];
		in.read(res);
		System.out.println(new String(res));//1234567890
		buf[0] = '4';
		in.reset();
		in.read(res);
		System.out.println(new String(res));//4234567890
	}
```

ByteArrayInputStream含有一个内部缓冲区包含了可能从流中读取的比特，一个内部计数器用来保持对下一个read方法读取的byte的跟踪。

关闭ByteArrayInputStream没有作用，所有方法都可以继续使用不会出现IOException，因为全部操作都在堆内存中不存在要关闭的句柄等。

从内部变量中可以看出，底层通过buf[]来存储流中的内容

```java
    /**
     * 构造时提供一个数组，buf中的比特是唯一能够从流中读取的，buf[pos]是下一个要被读取的byte
     */
    protected byte buf[];

    /**
     * 下一个要被读取的byte的下标，总是非负数，不能超过count，buf[pos]是下一个要被读取的byte
     */
    protected int pos;

    /**
     * 流中当前标记的位置，ByteArrayInputStream在构造时默认标记位置是0。可以通过mark()方法标记缓冲区中的其他位置，能够通过reset()回到当前标记的位置
     * 如果没有设置过mark，他的值会是传递给构造器的值，如果没有的话是0
     */
    protected int mark = 0;

    /**
     * 比当前有效字符的index大1，总是非负数，不超过buf数组的长度。比buf数组中最后一个能读取的byte的下标大1
     */
    protected int count;
```

构造函数必须要传入buf[]，传入后对于buf的引用无法改变，但可以通过缓存引用来直接修改其中的内容

```java
	//创建一个ByteArrayInputStream，使用buf作为缓冲区，缓冲区没有拷贝而是直接引用这个数组。pos初始值为0，count的初始值是buf的数组大小
    public ByteArrayInputStream(byte buf[]) {
        this.buf = buf;
        this.pos = 0;
        this.count = buf.length;
    }
	//创建一个ByteArrayInputStream，使用buf作为缓冲区，pos初始值是offset，count的初始值是offset+length和buf.length中的较小值。缓冲区没有拷贝，直接引用了buf，mark设为offset
    public ByteArrayInputStream(byte buf[], int offset, int length) {
        this.buf = buf;
        this.pos = offset;
        this.count = Math.min(offset + length, buf.length);
        this.mark = offset;
    }
```

读取单个byte时，读取输入流中的下一个byte，返回值是0-255的int，已经到达末端没有可读的byte返回-1

```java
    public synchronized int read() {
        return (pos < count) ? (buf[pos++] & 0xff) : -1;//pos==count时到达末端返回-1
    }
```

从流中读取最多len长度的bytes。如果pos==count，返回-1表示已经到达了文件末端，否则返回读取到的bytes数量k，k是len和count-pos中的较小值。如果k是正数，buf[pos]~buf[pos+k-1]将通过ystem.arraycopy被复制到b[off]~b[off+k-1]中，pos增加k，k被返回。

```java
    public synchronized int read(byte b[], int off, int len) {
        if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        }

        if (pos >= count) {
            return -1;
        }

        int avail = count - pos;
        if (len > avail) {
            len = avail;//最多读取到流中数据末端
        }
        if (len <= 0) {
            return 0;
        }
        System.arraycopy(buf, pos, b, off, len);//复制数据到b中
        pos += len;//增加pos
        return len;
    }
```

skip跳过输入流中的n个bytes，如果到达了末端跳过的bytes会少于n。实际跳过的bytes数量k等于n和count-pos中的较小值，pos增加k，k被返回

```java
    public synchronized long skip(long n) {
        long k = count - pos;
        if (n < k) {
            k = n < 0 ? 0 : n;
        }

        pos += k;
        return k;
    }
```

available返回输入流中剩余可以读取或者跳过的bytes数量，等于count-pos，也就是缓冲区中剩余的可被读取的数量

```java
    public synchronized int available() {
        return count - pos;
    }
```

markSupported测试这个InputStream是否支持mark/reset操作，ByteArrayInputStream总是支持该操作所以直接返回true

```java
    public boolean markSupported() {
        return true;
    }
```

mark将当前的位置标记，ByteArrayInputStream的标记位再构造时默认值是0，可以通过这个方法标记到缓冲区的其他位置，在构造时也可以通过传入offset设置其他mark，若没有传入则标记0。readAheadLimit这个参数在这里没有实际作用，InputStream中加这个参数是留给其他类中mark使用指出mark位置开始能读取的最大bytes数量。

```java
    public void mark(int readAheadLimit) {
        mark = pos;
    }
```

reset将位置重置到mark的位置

```java
    public synchronized void reset() {
        pos = mark;
    }
```

close方法什么都没做，所以依然可以照常使用，不会产生IOException

```java
    public void close() throws IOException {
    }
```