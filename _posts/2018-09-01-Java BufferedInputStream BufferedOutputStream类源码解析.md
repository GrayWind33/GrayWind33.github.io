---
layout:     post
title:      Java BufferedInputStream BufferedOutputStream类源码解析
subtitle:   分析缓冲字节流BufferedInputStream和BufferedOutputStream的实现原理
date:       2018-09-01
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

# BufferedInputStream

​	BufferedInputStream是一个缓冲输入流，继承的是FilterInputStream。FilterInputStream包含了另一个InputStream作为它的基础数据源，并且FilterInputStream重写了InputStream的所有方法。作为FilterInputStream需要重写其中的部分方法，如果没有重写的话默认调用InputStream的同名方法。

​	对于BufferedInputStream来说，常规的用法是以FileInputStream作为它的下层输入流，原因是FileInputStream是从文件中读取字节，而文件处于硬盘之中，每次从硬盘读取数据的速度相对于内存很慢。如果多次通过FileInputStream读取或跳过一段字符，就需要多次访问硬盘，如果能够通过一次访问将整块内容读取到内存中，再从内存中多次读取，效率就可以大幅度提高，这就是内存缓冲区。举个极端例子来说，连续调用FileInputStream.read()一千次，得到文件中一千个连续字节，访问硬盘1000次。如果使用FileInputStream.read(buff[1000])，然后从buff[]中读取1000个字节，只需要访问硬盘1-2（跨页）次，访问内存1000次，毫无疑问由于内存对硬盘的数量级速度优势是后者快得多。

​	BufferedInputStream对另一个输出流增加了功能，支持缓存输入和mark/reset操作。当创建BufferedInputStream时，一个内部缓冲数组被建立，随着字节从流中被读取或者跳过，内部缓冲区根据需要从包含的输入流中重新装填许多字节。mark操作记录了输入流中的一个位置，reset操作引起在最近一个mark操作前的所有读取的字节在流中新的字节被获取之前被重新读取。涉及数据的方法是线程同步的。

先来看一下内部变量，可以看到BufferedInputStream新增了很多内部变量来支持缓存以及mark/reset操作。buf数组作为缓冲区，其中的有效字节数是count，当前读取到的位置是pos，所以count-pos是剩余可以从缓冲区内直接读取的字节数。markpos是标记的位置，-1代表没有标记过，pos-markpos是标记之后的字节数，该值不能超过marklimit，超过的话会导致markpos被设为-1也就是mark内容被丢弃。

```java
    private static int DEFAULT_BUFFER_SIZE = 8192;//默认大小8K

    /**
     * 分配的最大数组大小。一些虚拟机保留了数组中一些头部字。
     * 试图分配更大的数组可能导致OutOfMemoryError：请求的数组大小超过了虚拟机限制
     */
    private static int MAX_BUFFER_SIZE = Integer.MAX_VALUE - 8;

    /**
     * 内部缓冲数组存储数据，必要时，它会被替换为另一个不同大小的数组
     */
    protected volatile byte buf[];

    /**
     * 比缓冲区最后一个有效字节的下标大1.这个值得范围总是在0和buf.length之间
     * 元素buf[0]到buf[count-1]含有从下层的输入流中缓冲的输入数据
     */
    protected int count;

    /**
     * 缓冲区的当前位置，下一个要从buf数组中被读取的字符的下标
     * 这个值的范围总是在0到count。如果它小于count，buf[pos]是下一个要作为输入提供的字节，
     * 如果它等于count，下一个read或者skip操作需要从输入流中读取更多的字节
     */
    protected int pos;

    /**
     * 上一次mark操作时pos的值
     * 这个值总是在-1到pos的范围之间。如果在输入流中没有标记的位置，这个值是-1。如果在输入流中有标记的位置，buf[markpos]在reset操作之后会是第一个作为输入提供的字节。
     * 如果markpos不是-1，从buf[markpos]到buf[pos-1]之间的所有字节需要保留在缓冲区数组中（尽管可能被移动到缓冲区数组的另一个位置使得对count、pos和markpos的值有合适的调整）
     * 它们直到pos和markpos之间的差值超过marklimit前不能被丢弃
     */
    protected int markpos = -1;

    /**
     * 在调用mark之后到随后调用reset之前能够读取的最大字节数
     * 无论何时pos和markpos之间的差值超过marklimit，标记会被丢弃，markpos被设为-1
     */
    protected int marklimit;
```

构造函数需要提供输入流，通常是FileInputStream，可选参数是缓冲区初始大小，该值不能为负数，不提供时使用默认值8K

```java
    /**
     * 创建一个BufferedInputStream并存储它的参数供之后使用，输入流是in。一个大小为8K内部的缓冲数组被建立存储在buf
     */
    public BufferedInputStream(InputStream in) {
        this(in, DEFAULT_BUFFER_SIZE);
    }

    /**
     * 创建一个指定缓冲区大小的BufferedInputStream并存储它的参数供之后使用，输入流是in。一个大小为size的内部的缓冲数组被建立存储在buf
     */
    public BufferedInputStream(InputStream in, int size) {
        super(in);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");
        }
        buf = new byte[size];
    }
```

BufferedInputStream中的方法除了close和markSupported外都需要保证流是打开的，所以有两个内部方法负责检查状态。调用方法在发现返回值为null时即表示流已经关闭。

```java
    /**
     * 检查确认下层输入流还没有因为关闭变成null，不是null的话返回输入流
     */
    private InputStream getInIfOpen() throws IOException {
        InputStream input = in;
        if (input == null)
            throw new IOException("Stream closed");
        return input;
    }

    /**
     * 检查确认缓冲区还没有因为关闭变成null，非null的话返回它
     */
    private byte[] getBufIfOpen() throws IOException {
        byte[] buffer = buf;
        if (buffer == null)
            throw new IOException("Stream closed");
        return buffer;
    }
```

下面分析read方法，先从读取单个字节开始，可以看到它从方法级别加了同步锁。如果缓冲区内的数据足够则直接返回数组中的元素，否则需要先通过fill()将下层输入流中的数据读取到缓冲区中，如果流中也没有数据了则返回-1。

```java
    public synchronized int read() throws IOException {
        if (pos >= count) {//pos>=count说明缓冲区内没有可读取的数据，需要从流中读取数据到缓冲区
            fill();
            if (pos >= count)//流中也没有数据可读取了
                return -1;
        }
        return getBufIfOpen()[pos++] & 0xff;
    }
```

然后来看一下fill方法，它会从下层输入流中读取字节填满缓冲区直到输入流中也没有有效的字节。如果当前没有标记，说明缓冲区不再需要存储已经读取过的内容，可以直接情况缓冲区。如果pos达到了缓冲区大小上限，此时如果markpos>0则丢弃缓冲区中markpos之前的部分；如果markpos等于0且缓冲区大小超过了marklimit需要丢弃标记和缓冲区内容；如果缓冲区大小达到了最大限制抛出OOM错误；如果缓冲区还没有达到大小限制，通过分配新数组再复制将缓冲区大小扩大两倍但不能超过最大限制，扩大仅在标记位置为0且读取到缓冲区最后一个字节，缓冲区大小没有超过marklimit和MAX_BUFFER_SIZE时才发生。

```java
    private void fill() throws IOException {
        byte[] buffer = getBufIfOpen();
        if (markpos < 0)
            pos = 0;            /* no mark: throw away the buffer 没有标记丢弃缓冲区*/
        else if (pos >= buffer.length)  /* no room left in buffer 缓冲区没有空间剩余*/
            if (markpos > 0) {  /* can throw away early part of the buffer 丢弃缓冲区早期部分*/
                int sz = pos - markpos;
                System.arraycopy(buffer, markpos, buffer, 0, sz);//被丢弃的是从0到markpos之间的数据
                pos = sz;
                markpos = 0;//标记位置设为0
            } else if (buffer.length >= marklimit) {
                markpos = -1;   /* buffer got too big, invalidate mark buffer太大，无效标记缓冲区*/
                pos = 0;        /* drop buffer contents 丢弃缓冲区内容*/
            } else if (buffer.length >= MAX_BUFFER_SIZE) {
                throw new OutOfMemoryError("Required array size too large");
            } else {            /* grow buffer buffer增长*/
                int nsz = (pos <= MAX_BUFFER_SIZE - pos) ?
                        pos * 2 : MAX_BUFFER_SIZE;//pos大小乘以2除非超过最大上限
                if (nsz > marklimit)
                    nsz = marklimit;//增长后的大小不能超过marklimit
                byte nbuf[] = new byte[nsz];
                System.arraycopy(buffer, 0, nbuf, 0, pos);//将原buffer中的内容复制到新的数组中
                if (!bufUpdater.compareAndSet(this, buffer, nbuf)) {
                    // Can't replace buf if there was an async close.如果有一个异步关闭，不能替换
                    // Note: This would need to be changed if fill()
                    // is ever made accessible to multiple threads.如果fill()曾被设置为多线程可进入，这里需要改变
                    // But for now, the only way CAS can fail is via close.
                    // assert buf == null;但是现在，CAS唯一失败的原因是关闭，断言buf==null
                    throw new IOException("Stream closed");
                }
                buffer = nbuf;
            }
        count = pos;//从开始pos到count都是有效字符
        int n = getInIfOpen().read(buffer, pos, buffer.length - pos);//从输入流中读取数据填满缓冲区
        if (n > 0)
            count = n + pos;//读取到数据则增加count
    }
```

int read(byte b[], int off, int len)从这个字节输入流读取字节到指定的字节数组中，从给出的偏移量开始。这个方法实现了InputStream.read(byte[], int, int)抽象方法。作为额外的便利，这个方法尝试读取尽可能多的字节，通过重复调用下层输入流的read方法。这个重复的read会一直持续直到以下几种情况：读取到了指定数量的字节；下层输入流read方法返回-1说明到达文件末尾；下层输入流的available方法返回0说明后续的读取请求会被阻塞。如果下层输入流的第一次read方法返回-1说明到达文件末尾，这个方法会返回-1，否则返回实际读取的字节数。

```java
    public synchronized int read(byte b[], int off, int len)
        throws IOException
    {
        getBufIfOpen(); // 检查流有没有关闭
        if ((off | len | (off + len) | (b.length - (off + len))) < 0) {//这里面有一个负数则或计算后得到负数
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }

        int n = 0;
        for (;;) {
            int nread = read1(b, off + n, len - n);
            if (nread <= 0)
                return (n == 0) ? nread : n;//什么都没读取到时返回-1
            n += nread;
            if (n >= len)
                return n;//达到需要读取的字节数，返回实际读取到的字节，正常情况下等于len
            // 如果没有关闭但是没有可读取的字节，返回
            InputStream input = in;
            if (input != null && input.available() <= 0)//下层输入流没有可以读取的字节
                return n;//返回读取到的字节数，该值小于len
        }
    }
```

上面的方法调用了内部方法read1，它将字符读入到数组的一部分，如果需要的话最多从下层输入流读取一次。先检查缓冲区内有没有数据，若没有数据且len超过了缓冲区大小上限并且缓冲区内没有标记，则直接从输入流读取到数组b，不再经过缓冲区并返回；否则从输入流读取数据填满缓冲区。除非已经返回，否则随后会从缓冲区复制数据到数组b，长度为len和缓冲区内剩余字节数的较小值。也就是说，read1会在不超过len和缓冲区上限的情况下读取尽可能多的字节。

```java
    private int read1(byte[] b, int off, int len) throws IOException {
        int avail = count - pos;//缓冲区内有效的字节数
        if (avail <= 0) {//缓冲区内没有数据了
            /*  如果需要的长度至少跟buffer一样大，并且没有mark/reset活动，不会打扰复制字节到本地缓冲区。这样缓冲流是级联无害的 */
            if (len >= getBufIfOpen().length && markpos < 0) {//流超过了缓冲区的数组长度且不存在标记
                return getInIfOpen().read(b, off, len);//从输入流中直接读取字节复制到数组b
            }
            fill();//从输入流中读取数据到缓冲区，在缓冲区数据被全部读取到b中且未超过marklimit的时候，会扩大缓冲区
            avail = count - pos;
            if (avail <= 0) return -1;//输入流中也没有数据了，返回-1
        }
        int cnt = (avail < len) ? avail : len;
        System.arraycopy(getBufIfOpen(), pos, b, off, cnt);//将缓冲区内的字节复制到数组b，数据量为len和缓冲区内剩余字节数的较小值
        pos += cnt;
        return cnt;//返回读取到数组b的字节数
    }
```

skip操作跳过指定数量的字节除非达到文件末尾。如果存在标记或者缓冲区存在有效内容则通过读取字节到缓冲区在设置pos的值来完成跳过，这个跳过的数量不能超过缓冲区的有效字节数。如果没有标记也没有缓冲区有效数据，直接通过下层输入流的skip来跳过，对于FileInputStream是通过底层方法IO_Lseek。

```java
    public synchronized long skip(long n) throws IOException {
        getBufIfOpen(); // 检查流是否关闭
        if (n <= 0) {
            return 0;
        }
        long avail = count - pos;

        if (avail <= 0) {
            // 如果没有标记则不再保留buffer
            if (markpos <0)
                return getInIfOpen().skip(n);//调用了下层输入流的skip方法

            // 填充缓冲区，保存用于reset的字节
            fill();
            avail = count - pos;
            if (avail <= 0)
                return 0;
        }

        long skipped = (avail < n) ? avail : n;//跳过的字节数最大不能超过缓冲区的可填充字节数
        pos += skipped;//通过读取到缓冲区并设置pos的位置来skip
        return skipped;//返回实际跳过的字节
    }
```

available()返回从输入流中不需要因为下一个操作阻塞的可以读取或者跳过的字节数的估计值。下一个操作可能由这个线程或者其他线程来调用。一个单位的读取或者跳过操作不会阻塞，但是可能读取或跳过更少的字节。这个方法返回缓冲区内剩余可读取的字节数count-pos加上FilterInputStream.available()

```java
    public synchronized int available() throws IOException {
        int n = count - pos;
        int avail = getInIfOpen().available();
        return n > (Integer.MAX_VALUE - avail)
                    ? Integer.MAX_VALUE
                    : n + avail;//不超过Interger范围时，返回值是缓冲区的有效数据加上输入流中的有效数据量
    }
```

mark/reset相关方法前面基本提到了，是通过直接修改pos的值来进行的

```java
    public synchronized void mark(int readlimit) {
        marklimit = readlimit;//marklimit设为给出的参数
        markpos = pos;//标记当前位置
    }

    /**
     * markpos是-1时没有标记，抛出IOException。否则pos=markpos
     */
    public synchronized void reset() throws IOException {
        getBufIfOpen(); // 如果关闭了会引起异常
        if (markpos < 0)
            throw new IOException("Resetting to invalid mark");
        pos = markpos;
    }

    /**
     * 检查这个输入流是否支持mark/reset操作，BufferedInputStream总是返回true
     */
    public boolean markSupported() {
        return true;
    }
```

close关闭这个输入流并释放任何关联的系统资源。一旦流被关闭，后面的read(), available(), reset(), skip()调用都会抛出IOException。关闭一个已经关闭的流没有作用

```java
    public void close() throws IOException {
        byte[] buffer;
        while ( (buffer = buf) != null) {
            if (bufUpdater.compareAndSet(this, buffer, null)) {//将buf设为null
                InputStream input = in;
                in = null;//将in设为null
                if (input != null)
                    input.close();//关闭下层输入流
                return;
            }
            // 或者一个新的buf在fill()中更新时重试
        }
    }
```

最后，简单讲一下前面提到的CAS，compareAndSet简单来说就是先获取内存中的某个值，比较是否与给出的原值相等，如果相等的话尝试将这个值更新为目标值，再检查内存中的值是否变为目标值，成功则返回，任何一个环节失败则直接返回false。这里主要涉及到的问题是多线程之间的不同步问题，CAS一定会保证这次操作是原子性的，并且这个操作是无锁算法所以比起synchronized在碰撞较少时更加高效。总之，CAS的作用是确保方法是线程安全的，对于BufferedInputStream来说，唯一可能造成线程不安全的地方是在其他方法没有进行完时进行关闭。对于close来说，它在发现有其他线程在通过fill修改buf时，会检测到buffer不相等或者修改后不是null，会继续循环尝试关闭。而对于fill方法，如果检查到冲突会直接抛出IO异常，结束这次操作。

```java
    /**
     * 原子更新操作为缓冲区提供CAS操作。这是必要的，因为关闭是异步的。
     * 我们使用null的buf[]作为流关闭的主要指示器，输入关闭时输入区域也是null
     */
    private static final
        AtomicReferenceFieldUpdater<BufferedInputStream, byte[]> bufUpdater =
        AtomicReferenceFieldUpdater.newUpdater
        (BufferedInputStream.class,  byte[].class, "buf");
```

# BufferedOutputStream

BufferedOutputStream继承了FilterOutputStream，FilterOutputStream是所有过滤输出流类的超级父类，它含有一个下层的输出流，并且简单重写了OutputStream的全部方法。

跟缓冲输入流相对应，BufferedOutputStream实现了一个缓冲输出流，通过设置这样的输出流，应用可以在写入字节到下层输出流时不需要在写每个字节都调用一次下层系统，它通常以FileOutputStream作为下层输出流，通过一个在构造时分配的字节数组作为缓冲区来存储数据，避免每次write都直接涉及到底层的写入操作。前面讲到过，直接读写硬盘的速度和内存读取速度差距极大，所以通过在内存中缓存数据再一口气按块写入可以提高写入到硬盘的速度。该类的刷新和写操作是方法级加锁同步的。

BufferedOutputStream的内部变量只有缓冲区buf[]，记录有效字节数量的count和继承自父类的下层输出流out

```java
    /**
     * 存储数据的内部缓冲区
     */
    protected byte buf[];

    /**
     * 缓冲区中有效字节的数量。这个值得范围总是从0到buf.length，元素buf[0]到buf[count-1]含有有效的字节数据
     */
    protected int count;

    protected OutputStream out;
```

构造函数必须提供OutputStream，一般是FileOutputStream，可选参数是数组大小，数组在构造时进行分配，之后不会再重新分配

```java
    /**
     * 创建一个新的缓冲输出流来写入数据到指定的下层输出流当中
     */
    public BufferedOutputStream(OutputStream out) {
        this(out, 8192);//默认缓冲区大小是8K
    }

    /**
     * 创建一个新的缓冲输出流来写入数据到指定的下层输出流当中，指定它的缓冲区大小
     */
    public BufferedOutputStream(OutputStream out, int size) {
        super(out);
        if (size <= 0) {
            throw new IllegalArgumentException("Buffer size <= 0");//size必须大于0
        }
        buf = new byte[size];
    }
```

write写单个字节先检查缓冲区有没有满，满了先将缓冲区内的数据全部写入到输出流中，然后将要写的数据存储到缓冲区数组中。而从指定的字节数组中写从off偏移开始len长度的字节到这个缓冲输出流中时，一般来说这个方法存储给出数组中的字节到这个流的缓冲区，必要的时候刷新缓冲区到下层输出流。 如果请求的长度大于鞥与缓冲区大小，这个方法会刷新缓冲区，然后将字节直接写入到下层输出流中。这样多余的BufferedOutputStream就不会无必要的复制数据。

```java
    public synchronized void write(int b) throws IOException {
        if (count >= buf.length) {
            flushBuffer();//如果缓冲区满了则将缓冲区内的数据全部写入到输出流中
        }
        buf[count++] = (byte)b;//将b存储到缓冲区中
    }

    public synchronized void write(byte b[], int off, int len) throws IOException {
        if (len >= buf.length) {
            /* 
               如果请求的长度超出了输出缓冲区的大小，刷新输出缓冲区然后直接写数据。这样缓冲流可以无损害地流动*/
            flushBuffer();
            out.write(b, off, len);//直接调用下层输出流的write方法
            return;
        }
        if (len > buf.length - count) {//要输出的字节长度超过了剩余缓冲区大小则先刷新清空缓冲区
            flushBuffer();
        }
        System.arraycopy(b, off, buf, count, len);//将要输出的内容存储到缓冲区
        count += len;
    }
```

flushBuffer这个内部方法的作用是刷新内部缓冲区，调用下层输出流的write方法将缓冲区内的数据全部写入

```java
    private void flushBuffer() throws IOException {
        if (count > 0) {//如果缓冲区内存在有效数据
            out.write(buf, 0, count);//将缓冲区内的有效数据全部写入到输出流
            count = 0;
        }
    }
```

flush刷新这个缓冲输出流，这个命令任何的缓冲区中的输出字节写出到底层的输出流中

```java
    public synchronized void flush() throws IOException {
        flushBuffer();//将缓冲区内的数据全部写入到输出流中
        out.flush();//触发下层输出流的刷新操作，对于FileOutputStream的话是什么也不做
    }
```

最后close方法直接调用的父类的close，刷新一次之后直接关闭下层输出流，这里的写法是JDK7之后的try-with-resource的写法，流只在try的代码块内打开，结束后自动关闭。

```java
    public void close() throws IOException {
        try (OutputStream ostream = out) {
            flush();
        }
    }
```

在关闭之后，仅对BufferedOutputStream中缓冲区的操作并不会报错，只有涉及到下层输出流才会报错，此时写入到缓冲区的数据是无法写入到文件的

```java
	public static void main(String args[]) throws IOException{	
		BufferedOutputStream bout = new BufferedOutputStream(new FileOutputStream("D:/test/file.txt"));
		bout.write("1234567890".getBytes());//1234567890
		bout.close();
		bout.write("1234567890".getBytes());//这里写入的数据在缓冲区的数组中所以不报错
		bout.flush();//这里会报错
	}
```