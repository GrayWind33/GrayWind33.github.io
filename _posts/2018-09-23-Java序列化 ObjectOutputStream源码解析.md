---
layout:     post
title:      Java序列化 ObjectOutputStream源码解析
subtitle:   分析序列化输出流ObjectOutputStream的实现原理
date:       2018-09-23
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

# 概述

众所周知，Java原生的序列化方法可以分为两种：

1. 实现Serializable接口
2. 实现Externalizable接口

其实还有一种，可以完全自己实现转为二进制内容，用Unsafe写到内存里面，然后写入文件

### Serializable

可以使用ObjectStream默认实现的writeObject和readObject方法并且可以通过transit关键字来使得变量不被序列化，开发简单

除了输出协议和包名类名外，会额外输出类的变量信息

有缓存机制，对于重复对象会直接输出所在位置，所以类较大且重复内容多时反而效率高，但会消耗额外内存空间

如果父类没有无参构造函数则不会序列化父类

### Externalizable

必须完全由自己来实现序列化规则所以可以直接控制哪些变量需要序列化，所以开发工作量较大

可以自己决定输出内容，只会固定输出协议和包名类名，较为简洁，对于小对象的序列化Externalizable会快一些

必须有无参构造函数否则编译会出错

------

​	但是，普遍实际项目开发中对于原生序列化的使用非常少，我觉得这里面的主要原因还是出在原生的对象流本身设计上一些是否安全的判断过多，加上缓冲区本身大小只有1K有点小，很明显一个16K的对象一次写入硬盘是比1K*16次快很多。尤其是大多数情况下重复对象判断就是在浪费时间，比如一个网站的一条用户信息，根本不会有几个重复字段。所以在很多网上的性能测试案例中，Serializable<Externalizable<一些常用的高性能序列化工具。

​	因为对象流篇幅过长，加上很多内容是系统安全或者是分隔符标志之类的东西，下面就只挑重点来说。

# ObjectOutputStream

先看一眼内部变量一大堆，光看注释根本不知道是干吗用的。大致分类一下，内部类Caches用于安全审计缓存。一面一块是用于输出的部分，bout是下层输出流，两个表是用于记录已输出对象的缓存便于之前说的重复输出的时候输出上一个相同内容的位置。接下来两个是writeObject()/writeExternal()上行调用记录上下文用的。debugInfoStack用于存储错误信息。

```java
    private static class Caches {
        /** cache of subclass security audit results 子类安全审计结果缓存*/
        static final ConcurrentMap<WeakClassKey,Boolean> subclassAudits =
            new ConcurrentHashMap<>();

        /** queue for WeakReferences to audited subclasses 对审计子类弱引用的队列*/
        static final ReferenceQueue<Class<?>> subclassAuditsQueue =
            new ReferenceQueue<>();
    }

    /** filter stream for handling block data conversion 解决块数据转换的过滤流*/
    private final BlockDataOutputStream bout;
    /** obj -> wire handle map obj->线性句柄映射*/
    private final HandleTable handles;
    /** obj -> replacement obj map obj->替代obj映射*/
    private final ReplaceTable subs;
    /** stream protocol version 流协议版本*/
    private int protocol = PROTOCOL_VERSION_2;
    /** recursion depth 递归深度*/
    private int depth;

    /** buffer for writing primitive field values 写基本数据类型字段值缓冲区*/
    private byte[] primVals;

    /** if true, invoke writeObjectOverride() instead of writeObject() 如果为true，调用writeObjectOverride()来替代writeObject()*/
    private final boolean enableOverride;
    /** if true, invoke replaceObject() 如果为true，调用replaceObject()*/
    private boolean enableReplace;

	//下面的值只在上行调用writeObject()/writeExternal()时有效
    /**
     * 上行调用类定义的writeObject方法时的上下文，持有当前被序列化的对象和当前对象描述符。在非writeObject上行调用时为null
     */
    private SerialCallbackContext curContext;
    /** current PutField object 当前PutField对象*/
    private PutFieldImpl curPut;

    /** custom storage for debug trace info 常规存储用于debug追踪信息*/
    private final DebugTraceInfoStack debugInfoStack;
```

构造函数有两个，第一个是自身的构造需要提供一个输出流，第二个实际上是提供给子类用的，创建一个自身相关内部变量全为空的对象输出流。但是，两个构造器都会进行安全检查，检查序列化的类是否重写了安全敏感方法，如果违反了规则会抛出异常。正常的构造类还会直接输出头部信息，包括对象输出流的魔数和协议版本信息，所以即使只新建一个对象输出流就会输出头部信息。

```java
    /**
     * 创建一个ObjectOutputStream写到指定的OutputStream。这个构造器写序列化流头部到下层流中，
     * 调用者可能希望立即刷新流来确保接收的ObjectInputStreams构造器不会再读取头部时阻塞。
     * 如果一个安全管理器被安装，这个构造器将会在被直接调用和被子类的构造器间接调用时检查enableSubclassImplementation序列化许可，
     * 如果这个子类重写了ObjectOutputStream.putFields或者ObjectOutputStream.writeUnshared方法
     */
    public ObjectOutputStream(OutputStream out) throws IOException {
    	verifySubclass();
        bout = new BlockDataOutputStream(out);//通过下层流out创建一个块输出流
        handles = new HandleTable(10, (float) 3.00);
        subs = new ReplaceTable(10, (float) 3.00);
        enableOverride = false;
        writeStreamHeader();
        bout.setBlockDataMode(true);//默认采用块模式
        if (extendedDebugInfo) {
            debugInfoStack = new DebugTraceInfoStack();
        } else {
            debugInfoStack = null;
        }
    }

    /**
     * 给子类提供一个路径完全重新实现ObjectOutputStream，不会分配任何用于实现ObjectOutputStream的私有数据
     * 如果安装了一个安全管理器，这个方法会先调用安全管理器的checkPermission方法来检查序列化许可来确保可以使用子类
     */
    protected ObjectOutputStream() throws IOException, SecurityException {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
        bout = null;
        handles = null;
        subs = null;
        enableOverride = true;
        debugInfoStack = null;
    }

    /**
     * 验证这个实例(可能是子类)可以不用违背安全约束被构造：子类不能重写安全敏感的非final方法，或者其他enableSubclassImplementation序列化许可检查
     * 这个检查会增加运行时开支
     */
    private void verifySubclass() {
    	Class<?> cl = getClass();
        if (cl == ObjectOutputStream.class) {
            return;//不是子类直接返回
        }
        SecurityManager sm = System.getSecurityManager();
        if (sm == null) {
        	return;//没有安全管理器直接返回
        }
        processQueue(Caches.subclassAuditsQueue, Caches.subclassAudits);//从弱引用队列中出队所有类，并移除缓存中相同的类
        WeakClassKey key = new WeakClassKey(cl, Caches.subclassAuditsQueue);
        Boolean result = Caches.subclassAudits.get(key);//缓存中是否已有这个类
        if (result == null) {
        	result = Boolean.valueOf(auditSubclass(cl));//检查这个子类是否安全
            Caches.subclassAudits.putIfAbsent(key, result);//将结果存储到缓存
        }
        if (result.booleanValue()) {
            return;//子类安全直接返回
        }
        sm.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);//检查子类实现许可
    }

    /**
     * 提供writeStreamHeader方法这样子类可以扩展或者预先考虑它们自己的流头部。
     * 这个方法写魔数和版本到流中。
     *
     * @throws  IOException if I/O errors occur while writing to the underlying
     *          stream
     */
    protected void writeStreamHeader() throws IOException {
        bout.writeShort(STREAM_MAGIC);//流魔数
        bout.writeShort(STREAM_VERSION);//流版本
    }
```

接下来开始关键部分，来分析writeObject到底做了什么，首先看这个方法本身是final方法也就是说即使继承了ObjectOutputStream也不能重写这个方法而是重写writeObjectOverride并且enableOverride=true

```java
    public final void writeObject(Object obj) throws IOException {
        if (enableOverride) {
            writeObjectOverride(obj);//如果流子类重写了writeObject则调用这里的方法
            return;
        }
        try {
            writeObject0(obj, false);
        } catch (IOException ex) {
            if (depth == 0) {
                writeFatalException(ex);
            }
            throw ex;
        }
    }
```

writeObject0这个方法代码很长，一部分一部分来看，首先我们注意到上面的都是缓存替换部分，第一次进入这个方法是不需要考虑的，直接看到writeOrdinaryObject这里，因为用于数据化的类是实现了Serializable接口，所以会进入这个分支。

```java
    private void writeObject0(Object obj, boolean unshared)
        throws IOException
    {
        boolean oldMode = bout.setBlockDataMode(false);//将输出流设置为非块模式
        depth++;//增加递归深度
        try {
            // handle previously written and non-replaceable objects处理之前写的不可替换对象
            int h;
            if ((obj = subs.lookup(obj)) == null) {
                writeNull();//替代对象映射中这个对象为null时，写入null代码
                return;
            } else if (!unshared && (h = handles.lookup(obj)) != -1) {
                writeHandle(h);//不是非共享模式且这个对象在对句柄的映射表中已有缓存，写入该对象在缓存中的句柄值
                return;
            } else if (obj instanceof Class) {
                writeClass((Class) obj, unshared);//写类名
                return;
            } else if (obj instanceof ObjectStreamClass) {
                writeClassDesc((ObjectStreamClass) obj, unshared);//写类描述
                return;
            }

            // check for replacement object检查替代对象，要求对象重写了writeReplace方法
            Object orig = obj;
            Class<?> cl = obj.getClass();
            ObjectStreamClass desc;
            for (;;) {
                // REMIND: skip this check for strings/arrays?
                Class<?> repCl;
                desc = ObjectStreamClass.lookup(cl, true);
                if (!desc.hasWriteReplaceMethod() ||
                    (obj = desc.invokeWriteReplace(obj)) == null ||
                    (repCl = obj.getClass()) == cl)
                {
                    break;
                }
                cl = repCl;
            }
            if (enableReplace) {
                Object rep = replaceObject(obj);//如果不重写这个方法直接返回了obj也就是什么也没做
                if (rep != obj && rep != null) {
                    cl = rep.getClass();
                    desc = ObjectStreamClass.lookup(cl, true);
                }
                obj = rep;
            }

            // if object replaced, run through original checks a second time如果对象被替换，第二次运行原本的检查，大部分情况下不执行此段
            if (obj != orig) {
                subs.assign(orig, obj);//将原本对象和替代对象作为一个键值对存入缓存
                if (obj == null) {
                    writeNull();
                    return;
                } else if (!unshared && (h = handles.lookup(obj)) != -1) {
                    writeHandle(h);
                    return;
                } else if (obj instanceof Class) {
                    writeClass((Class) obj, unshared);
                    return;
                } else if (obj instanceof ObjectStreamClass) {
                    writeClassDesc((ObjectStreamClass) obj, unshared);
                    return;
                }
            }

            // remaining cases剩下的情况
            if (obj instanceof String) {
                writeString((String) obj, unshared);
            } else if (cl.isArray()) {
                writeArray(obj, desc, unshared);
            } else if (obj instanceof Enum) {
                writeEnum((Enum<?>) obj, desc, unshared);
            } else if (obj instanceof Serializable) {
                writeOrdinaryObject(obj, desc, unshared);//传入流的对象第一次执行这个方法
            } else {
                if (extendedDebugInfo) {
                    throw new NotSerializableException(
                        cl.getName() + "\n" + debugInfoStack.toString());
                } else {
                    throw new NotSerializableException(cl.getName());
                }
            }
        } finally {
            depth--;
            bout.setBlockDataMode(oldMode);
        }
    }
```

writeOrdinaryObject这个方法主要是在Externalizable和Serializable的接口出现分支，如果实现了Externalizable接口并且类描述符非动态代理，则执行writeExternalData，否则执行writeSerialData。同时，**这个方法会写类描述信息**。

```java
    private void writeOrdinaryObject(Object obj,
                                     ObjectStreamClass desc,
                                     boolean unshared)
        throws IOException
    {
        if (extendedDebugInfo) {
            debugInfoStack.push(
                (depth == 1 ? "root " : "") + "object (class \"" +
                obj.getClass().getName() + "\", " + obj.toString() + ")");
        }
        try {
            desc.checkSerialize();

            bout.writeByte(TC_OBJECT);
            writeClassDesc(desc, false);//写类描述
            handles.assign(unshared ? null : obj);//如果是share模式把这个对象加入缓存
            if (desc.isExternalizable() && !desc.isProxy()) {
                writeExternalData((Externalizable) obj);
            } else {
                writeSerialData(obj, desc);
            }
        } finally {
            if (extendedDebugInfo) {
                debugInfoStack.pop();
            }
        }
    }
```

writeExternalData和writeSerialData(Object, ObjectStreamClass)这里有个上下文的操作，目的是保证序列化操作同一时间只能由一个线程调用。前者直接调用writeExternal，后者如果重写了writeObject则调用它，否则调用defaultWriteFields。defaultWriteFields会先输出基本数据类型，对于非基本数据类型的部分会再递归调用writeObject0，所以这里也就会增加递归深度depth。

```java
    private void writeExternalData(Externalizable obj) throws IOException {
        PutFieldImpl oldPut = curPut;
        curPut = null;

        if (extendedDebugInfo) {
            debugInfoStack.push("writeExternal data");
        }
        SerialCallbackContext oldContext = curContext;//存储上下文
        try {
            curContext = null;
            if (protocol == PROTOCOL_VERSION_1) {
                obj.writeExternal(this);
            } else {//默认协议是2，所以会使用块输出流
                bout.setBlockDataMode(true);
                obj.writeExternal(this);//这里取决于类的方法怎么实现
                bout.setBlockDataMode(false);
                bout.writeByte(TC_ENDBLOCKDATA);
            }
        } finally {
            curContext = oldContext;//恢复上下文
            if (extendedDebugInfo) {
                debugInfoStack.pop();
            }
        }

        curPut = oldPut;
    }

    private void writeSerialData(Object obj, ObjectStreamClass desc)
        throws IOException
    {
        ObjectStreamClass.ClassDataSlot[] slots = desc.getClassDataLayout();
        for (int i = 0; i < slots.length; i++) {
            ObjectStreamClass slotDesc = slots[i].desc;
            if (slotDesc.hasWriteObjectMethod()) {//重写了writeObject方法
                PutFieldImpl oldPut = curPut;
                curPut = null;
                SerialCallbackContext oldContext = curContext;

                if (extendedDebugInfo) {
                    debugInfoStack.push(
                        "custom writeObject data (class \"" +
                        slotDesc.getName() + "\")");
                }
                try {
                    curContext = new SerialCallbackContext(obj, slotDesc);
                    bout.setBlockDataMode(true);
                    slotDesc.invokeWriteObject(obj, this);//调用writeObject方法
                    bout.setBlockDataMode(false);
                    bout.writeByte(TC_ENDBLOCKDATA);
                } finally {
                    curContext.setUsed();
                    curContext = oldContext;
                    if (extendedDebugInfo) {
                        debugInfoStack.pop();
                    }
                }

                curPut = oldPut;
            } else {
                defaultWriteFields(obj, slotDesc);//如果没有重写writeObject则输出默认内容
            }
        }
    }
    
    private void defaultWriteFields(Object obj, ObjectStreamClass desc)
        throws IOException
    {
        Class<?> cl = desc.forClass();
        if (cl != null && obj != null && !cl.isInstance(obj)) {
            throw new ClassCastException();
        }

        desc.checkDefaultSerialize();

        int primDataSize = desc.getPrimDataSize();
        if (primVals == null || primVals.length < primDataSize) {
            primVals = new byte[primDataSize];
        }
        desc.getPrimFieldValues(obj, primVals);//将基本类型数据的字段值存入缓冲区
        bout.write(primVals, 0, primDataSize, false);//输出缓冲区内容

        ObjectStreamField[] fields = desc.getFields(false);
        Object[] objVals = new Object[desc.getNumObjFields()];//获取非基本数据类型对象
        int numPrimFields = fields.length - objVals.length;
        desc.getObjFieldValues(obj, objVals);
        for (int i = 0; i < objVals.length; i++) {
            if (extendedDebugInfo) {
                debugInfoStack.push(
                    "field (class \"" + desc.getName() + "\", name: \"" +
                    fields[numPrimFields + i].getName() + "\", type: \"" +
                    fields[numPrimFields + i].getType() + "\")");
            }
            try {
                writeObject0(objVals[i],
                             fields[numPrimFields + i].isUnshared());//递归输出
            } finally {
                if (extendedDebugInfo) {
                    debugInfoStack.pop();
                }
            }
        }
    }
```

然后看一下类描述信息是怎么写的，动态代理类和普通类有一些区别，但都是先写这个类本身的信息再写入父类的信息。

```java
    private void writeClassDesc(ObjectStreamClass desc, boolean unshared)
        throws IOException
    {
        int handle;
        if (desc == null) {
            writeNull();//描述符不存在时写null
        } else if (!unshared && (handle = handles.lookup(desc)) != -1) {
            writeHandle(handle);//共享模式且缓存中已有该类描述符时，写对应句柄值
        } else if (desc.isProxy()) {
            writeProxyDesc(desc, unshared);//描述符是动态代理类时
        } else {
            writeNonProxyDesc(desc, unshared);//描述符是标准类时
        }
    }
    
    private void writeProxyDesc(ObjectStreamClass desc, boolean unshared)
        throws IOException
    {
        bout.writeByte(TC_PROXYCLASSDESC);
        handles.assign(unshared ? null : desc);//存入缓存
        //获取类实现的接口，然后写入接口个数和接口名
        Class<?> cl = desc.forClass();
        Class<?>[] ifaces = cl.getInterfaces();
        bout.writeInt(ifaces.length);
        for (int i = 0; i < ifaces.length; i++) {
            bout.writeUTF(ifaces[i].getName());
        }

        bout.setBlockDataMode(true);
        if (cl != null && isCustomSubclass()) {
            ReflectUtil.checkPackageAccess(cl);
        }
        annotateProxyClass(cl);//装配动态代理类，子类可以重写这个方法存储类信息到流中，默认什么也不做
        bout.setBlockDataMode(false);
        bout.writeByte(TC_ENDBLOCKDATA);

        writeClassDesc(desc.getSuperDesc(), false);//写入父类的描述符
    }
    
    private void writeNonProxyDesc(ObjectStreamClass desc, boolean unshared)
        throws IOException
    {
        bout.writeByte(TC_CLASSDESC);
        handles.assign(unshared ? null : desc);

        if (protocol == PROTOCOL_VERSION_1) {
            // do not invoke class descriptor write hook with old protocol
            desc.writeNonProxy(this);
        } else {
            writeClassDescriptor(desc);
        }

        Class<?> cl = desc.forClass();
        bout.setBlockDataMode(true);
        if (cl != null && isCustomSubclass()) {
            ReflectUtil.checkPackageAccess(cl);
        }
        annotateClass(cl);//子类可以重写这个方法存储类信息到流中，默认什么也不做
        bout.setBlockDataMode(false);
        bout.writeByte(TC_ENDBLOCKDATA);

        writeClassDesc(desc.getSuperDesc(), false);//写入父类的描述信息
    }
```

最后看一下几个写方法，写字符串是写入UTF编码的二进制流数据。写枚举会额外写入一次枚举的类描述，然后将枚举名作为字符串写入。如果是写一个数组，先写入数组长度，然后如果数组是基本数据类型则可以直接写入，否则需要递归调用writeObject0

```java
    private void writeString(String str, boolean unshared) throws IOException {
        handles.assign(unshared ? null : str);
        long utflen = bout.getUTFLength(str);//获得UTF编码长度
        if (utflen <= 0xFFFF) {
            bout.writeByte(TC_STRING);
            bout.writeUTF(str, utflen);
        } else {
            bout.writeByte(TC_LONGSTRING);
            bout.writeLongUTF(str, utflen);
        }
    }

    private void writeEnum(Enum<?> en,
                           ObjectStreamClass desc,
                           boolean unshared)
        throws IOException
    {
        bout.writeByte(TC_ENUM);
        ObjectStreamClass sdesc = desc.getSuperDesc();
        writeClassDesc((sdesc.forClass() == Enum.class) ? desc : sdesc, false);
        handles.assign(unshared ? null : en);
        writeString(en.name(), false);
    }
```

## BlockDataOutputStream

BlockDataOutputStream是一个内部类，它继承了OutputStream并实现了DataOutput接口，缓冲输出流有两种模式：在默认模式下，输出数据和DataOutputStream使用同样模式；在块数据模式下，使用一个缓冲区来缓存数据到达最大长度或者手动刷新时将内容写入下层输入流，这点和BufferedOutputStream类似。不同之处在于，块模式在写数据之前，要先写入一个头部来表示当前块的长度。

从内部变量和构造函数中可以看出，缓冲区的大小是固定且不可修改的，其中包含了一个下层输入流和一个数据输出流以及是否采用块模式的标识，在构造时默认不采用块数据模式。

```java
        /** maximum data block length 最大数据块长度1K*/
        private static final int MAX_BLOCK_SIZE = 1024;
        /** maximum data block header length 最大数据块头部长度*/
        private static final int MAX_HEADER_SIZE = 5;
        /** (tunable) length of char buffer (for writing strings) 字符缓冲区的可变长度，用于写字符串*/
        private static final int CHAR_BUF_SIZE = 256;

        /** buffer for writing general/block data 用于写一般/块数据的缓冲区*/
        private final byte[] buf = new byte[MAX_BLOCK_SIZE];
        /** buffer for writing block data headers 用于写块数据头部的缓冲区*/
        private final byte[] hbuf = new byte[MAX_HEADER_SIZE];
        /** char buffer for fast string writes 用于写快速字符串的缓冲区*/
        private final char[] cbuf = new char[CHAR_BUF_SIZE];

        /** block data mode 块数据模式*/
        private boolean blkmode = false;
        /** current offset into buf buf中的当前偏移量*/
        private int pos = 0;

        /** underlying output stream 下层输出流*/
        private final OutputStream out;
        /** loopback stream (for data writes that span data blocks) 回路流用于写跨越数据块的数据*/
        private final DataOutputStream dout;

        /**
         * 在给定的下层流上创建一个BlockDataOutputStream，块数据模式默认关闭
         */
        BlockDataOutputStream(OutputStream out) {
            this.out = out;
            dout = new DataOutputStream(this);
        }
```

setBlockDataMode可以改变当前的数据模式，从块数据模式切换到非块数据模式时，要讲缓冲区内的数据写入到下层输入流中。getBlockDataMode可以查询当前的数据模式。

```java
        /**
         * 设置块数据模式为给出的模式true是开启，false是关闭，并返回之前的模式值。
         * 如果新的模式和旧的一样，什么都不做。
         * 如果新的模式和旧的模式不同，所有的缓冲区数据要在转换到新模式之前刷新。
         */
        boolean setBlockDataMode(boolean mode) throws IOException {
            if (blkmode == mode) {
                return blkmode;
            }
            drain();//将缓冲区内的数据全部写入下层输入流
            blkmode = mode;
            return !blkmode;
        }

        /**
         * 当前流为块数据模式返回true，否则返回false
         */
        boolean getBlockDataMode() {
            return blkmode;
        }
```

drain这个方法在多个方法中被调用，作用是将缓冲区内的数据全部写入下层输入流，但不会刷新下层输入流，在写入实际数据前要先用writeBlockHeader写入块头部，头部包含1字节标志位和1字节或4字节的长度大小

```java
        void drain() throws IOException {
            if (pos == 0) {
                return;//pos为0说明当前缓冲区为空
            }
            if (blkmode) {
                writeBlockHeader(pos);//块数据模式下要先写入头部
            }
            out.write(buf, 0, pos);//写入缓冲区数据
            pos = 0;//缓冲区被清空
        }

        /**
         * 写入块数据头部。数据块小于256字节会增加2字节头部前缀，其他会增加5字节头部。
         * 第一字节是标识长度范围，因为255字节以内可以用1字节来表示长度，4字节可以表示int范围内的最大整数
         */
        private void writeBlockHeader(int len) throws IOException {
            if (len <= 0xFF) {
                hbuf[0] = TC_BLOCKDATA;
                hbuf[1] = (byte) len;
                out.write(hbuf, 0, 2);
            } else {
                hbuf[0] = TC_BLOCKDATALONG;
                Bits.putInt(hbuf, 1, len);
                out.write(hbuf, 0, 5);
            }
        }
```

下面的方法等价于他们在OutputStream中的对应方法，除了他们参与在块数据模式下写入数据到数据块中的部分有所不同。写入都需要先检查缓冲区有没有达到上限，达到时需要先刷新，然后再将数据复制到缓冲区。刷新和关闭操作都不难理解。

```java
        public void write(int b) throws IOException {
            if (pos >= MAX_BLOCK_SIZE) {
                drain();//达到块数据上限时，将缓冲区内的数据全部写入下层流
            }
            buf[pos++] = (byte) b;//存储b到buf中
        }

        public void write(byte[] b) throws IOException {
            write(b, 0, b.length, false);
        }

        public void write(byte[] b, int off, int len) throws IOException {
            write(b, off, len, false);
        }
        /**
         * 将指定的字节段从数组中写出。如果copy是true，复制值到一个中间缓冲区在将它们写入下层流之前，来避免暴露一个对原字节数组的引用
         */
        void write(byte[] b, int off, int len, boolean copy)
            throws IOException
        {
            if (!(copy || blkmode)) {// 非copy也非块数据模式直接写入下层输入流
                drain();
                out.write(b, off, len);
                return;
            }

            while (len > 0) {
                if (pos >= MAX_BLOCK_SIZE) {
                    drain();
                }
                if (len >= MAX_BLOCK_SIZE && !copy && pos == 0) {
                    // 长度大于缓冲区非copy模式且缓冲区为空直接写，避免不必要的复制
                    writeBlockHeader(MAX_BLOCK_SIZE);
                    out.write(b, off, MAX_BLOCK_SIZE);
                    off += MAX_BLOCK_SIZE;
                    len -= MAX_BLOCK_SIZE;
                } else {
                    //剩余内容在缓冲区内放得下或者缓冲区不为空或者是copy模式，则将数据复制到缓冲区
                	int wlen = Math.min(len, MAX_BLOCK_SIZE - pos);
                    System.arraycopy(b, off, buf, pos, wlen);
                    pos += wlen;
                    off += wlen;
                    len -= wlen;
                }
            }
        }
        
        /**
         * 将缓冲区数据刷新到下层流，同时会刷新下层流
         */
        public void flush() throws IOException {
            drain();
            out.flush();
        }

        /**
         * 刷新之后关闭下层输出流
         */
        public void close() throws IOException {
            flush();
            out.close();
        }
```

上面的方法等价于他们在DataOutputStream中的对应方法，除了他们参与在块数据模式下写入数据到数据块中部分有所不同。基本上逻辑都是先检查空间是否足够，不足的话先刷新缓冲区，然后将数据存储到缓冲区中。因为篇幅原因，这里只贴几个方法为例。写一个字符串时，需要先将字符串中的字符存储到字符缓冲数组中，然后再转换成字节存储到buf中。

```java
        public void writeBoolean(boolean v) throws IOException {
            if (pos >= MAX_BLOCK_SIZE) {
                drain();
            }
            Bits.putBoolean(buf, pos++, v);
        }

        public void writeByte(int v) throws IOException {
            if (pos >= MAX_BLOCK_SIZE) {
                drain();
            }
            buf[pos++] = (byte) v;
        }

        /**
         * 写入单个字符，块未满时存储到缓冲区，块满时调用的是BlockDataOutputStream.write(int v)方法
         */
        public void writeChar(int v) throws IOException {
            if (pos + 2 <= MAX_BLOCK_SIZE) {
                Bits.putChar(buf, pos, (char) v);
                pos += 2;
            } else {
                dout.writeChar(v);
            }
        }

        /**
         * 先将String中的内容复制到字符缓冲区，再将其中的内容转为字节复制到块数据缓冲区
         */
        public void writeBytes(String s) throws IOException {
            int endoff = s.length();
            int cpos = 0;//当前字符串开始位置
            int csize = 0;//当前字符串大小
            for (int off = 0; off < endoff; ) {
                if (cpos >= csize) {
                    cpos = 0;
                    csize = Math.min(endoff - off, CHAR_BUF_SIZE);
                    s.getChars(off, off + csize, cbuf, 0);//将字符串中指定位置的片段复制到字符数组缓冲区
                }
                if (pos >= MAX_BLOCK_SIZE) {
                    drain();
                }
                int n = Math.min(csize - cpos, MAX_BLOCK_SIZE - pos);
                int stop = pos + n;
                while (pos < stop) {
                    buf[pos++] = (byte) cbuf[cpos++];//将字符数组中的内容复制到块数据缓冲区
                }
                off += n;
            }
        }
```

下面的方法写出连贯的原始数据值。尽管和重复调用对应的原始写方法结果相同，这些方法对于写一组原始数据值进行了效率优化。优化的方式是先计算出缓冲区内的剩余大小，计算可以写入的个数，然后直接写入而不是每次写入之前检查缓冲区是否有空间，减少判断次数。写UTF编码字符串时，如果能够提前知道编码长度，可以省去一次遍历字符串确定大小的过程，因为UTF编码中单个字符可能是一个1-3个字节不等。

```java
        void writeBooleans(boolean[] v, int off, int len) throws IOException {
            int endoff = off + len;
            while (off < endoff) {
                if (pos >= MAX_BLOCK_SIZE) {
                    drain();
                }
                int stop = Math.min(endoff, off + (MAX_BLOCK_SIZE - pos));
                while (off < stop) {//连续存储数据到缓冲区，减少了判断缓冲区是否满的次数
                    Bits.putBoolean(buf, pos++, v[off++]);
                }
            }
        }

        void writeChars(char[] v, int off, int len) throws IOException {
            int limit = MAX_BLOCK_SIZE - 2;
            int endoff = off + len;
            while (off < endoff) {
                if (pos <= limit) {
                    int avail = (MAX_BLOCK_SIZE - pos) >> 1;//一个字符=2个字节所以要除以2
                    int stop = Math.min(endoff, off + avail);
                    while (off < stop) {
                        Bits.putChar(buf, pos, v[off++]);
                        pos += 2;
                    }
                } else {
                    dout.writeChar(v[off++]);
                }
            }
        }

        /**
         * 返回给定字符串在UTF编码下的字节长度
         */
        long getUTFLength(String s) {
            int len = s.length();
            long utflen = 0;
            for (int off = 0; off < len; ) {
                int csize = Math.min(len - off, CHAR_BUF_SIZE);
                s.getChars(off, off + csize, cbuf, 0);
                for (int cpos = 0; cpos < csize; cpos++) {
                    char c = cbuf[cpos];
                    if (c >= 0x0001 && c <= 0x007F) {
                        utflen++;
                    } else if (c > 0x07FF) {
                        utflen += 3;
                    } else {
                        utflen += 2;
                    }
                }
                off += csize;
            }
            return utflen;
        }

        /**
         * 写给定字符串的UTF格式。这个方法用于字符串的UTF编码长度已知的情况，这样可以避免提前扫描一遍字符串来确定UTF长度
         */
        void writeUTF(String s, long utflen) throws IOException {
            if (utflen > 0xFFFFL) {
                throw new UTFDataFormatException();
            }
            writeShort((int) utflen);//先写长度
            if (utflen == (long) s.length()) {
                writeBytes(s);//没有特殊字符
            } else {
                writeUTFBody(s);//有特殊字符
            }
        }
```

## HandleTable

HandleTable是一个轻量的hash表，它的作用是缓存写过的共享class便于下次查找，内部含有3个数组，spine、next和objs。objs存储的是对象也就是class，spine是hash桶，next是冲突链表，每有一个新的元素插入需要计算它的hash值然后用spine的大小取模，找到它的链表，新对象会被插入到链表的头部，它在objs和next中对应的数据是根据加入的序号顺序存储，spine存储它的handle值也就是在另外两个数组中的下标。

![HandleTable](/img/blog/2018-09-23/HandleTable.png)

```java
        /** number of mappings in table/next available handle 表中映射的个数或者下一个有效的句柄*/
        private int size;
        /** size threshold determining when to expand hash spine 决定什么时候扩展hash脊柱的大小阈值*/
        private int threshold;
        /** factor for computing size threshold 计算大小阈值的因子*/
        private final float loadFactor;
        /** maps hash value -> candidate handle value 映射hash值->候选句柄值*/
        private int[] spine;
        /** maps handle value -> next candidate handle value 映射句柄值->下一个候选句柄值*/
        private int[] next;
        /** maps handle value -> associated object 映射句柄值->关联的对象*/
        private Object[] objs;

        /**
         * 创建一个新的hash表使用给定的容量和负载因子
         */
        HandleTable(int initialCapacity, float loadFactor) {
            this.loadFactor = loadFactor;
            spine = new int[initialCapacity];
            next = new int[initialCapacity];
            objs = new Object[initialCapacity];
            threshold = (int) (initialCapacity * loadFactor);
            clear();
        }
```

assign就是插入操作，它会检查3个数组大小是否足够，其中spine是根据next.length*负载因子来决定阈值的，数组大小扩大是乘以2加1，这个和HashTable时同样的设计。插入的时候注意到next的值被赋为原本的spine[index]值，说明之前的链表头成为了新结点的后驱，也就是说结点被插入链表头部。

```java
        /**
         * 分配下一个有效的句柄给给出的对象并返回句柄值。句柄从0开始升序被分配。相当于put操作
         */
        int assign(Object obj) {
            if (size >= next.length) {
                growEntries();
            }
            if (size >= threshold) {
                growSpine();
            }
            insert(obj, size);
            return size++;
        }

        /**
         * 通过延长条目数组增加hash表容量，next和objs大小变为旧大小*2+1
         */
        private void growEntries() {
            int newLength = (next.length << 1) + 1;//长度=旧长度*2+1
            int[] newNext = new int[newLength];
            System.arraycopy(next, 0, newNext, 0, size);//复制旧数组元素到新数组中
            next = newNext;

            Object[] newObjs = new Object[newLength];
            System.arraycopy(objs, 0, newObjs, 0, size);
            objs = newObjs;
        }

        /**
         * 扩展hash脊柱，等效于增加常规hash表的桶数
         */
        private void growSpine() {
            spine = new int[(spine.length << 1) + 1];//新大小=旧大小*2+1
            threshold = (int) (spine.length * loadFactor);//扩展阈值=spine大小*负载因子
            Arrays.fill(spine, -1);//spine中全部填充-1
            for (int i = 0; i < size; i++) {
                insert(objs[i], i);
            }
        }

        /**
         * 插入映射对象->句柄到表中，假设表足够大来容纳新的映射
         */
        private void insert(Object obj, int handle) {
            int index = hash(obj) % spine.length;//hash值%spine数组大小
            objs[handle] = obj;//objs顺序存储对象
            next[handle] = spine[index];//next存储spine[index]原本的handle值，也就是说新的冲突对象插入在链表头部
            spine[index] = handle;//spine中存储handle大小
        }
```

hash值计算就是通过系统函数计算出hash值然后去有符号int的有效位

```java
        private int hash(Object obj) {
            return System.identityHashCode(obj) & 0x7FFFFFFF;//取系统计算出的hash值得有效整数值部分
        }
```

lookup是查找hash表中是否含有指定对象，这里相等必须是==，因为class在完整类名相等时就是==

```java
        /**
         * 查找并返回句柄值关联给与的对象，如果没有映射返回-1
         */
        int lookup(Object obj) {
            if (size == 0) {
                return -1;
            }
            int index = hash(obj) % spine.length;//通过hash值寻找在spine数组中的位置
            for (int i = spine[index]; i >= 0; i = next[i]) {
                if (objs[i] == obj) {//遍历spine[index]位置的链表，必须是对象==才是相等
                    return i;
                }
            }
            return -1;
        }
```

clear是清空hash表，size返回当前表中映射对数

```java
        /**
         * 重置表为初始状态，next不需要重新赋值是因为插入第一个元素时，原本的spine[index]一定是-1，链表中不会出现之前存在的值
         */
        void clear() {
            Arrays.fill(spine, -1);
            Arrays.fill(objs, 0, size, null);
            size = 0;
        }

        /**
         * 返回当前表中的映射数量
         */
        int size() {
            return size;
        }
```

