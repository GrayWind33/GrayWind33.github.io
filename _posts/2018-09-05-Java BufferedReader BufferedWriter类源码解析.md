---
layout:     post
title:      Java BufferedReader BufferedWriter类源码解析
subtitle:   分析缓冲字符流BufferedReader和BufferedWriter的实现原理
date:       2018-09-05
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

# BufferedReader

像BufferedInputStream为FileInputStream提供了缓冲区一样，BufferedReader为InputStreamReader提供了缓冲区。一般情况下，所有的读取都是先从下层输入流读取到缓冲区，然后再从缓冲区读取到目标数组，除非出现要读取的长度超过了缓冲区大小且缓冲区没有有效数据和有效mark，可以直接从下层输入流读取字符。

来看一下这个类的头部注释：从字符输入流读取文本，缓冲字符来提供高效地读取字符、数组和行。缓冲区的大小可能是指定的或者是默认大小，默认大小对于大部分情况足够大了。一般来说，Reader的每一个读取请求引起一个下层的字节或者字符输入流的读取请求。所以建议用BufferedReader来包裹一个read方法花费大的Reader，比如FileReaders和InputStreamReaders。例如 BufferedReader in = new BufferedReader(new FileReader("foo.in"));将会缓存从特定文件来的输入。没有缓冲，每一次调用read()或者readLine()可能引起从文件读取字节，转换为字符后返回，这样效率不高。使用DataInputStreams进行文本输入的程序可以通过替换每一个DataInputStream为合适的BufferedReader进行局部化。

前面已经解释过为什么可以通过从文件读取大块内容存储到内存缓冲区来提高整体读取速度，BufferedReader就是字符缓冲输入流。它提供了一个字符数组来作为缓冲区，默认大小是8K，在构造时初始化，fill()方法可能会出现需要重新分配一个更大数组的情况。从构造函数和内部变量中，我们可以看到它也具有内部包裹的下层输入流并且需要是Reader及其子类。

```java
	/**
	 * 下层输入流
	 */
    private Reader in;
    /**
     * 缓冲区
     */
    private char cb[];
    /**
     * 最后一个有效字符的位置
     */
    private int nChars;
    /**
     * 下一个要读取的字符位置
     */
    private int nextChar;

    private static final int INVALIDATED = -2;
    private static final int UNMARKED = -1;
    /**
     * 标记的位置，小于等于-1时代表不存在mark
     */
    private int markedChar = UNMARKED;
    private int readAheadLimit = 0; /* 只有markedChar > 0时是有效的 */

    /** 如果下一个字符是换行符，跳过它 */
    private boolean skipLF = false;

    /** 设定mark时skipLF的标记 */
    private boolean markedSkipLF = false;

    private static int defaultCharBufferSize = 8192;
    private static int defaultExpectedLineLength = 80;

    /**
     * 创建一个缓冲字符输入流使用一个指定大小的输入缓冲区
     */
    public BufferedReader(Reader in, int sz) {
        super(in);
        if (sz <= 0)
            throw new IllegalArgumentException("Buffer size <= 0");
        this.in = in;
        cb = new char[sz];
        nextChar = nChars = 0;
    }

    /**
     * 创建一个缓冲字符输入流使用一个默认大小8K的输入缓冲区
     */
    public BufferedReader(Reader in) {
        this(in, defaultCharBufferSize);
    }
```

ensureOpen这个内部方法的作用就是检查流有没有被关闭，标志是下层输入流为null

```java
    private void ensureOpen() throws IOException {
        if (in == null)
            throw new IOException("Stream closed");
    }
```

无参数的read()读取单个字符，返回的是转为int的字符，返回是0-0xffff，因为存在两个字节的字符，到达文件末尾返回-1。这里存在一种情况，所有提到跳过换行符都是这个意思，**上一次读取到的是'\r'此时如果当前要读取的是'\n'则直接跳过读取下一个字符，这是LRLF代表换行的情况，Windows平台下输入的文本很多是这种换行方式**。

```java
    public int read() throws IOException {
        synchronized (lock) {
            ensureOpen();
            for (;;) {
                if (nextChar >= nChars) {
                    fill();//读取到最后一个有效字符时，从输入流读取字符到缓冲区
                    if (nextChar >= nChars)
                        return -1;
                }
                if (skipLF) {//如果下一个字符是换行符要跳过
                    skipLF = false;
                    if (cb[nextChar] == '\n') {
                        nextChar++;
                        continue;
                    }
                }
                return cb[nextChar++];//从缓冲区返回要读取的字符
            }
        }
    }
```

内部方法fill()填充输入缓冲区，如果mark是有效的需要将它加入考虑，只有在可被读取的字符被全部读取完后才会调用这个方法。**该类中所有要从下层输入流读取字符到缓冲区都需要经过该方法**。没有mark时，会直接将内容从数组头部开始尽可能填满缓冲区。存在mark时，如果从mark开始读取的内容超过了readAheadLimit的限制，mark失效。mark未超过范围时，检查readAheadLimit是否超过了缓冲区大小，没有超过时将mark开始的部分移动到缓冲区头部，然后读取数据尽可能填满缓冲区；mark超过了缓冲区大小，需要分配一个大小等于readAheadLimit的新数组，复制mark开始的内容，然后再读取字符，**这是唯一引起分配新数组的方法**。所有调用fill()的方法需要保证线程的安全性，我们可以看到这里分配新数组是简单赋值而不是CAS方法。

```java
    private void fill() throws IOException {
        int dst;
        if (markedChar <= UNMARKED) {
            /* 没有mark */
            dst = 0;
        } else {
            /* 存在mark */
            int delta = nextChar - markedChar;
            if (delta >= readAheadLimit) {
                /* 从mark开始到读取位置的字节数超过了限制，将markedChar设为-2使它失效 */
                markedChar = INVALIDATED;
                readAheadLimit = 0;
                dst = 0;
            } else {
                if (readAheadLimit <= cb.length) {//能够保留的最大mark部分长度超过了缓冲区大小
                    /* 在当前缓冲区中洗牌 */
                    System.arraycopy(cb, markedChar, cb, 0, delta);//将从markedChar开始的所有有效字符移到缓冲区头部
                    markedChar = 0;
                    dst = delta;
                } else {
                    /* 重新分配缓冲区的大小来满足预读要求 */
                    char ncb[] = new char[readAheadLimit];//新大小是readAheadLimit
                    System.arraycopy(cb, markedChar, ncb, 0, delta);//将有效字符复制到新的数组
                    cb = ncb;
                    markedChar = 0;//因为从标记位开始的有效字符被移动到了头上，所以标记位从0开始
                    dst = delta;
                }
                nextChar = nChars = delta;
            }
        }

        int n;
        do {
            n = in.read(cb, dst, cb.length - dst);//从下层输入流读取字符
        } while (n == 0);
        if (n > 0) {
            nChars = dst + n;
            nextChar = dst;
        }
    }
```

int read(char cbuf[], int off, int len)读取字符到数组的一部分中，这个方法实现了Reader.read(char[], int, int)方法的一般约束。作为额外的便利，它会试图读取尽可能多的字符，通过重复调用下层输入流的read方法。这个重复的read会持续到一下一种情况为true：指定的字符数已经被读取，下层输入流的read方法返回-1说明到达了文件结束符，或者下层输入流的ready方法返回false说明后面的输入请求会阻塞。如果第一个下层输入流的read方法返回-1，这个方法会返回-1.否则这个方法返回实际读取的字符数。这个类的子类鼓励但不是必须去尝试用同样的方法读取尽可能多的字符。一般这个方法从这个输入流的字符缓冲区获取字符，如果需要的话从下层输入流来填充缓冲区。但是，如果缓冲区是空的，mark是无效的，请求的长度超过了缓冲区的长度，这个方法会从下层输入流直接读取字符到给定的数组。因此多余的BufferedReader不会进行不必要的复制。

```java
    public int read(char cbuf[], int off, int len) throws IOException {
        synchronized (lock) {//同步操作
            ensureOpen();
            if ((off < 0) || (off > cbuf.length) || (len < 0) ||
                ((off + len) > cbuf.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return 0;
            }

            int n = read1(cbuf, off, len);//从缓冲区读取数据到数组，如果调用时缓冲区就已经没有有效数据从下层输入流读取
            if (n <= 0) return n;
            while ((n < len) && in.ready()) {
                int n1 = read1(cbuf, off + n, len - n);//循环尝试从下层输入流读取数据到缓冲区
                if (n1 <= 0) break;//下层输入流没有数据时返回
                n += n1;
            }
            return n;
        }
    }

    private int read1(char[] cbuf, int off, int len) throws IOException {
        if (nextChar >= nChars) {
            /* If the requested length is at least as large as the buffer, and
               if there is no mark/reset activity, and if line feeds are not
               being skipped, do not bother to copy the characters into the
               local buffer.  In this way buffered streams will cascade
               harmlessly.如果需要的长度超过了缓冲区大小，并且没有mark/reset活动，并且换行没有被跳过
               不要麻烦地将字符复制到本地缓冲区。这样可以避免没必要的损耗 */
            if (len >= cb.length && markedChar <= UNMARKED && !skipLF) {
                return in.read(cbuf, off, len);
            }
            fill();//读取到最后一个缓冲区内的有效字符时，从下层输入流读取字符到缓冲区
        }
        if (nextChar >= nChars) return -1;//下层输入流没有有效字符时返回-1
        if (skipLF) {
            skipLF = false;
            if (cb[nextChar] == '\n') {
                nextChar++;//下一个字符时换行符时跳过
                if (nextChar >= nChars)//如果跳过换行后有效字符不足则尝试从下层输入流中读取
                    fill();
                if (nextChar >= nChars)
                    return -1;
            }
        }
        int n = Math.min(len, nChars - nextChar);//读取的字符长度是len和缓冲区内有效字符的较小值
        System.arraycopy(cb, nextChar, cbuf, off, n);//从缓冲区复制字符到目标数组中
        nextChar += n;
        return n;//返回实际读取的字符数
    }
```

readLine读取一行文本。一行通过一个换行符或者一个回车或者一个回车跟着一个换行符来终止。返回一个包含行内容但是不包含行终止字符的字符串，如果已经到达流结尾的话返回null。

```java
    public String readLine() throws IOException {
        return readLine(false);
    }

    String readLine(boolean ignoreLF) throws IOException {
        StringBuffer s = null;
        int startChar;

        synchronized (lock) {
            ensureOpen();
            boolean omitLF = ignoreLF || skipLF;

        bufferLoop:
            for (;;) {

                if (nextChar >= nChars)
                    fill();//缓冲区内没有有效字符时，从下层输入流读取字符到缓冲区
                if (nextChar >= nChars) { /* 到达了EOF */
                    if (s != null && s.length() > 0)
                        return s.toString();//读取到了字符则返回字符串
                    else
                        return null;//没有读取到字符返回null
                }
                boolean eol = false;//缓冲区内有有效数据，还没有到达行结束
                char c = 0;
                int i;

                /* 如果有必要的话跳过一个 '\n'*/
                if (omitLF && (cb[nextChar] == '\n'))
                    nextChar++;
                skipLF = false;//只有第一个字符是'\n'时才会跳过
                omitLF = false;

            charLoop:
                for (i = nextChar; i < nChars; i++) {//读取范围不超过缓冲区内的所有有效字符
                    c = cb[i];
                    if ((c == '\n') || (c == '\r')) {
                        eol = true;//到达了行结束
                        break charLoop;//这里跳出了循环，所以nextChar依然在换行符或者回车的位置上
                    }
                }

                startChar = nextChar;
                nextChar = i;

                if (eol) {//到达了行结束
                    String str;
                    if (s == null) {//将新读取的内容添加到返回结果中
                        str = new String(cb, startChar, i - startChar);
                    } else {
                        s.append(cb, startChar, i - startChar);
                        str = s.toString();
                    }
                    nextChar++;//此时nextChar向前移动一位，经过了之前的换行符或回车
                    if (c == '\r') {//如果行结束标记是回车，跳过下一个换行符对应LRLF这种情况
                        skipLF = true;
                    }
                    return str;
                }

                if (s == null)
                    s = new StringBuffer(defaultExpectedLineLength);
                s.append(cb, startChar, i - startChar);//没有到达行结尾时，直接将新增内容添加到返回的字符串中
            }
        }
    }
```

skip跳过字符的操作是通过读取字符到缓冲区然后改变缓冲区的当前读取位置下标nextChar实现的，跳过的字符数为参数n和缓冲区内剩余有效字符的较小值

```java
    public long skip(long n) throws IOException {
        if (n < 0L) {
            throw new IllegalArgumentException("skip value is negative");
        }
        synchronized (lock) {
            ensureOpen();
            long r = n;
            while (r > 0) {
                if (nextChar >= nChars)
                    fill();//缓冲区内没有有效字符时尝试从下层输入流读取字符到缓冲区
                if (nextChar >= nChars) /* 到达了EOF */
                    break;
                if (skipLF) {
                    skipLF = false;
                    if (cb[nextChar] == '\n') {
                        nextChar++;//调用时第一个有效字符时'\n'时跳过它
                    }
                }
                long d = nChars - nextChar;//当前缓冲区内的有效字符个数
                if (r <= d) {//缓冲区内的有效字符数超过了要跳过的字符数，直接增加nextChar
                    nextChar += r;
                    r = 0;
                    break;
                }
                else {//缓冲区内数据不足时，nextChar移动到最后一个有效字符的位置
                    r -= d;
                    nextChar = nChars;
                }
            }
            return n - r;//返回跳过的字符数
        }
    }
```

ready告知这个流是否准备完读取。一个字符缓冲流在缓冲区非空或者下层输入流准备完时是准备完成的状态。

```java
    public boolean ready() throws IOException {
        synchronized (lock) {
            ensureOpen();

            /*
             * 如果一个新行需要跳过并且下一个要读取的字符是新行的字符，立刻跳过它
             */
            if (skipLF) {
                /* 
                 * in.ready()仅当流中的下一个读取不会被阻塞时会返回true
                 */
                if (nextChar >= nChars && in.ready()) {
                    fill();//缓冲区没有有效数据且下层输入流准备完时进行读取
                }
                if (nextChar < nChars) {
                    if (cb[nextChar] == '\n')//如果缓冲区内第一个有效字符是'\n'则跳过
                        nextChar++;
                    skipLF = false;
                }
            }
            return (nextChar < nChars) || in.ready();
        }
    }
```

mark/reset操作只需要操作内部变量就行了，另外readAheadLimit是在mark时输入的，当这个限制超过了缓冲区大小时会引起新分配一个不小于这个大小的缓冲区，因此要小心输入一个大值。

```java
    public void mark(int readAheadLimit) throws IOException {
        if (readAheadLimit < 0) {
            throw new IllegalArgumentException("Read-ahead limit < 0");
        }
        synchronized (lock) {
            ensureOpen();
            this.readAheadLimit = readAheadLimit;
            markedChar = nextChar;
            markedSkipLF = skipLF;
        }
    }

    public void reset() throws IOException {
        synchronized (lock) {
            ensureOpen();
            if (markedChar < 0)
                throw new IOException((markedChar == INVALIDATED)
                                      ? "Mark invalid"
                                      : "Stream not marked");
            nextChar = markedChar;
            skipLF = markedSkipLF;
        }
    }
```

close方法会关闭下层输入流并将in和cb设为null，所以关闭之后所有操作都会抛出异常，即使只涉及缓冲区，因为缓冲区被清除了。

```java
    public void close() throws IOException {
        synchronized (lock) {
            if (in == null)
                return;
            try {
                in.close();
            } finally {
                in = null;
                cb = null;
            }
        }
    }
```

最后lines()方法是JDK1.8引入了流式操作，具体流式操作的内容以后再讲吧，这里就翻译下注释：返回一个流，元素是从这个BufferedReader读取的行。流是延迟构成的，比如读取只发生在终止流操作中。reader在执行终止流操作期间不能被操作，否则，终止流的结果会不确定。如果访问下层BufferedReader时抛出了IOException，它会被包装为UncheckedIOException由Stream抛出。这个方法如果调用在一个关闭的BufferedReader上会返回stream，任何请求从关闭后的BufferedReader中读取的请求都会引起抛出UncheckedIOException。

# BufferedWriter

BufferedWriter是对应的缓冲字符输出流。将文本写到字符输出流，缓冲字符来提供单个字符、数组、字符串的高效写入。缓冲区的大小可能是指定的，或者默认大小8K，默认大小对于大多数情况足够大。

newLine()方法使用平台定义的行分隔符line.separator。不是所有平台使用'\n'来表示行结束。调用这个方法来结束每个输入行比直接写一个'\n'字符更好。

一般来说，一个Writer发送它的输出直接到下层的字符流或者字节流。除非要求立刻输出，否则建议用BufferedWriter来包装一个write()操作花费大的Writer，比如FileWriters和OutputStreamWriters。例如，PrintWriter out = new PrintWriter(new BufferedWriter(new FileWriter("foo.out")));会缓冲PrintWriter到一个文件的输出。没有缓冲，每一次调用print()方法会引起字符被转换成字节然后直接写入到文件，这样效率低。

含有一个底层输入流Writer，在构造时提供，并且Writer的锁对象会是它自身。

```java
	/**
	 * 下层输出流
	 */
    private Writer out;
    /**
     * 缓冲区
     */
    private char cb[];
    /**
     * 缓冲区大小
     */
    private int nChars;
    /**
     * 下一个有效字符
     */
    private int nextChar;

    private static int defaultCharBufferSize = 8192;

    /**
     * 行分隔符，在这个流创建时行分隔符的值
     */
    private String lineSeparator;

    /**
     * 创建一个使用默认大小8K输出缓冲区的缓冲字符输出流
     */
    public BufferedWriter(Writer out) {
        this(out, defaultCharBufferSize);
    }

    /**
     * 创建一个使用指定大小sz输出缓冲区的缓冲字符输出流
     */
    public BufferedWriter(Writer out, int sz) {
        super(out);
        if (sz <= 0)
            throw new IllegalArgumentException("Buffer size <= 0");
        this.out = out;
        cb = new char[sz];
        nChars = sz;
        nextChar = 0;

        lineSeparator = java.security.AccessController.doPrivileged(
            new sun.security.action.GetPropertyAction("line.separator"));
    }
```

ensureOpen内部方法用于确认这个流是否被关闭，标志是out为null

```java
    private void ensureOpen() throws IOException {
        if (out == null)
            throw new IOException("Stream closed");
    }
```

write(int c)写入单个字符，在缓冲区满了时需要将缓冲区内的字符写入到下层输入流

```java
    public void write(int c) throws IOException {
        synchronized (lock) {
            ensureOpen();
            if (nextChar >= nChars)
                flushBuffer();//如果缓冲区已满，将缓冲区内的字符写入到下层输出流
            cb[nextChar++] = (char) c;//将c复制到数组中
        }
    }
```

flushBuffer将这个输出缓冲区刷新到下层字符流中，但不刷新下层流本身。这个类不是private，所以可以被PrintStream调用

```java
    void flushBuffer() throws IOException {
        synchronized (lock) {
            ensureOpen();
            if (nextChar == 0)
                return;
            out.write(cb, 0, nextChar);//下层输出流输出缓冲区内的所有有效字符
            nextChar = 0;
        }
    }
```

write(char cbuf[], int off, int len)写入一个字符数组的一部分。一般来说这个方法从数组存储字符到流的缓冲区，如果需要的话刷新缓冲区到下层输出流。如果请求的长度超过了缓冲区大小，这个方法会刷新缓冲区并将字符直接写入到下层输出流。因此多余的BufferedWriter不会不必要的复制数据。

```java
    public void write(char cbuf[], int off, int len) throws IOException {
        synchronized (lock) {
            ensureOpen();
            if ((off < 0) || (off > cbuf.length) || (len < 0) ||
                ((off + len) > cbuf.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
            } else if (len == 0) {
                return;
            }

            if (len >= nChars) {
                /* 如果请求长度超过了输出缓冲区大小，刷新缓冲区直接写数据。这样缓冲输出流是级联无害的 */
                flushBuffer();
                out.write(cbuf, off, len);//直接使用下层输出流的write方法
                return;
            }

            int b = off, t = off + len;
            while (b < t) {
                int d = min(nChars - nextChar, t - b);
                System.arraycopy(cbuf, b, cb, nextChar, d);//复制字符到缓冲区，长度为缓冲区剩余空间和剩余要输入字符的较小值
                b += d;
                nextChar += d;
                if (nextChar >= nChars)
                    flushBuffer();//缓冲区满了时刷新
            }
        }
    }
```

write(String s, int off, int len)写一个字符串的一部分，一定会经过复制到缓冲区这个过程。需要注意的是，如果len参数是负数，没有字符会被写入。这个和java.io.Writer.write(java.lang.String,int,int)超类是相反的，它要求抛出IndexOutOfBoundsException

```java
    public void write(String s, int off, int len) throws IOException {
        synchronized (lock) {
            ensureOpen();

            int b = off, t = off + len;
            while (b < t) {
                int d = min(nChars - nextChar, t - b);
                s.getChars(b, b + d, cb, nextChar);
                b += d;
                nextChar += d;
                if (nextChar >= nChars)
                    flushBuffer();
            }
        }
    }
```

在上面的写入方法中，我们注意到有个min方法来获得两个数中的最小值，这里的min是自身的内部方法，避免加载java.lang.Math在用尽文件描述符和尝试打印堆栈追踪时

```java
    private int min(int a, int b) {
        if (a < b) return a;
        return b;
    }
```

newLine()写一个行分隔符，行分隔符通过系统属性line.separator来定义，不一定是一个'\n'字符

```java
    public void newLine() throws IOException {
        write(lineSeparator);
    }
```

flush刷新流，同时会刷新下层输入流

```java
    public void flush() throws IOException {
        synchronized (lock) {
            flushBuffer();
            out.flush();//下层输入流的刷新在这里调用
        }
    }
```

close关闭这个输出流和下层输出流，重复关闭没有效果。关闭后由于out和cb都为null，所以一切写入操作都会抛出异常。这里关闭out也是使用了JDK7引入的try-with-resources写法

```java
    public void close() throws IOException {
        synchronized (lock) {
            if (out == null) {//out为null说明已经被关闭了
                return;
            }
            try (Writer w = out) {//out随着这个代码块结束而关闭
                flushBuffer();//先做一次刷新，避免有没有输入到下层输出流的数据
            } finally {
                out = null;
                cb = null;//缓冲区被移除
            }
        }
    }
```

