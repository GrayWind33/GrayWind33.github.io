---
layout:     post
title:      Java PipedInputStream PipedOutputStream类源码解析
subtitle:   分析管道字节流PipedInputStream和PipedOutputStream的实现原理
date:       2018-09-10
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

管道流主要是用于不同线程间的数据交互，可以通过一个PipedInputStream和一个PipedOutputStream相互连接来进行通信，**从PipedOutputStream写入字节到PipedInputStream中，所以PipedOutputStream是writer端，PipedInputStream是reader端**。

**一个PipedInputStream只能与一个PipedOutputStream连接**，但是可以通过多个线程共享同一个管道来达到复数生产者和复数消费者的模式，同一个角色相互之间会存在竞争。下面的例子是一个writer和两个reader共用同一个管道流，可以看到两个reader之间根据线程调度随机接收一部分数据。

```java
public class PipeStreamTest implements Runnable {
	private PipedInputStream in;
	private PipedOutputStream out;

	public PipeStreamTest(PipedInputStream in) {
		this.in = in;
	}

	public PipeStreamTest(PipedOutputStream out) {
		this.out = out;
	}

	@Override
	public void run() {
		if (this.in != null) {
			boolean flag = true;
			while (flag) {
				try {
					int a = in.read();
					if (a == -1) {
						flag = false;
						break;
					}
					System.out.println(Thread.currentThread().getName() + " calculate " + String.valueOf(2 * a + 1));
				} catch (IOException e) {
					e.printStackTrace();
					flag = false;
				}
			}
		} else {
			try {
				Thread.sleep(1000);// 这里阻塞了out，导致in内没有可读取数据被阻塞
			} catch (InterruptedException e1) {
				e1.printStackTrace();
			}
			int a = 5;
			try {
				for (int i = 0; i < 5; i++) {
					out.write(a + i);
					System.out.println(Thread.currentThread().getName() + " write " + String.valueOf(a));
				}
				out.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}

	public static void main(String args[]) throws IOException {
		PipedInputStream in = new PipedInputStream(10);
		PipedOutputStream out = new PipedOutputStream(in);
		new Thread(new PipeStreamTest(in)).start();
		new Thread(new PipeStreamTest(in)).start();
		new Thread(new PipeStreamTest(out)).start();
		/*Thread-0 calculate 11
		Thread-2 write 5
		Thread-2 write 5
		Thread-0 calculate 13
		Thread-2 write 5
		Thread-2 write 5
		Thread-2 write 5
		Thread-0 calculate 15
		Thread-0 calculate 19
		Thread-1 calculate 17*/
	}
}
```

对run()进行一些修改，阻塞reader线程，使得writer可以写满缓存区，此时writer会被阻塞，等待有reader读取数据后缓冲区有空余空间再写入，由于Thread.sleep不会被notifyAll()影响，所以在写入了10bytes数据后会存在5s的明显间隔

```java
	public void run() {
		if (this.in != null) {
			try {
				Thread.sleep(5000);// 这里阻塞了in，使得out会因没有空间写入而阻塞
			} catch (InterruptedException e1) {
				e1.printStackTrace();
			}
			boolean flag = true;
			while (flag) {
				try {
					int a = in.read();
					if (a == -1) {
						flag = false;
						break;
					}
					System.out.println(Thread.currentThread().getName() + " calculate " + String.valueOf(2 * a + 1));
				} catch (IOException e) {
					e.printStackTrace();
					flag = false;
				}
			}
		} else {
			try {
				Thread.sleep(1000);// 这里阻塞了out，导致in内没有可读取数据被阻塞
			} catch (InterruptedException e1) {
				e1.printStackTrace();
			}
			int a = 5;
			try {
				for (int i = 0; i < 15; i++) {
					out.write(a + i);
					System.out.println(Thread.currentThread().getName() + " write " + String.valueOf(a));
				}
				out.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}
/*
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
Thread-2 write 5
这里出现明显的间隔
Thread-0 calculate 11
Thread-0 calculate 15
Thread-2 write 5
Thread-1 calculate 13
Thread-1 calculate 19
Thread-1 calculate 21
Thread-1 calculate 23
Thread-1 calculate 25
Thread-1 calculate 27
Thread-1 calculate 29
Thread-2 write 5
Thread-2 write 5
Thread-0 calculate 17
Thread-0 calculate 33
Thread-0 calculate 35
Thread-0 calculate 37
Thread-2 write 5
Thread-2 write 5
Thread-1 calculate 31
Thread-0 calculate 39*/
```

# PipedInputStream

下面先来分析一下管道字节输入流PipedInputStream，它继承了InputStream。一个管道输入流需要连接一个管道输出流，管道输入流会提供任何数据字节写入管道输出流。典型地，数据被一个线程从一个PipedInputStream对象中读取，然后数据被另一个线程写到对应的PipedOutputStream。试图由一个线程使用所有对象是不推荐的，这可能导致线程死锁。

管道输入流含有一个缓冲区来解耦一个读操作和写操作。一个管道如果提供数据给相连接的管道输出流的线程不再活动，这个管道状态被称为BROKEN，也就是说如果上面的例子中把in对应的线程循环去掉使它运行完后被终止，此时out再尝试写入就会出现管道BROKEN的异常。

先来看一下跟管道状态有关的内部变量，流的关闭需要输入和输出端都关闭，所以需要两个变量来标志。同时，所有读写操作都需要管道两端建立连接，connected表示是否有连接建立。最后两个线程是输入和输出的操作线程，由于接收需要读取线程是运行状态，读取在缓冲区内为空时需要

```java
    /**
     * 输出流关闭，仅PipedOutputStream.close()调用receivedLast()方法会将它置为true
     */
	boolean closedByWriter = false;
	/**
	 * 输入流关闭，仅PipedInputStream.close()会将它置为true
	 */
    volatile boolean closedByReader = false;
    /**
     * 是否有一对输入输出流相互连接
     */
    boolean connected = false;

        /* 识别读和写两边需要更加复杂。使用线程组（但是管道在一个线程中怎么办）或者使用final化（但是可能到下一次GC的时间更长） */
    Thread readSide;//input流的线程
    Thread writeSide;//output流的线程
```

下面这些内部变量和缓冲区有关。buffer数组是循环缓冲区，这里有in和out两个下标指针，in负责接收，out负责读取，两个下标都是循环的如果达到末端会重新回到0的位置，读取不能超过接收数据的范围。

```java
    private static final int DEFAULT_PIPE_SIZE = 1024;

    /**
     * 管道循环输入缓冲区默认大小1K
     */
    // 这个值在管道大小允许改变前被用作常数。这个值会为了向下兼容性被持续保持
    protected static final int PIPE_SIZE = DEFAULT_PIPE_SIZE;

    /**
     * 循环缓冲区，接下来的数据会被放入其中
     */
    protected byte buffer[];

    /**
     * 循环缓冲区中下一个从连接的管道输出流接收的字节数据将会被存储的位置下标。in<0说明缓冲区是空的，in==out说明缓冲区满了
     */
    protected int in = -1;

    /**
     * 循环缓冲区中下一个将会被这个管道输入流读取的字节下标位置。
     */
    protected int out = 0;
```

构造函数方面，可以直接给出管道输出流直接在构造时建立连接，也可以先构造输入流在需要使用时再连接。循环缓冲区的大小可以使用默认的1K，也可以手动指定大小。connect方法用于建立两端的连接，可以在构造时调用也可以手动调用。

```java
    /**
     * 创建一个PipedInputStream连接到管道输出流src。数据字节写入到src中将会成为这个流的输入。
     */
    public PipedInputStream(PipedOutputStream src) throws IOException {
        this(src, DEFAULT_PIPE_SIZE);
    }

    /**
     * 跟上面相比这个管道缓冲区的大小是指定的，其他相同
     */
    public PipedInputStream(PipedOutputStream src, int pipeSize)
            throws IOException {
         initPipe(pipeSize);
         connect(src);
    }

    /**
     * 创建一个还没有连接的PipedInputStream，在使用前必须连接到一个PipedOutputStream
     */
    public PipedInputStream() {
        initPipe(DEFAULT_PIPE_SIZE);
    }

    /**
     * 创建一个还没有连接的PipedInputStream指定它的缓冲区大小，在使用前必须连接到一个PipedOutputStream
     */
    public PipedInputStream(int pipeSize) {
        initPipe(pipeSize);
    }

    private void initPipe(int pipeSize) {
         if (pipeSize <= 0) {
            throw new IllegalArgumentException("Pipe Size <= 0");
         }
         buffer = new byte[pipeSize];
    }

    /**
     * 引起这个管道输入流连接到管道输出流src。如果这个对象已经连接到某个其他的管道输出流会抛出IOException。
     * 如果src是一个未连接的管道输出流，snk是一个未连接的管道输入流，它们可以通过snk.connect(src)或者src.connect(snk)连接，两者效果相同。
     */
    public void connect(PipedOutputStream src) throws IOException {
        src.connect(this);
    }
```

receive方法将字节写入到缓冲区，这是protected方法，再没有继承类的情况下由PipedOutputStream.write方法调用，接收字节会导致下标指针in增加，如果到达右边界，会重置为0。如果当前缓冲区没有空余的空间，这个方法会被阻塞，直到有线程读取了数据使得有空间写入时再继续。

```java
    /**
     * 接收一字节数据，这个方法如果没有有效输入时会阻塞。
     */
    protected synchronized void receive(int b) throws IOException {
        checkStateForReceive();//检查管道状态
        writeSide = Thread.currentThread();//写线程设为当前线程
        if (in == out)
            awaitSpace();//缓冲区满了，通知所有线程使得读取端读取字节给当前流缓冲区空出位置来接收
        if (in < 0) {//in<0表示当前缓冲区为空
            in = 0;
            out = 0;
        }
        buffer[in++] = (byte)(b & 0xFF);//将b存储到缓冲区中
        if (in >= buffer.length) {
            in = 0;//因为是循环缓冲区，所以in超过buffer.length时重新回到0的位置
        }
    }

    /**
     * 将数据接收到一个字节数组中，这个方法会阻塞到一些输入变得有效。
     */
    synchronized void receive(byte b[], int off, int len)  throws IOException {
        checkStateForReceive();//检查管道状态
        writeSide = Thread.currentThread();//写线程设为当前线程
        int bytesToTransfer = len;
        while (bytesToTransfer > 0) {//还有需要写入的字节
            if (in == out)
                awaitSpace();//缓冲区满了，通知其他线程读取字节空出空间
            int nextTransferAmount = 0;
            if (out < in) {//out<in说明out到in这段是未读取的数据，所以空余空间是buffer.length-in
                nextTransferAmount = buffer.length - in;
            } else if (in < out) {
                if (in == -1) {
                	//当前缓冲区为空，可用空间为buffer.length
                    in = out = 0;
                    nextTransferAmount = buffer.length - in;
                } else {
                	//in已经到达一次右边界重置为0之后in<out，out到右边界是未读取内容，in不能超过out的值，所以空余空间为out-in
                    nextTransferAmount = out - in;
                }
            }
            if (nextTransferAmount > bytesToTransfer)
                nextTransferAmount = bytesToTransfer;//要读取的字节数是剩余空间和参数指定长度间的较小值
            assert(nextTransferAmount > 0);
            System.arraycopy(b, off, buffer, in, nextTransferAmount);//将接收的字节复制到缓冲区in开始的位置
            bytesToTransfer -= nextTransferAmount;
            off += nextTransferAmount;
            in += nextTransferAmount;//in增加读取的字节数
            if (in >= buffer.length) {
                in = 0;//in到达缓冲区边界时重置为0
            }
        }
    }
```

然后看下这用到的两个内部方法。checkStateForReceive这个内部方法，确定当前有连接，管道两端都没有被关闭，有活动的读取端线程。当in==out时说明缓冲区满了不能再写入，需要有线程读取使得out增大之后才能再写入，awaitSpace会唤醒所有的等待读取线程，然后等待1s，再检查in==out是否成立，不断循环这个过程。

```java
    private void checkStateForReceive() throws IOException {
        if (!connected) {
            throw new IOException("Pipe not connected");
        } else if (closedByWriter || closedByReader) {
            throw new IOException("Pipe closed");
        } else if (readSide != null && !readSide.isAlive()) {
            throw new IOException("Read end dead");
        }
    }

    private void awaitSpace() throws IOException {
        while (in == out) {
            checkStateForReceive();

            /* full: kick any waiting readers */
            notifyAll();//唤醒所有线程，使得有reader读取字节将当前流缓冲区空出来
            try {
                wait(1000);//当前线程等待1s
            } catch (InterruptedException ex) {
                throw new java.io.InterruptedIOException();
            }
        }
    }
```

read方法从缓冲区中读取字节，要求存在连接且读取端没有关闭，如果缓冲区内没有可读取的字节则还需要writer端线程是活跃的。读取多个字节时会先尝试允许阻塞读取一个字节，成功后再读取后面部分，此时不会再阻塞等待所以如果要读取的长度太长则只读取缓冲区内所有可读取的内容，返回的是实际读取的字节数。

```java
    /**
     * 从这个管道输入流读取下一个字节。返回值作为一个整数范围在0-255之间。这个方法一直阻塞到输入数据变得有效，或者探知到流结束或者抛出异常。
     */
    public synchronized int read()  throws IOException {
        //存在连接，读取端没有关闭，缓存区为空时写入端线程必须活动
    	if (!connected) {
            throw new IOException("Pipe not connected");
        } else if (closedByReader) {
            throw new IOException("Pipe closed");
        } else if (writeSide != null && !writeSide.isAlive()
                   && !closedByWriter && (in < 0)) {
            throw new IOException("Write end dead");
        }

        readSide = Thread.currentThread();//读取端线程设为当前线程
        int trials = 2;//加起来等待2s，第三次循环缓冲区依然为空没有写入数据则认为管道BROKEN
        while (in < 0) {
            if (closedByWriter) {
                /* 输出管道被writer关闭返回-1 */
                return -1;
            }
            if ((writeSide != null) && (!writeSide.isAlive()) && (--trials < 0)) {
                throw new IOException("Pipe broken");
            }
            /* 可能有一个writer在等待 */
            notifyAll();
            try {
                wait(1000);
            } catch (InterruptedException ex) {
                throw new java.io.InterruptedIOException();
            }
        }
        int ret = buffer[out++] & 0xFF;//读取out位置的字节并增加out
        if (out >= buffer.length) {
            out = 0;//如果out到达buffer右边界，将它重置为0
        }
        if (in == out) {
            /* in==out说明缓冲区空了 */
            in = -1;
        }

        return ret;
    }

    /**
     * 从这个管道输入流读取最大len程度的字节数据到字节数组中。如果到达了数据量末端或者len超过了管道缓冲区大小，少于len字节的数据会被读取。
     * 如果len是0，没有数据会被读取返回0；否则这个方法会阻塞直到至少1字节输入是有效的，或者到达了流末端或者抛出异常。
     */
    public synchronized int read(byte b[], int off, int len)  throws IOException {
    	//存在连接，读取端没有关闭，缓存区为空时写入端线程必须活动
    	if (b == null) {
            throw new NullPointerException();
        } else if (off < 0 || len < 0 || len > b.length - off) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return 0;
        }

        /* 可能等待第一个字节 */
        int c = read();//尝试读取一个字节
        if (c < 0) {
            return -1;//没有读取到直接返回-1
        }
        b[off] = (byte) c;//将读取到的字节存入数组
        int rlen = 1;
        while ((in >= 0) && (len > 1)) {

            int available;

            if (in > out) {
                available = Math.min((buffer.length - out), (in - out));//in>out则可读取的数据是in-out，正常情况下in不能超过buffer.length
            } else {
                available = buffer.length - out;//in<out则可读取数据是out到buffer右边界，一次循环只能读取到右边界，从头开始的部分要下一次循环读取
            }

            // 在循环外事先读取的字节
            if (available > (len - 1)) {
                available = len - 1;//读取的字节不能超过参数指定的数量-1因为循环外已经读取了一个字节
            }
            System.arraycopy(buffer, out, b, off + rlen, available);//复制可读取字节
            out += available;//out位置移动读取的字节数
            rlen += available;//目标位置移动
            len -= available;//待读取长度减少

            if (out >= buffer.length) {
                out = 0;
            }
            if (in == out) {
                /* in==out缓冲区为空，将in置为-1，所以循环会跳出 */
                in = -1;
            }
        }
        return rlen;
    }
```

available返回从这个输入流可以读取多少字节不用阻塞

```java
    public synchronized int available() throws IOException {
        if(in < 0)
            return 0;
        else if(in == out)
            return buffer.length;//因为在接收和读取时没有可读取的字节会把in置为-1，所以in==out是缓冲区满了的状态
        else if (in > out)
            return in - out;//in>out时可读取内容是in到out之间
        else
            return in + buffer.length - out;//in<out时是out到buffer右边界再加上buffer左边界到in的内容
    }
```

close关闭管道输入流，释放任何和这个流关联的资源

```java
    public void close()  throws IOException {
        closedByReader = true;//输入流关闭
        synchronized (this) {
            in = -1;//缓冲区所有数据失效
        }
    }
```

# PipedOutputStream

PipedOutputStream继承了OutputStream。管道输出流可以连接到一个管道输入流来创建一个通信管道。管道输出流是发送端。一般来说，一个线程将数据写入到一个PipedOutputStream对象，其他线程从连接的PipedInputStream对象读取数据。不推荐使用同一个线程来同时使用两个对象，可能会引起这个线程死锁。如果一个从连接的管道输入流读取数据的线程不再活动，这个管道被称为broken状态。

PipedOutputStream的代码很少，基本上全部都是调用PipedInputStream的方法。

内部变量只有一个private PipedInputStream sink用来判定输入端的状态。

构造方法可以指定输入流，也可以直接用无参构造使用前再手动调用连接方法

```java
    /**
     * 创建一个管道输出流连接到指定的管道输入流。数据字节写入到这个流中将会称为snk的有效输入。
     */
    public PipedOutputStream(PipedInputStream snk)  throws IOException {
        connect(snk);
    }

    /**
     * 创建一个管道输出流还没有连接管道输入流。它在使用前必须通过接收者或者发送者连接到一个管道输入流。
     */
    public PipedOutputStream() {
    }
```

connect方法要求两端都不能有已有的连接，否则会抛出异常

```java
    /**
     * 连接这个管道输出流到一个接收者。如果这个对象已经连接到了某个其他的管道输入流，会抛出IOException。
     * 如果snk是一个未连接的管道输入流，src是一个未连接的管道输出流，它们可以通过两种方式连接：
     * src.connect(snk)或者snk.connect(src)。这两种方式是同样的效果。
     */
    public synchronized void connect(PipedInputStream snk) throws IOException {
        if (snk == null) {
            throw new NullPointerException();
        } else if (sink != null || snk.connected) {
            throw new IOException("Already connected");//如果自身或者snk已经连接到某个管道，则抛出异常
        }
        sink = snk;//因为snk内的属性是protected所以可以被同package的PipedOutputStream直接修改
        snk.in = -1;
        snk.out = 0;
        snk.connected = true;//连接状态修改
    }
```

write方法写入数据到缓冲区，需要检查是否有连接的输入端，输入字符数组时还要检查数组和位置参数是否在正确范围内，最后调用输入流的receive方法

```java
    /**
     * 将指定的字节写入到管道输出流，实现了OutputStream.write
     */
    public void write(int b)  throws IOException {
        if (sink == null) {
            throw new IOException("Pipe not connected");
        }
        sink.receive(b);//调用PipedInputStream.receive
    }

    /**
     * 将指定的数组中从off偏移开始长度len的字节写入到这个管道输出流。这个方法会阻塞，直到所有的字节都写入到输出流中。
     */
    public void write(byte b[], int off, int len) throws IOException {
        if (sink == null) {
            throw new IOException("Pipe not connected");
        } else if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        sink.receive(b, off, len);
    }
```

flush()刷新这个输出流，并且促使任何缓冲的输出数据被写出。这个方法会唤醒所有的字节在管道中等待的读取端进入就绪状态。

```java
    public synchronized void flush() throws IOException {
        if (sink != null) {
            synchronized (sink) {
                sink.notifyAll();
            }
        }
    }
```

close关闭这个管道输出流并释放所有关联的系统资源。这个流不能再用于写字节。

```java
    public void close()  throws IOException {
        if (sink != null) {
            sink.receivedLast();
        }
    }

    /**
     * 通知所有等待的线程最后一个字节数据已经被接收
     */
    synchronized void receivedLast() {
        closedByWriter = true;
        notifyAll();
    }
```
# PipedReader

PipedReader是管道字符输入流，总体设计逻辑基本上和PipedInputStream是一致的，它继承了Reader。因为总体上都是一样的，就只挑区别来讲了。

第一个区别，因为是字符流，缓冲区变为字符数组

```java
    char buffer[];
```

receive单个字符把PipedInputStream里两个内部方法直接写到代码里了，然后因为类型的改变这里有强制转换类型的区别。

```java
    synchronized void receive(int c) throws IOException {
        if (!connected) {
            throw new IOException("Pipe not connected");
        } else if (closedByWriter || closedByReader) {
            throw new IOException("Pipe closed");
        } else if (readSide != null && !readSide.isAlive()) {
            throw new IOException("Read end dead");
        }

        writeSide = Thread.currentThread();
        while (in == out) {
            if ((readSide != null) && !readSide.isAlive()) {
                throw new IOException("Pipe broken");
            }
            /* full: kick any waiting readers */
            notifyAll();
            try {
                wait(1000);
            } catch (InterruptedException ex) {
                throw new java.io.InterruptedIOException();
            }
        }
        if (in < 0) {
            in = 0;
            out = 0;
        }
        buffer[in++] = (char) c;//int转为char
        if (in >= buffer.length) {
            in = 0;
        }
    }
```

receive多个字符代码很简单，前面分析过PipedInputStream是在缓冲区满时等待读取，然后写入直到再次填满缓冲区或者写完所有数据，而PipedReader则是一个个字符存储到缓冲区，每一次写入都有可能发生阻塞等待。

```java
    synchronized void receive(char c[], int off, int len)  throws IOException {
        while (--len >= 0) {
            receive(c[off++]);
        }
    }
```

read设计逻辑上没有区别，也是单个字符读取最多等待2个循环也就是2s。多个字符读取会先尝试读取一个字符，后面无阻塞的尽可能读取多的字符来满足要求的长度，返回实际读取的字符个数。

PipedWriter设计和PipedOutputStream逻辑上无区别，就不说了。