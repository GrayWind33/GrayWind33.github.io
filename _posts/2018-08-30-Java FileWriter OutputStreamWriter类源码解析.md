---
layout:     post
title:      Java FileWriter OutputStreamWriter类源码解析
subtitle:   分析字符输出流FileWriter和OutputStreamWriter的实现原理
date:       2018-08-30
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

# FileWriter

因为篇幅原因，上一篇直接了字符输入流，今天来分析一下跟FileReader相对应的字符输出流FileWriter。FileWriter是将字符写入文件的通用类，**构造函数假定使用默认的字符编码和默认的字节缓冲区大小8K是使用者可以接受的，如果要指定这些值，需要通过一个FileOutputStream来构造FileWriter的父类OutputStreamWriter**。

文件是否有效或者是否能够被创建取决于平台，在一些平台上，对于同一个文件同一时间只允许一个FileWriter或者其他文件写入对象打开。在这种情况下，如果一个文件已经被打开，构造函数会抛出异常。

和FileReader类似，FileWriter也是除了构造函数以外全部是继承了父类的方法。先创建一个FileOutputStream，如果不给出append参数或者append为false则清空原文件从头开始写入，否则是从尾部开始扩展文件内容，使用文件描述符创建时必定是从文件头部开始写。然后通过FileOutputStream创建OutputStreamWriter

```java
    public FileWriter(String fileName) throws IOException {
        super(new FileOutputStream(fileName));
    }

    public FileWriter(String fileName, boolean append) throws IOException {
        super(new FileOutputStream(fileName, append));
    }

    public FileWriter(File file) throws IOException {
        super(new FileOutputStream(file));
    }

    public FileWriter(File file, boolean append) throws IOException {
        super(new FileOutputStream(file, append));
    }

    public FileWriter(FileDescriptor fd) {
        super(new FileOutputStream(fd));
    }
```

# OutputStreamWriter

OutputStreamWriter继承了抽象类Writer，跟OutputStreamReader类似，主要重写的方法都是基于StreamEncoder来完成，StreamEncoder在构造函数中通过工厂方法构造。

在构造函数中存在super(out)，作用是构造父类Writer，将OutputStream作为加锁的对象

```java
    //指定字符集名字
	public OutputStreamWriter(OutputStream out, String charsetName)
        throws UnsupportedEncodingException
    {
        super(out);
        if (charsetName == null)
            throw new NullPointerException("charsetName");
        se = StreamEncoder.forOutputStreamWriter(out, this, charsetName);//传给StreamEncoder的加锁对象是OutputStreamWriter对象自身
    }
    //使用默认字符集
    public OutputStreamWriter(OutputStream out) {
        super(out);
        try {
            se = StreamEncoder.forOutputStreamWriter(out, this, (String)null);
        } catch (UnsupportedEncodingException e) {
            throw new Error(e);
        }
    }
	//使用指定的字符集
    public OutputStreamWriter(OutputStream out, Charset cs) {
        super(out);
        if (cs == null)
            throw new NullPointerException("charset");
        se = StreamEncoder.forOutputStreamWriter(out, this, cs);
    }
    //使用指定的CharsetEncoder
    public OutputStreamWriter(OutputStream out, CharsetEncoder enc) {
        super(out);
        if (enc == null)
            throw new NullPointerException("charset encoder");
        se = StreamEncoder.forOutputStreamWriter(out, this, enc);
    }
```

OutputStreamWriter重写了Writer中的write, flush, close，此外Writer中的append方法最终也是基于write来完成的，这些方法都是直接调用StreamEncoder中的对应方法。

```java
	//写入单个字符
	public void write(int c) throws IOException {
        se.write(c);
    }
	//写入一部分字符数组
    public void write(char cbuf[], int off, int len) throws IOException {
        se.write(cbuf, off, len);
    }
	//写入一部分字符串
    public void write(String str, int off, int len) throws IOException {
        se.write(str, off, len);
    }
	//刷新
    public void flush() throws IOException {
        se.flush();
    }
	//关闭输出流
    public void close() throws IOException {
        se.close();
    }
```

getEncoding和flushBuffer是OutputStreamWriter相比于Writer新增的两个调用StreamEncoder中的中间接口

```java
	//返回字符集的名称
    public String getEncoding() {
        return se.getEncoding();
    }
	//只有PrintStream这样的类可以调用这个方法，将输出缓冲区的内容刷新到字节流中，但是字节流不会刷新到文件中
    void flushBuffer() throws IOException {
        se.flushBuffer();
    }
```

# StreamEncoder

跟上一篇一样，要分析字符输出流关键还是要分析StreamEncoder，很多方法可以对照StreamDecoder进行比较，它们从设计上除了输入和输出外是相近的。StreamEncoder同样一次至少操作两个字符，避免出现代替对，也就是2个字节码表示一个字符的特殊情况，当然前面提到过，这种通常是一些机器或者数学上的特殊符号，键盘输入是不会出现的。StreamEncoder继承了抽象类Writer，并重写了其中的write、flush、close方法。

StreamEncoder的构造函数也是private函数，外部只能通过工厂方法类调用

```java
	private Charset cs;
	private CharsetEncoder encoder;
	private ByteBuffer bb;

	// Exactly one of these is non-null至少有一个不为null
	private final OutputStream out;
	private WritableByteChannel ch;

	// Leftover first char in a surrogate pair代理对中剩下的第一个字符
	private boolean haveLeftoverChar = false;
	private char leftoverChar;
	private CharBuffer lcb = null;

	private StreamEncoder(OutputStream out, Object lock, Charset cs) {
		this(out, lock, cs.newEncoder().onMalformedInput(CodingErrorAction.REPLACE)// 有畸形输入错误时解码器丢弃错误的输入，替换为替代值然后继续后面的操作
				.onUnmappableCharacter(CodingErrorAction.REPLACE));// 有不可用图形表示的字符错误出现时解码器丢弃错误的输入，替换为替代值然后继续后面的操作
	}

	private StreamEncoder(OutputStream out, Object lock, CharsetEncoder enc) {
		super(lock);// lock是OutputStream对象本身
		this.out = out;
		this.ch = null;
		this.cs = enc.charset();
		this.encoder = enc;

		// 在堆外内存速度更快之前不使用这段代码
		if (false && out instanceof FileOutputStream) {
			ch = ((FileOutputStream) out).getChannel();
			if (ch != null)
				bb = ByteBuffer.allocateDirect(DEFAULT_BYTE_BUFFER_SIZE);
		}
		if (ch == null) {
			bb = ByteBuffer.allocate(DEFAULT_BYTE_BUFFER_SIZE);//分配一个8K的堆内ByteBuffer
		}
	}

	private StreamEncoder(WritableByteChannel ch, CharsetEncoder enc, int mbc) {
		this.out = null;
		this.ch = ch;
		this.cs = enc.charset();
		this.encoder = enc;
		this.bb = ByteBuffer.allocate(mbc < 0 ? DEFAULT_BYTE_BUFFER_SIZE : mbc);//分配一个大小mbc的堆内ByteBuffer
	}
```

在看工厂方法，这里传入的lock对象是OutputStreamWriter本身

```java
	// java.io.OutputStreamWriter工厂模式
	public static StreamEncoder forOutputStreamWriter(OutputStream out, Object lock, String charsetName)
			throws UnsupportedEncodingException {
		String csn = charsetName;
		if (csn == null)
			csn = Charset.defaultCharset().name();
		try {
			if (Charset.isSupported(csn))
				return new StreamEncoder(out, lock, Charset.forName(csn));
		} catch (IllegalCharsetNameException x) {
		}
		throw new UnsupportedEncodingException(csn);
	}

	public static StreamEncoder forOutputStreamWriter(OutputStream out, Object lock, Charset cs) {
		return new StreamEncoder(out, lock, cs);
	}

	public static StreamEncoder forOutputStreamWriter(OutputStream out, Object lock, CharsetEncoder enc) {
		return new StreamEncoder(out, lock, enc);
	}

	// java.nio.channels.Channels.newWriter工厂模式

	public static StreamEncoder forEncoder(WritableByteChannel ch, CharsetEncoder enc, int minBufferCap) {
		return new StreamEncoder(ch, enc, minBufferCap);
	}
```

内部属性isOpen标记流当前是否打开，在关闭时会被设置为close，实际操作方法都会先检查流是否开启，否则会抛出异常

```java
	private volatile boolean isOpen = true;

	private void ensureOpen() throws IOException {
		if (!isOpen)
			throw new IOException("Stream closed");
	}

	private boolean isOpen() {
		return isOpen;
	}
```

getEncoding返回字符集的历史名，没有的话返回官方名，这里的名字可能和构造时传入的有不同

```java
	public String getEncoding() {
		if (isOpen())
			return encodingName();
		return null;
	}

	String encodingName() {
		return ((cs instanceof HistoricallyNamedCharset) ? ((HistoricallyNamedCharset) cs).historicalName()
				: cs.name());
	}
```

接下来分析write部分，通过将字符编码为字节放入ByteBuffer中，然后通过文件通道或者输出流进行输出

```java
	public void write(int c) throws IOException {
		char cbuf[] = new char[1];
		cbuf[0] = (char) c;
		write(cbuf, 0, 1);
	}
	//这个是实际调用写入的方法
	public void write(char cbuf[], int off, int len) throws IOException {
		synchronized (lock) {
			ensureOpen();
			if ((off < 0) || (off > cbuf.length) || (len < 0) || ((off + len) > cbuf.length) || ((off + len) < 0)) {
				throw new IndexOutOfBoundsException();
			} else if (len == 0) {
				return;
			}
			implWrite(cbuf, off, len);
		}
	}

	public void write(String str, int off, int len) throws IOException {
		/* 创建字符缓冲区前检查长度 */
		if (len < 0)
			throw new IndexOutOfBoundsException();
		char cbuf[] = new char[len];
		str.getChars(off, off + len, cbuf, 0);// 将str中的value从off开始长度len的内容复制到cbuf中
		write(cbuf, 0, len);
	}

	void implWrite(char cbuf[], int off, int len) throws IOException {
		CharBuffer cb = CharBuffer.wrap(cbuf, off, len);// 将字符数组组装成一个堆内CharBuffer，数组中的内容不存在复制

		if (haveLeftoverChar)
			flushLeftoverChar(cb, false);//如果有的话，将leftoverChar写入输出流

		while (cb.hasRemaining()) {
			CoderResult cr = encoder.encode(cb, bb, false);//将字符编码为二进制字节直到ByteBuffer满或者CharBuffer中没有更多内容
			if (cr.isUnderflow()) {//ByteBuffer没有满，说明CharBuffer内的内容全部编码完成
				assert (cb.remaining() <= 1) : cb.remaining();
				if (cb.remaining() == 1) {
					//如果当前缓冲区仅剩一个字符，保存到leftoverChar并修改haveLeftoverChar状态，结束输出
					haveLeftoverChar = true;
					leftoverChar = cb.get();
				}
				break;
			}
			if (cr.isOverflow()) {//ByteBuffer满了
				assert bb.position() > 0;
				writeBytes();//将ByteBuffer的内容写入到输出流里面
				continue;
			}
			cr.throwException();
		}
	}

	private void writeBytes() throws IOException {
		bb.flip();//ByteBuffer准备输出当前内容，将limit设为当前位置，pos设为0
		int lim = bb.limit();
		int pos = bb.position();
		assert (pos <= lim);
		int rem = (pos <= lim ? lim - pos : 0);

		if (rem > 0) {//输出ByteBuffer中全部内容
			if (ch != null) {
				if (ch.write(bb) != rem)
					assert false : rem;
			} else {
				out.write(bb.array(), bb.arrayOffset() + pos, rem);
			}
		}
		bb.clear();//清空ByteBuffer
	}
```

implWrite调用了内部方法flushLeftoverChar，作用是将缓存的字符写入输出流，这个方法同时在close时也被调用，因为leftoverChar只有一个字符，而一次输出至少是两个字符，所以还要从cb中读取字符，保证写入的是2个字符

```java
	private void flushLeftoverChar(CharBuffer cb, boolean endOfInput) throws IOException {
		if (!haveLeftoverChar && !endOfInput)
			return;
		if (lcb == null)//lcb内一开始是空的
			lcb = CharBuffer.allocate(2);
		else
			lcb.clear();
		if (haveLeftoverChar)
			lcb.put(leftoverChar);//将leftoverChar放入lcb中
		if ((cb != null) && cb.hasRemaining())
			lcb.put(cb.get());//cb的内容复制到lcb中
		lcb.flip();//将limit设为当前位置，pos设为0，所以现在lcb要输出的内容就是刚才从leftoverChar（如果有的话）和cb里读入的
		while (lcb.hasRemaining() || endOfInput) {
			CoderResult cr = encoder.encode(lcb, bb, endOfInput);//将lcb的内容编码为字节尽可能多的放入到ByteBuffer中
			if (cr.isUnderflow()) {//cr未溢出，ByteBuffer还有剩余的空间
				if (lcb.hasRemaining()) {//lcb还有剩余的数据
					leftoverChar = lcb.get();
					if (cb != null && cb.hasRemaining())
						flushLeftoverChar(cb, endOfInput);
					return;
				}
				break;
			}
			if (cr.isOverflow()) {//cr溢出，超出了ByteBuffer的上限
				assert bb.position() > 0;//ByteBuffer中存在数据
				writeBytes();//将ByteBuffer中的数据写入到输出流后清空ByteBuffer
				continue;
			}
			cr.throwException();
		}
		haveLeftoverChar = false;
	}
```

flushBuffer()和flush()将ByteBuffer中的数据写入输出流，flush()同时还会进行输出流的刷新，具体操作取决于输出流的实现，比如FileOutputStream是什么也不做，因为没有缓冲区。

```java
	public void flushBuffer() throws IOException {
		synchronized (lock) {
			if (isOpen())
				implFlushBuffer();
			else
				throw new IOException("Stream closed");
		}
	}

	public void flush() throws IOException {
		synchronized (lock) {
			ensureOpen();
			implFlush();
		}
	}

	void implFlush() throws IOException {
		implFlushBuffer();
		if (out != null)
			out.flush();//这里out的刷盘操作取决于子类的具体实现
	}

	void implFlushBuffer() throws IOException {
		if (bb.position() > 0)// 如果ByteBuffer内还有剩余的数据，将它们写入文件
			writeBytes();
	}
```

close()同样也只能关闭一次，并且是线程同步方法，在关闭之前需要先将缓冲区的内容全部输入到输入流中

```java
	public void close() throws IOException {
		synchronized (lock) {
			if (!isOpen)
				return;
			implClose();
			isOpen = false;//只能关闭一次
		}
	}

	void implClose() throws IOException {
		flushLeftoverChar(null, true);//将leftoverChar的内容写入的输出流
		try {
			for (;;) {
				CoderResult cr = encoder.flush(bb);
				if (cr.isUnderflow())//cr未溢出说明ByteBuffer中的数据全部写入到输入流了
					break;
				if (cr.isOverflow()) {//cr溢出说明ByteBuffer仍然存在数据
					assert bb.position() > 0;
					writeBytes();//将ByteBuffer中的数据写入输出流
					continue;
				}
				cr.throwException();
			}

			if (bb.position() > 0)
				writeBytes();
			if (ch != null)//关闭文件通道或者输出流
				ch.close();
			else
				out.close();
		} catch (IOException x) {
			encoder.reset();
			throw x;
		}
	}
```

# 总结

基于OutputStreamWriter和StreamEncoder来实现字符输出时，保证字符编码为字节后输入到输入流中的操作是可靠的，并且直到最后一个字符前一次会传递成对的字符来解决代理对的问题，后续的部分由输入流来完成，而OutputStreamWriter在通过FileWriter构建时是基于FileOutputStream，也就是没有缓冲区，有多少内容就写多少。