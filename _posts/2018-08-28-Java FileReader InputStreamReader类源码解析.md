---
layout:     post
title:      Java FileReader InputStreamReader类源码解析
subtitle:   分析字符输入流FileReader和InputStreamReader的实现原理
date:       2018-08-28
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
    - IO流
---

# FileReader

前面介绍FileInputStream的时候提到过，它是从文件读取字节，如果要从文件读取字符的话可以使用FileReader。FileReader是可以便利读取字符文件的类，构造器只能使用默认的字符集编码（系统的默认字符集）、默认的bytebuffer大小8KB。如果想要自己指定这些值的话，可以直接通过FileInputStream构造一个InputStreamReader而不使用FileInputStream。

FileReader本身的代码其实没有什么可以分析的，就只有下面几行，它的操作全部是基于父类来进行的。传入文件路径名、具体的文件或者文件描述符来构造一个文件字节输入流，字符输入流是基于字节流上转换的。

```java
public class FileReader extends InputStreamReader {

    public FileReader(String fileName) throws FileNotFoundException {
        super(new FileInputStream(fileName));
    }

    public FileReader(File file) throws FileNotFoundException {
        super(new FileInputStream(file));
    }

    public FileReader(FileDescriptor fd) {
        super(new FileInputStream(fd));
    }

}
```

# InputStreamReader

然后我们来看下FileReader的父类InputStreamReader，它是从字节流到字符流的桥梁：它读取字节并使用特定的字符集解码成字符。字符集可能通过名字来确定或者直接特别给出或者是平台的默认字符集。

InputStreamReader的每一个read方法的调用可能会引起一个或多个字节从字节输出流中被读取。为了使字节能够有效的转换为字符，可能会提前从流中读取比当前读取操作所需字节数更多的字节。

为了提高效率，考虑将InputStreamReader嵌入到BufferedReader中，比如BufferedReader in = new BufferedReader(new InputStreamReader(System.in));

InputStreamReader继承了抽象类Reader，Reader中实现了一些具体方法，这些方法没有在InputStreamReader中重写，比如skip方法。InputStreamReader有一个核心内部变量StreamDecoder，这个类的作用是将输入的字节转换为字符，后面会具体分析。

InputStreamReader的构造函数有4种重载，必须的参数是InputStream，可选的是字符集参数可以输入字符集的名字、直接指定字符集或者构造一个CharsetDecoder作为参数

```java
	//创建一个使用默认字符集的InputStreamReader
    public InputStreamReader(InputStream in) {
        super(in);//Reader的lock是InputStream
        try {
            sd = StreamDecoder.forInputStreamReader(in, this, (String)null); // ## check lock object
        } catch (UnsupportedEncodingException e) {
            // The default encoding should always be available默认的字符集总是有效的，所以无参构造不会抛出
            throw new Error(e);
        }
    }
	//创建一个使用指定名字字符集的InputStreamReader
    public InputStreamReader(InputStream in, String charsetName)
        throws UnsupportedEncodingException
    {
        super(in);
        if (charsetName == null)
            throw new NullPointerException("charsetName");
        sd = StreamDecoder.forInputStreamReader(in, this, charsetName);
    }
    //创建一个使用给出的字符集的InputStreamReader
    public InputStreamReader(InputStream in, Charset cs) {
        super(in);
        if (cs == null)
            throw new NullPointerException("charset");
        sd = StreamDecoder.forInputStreamReader(in, this, cs);
    }
    //创建一个使用给出的字符集解码器的InputStreamReader
    public InputStreamReader(InputStream in, CharsetDecoder dec) {
        super(in);
        if (dec == null)
            throw new NullPointerException("charset decoder");
        sd = StreamDecoder.forInputStreamReader(in, this, dec);
    }
```

getEncoding方法通过StreamDecoder提供的getEncoding()返回这个流使用的字符编码名字，如果编码有历史名则返回它，如果没有的话返回官方名字。如果这个对象是通过InputStreamReader(InputStream, String)构造的，返回的名字可能给传给构造函数的不同，如果流已经被关闭会返回null。

```java
    public String getEncoding() {
        return sd.getEncoding();
    }
```

read、ready和close方法都是直接调用StreamDecoder对应的方法

```java
	//读取单个字符
    public int read() throws IOException {
        return sd.read();
    }
	//读取字符到一个数组中，返回读取的字符数，如果开始前就已经到达末尾则返回-1
    public int read(char cbuf[], int offset, int length) throws IOException {
        return sd.read(cbuf, offset, length);
    }
	//该流是否准备完毕读取。当输入缓冲区是非空时，或者字节能够从下方的字节流读取时，InputStreamReader是准备完的
    public boolean ready() throws IOException {
        return sd.ready();
    }
	//关闭输入流，释放资源
    public void close() throws IOException {
        sd.close();
    }
```

# Reader

再来看InputStreamReader的父类Reader，它是一个字符读取流的抽象类，子类必须实现的方法只有read(char[], int, int)和close()。但是大部分子类会重写这里定义的一些方法来获取更高的效率或者更多的功能。实现了两个接口：Readable是字符来源，实现了这个接口字符可以通过CharBuffer来读取，Closeable接口代表实现类对象可以关闭来释放资源，比如打开的文件。根据对这个类和InputStreamReader的分析，**InputStreamReader直接使用了Reader的skip方法也就是读取到内存后丢弃，所以比起能够直接位移的实现方法效率不佳，此外也不能往回跳和使用mark/reset**。

存在一个内部变量lock，这个对象用于流中的同步操作。为了提高效率，一个字符流对象可能使用一个对象而不是它自己来保护临界区。因此子类应该使用这个对象而不是this或者synchronized方法。InputStreamReader传入作为lock对象的是自己对象本身。

构造函数可以传入指定的lock对象，不传入的话使用对象自身

```java
    protected Object lock;
	//创建一个新的字符流reader，它的临界区依靠它自己来同步
    protected Reader() {
        this.lock = this;
    }
	//创建一个新的字符流reader，它的临界区依靠提供的对象来同步
    protected Reader(Object lock) {
        if (lock == null) {
            throw new NullPointerException();
        }
        this.lock = lock;
    }
```

read方法根据重载的输入参数不同，可以读取单个字符也一个读取多个字符到数组或者抽象类CharBuffer中，但是最终这些方法都是基于抽象方法read(char cbuf[], int off, int len)，也是子类必须要实现的方法之一

```java
	//尝试读取字符到具体的字符缓冲区中，缓冲区作为字符的仓库：唯一的变更是put操作的结果，没有翻转或者倒回的操作。
    public int read(java.nio.CharBuffer target) throws IOException {
        int len = target.remaining();//读取长度是缓冲区剩余的空间，也就是尽量填满缓冲区
        char[] cbuf = new char[len];
        int n = read(cbuf, 0, len);//将字符读取到char数组中，实现随子类决定，n=读取到的字符数
        if (n > 0)
            target.put(cbuf, 0, n);//将数组中的字符复制到CharBuffer，根据CharBuffer子类的实现方法具体操作不同
        return n;
    }
	//读取一个单一的字符，这个方法会阻塞，直到有一个有效字符，或者一个IO错误发生或者，或者到达了流的尾端
    public int read() throws IOException {
        char cb[] = new char[1];
        if (read(cb, 0, 1) == -1)
            return -1;
        else
            return cb[0];
    }
	//将字符读取到数组中
    public int read(char cbuf[]) throws IOException {
        return read(cbuf, 0, cbuf.length);
    }

    abstract public int read(char cbuf[], int off, int len) throws IOException;
```

Reader默认实现了skip方法，跳过n个字符，最大一次跳8KB，不能回跳，InputStreamReader没有重写这个方法。通过将字符读取到skipBuffer数组然后丢弃来实现，所以会增加gc的工作量，效率不佳。

```java
    /** 最大跳过缓冲区大小8K */
    private static final int maxSkipBufferSize = 8192;

    /** 跳过缓冲区在分配前为null */
    private char skipBuffer[] = null;

    public long skip(long n) throws IOException {
        if (n < 0L)//跳过负数会抛出异常，也就是不能回跳，这点和FileInputStream不同
            throw new IllegalArgumentException("skip value is negative");
        int nn = (int) Math.min(n, maxSkipBufferSize);//最多跳过8KB
        synchronized (lock) {//skip操作是同步的
            if ((skipBuffer == null) || (skipBuffer.length < nn))
                skipBuffer = new char[nn];
            long r = n;
            while (r > 0) {
                int nc = read(skipBuffer, 0, (int)Math.min(r, nn));//通过读取后丢弃来实现跳跃
                if (nc == -1)
                    break;
                r -= nc;
            }
            return n - r;
        }
    }
```

ready告知这个流是否准备完读取数据，若不重写则永远返回false

```java
    public boolean ready() throws IOException {
        return false;
    }
```

Reader默认也不支持mark/reset，所以以下几个方法不重写无法使用

```java
	//告知这个流是否支持mark()操作
    public boolean markSupported() {
        return false;
    }
	//标记当前位置，然后可以通过reset()回跳
    public void mark(int readAheadLimit) throws IOException {
        throw new IOException("mark() not supported");
    }

    public void reset() throws IOException {
        throw new IOException("reset() not supported");
    }
```

close关闭流并释放相关的系统资源，一旦流被关闭，其他操作会抛出异常IOException。关闭一个已经关闭的流没有作用。是必须实现的两个类之一。

```java
     abstract public void close() throws IOException;
```

# StreamDecoder

最后要来分析的是sun.nio.cs.StreamDecoder，这个包里的源码在OracleJDK里是没有的，所以我去找了OpenJDK8里对应的源码来分析。这个类的作用是将字节解析为字符的解码器，继承了抽象类Reader。因为eclipse无法在sun包里的代码中加断点，所以只能肉眼调试了，可能以下分析的具体操作会有些出入，但总体思路应该问题不大。

StreamDecoder一次至少要读取两个字符，如果调用者只需要一个字符，那么会将一个字符缓存在leftoverChar，下次需要读取时再加入到返回内容中。所以，**使用了这个StreamDecoder的对象只能采取读取后丢弃的方式进行skip，否则会出现内容错乱**。

```java
	// 为了解决替换问题我们决不能尝试一次产生少于两个字符。如果我们只要求返回一个字符，另外一个会存在这里之后再返回
	private boolean haveLeftoverChar = false;
	private char leftoverChar;
```

StreamDecoder只有在流关闭前才能工作，关闭后的操作会抛出IOException

```java
	private volatile boolean isOpen = true;// 流是否打开

	private void ensureOpen() throws IOException {
		if (!isOpen)
			throw new IOException("Stream closed");
	}
```

StreamDecoder自己的构造函数是一个包所有权的方法，所以包外的类不能够直接使用构造函数

```java
	private Charset cs;
	private CharsetDecoder decoder;
	private ByteBuffer bb;

	// 下面两个有一个不是null
	private InputStream in;
	private ReadableByteChannel ch;

	StreamDecoder(InputStream in, Object lock, Charset cs) {
		this(in, lock, cs.newDecoder().onMalformedInput(CodingErrorAction.REPLACE)// 有畸形输入错误时解码器丢弃错误的输入，替换为替代值然后继续后面的操作
				.onUnmappableCharacter(CodingErrorAction.REPLACE));// 有不可用图形表示的字符错误出现时解码器丢弃错误的输入，替换为替代值然后继续后面的操作
	}
	//实际执行的构造函数
	StreamDecoder(InputStream in, Object lock, CharsetDecoder dec) {
		super(lock);// lock是InputStreamReader对象本身
		this.cs = dec.charset();
		this.decoder = dec;

		// 在directbuffer更快前不会进入这个代码块，实际上因为堆外内存的操作速度不如堆内内存所以这段是被弃用的
		if (false && in instanceof FileInputStream) {
			ch = getChannel((FileInputStream) in);
			if (ch != null)
				bb = ByteBuffer.allocateDirect(DEFAULT_BYTE_BUFFER_SIZE);
		}
		if (ch == null) {
			this.in = in;
			this.ch = null;
			bb = ByteBuffer.allocate(DEFAULT_BYTE_BUFFER_SIZE);//分配一个大小为8K的堆内ByteBuffer
		}
		bb.flip(); // 为初始状态为空
		/*flip的作用有两个：
		1. 把limit设置为当前的position值
		2. 把position设置为0
		然后处理的数据就是从position到limit直接的数据，也就是你刚刚读取过来的数据*/
	}
	//从ReadableByteChannel读取数据
	StreamDecoder(ReadableByteChannel ch, CharsetDecoder dec, int mbc) {
		this.in = null;
		this.ch = ch;
		this.decoder = dec;
		this.cs = dec.charset();
		this.bb = ByteBuffer.allocate(
				mbc < 0 ? DEFAULT_BYTE_BUFFER_SIZE : (mbc < MIN_BYTE_BUFFER_SIZE ? MIN_BYTE_BUFFER_SIZE : mbc));//mbc是ByteBuffer初始大小，为负数时取8KB，小于32B时取32B
		bb.flip();
	}
```

我们可以看到在InputStreamReader中，构造是通过工厂模式StreamDecoder.forInputStreamReader(in, this, charsetName)来完成的，这里会调用构造函数来构造StreamDecoder对象。这里forInputStreamReader是用于InputStreamReader的，而forDecoder则是用于java.nio.channels.Channels.newReader

```java
	// java.io.InputStreamReader工厂模式

	public static StreamDecoder forInputStreamReader(InputStream in, Object lock, String charsetName)
			throws UnsupportedEncodingException {
		String csn = charsetName;
		if (csn == null)
			csn = Charset.defaultCharset().name();// 若没有给出字符集名字则使用平台默认字符集
		try {
			if (Charset.isSupported(csn))
				return new StreamDecoder(in, lock, Charset.forName(csn));// lock是InputStreamReader对象本身
		} catch (IllegalCharsetNameException x) {
		}
		throw new UnsupportedEncodingException(csn);
	}

	public static StreamDecoder forInputStreamReader(InputStream in, Object lock, Charset cs) {
		return new StreamDecoder(in, lock, cs);
	}

	public static StreamDecoder forInputStreamReader(InputStream in, Object lock, CharsetDecoder dec) {
		return new StreamDecoder(in, lock, dec);
	}

	// java.nio.channels.Channels.newReader工厂模式

	public static StreamDecoder forDecoder(ReadableByteChannel ch, CharsetDecoder dec, int minBufferCap) {
		return new StreamDecoder(ch, dec, minBufferCap);
	}
```

read是一个线程安全的操作，可以读取单个字符，也可以读取多个字符，读取前要先检查是否有缓存的字符，如果除了缓存的字符外没有其他需求则直接返回，否则要使用implRead(char[], int, int)来进行实际的读取

```java
	public int read() throws IOException {
		return read0();
	}

	@SuppressWarnings("fallthrough")
	private int read0() throws IOException {
		synchronized (lock) {

			// 如果缓存中有未返回的字符则直接返回并清空缓存
			if (haveLeftoverChar) {
				haveLeftoverChar = false;
				return leftoverChar;
			}

			// Convert more bytes
			char cb[] = new char[2];
			int n = read(cb, 0, 2);// 尝试读取两个字符
			switch (n) {
			case -1:
				return -1;// 已到文件结束符返回-1
			case 2:
				leftoverChar = cb[1];// 读取了2个字符，缓存第二个字符
				haveLeftoverChar = true;
				// FALL THROUGH继续进入case1
			case 1:
				return cb[0];// 返回第一个字符
			default:
				assert false : n;
				return -1;
			}
		}
	}

	public int read(char cbuf[], int offset, int length) throws IOException {
		int off = offset;
		int len = length;
		synchronized (lock) {// 同时只能有一个线程进行read操作
			ensureOpen();// 确保流是打开的
			if ((off < 0) || (off > cbuf.length) || (len < 0) || ((off + len) > cbuf.length) || ((off + len) < 0)) {
				throw new IndexOutOfBoundsException();
			}
			if (len == 0)
				return 0;

			int n = 0;

			if (haveLeftoverChar) {
				// 将leftover缓存中的字符复制到数组中
				cbuf[off] = leftoverChar;
				off++;
				len--;
				haveLeftoverChar = false;
				n = 1;
				if ((len == 0) || !implReady())
					// 如果不需要更多字符或者读到没有剩余数据则返回
					return n;
			}

			if (len == 1) {
				// 只读一个字符时调用read0，视为读取两个缓存一个
				int c = read0();
				if (c == -1)
					return (n == 0) ? -1 : n;
				cbuf[off] = (char) c;
				return n + 1;
			}

			return n + implRead(cbuf, off, off + len);// 直接调用read()不会进入前两个if代码块，返回实际读取的字符数
		}
	}
```

read方法调用了implRead进行读取，先通过readBytes将字节尽可能多地读取到ByteBuffer中，然后通过CharsetDecoder.decode解码为字符

```java
	int implRead(char[] cbuf, int off, int end) throws IOException {

		//为了处理替代对，这个方法要求调用者试图读取至少两个字符，如果有的话保存多余字符，在更高的层次比在这里更容易处理这个问题。
		assert (end - off > 1);

		CharBuffer cb = CharBuffer.wrap(cbuf, off, end - off);
		if (cb.position() != 0)
			// Ensure that cb[0] == cbuf[off]
			cb = cb.slice();

		boolean eof = false;//decode的第三个参数只有在调用者确保除了buffer中的字节外没有其他字节了才是true
		for (;;) {
			CoderResult cr = decoder.decode(bb, cb, eof);//将ByteBuffer内的字节解码存入CharBuffer
			if (cr.isUnderflow()) {//向下溢出，CharBuffer没有填满
				if (eof)
					break;
				if (!cb.hasRemaining())
					break;
				if ((cb.position() > 0) && !inReady())
					break; // 最多阻塞一次
				int n = readBytes();//将字节尽可能多地读取到ByteBuffer，返回读取的字节数
				if (n < 0) {
					eof = true;//已经到结束符了
					if ((cb.position() == 0) && (!bb.hasRemaining()))
						break;
					decoder.reset();//重置decoder，清除内部状态
				}
				continue;
			}
			if (cr.isOverflow()) {//向上溢出，CharBuffer满了
				assert cb.position() > 0;
				break;
			}
			cr.throwException();
		}

		if (eof) {
			// ## Need to flush decoder
			decoder.reset();
		}

		if (cb.position() == 0) {
			if (eof)
				return -1;
			assert false;
		}
		return cb.position();
	}

	private int readBytes() throws IOException {
		bb.compact();//使ByteBuffer中的字节变得紧密连接，如果从有字节是在position到limit的位置，把它们复制到头上去，使得position和capacity保持一致
		try {
			if (ch != null) {
				// 从通道中读取字节填满ByteBuffer或者文件中没有剩余数据
				int n = ch.read(bb);
				if (n < 0)
					return n;
			} else {
				// 从流中读取，更新缓冲区
				int lim = bb.limit();
				int pos = bb.position();
				assert (pos <= lim);//pos>lim会直接抛出异常
				int rem = (pos <= lim ? lim - pos : 0);//剩余的字节数
				assert rem > 0;
				int n = in.read(bb.array(), bb.arrayOffset() + pos, rem);//从InputStream中将全部剩余字节读取到ByteBuffer直到缓冲区所需的内容装满
				if (n < 0)
					return n;
				if (n == 0)
					throw new IOException("Underlying input stream returned zero bytes");//返回0说明有异常发生，流中没数据返回的是-1
				assert (n <= rem) : "n = " + n + ", rem = " + rem;
				bb.position(pos + n);
			}
		} finally {
			// Flip even when an IOException is thrown,
			// otherwise the stream will stutter
			bb.flip();
		}

		int rem = bb.remaining();
		assert (rem != 0) : rem;
		return rem;
	}
```

ready方法被重写了，检查缓冲区或者文件中是否有可以读取的数据

```java
	public boolean ready() throws IOException {
		synchronized (lock) {
			ensureOpen();
			return haveLeftoverChar || implReady();// 缓存中有字符或者文件中还有剩余数据
		}
	}

	boolean implReady() {
		return bb.hasRemaining() || inReady();//ByteBuffer中有剩余内容或者输入流中还有剩余内容
	}

	private boolean inReady() {
		try {
			return (((in != null) && (in.available() > 0)) || (ch instanceof FileChannel)); // ## RBC.available()?
		} catch (IOException x) {
			return false;
		}
	}
```

close方法将isOpen设为false，然后关闭ReadableByteChannel或者InputStream，重复调用不会生效。

```java
	public void close() throws IOException {
		synchronized (lock) {
			if (!isOpen)
				return;
			implClose();
			isOpen = false;
		}
	}

	void implClose() throws IOException {
		if (ch != null)
			ch.close();
		else
			in.close();
	}
```

getChannel获取文件通道有重复调用失败立即退出的机制

```java
	// 在早期版本中还没有构建完NIO的native代码，为了保证第一次尝试中捕捉到UnsatisfiedLinkError后面再尝试会立即失败，所以有了这个标记
	private static volatile boolean channelsAvailable = true;

	private static FileChannel getChannel(FileInputStream in) {
		if (!channelsAvailable)
			return null;
		try {
			return in.getChannel();
		} catch (UnsatisfiedLinkError x) {
			channelsAvailable = false;
			return null;
		}
	}
```

总之StreamDecoder就是将读取到的字节转换为字符，然后返回。理论上来说除了解码的具体实现需要依赖底层实现外其他自己重写应该问题不大，一次至少读取两个字符是为了处理替代对。关于用InputStreamReader和使用字节流读取后再new String转换成字符哪个比较快有待测试。