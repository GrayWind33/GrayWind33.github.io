---
layout:     post
title:      Java FileInputStream FileOutputStream类源码解析
subtitle:   分析文件字节输入流FileInputStream和FileOutputStream的实现原理
date:       2018-08-25
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

FileInputStream和FileOutputStream是匹配的文件输出输出流，读取和写入的是byte，所以适合用来处理一些非字符的数据，比如图片数据。因为涉及到大量关于文件的操作，所以存在很多的native方法和利用操作系统的文件系统实现，所以要深入了解文件输入输出流还是需要加强操作系统和native源码的知识。先来看一下简单的使用示例：

```java
	public static void main(String args[]) throws IOException{
		FileOutputStream out = new FileOutputStream("D:/test/file.txt");
		out.write("1234567890".getBytes());//1234567890
		out.close();
		out = new FileOutputStream("D:/test/file.txt");
		out.write("765".getBytes());//765
		out.close();
		out = new FileOutputStream("D:/test/file.txt", true);
		out.write("asdfgh".getBytes());//765asdfgh
		out.close();
		new File("D:/test/file.txt");//765asdfgh
		out = new FileOutputStream("D:/test/file.txt");
		out.close();//内容为空
		
		//测试filechannel位置改变对stream的影响
		out = new FileOutputStream("D:/test/file.txt");
		out.write("1234567890".getBytes());//1234567890
		out.close();
		
		FileInputStream in = new FileInputStream("D:/test/file.txt");
		
		System.out.print(String.valueOf((byte)in.read() & 0xf));//1
		System.out.print(String.valueOf((byte)in.read() & 0xf));//2
		System.out.print(String.valueOf((byte)in.read() & 0xf));//3
		
		in.getChannel().position(5);
		System.out.print(String.valueOf((byte)in.read() & 0xf));//6
		in.getChannel().position(0);
		System.out.print(String.valueOf((byte)in.read() & 0xf));//1
		
		in.skip(-1);//向前跳一位
		System.out.print(String.valueOf((byte)in.read() & 0xf));//1
	}
```

# FileInputStream

实现了抽象类InputStream，FileInputStream从文件系统中的文件获取bytes，文件是否有效取决于主机环境。FileInputStream对于直接从流中读取bytes数据非常有意义如图像数据。如果要读取字符信息，考虑使用FileReader。我们可以看到除了close以外，没有出现任何锁，但是实际上如果我们使用多个线程读取同一个文件时，单次读取是原子操作，见下方代码：

```java
package test;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class FileThreadTest implements Runnable {
	private int type;// 0做skip操作，1做读取操作
	
	private int gap;

	private FileInputStream in;

	public FileThreadTest(int type, FileInputStream in, int gap) {
		this.type = type;
		this.in = in;
		this.gap = gap;
	}

	@Override
	public void run() {
		byte[] body = new byte[gap];
		if (this.type == 0) {
			try {
				for(int i = 0; i < 4; i++) {
					in.skip(gap);
				}
			} catch (IOException e) {
				e.printStackTrace();
			}
		} else {
			try {
				for(int i = 0; i < 10; i++) {
					in.read(body);
					System.out.println(Thread.currentThread().getName() + "-" + new String(body));
				}
			} catch (IOException e) {
				e.printStackTrace();
			}
			
		}
	}

	public static void main(String args[]) throws IOException, InterruptedException {
		FileOutputStream out = new FileOutputStream("D:/test/file.txt");
		for (int i = 0; i < 1000; i++) {
			out.write("1234567890".getBytes());// 写入测试数据
		}
		out.close();
		FileInputStream in = new FileInputStream("D:/test/file.txt");
		FileThreadTest t1 = new FileThreadTest(0, in, 2);
		FileThreadTest t2 = new FileThreadTest(1, in, 4);
		FileThreadTest t3 = new FileThreadTest(1, in, 3);
		Thread thread1 = new Thread(t1,"线程1");
		Thread thread2 = new Thread(t2,"线程2");
		Thread thread3 = new Thread(t3,"线程3");
		thread2.start();
		thread3.start();
		thread1.start();
		thread1.join();
		thread2.join();
		thread3.join();
		in.close();
		/*
		线程3-567
		线程3-678
		线程2-1234
		线程3-901
		线程3-678
		线程2-2345
		线程2-2345
		线程2-6789
		线程2-0123
		线程2-4567
		线程2-8901
		线程2-2345
		线程2-6789
		线程3-901
		线程3-456
		线程3-789
		线程2-0123
		线程3-012
		线程3-345
		线程3-678
		*/
	}
}
```

运行结果试机器配置每次运行会有所不同，但是我们可以看到无论怎么运行，每一次read操作本身是不会被其他线程抢占而中断的，它一定会完整的读取到这次要读取的内容，但是由于其他线程可以改变输入流的位置，所以每个线程读取时开始的位置是不可预知的，每个线程的read和skip操作都会改变流的位置。

先来看下内部变量，一个文件输出流有文件路径名、文件描述符、文件通道属性，若由文件描述符来创建，则文件路径名为null

```java
    /* 文件描述符，用来打开文件*/
    private final FileDescriptor fd;

    /**
     * 引用文件的路径，如果流是通过文件描述符创建时该值为null
     */
    private final String path;

    private FileChannel channel = null;
    //用于保证close操作的线程安全性
    private final Object closeLock = new Object();
    private volatile boolean closed = false;
```

然后是构造函数，主要分为通过文件路径名、文件描述符、具体文件，通过文件描述符创建时文件路径名为null

```java
    /**
     * 通过打开一个连接实际文件的连接来创建一个FileInputStream，文件通过文件系统中的路径名name来命名。
     * 一个新的文件描述符对象会被创建来表示这个文件连接
     * 如果存在安全管理器，checkRead方法会被调用，name作为参数传入
     * 如果文件名不存在，或者是一个目录而不是规则文件，或者因为其他原因无法打开读取，抛出FileNotFoundException
     */
    public FileInputStream(String name) throws FileNotFoundException {
        this(name != null ? new File(name) : null);
    }
	//通过具体文件来创建，其他情况同上
    public FileInputStream(File file) throws FileNotFoundException {
        String name = (file != null ? file.getPath() : null);//name是file的路径名
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkRead(name);//检查是否对文件有读取权限
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        fd = new FileDescriptor();
        fd.attach(this);//绑定文件描述符便于关闭对象
        path = name;
        open(name);//打开文件
    }

    /**
     * 通过文件描述符fdObj来创建FileInputStream，代表了对一个文件系统中实际存在文件的连接
     * 如果FileInputStream是null会抛出NullPointerException
     * 如果fdObj是无效的，构造器不会抛出异常，但是，如果调用这个流的IO方法，会抛出IOException
     */
    public FileInputStream(FileDescriptor fdObj) {
        SecurityManager security = System.getSecurityManager();
        if (fdObj == null) {
            throw new NullPointerException();
        }
        if (security != null) {
            security.checkRead(fdObj);
        }
        fd = fdObj;
        path = null;

        //文件描述符被流共享，将这个流注册到文件描述符的追踪器
        fd.attach(this);
    }
```

文件本身需要打开才能进行操作，open只能由构造函数来调用，最终是由native方法open0来完成的系统的交互

```java
    private void open(String name) throws FileNotFoundException {
        open0(name);
    }

    private native void open0(String name) throws FileNotFoundException;
```

读取根据参数重载分为读取单个字节和读取字符数组，它们分别基于native方法read0和readBytes，前面的测试中，我们可以看到被多个线程共享的FileInputStream依然能够保证单次的read操作读取的信息是完整的，这应该与readBytes的实现有关

```java
	//从流中读取byte，如果没有有输出没有完成会阻塞方法
    public int read() throws IOException {
        return read0();
    }
	//从流中读取最大b.length的bytes数据到数组b中。该方法会被阻塞知道某些输入完成
    public int read(byte b[]) throws IOException {
        return readBytes(b, 0, b.length);
    }
	//从流中读取长度为len的数据，放到b中从off开始的位置
    public int read(byte b[], int off, int len) throws IOException {
        return readBytes(b, off, len);
    }

    private native int read0() throws IOException;
    private native int readBytes(byte b[], int off, int len) throws IOException;
```

skip也是native方法跳过并废弃输入流中n个bytes数据，skip方法可能因为一些不同的原因以跳过更少的bytes数结束，可能这个数量是0.如果n是负数，方法会尝试往回跳（见本文最上方例子）。如果文件不支持从当前位置往回掉，会抛出IOException。返回的是实际跳过的bytes数量，为正是往后跳，为负是往前跳。可能会跳过比文件中剩余数量更多的bytes数，这个过程不会产生异常，跳过的bytes数可能包括了一些文件中超过了EOF文件结束符的bytes数，跳过了文件结束符再尝试读取会返回-1，说明已经到达文件末尾。

```java

```

available也是native方法，返回剩余可读取的或者可跳过的bytes数的估计值，这个过程不会阻塞下一个操作。文件超过EOF时返回0。下一个调用可能是相同的线程或不同的线程，一个读取或者跳过这么多bytes的操作不会被阻塞，但是可能读取或跳过更少的bytes。一些情况下，一个非阻塞的读取或者跳过可能在非常慢时被阻塞，比如从一个很慢的网络中读取大文件。

```java
    public native int available() throws IOException;
```

close关闭文件输入流并释放相关联的任何系统资源，如果流关联通道则通道也要关闭。通过加锁保证只能进行一次，避免重复关闭。最终由文件描述符通过close0来关闭文件。

```java
    public void close() throws IOException {
        synchronized (closeLock) {//只能由一个线程来执行关闭一次
            if (closed) {
                return;
            }
            closed = true;
        }
        if (channel != null) {
           channel.close();//关闭关联的通道
        }

        fd.closeAll(new Closeable() {//通知文件描述符关闭文件
            public void close() throws IOException {
               close0();
           }
        });
    }

    private native void close0() throws IOException;
```

getChannel返回关联的通道，初始化通道的位置是目前位置从文件中读取的bytes数量。从流中读取bytes会增加通道的位置，改变通道的位置会改变流中的文件位置。延迟初始化，第一次调用该方法才会打开文件通道。

```java
    public FileChannel getChannel() {
        synchronized (this) {//初始化单例
            if (channel == null) {
                channel = FileChannelImpl.open(fd, path, true, false, this);
            }
            return channel;
        }
    }
```

finalize是protected方法必须通过继承FileInputStram的类来使用，作用是确保close在没有任何对流的引用时被调用，也就是避免其他线程中流还在读取，另一个线程发起了close，因为可以通过文件描述符来判断是否是in状态也就是正在读取。

```java
    protected void finalize() throws IOException {
        if ((fd != null) &&  (fd != FileDescriptor.in)) {
            /* 
             * 如果fd被共享，FileDescriptor中的引用会确保终结方法只会在安全的时候调用。
             * 所有使用fd的引用都不可达时，我们调用close
             */
            close();
        }
    }
```

# FileOutputStream

FileOutputStream是对应的文件输出流，文件输出流将数据也到一个文件或者是一个文件描述符中，无论文件是否有效或者可能根据所在平台创建一个新的文件。在一些平台上，一个文件只允许被一个FileOutputStream或者其他文件写入对象进行操作，在这种环境下，本类的构建在文件已经被打开时可能会出错。FileOutputStream写入的是比特，可以用于写入图片数据，如果要写入字符的话可以考虑使用FileWriter。

和上面的输入流很相似，FileOutputStream除了close以外也是不加锁的，但是write是一个原子性操作，必须在前一个byte串输出完之后，下一个输出才能开始。多线程可以强制输出流进行输出，但不能中断未进行完的write。

```java
package test;

import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;

public class FileOutputThreadTest implements Runnable {
	private byte[] txt;

	private FileOutputStream out;

	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			try {
				out.write(txt);
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}

	public FileOutputThreadTest(String txt, FileOutputStream out) {
		this.txt = txt.getBytes();
		this.out = out;
	}

	public static void main(String args[]) throws InterruptedException, IOException {
		FileOutputStream out = new FileOutputStream("D:/test/file.txt");
		StringBuilder build = new StringBuilder();
		Thread[] t = new Thread[10];
		for (int i = 0; i < 10; i++) {
			build.setLength(0);
			for (int j = 0; j < 5; j++) {
				build.append(i);
			}
			t[i] = new Thread(new FileOutputThreadTest(build.toString(), out));
		}
		for (int i = 0; i < 10; i++) {
			t[i].start();
		}
		for (int i = 0; i < 10; i++) {
			t[i].join();
		}
		out.close();

		// 根据输出结果，每个字符应该连续出现5的倍数
		FileInputStream in = new FileInputStream("D:/test/file.txt");
		int pos = 0;
		while (in.available() > 0) {
			int text = in.read();
			pos++;
			for (int i = 1; i < 5; i++) {
				if (text != in.read()) {
					System.out.println("error" + String.valueOf(pos));// 没有出现
					return;
				}
				pos++;
			}
		}
		/*
		 * 00000000000000000000000000000000000000000000000000
		 * 22222222222222211111222222222222222222222222222222输入有交错
		 * 22222111111111111111111111111111111111111111111111
		 * 33333333333333333333333333333333333333333333333333
		 * 55555555555555555555555555555555555555555555555555
		 * 44444444444444444444444444444444444444444444444444
		 * 66666666666666666666666666666666666666666666666666
		 * 99999999999999999999999998888888888777777777777777
		 * 77777777777777777777777777777777777999999999999999
		 * 99999999998888888888888888888888888888888888888888
		 */
	}

}

```

FileOutputStream有两种模式，清空文件从头开始输入和保留原本内容从文件末尾开始添加，这取决于内部属性append为true时是添加模式

```java
    /**
     * 系统依赖的文件描述符
     */
    private final FileDescriptor fd;

    /**
     * 文件为扩展模式在末尾添加时为true
     */
    private final boolean append;

    /**
     * 相关联的文件通道，延迟初始化
     */
    private FileChannel channel;

    /**
     * 文件路径，如果该流是通过文件描述符来创建，为null
     */
    private final String path;

    private final Object closeLock = new Object();
    private volatile boolean closed = false;
```

构造方法传入的参数同样是三类：文件路径名、具体文件和文件描述符，append参数也在构造中指定，如果不输入默认是false

```java
    /**
     * 通过具体的名字创建一个文件输出流来写入到文件。一个新的文件描述符被创建来代表这个文件.
     * 如果有一个安全管理器，它的checkWrite方法需要传入name参数
     * 如果文件存在但它是一个目录而不是规则的文件，或者文件不出在但不能被创建，或者文件因为其他原因不能被打开，抛出FileNotFoundException
     */
    public FileOutputStream(String name) throws FileNotFoundException {
        this(name != null ? new File(name) : null, false);//默认输出流从文件头部开始写入，会导致文本被清空
    }

    public FileOutputStream(String name, boolean append)
        throws FileNotFoundException
    {
        this(name != null ? new File(name) : null, append);
    }
    //创建一个文件输出流来向一个具体的file中写入数据。一个新的文件描述符被创建来代表这个文件连接。
    public FileOutputStream(File file) throws FileNotFoundException {
        this(file, false);//清空文件并且从头开始输入
    }
    
    public FileOutputStream(File file, boolean append)
        throws FileNotFoundException
    {
        String name = (file != null ? file.getPath() : null);//file的路径和文件名
        SecurityManager security = System.getSecurityManager();//获取操作系统的安全管理器
        if (security != null) {
            security.checkWrite(name);//检查对文件是否有写入权限
        }
        if (name == null) {
            throw new NullPointerException();
        }
        if (file.isInvalid()) {
            throw new FileNotFoundException("Invalid file path");
        }
        this.fd = new FileDescriptor();
        fd.attach(this);//便于文件描述符关闭文件
        this.append = append;
        this.path = name;

        open(name, append);
    }
    
    /**
     * 创建一个文件输入流写入到具体的文件描述符中，该文件描述符表示了对一个文件系统中实际文件存在的链接
     * 安全管理器的checkWrite参数是文件描述符fdObj
     * 如果fdObj是null会抛出NullPointerException
     * 如果fdObj不可用不会抛出异常，但是，如果此时尝试调用该流的IO方法会抛出IOException
     */
    public FileOutputStream(FileDescriptor fdObj) {
        SecurityManager security = System.getSecurityManager();
        if (fdObj == null) {
            throw new NullPointerException();
        }
        if (security != null) {
            security.checkWrite(fdObj);
        }
        this.fd = fdObj;
        this.append = false;//从头写入
        this.path = null;//使用文件描述符时没有路径

        fd.attach(this);
    }
```

同样，文件需要打开才能进行写入，open方法只能由构造函数调用，基于native方法open0完成

```java
    private void open(String name, boolean append)
        throws FileNotFoundException {
        open0(name, append);//调用native方法打开文件
    }

    private native void open0(String name, boolean append)
        throws FileNotFoundException;
```

write操作上面提到过写入byte数组的操作是原子的，也就是native方法writeBytes是不可中断的。append变量作用在两个native方法中

```java
	//将具体的byte写入到文件输出流中，实现了OutputStream.write方法
    public void write(int b) throws IOException {
        write(b, append);
    }

    public void write(byte b[]) throws IOException {
        writeBytes(b, 0, b.length, append);
    }

    public void write(byte b[], int off, int len) throws IOException {
        writeBytes(b, off, len, append);
    }

    private native void writeBytes(byte b[], int off, int len, boolean append)
        throws IOException;

    private native void write(int b, boolean append) throws IOException;
```

close关闭这个文件输出流并释放任何关联的系统资源，这个输出流不能再用于写入bytes，如果流关联到了通道，则通道也关闭。通过加锁保证只会被关闭一次。

```java
    public void close() throws IOException {
        synchronized (closeLock) {//close只能进行一次所以需要是线程安全的，只能由一个线程进行
            if (closed) {
                return;
            }
            closed = true;
        }

        if (channel != null) {
            channel.close();//关闭文件通道
        }

        fd.closeAll(new Closeable() {
            public void close() throws IOException {
               close0();
           }
        });
    }

    private native void close0() throws IOException;
```

getChannel获取的文件通道，返回通道的初始化等于到目前为止写入文件的bytes数量，除非当前的流是扩展模式，该模式下等于文件的大小。写入bytes将会增加通道的位置，无论是通过写入或者指明来改变通道的位置都会改变流的文件位置。文件通道是延迟初始化的设计，在调用时才进行初始化。

```java
    public FileChannel getChannel() {
        synchronized (this) {//确保只初始化一次
            if (channel == null) {
                channel = FileChannelImpl.open(fd, path, false, true, append, this);
            }
            return channel;
        }
    }
```

finalize和FIleInputStream一样也是要通过继承类来调用的protected方法，清除所有到文件的连接，确保当没有其他对这个流的引用时，close方法被调用。通过文件描述符的状态来判断当前是否在输出。

```java
    protected void finalize() throws IOException {
        if (fd != null) {
            if (fd == FileDescriptor.out || fd == FileDescriptor.err) {
                flush();
            } else {
                /* 
                 * 如果fd被共享，FileDescriptor中的引用会确保终结器只在安全的时候被调用。
                 * 所有使用fd的引用都不可达时，我们调用close
                 */
                close();
            }
        }
    }
```