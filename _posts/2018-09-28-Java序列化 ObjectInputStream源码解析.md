---
layout:     post
title:      Java序列化 ObjectInputStream源码解析
subtitle:   分析序列化输入流ObjectInputStream的实现原理
date:       2018-09-28
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

上一篇讲了类的序列化，今天要讲类的反序列化，ObjectInputStream。

从内部变量中我们可以看出，内部包含一个块输入流，因为有handle机制所以也有一个内部缓存表但不是hash表

```java
    /** 处理数据块转换的过滤流 */
    private final BlockDataInputStream bin;
    /** 确认调用返回列表 */
    private final ValidationList vlist;
    /** 递归深度 */
    private long depth;
    /** 对任何种类的对象、类、枚举、代理的引用总数 */
    private long totalObjectRefs;
    /** 流是否关闭 */
    private boolean closed;

    /** 线柄->obj/exception映射 */
    private final HandleTable handles;
    /** 传递句柄值上下调用栈的草稿区 */
    private int passHandle = NULL_HANDLE;
    /** 当在字段值末尾因没有TC_ENDBLOCKDATA阻塞时设置的标志 */
    private boolean defaultDataEnd = false;

    /** b读取原始字段值的缓冲区 */
    private byte[] primVals;

    /** 如果为true，调用readObjectOverride()来替代readObject() */
    private final boolean enableOverride;
    /** if true, invoke resolveObject()如果为true调用resolveObject() */
    private boolean enableResolve;

    /**
     * 向上调用类定义的readObject方法期间的上下文，保持当前在被反序列化的对象和当前类的描述符。非readObject向上调用期间为null。
     */
    private SerialCallbackContext curContext;

    /**
     * 类描述符过滤器和从流中读取的类，可能是null
     */
    private ObjectInputFilter serialFilter;
```

从构造函数中可以看出，同样需要验证类的安全性，同时在初始化时会自动读取Java序列化头部信息。跟输出流一样，预留了一个生成全空的类输入流初始化方法，用于继承ObjectInputStream的子类进行重写。安全验证和输出流中是相同的，而读取头部信息需要读取魔数和版本号并验证与当前JDK中的是否一致。当然，如果两端使用的JDK中这个类版本号不一致就会出现异常。

```java
    /**
     * 创建一个ObjectInputStream从指定的InputStream中读取。一个序列化流头部从这个流读取和验证。这个构造器会阻塞直到对应的ObjectOutputStream已经写并刷新了头部。
     * 如果安装了安全管理器，这个构造器会在被重写了ObjectInputStream.readFields或ObjectInputStream.readUnshared的子类构造器
     * 直接或间接调用的时候检查enableSubclassImplementation序列化许可
     *
     * @param   in input stream to read from
     * @throws  StreamCorruptedException 如果流头部错误
     * @throws  IOException 在读取流头部是发生了IO错误
     * @throws  SecurityException 如果不被信任的子类非法重写了安全敏感方法
     * @throws  NullPointerException 如果in是null
     */
    public ObjectInputStream(InputStream in) throws IOException {
        verifySubclass();
        bin = new BlockDataInputStream(in);
        handles = new HandleTable(10);
        vlist = new ValidationList();
        serialFilter = ObjectInputFilter.Config.getSerialFilter();
        enableOverride = false;
        readStreamHeader();
        bin.setBlockDataMode(true);
    }

    protected ObjectInputStream() throws IOException, SecurityException {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(SUBCLASS_IMPLEMENTATION_PERMISSION);
        }
        bin = null;
        handles = null;
        vlist = null;
        serialFilter = ObjectInputFilter.Config.getSerialFilter();
        enableOverride = true;
    }

    protected void readStreamHeader()
        throws IOException, StreamCorruptedException
    {
        short s0 = bin.readShort();
        short s1 = bin.readShort();
        if (s0 != STREAM_MAGIC || s1 != STREAM_VERSION) {
            throw new StreamCorruptedException(
                String.format("invalid stream header: %04X%04X", s0, s1));
        }
    }
```

下面直接切入正题，来看一下readObject，读和写一样也是final方法，所以子类要重写必须通过readObjectOverride方法来完成。如果类里面定义了非基本数据类型变量，需要进行嵌套读取，outerHandle用来存储上一层读取对象的句柄，然后调用readObject0读取生成对象。完成之后要记录句柄依赖，并检查有无异常产生。最上层读取完成之后要发起回调。

```java
    public final Object readObject()
        throws IOException, ClassNotFoundException
    {
        if (enableOverride) {
            return readObjectOverride();//因为是final方法，子类要重写只能通过这里
        }

        // 如果是嵌套读取，passHandle包含封闭对象的句柄
        int outerHandle = passHandle;
        try {
            Object obj = readObject0(false);
            handles.markDependency(outerHandle, passHandle);//记录句柄依赖
            ClassNotFoundException ex = handles.lookupException(passHandle);
            if (ex != null) {
                throw ex;
            }
            if (depth == 0) {
                vlist.doCallbacks();//最上层调用的读取完成后，发起回调
            }
            return obj;
        } finally {
            passHandle = outerHandle;
            if (closed && depth == 0) {
                clear();//最上层调用退出时清除调用返回列表和句柄缓存
            }
        }
    }
```

readObject0必须在块输入流内为空也就是上一次反序列化完全结束后才能开始，否则会抛出异常。首先，从流中读取一个标志位，判断当前下一个内容是什么类型，上次我们讲到过，输出的时候都会先输出TC_OBJECT，所以在默认情况下也是先读取到TC_OBJECT，也就是先执行readOrdinaryObject

```java
    private Object readObject0(boolean unshared) throws IOException {
        boolean oldMode = bin.getBlockDataMode();
        if (oldMode) {//之前是块输入模式且有未消费的数据会抛出异常
            int remain = bin.currentBlockRemaining();
            if (remain > 0) {
                throw new OptionalDataException(remain);
            } else if (defaultDataEnd) {
                /*
                 * 流当前在字段值通过默认序列化块写的末尾，因为没有终止TC_ENDBLOCKDATA标记，模拟通常数据结尾的行为
                 */
                throw new OptionalDataException(true);
            }
            bin.setBlockDataMode(false);
        }

        byte tc;
        while ((tc = bin.peekByte()) == TC_RESET) {//读到流reset标志
            bin.readByte();//消耗缓存的字节
            handleReset();//depth为0时执行clear方法，否则抛出异常
        }

        depth++;//增加递归深度
        totalObjectRefs++;//增加引用对象数
        try {
            switch (tc) {
                case TC_NULL:
                    return readNull();

                case TC_REFERENCE:
                    return readHandle(unshared);

                case TC_CLASS:
                    return readClass(unshared);

                case TC_CLASSDESC:
                case TC_PROXYCLASSDESC:
                    return readClassDesc(unshared);

                case TC_STRING:
                case TC_LONGSTRING:
                    return checkResolve(readString(unshared));

                case TC_ARRAY:
                    return checkResolve(readArray(unshared));

                case TC_ENUM:
                    return checkResolve(readEnum(unshared));

                case TC_OBJECT:
                    return checkResolve(readOrdinaryObject(unshared));//一般会先进入这里

                case TC_EXCEPTION:
                    IOException ex = readFatalException();
                    throw new WriteAbortedException("writing aborted", ex);

                case TC_BLOCKDATA:
                case TC_BLOCKDATALONG:
                    if (oldMode) {
                        bin.setBlockDataMode(true);
                        bin.peek();             // force header read
                        throw new OptionalDataException(
                            bin.currentBlockRemaining());
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected block data");
                    }

                case TC_ENDBLOCKDATA:
                    if (oldMode) {
                        throw new OptionalDataException(true);
                    } else {
                        throw new StreamCorruptedException(
                            "unexpected end of block data");
                    }

                default:
                    throw new StreamCorruptedException(
                        String.format("invalid type code: %02X", tc));
            }
        } finally {
            depth--;
            bin.setBlockDataMode(oldMode);
        }
    }
```

readOrdinaryObject是读取一个自定义类最上层直接调用的方法。几个关键的点，首先是要读取类的描述信息，然后根据类描述符创建一个实例，将对象实例存储到句柄缓存中并将句柄存储到passHandle，然后读取内部变量数据，最后检查有没有替换对象

```java
    private Object readOrdinaryObject(boolean unshared)
        throws IOException
    {
        if (bin.readByte() != TC_OBJECT) {
            throw new InternalError();
        }

        ObjectStreamClass desc = readClassDesc(false);//读取类描述信息
        desc.checkDeserialize();

        Class<?> cl = desc.forClass();
        if (cl == String.class || cl == Class.class
                || cl == ObjectStreamClass.class) {
            throw new InvalidClassException("invalid class descriptor");
        }

        Object obj;
        try {
            obj = desc.isInstantiable() ? desc.newInstance() : null;//根据类描述新建一个实例
        } catch (Exception ex) {
            throw (IOException) new InvalidClassException(
                desc.forClass().getName(),
                "unable to create instance").initCause(ex);
        }

        passHandle = handles.assign(unshared ? unsharedMarker : obj);//将对象存储到句柄缓存中
        ClassNotFoundException resolveEx = desc.getResolveException();
        if (resolveEx != null) {
            handles.markException(passHandle, resolveEx);
        }
        //读取序列化内部变量数据
        if (desc.isExternalizable()) {
            readExternalData((Externalizable) obj, desc);
        } else {
            readSerialData(obj, desc);
        }

        handles.finish(passHandle);//标记句柄对应的对象读取完成

        if (obj != null &&
            handles.lookupException(passHandle) == null &&
            desc.hasReadResolveMethod())//存在替换对象
        {
            Object rep = desc.invokeReadResolve(obj);
            if (unshared && rep.getClass().isArray()) {
                rep = cloneArray(rep);
            }
            if (rep != obj) {
                // Filter the replacement object
                if (rep != null) {
                    if (rep.getClass().isArray()) {
                        filterCheck(rep.getClass(), Array.getLength(rep));
                    } else {
                        filterCheck(rep.getClass(), -1);
                    }
                }
                handles.setObject(passHandle, obj = rep);
            }
        }

        return obj;
    }
```

那么先来看下怎么读取类的描述信息readClassDesc，从上面方法的case中不能分析出，在没有发生异常使用的情况下，此时的case只会进入TC_PROXYCLASSDESC或者TC_CLASSDESC也就是动态代理类的描述符合普通类的描述符。对于readProxyDesc读取并返回一个动态代理类的类描述符，而readNonProxyDesc则是返回一个普通类的描述符，两个方法都会设置passHandle到动态类描述符的设置句柄。如果类描述符不能被解析为本地虚拟机中的一个类，一个ClassNotFoundException会被关联到描述符的句柄中。关于动态代理和反射的部分下次要再仔细看一下。不过这里又是读取类描述，又是安全检查还有类合法性检查，跟直接在读取端指定类相比额外多出很多开销。

```java
    private ObjectStreamClass readClassDesc(boolean unshared)
        throws IOException
    {
        byte tc = bin.peekByte();
        ObjectStreamClass descriptor;
        switch (tc) {
            case TC_NULL:
                descriptor = (ObjectStreamClass) readNull();
                break;
            case TC_REFERENCE:
                descriptor = (ObjectStreamClass) readHandle(unshared);
                break;
            case TC_PROXYCLASSDESC:
                descriptor = readProxyDesc(unshared);
                break;
            case TC_CLASSDESC:
                descriptor = readNonProxyDesc(unshared);
                break;
            default:
                throw new StreamCorruptedException(
                    String.format("invalid type code: %02X", tc));
        }
        if (descriptor != null) {
            validateDescriptor(descriptor);
        }
        return descriptor;
    }
    
    private ObjectStreamClass readProxyDesc(boolean unshared)
        throws IOException
    {
        if (bin.readByte() != TC_PROXYCLASSDESC) {
            throw new InternalError();
        }

        ObjectStreamClass desc = new ObjectStreamClass();
        int descHandle = handles.assign(unshared ? unsharedMarker : desc);//添加描述符到句柄缓存中
        passHandle = NULL_HANDLE;

        int numIfaces = bin.readInt();//接口实现类数量
        if (numIfaces > 65535) {
            throw new InvalidObjectException("interface limit exceeded: "
                    + numIfaces);
        }
        String[] ifaces = new String[numIfaces];
        for (int i = 0; i < numIfaces; i++) {
            ifaces[i] = bin.readUTF();//读取接口名
        }

        Class<?> cl = null;
        ClassNotFoundException resolveEx = null;
        bin.setBlockDataMode(true);
        try {
            if ((cl = resolveProxyClass(ifaces)) == null) {
                resolveEx = new ClassNotFoundException("null class");
            } else if (!Proxy.isProxyClass(cl)) {
                throw new InvalidClassException("Not a proxy");
            } else {
            	//这个检查等价于isCustomSubclass
                ReflectUtil.checkProxyPackageAccess(
                        getClass().getClassLoader(),
                        cl.getInterfaces());
                // Filter the interfaces过滤接口
                for (Class<?> clazz : cl.getInterfaces()) {
                    filterCheck(clazz, -1);
                }
            }
        } catch (ClassNotFoundException ex) {
            resolveEx = ex;
        }

        // Call filterCheck on the class before reading anything else
        filterCheck(cl, -1);

        skipCustomData();

        try {
            totalObjectRefs++;
            depth++;
            desc.initProxy(cl, resolveEx, readClassDesc(false));//初始化类描述符
        } finally {
            depth--;
        }

        handles.finish(descHandle);
        passHandle = descHandle;
        return desc;
    }
    
    private ObjectStreamClass readNonProxyDesc(boolean unshared)
        throws IOException
    {
        if (bin.readByte() != TC_CLASSDESC) {
            throw new InternalError();
        }

        ObjectStreamClass desc = new ObjectStreamClass();
        int descHandle = handles.assign(unshared ? unsharedMarker : desc);//将描述符存储到句柄缓存中，返回句柄值
        passHandle = NULL_HANDLE;

        ObjectStreamClass readDesc = null;
        try {
            readDesc = readClassDescriptor();
        } catch (ClassNotFoundException ex) {
            throw (IOException) new InvalidClassException(
                "failed to read class descriptor").initCause(ex);
        }

        Class<?> cl = null;
        ClassNotFoundException resolveEx = null;
        bin.setBlockDataMode(true);
        final boolean checksRequired = isCustomSubclass();
        try {
            if ((cl = resolveClass(readDesc)) == null) {//通过类描述解析类
                resolveEx = new ClassNotFoundException("null class");
            } else if (checksRequired) {
                ReflectUtil.checkPackageAccess(cl);
            }
        } catch (ClassNotFoundException ex) {
            resolveEx = ex;
        }

        // Call filterCheck on the class before reading anything else
        filterCheck(cl, -1);

        skipCustomData();

        try {
            totalObjectRefs++;
            depth++;
            desc.initNonProxy(readDesc, cl, resolveEx, readClassDesc(false));//初始化类描述符
        } finally {
            depth--;
        }

        handles.finish(descHandle);
        passHandle = descHandle;

        return desc;
    }
```

接下来是读取序列化内部变量数据，由于readExternalData除了一些安全性的判断外，直接调用了类中的方法，所以就不看了，直接看readSerialData中默认的读取数据部分defaultReadFields。注意到基本数据类型可以直接通过字节输入流来进行读取，而非基本数据类型则需要递归调用readObject0来读取。

```java
    private void defaultReadFields(Object obj, ObjectStreamClass desc)
        throws IOException
    {
        Class<?> cl = desc.forClass();
        if (cl != null && obj != null && !cl.isInstance(obj)) {
            throw new ClassCastException();
        }

        int primDataSize = desc.getPrimDataSize();
        if (primVals == null || primVals.length < primDataSize) {
            primVals = new byte[primDataSize];//第一次进行需要初始化缓冲区，缓冲区不足时需要新建一个更大的缓冲区
        }
            bin.readFully(primVals, 0, primDataSize, false);//将字节流读取到缓冲区
        if (obj != null) {//从readSerialData进入这个方法时obj为null
            desc.setPrimFieldValues(obj, primVals);
        }

        int objHandle = passHandle;
        ObjectStreamField[] fields = desc.getFields(false);
        Object[] objVals = new Object[desc.getNumObjFields()];
        int numPrimFields = fields.length - objVals.length;
        for (int i = 0; i < objVals.length; i++) {
            ObjectStreamField f = fields[numPrimFields + i];
            objVals[i] = readObject0(f.isUnshared());//递归读取非基本数据类型类
            if (f.getField() != null) {
                handles.markDependency(objHandle, passHandle);//记录依赖
            }
        }
        if (obj != null) {
            desc.setObjFieldValues(obj, objVals);
        }
        passHandle = objHandle;
    }
```

然后看一下其他非基本数据类的读取。readNull读取null代码并将句柄设置为NULL_HANDLE，readString读取UTF编码，readArray需要先读取数组长度，然后根据类的类型进行读取。

```java
    private Object readNull() throws IOException {
        if (bin.readByte() != TC_NULL) {
            throw new InternalError();
        }
        passHandle = NULL_HANDLE;
        return null;
    }

    private String readString(boolean unshared) throws IOException {
        String str;
        byte tc = bin.readByte();
        switch (tc) {
            case TC_STRING:
                str = bin.readUTF();
                break;

            case TC_LONGSTRING:
                str = bin.readLongUTF();
                break;

            default:
                throw new StreamCorruptedException(
                    String.format("invalid type code: %02X", tc));
        }
        passHandle = handles.assign(unshared ? unsharedMarker : str);
        handles.finish(passHandle);
        return str;
    }

    private Object readArray(boolean unshared) throws IOException {
        if (bin.readByte() != TC_ARRAY) {
            throw new InternalError();
        }

        ObjectStreamClass desc = readClassDesc(false);
        int len = bin.readInt();

        filterCheck(desc.forClass(), len);

        Object array = null;
        Class<?> cl, ccl = null;
        if ((cl = desc.forClass()) != null) {
            ccl = cl.getComponentType();
            array = Array.newInstance(ccl, len);
        }

        int arrayHandle = handles.assign(unshared ? unsharedMarker : array);
        ClassNotFoundException resolveEx = desc.getResolveException();
        if (resolveEx != null) {
            handles.markException(arrayHandle, resolveEx);
        }

        if (ccl == null) {
            for (int i = 0; i < len; i++) {
                readObject0(false);
            }
        } else if (ccl.isPrimitive()) {
            if (ccl == Integer.TYPE) {
                bin.readInts((int[]) array, 0, len);
            } else if (ccl == Byte.TYPE) {
                bin.readFully((byte[]) array, 0, len, true);
            } else if (ccl == Long.TYPE) {
                bin.readLongs((long[]) array, 0, len);
            } else if (ccl == Float.TYPE) {
                bin.readFloats((float[]) array, 0, len);
            } else if (ccl == Double.TYPE) {
                bin.readDoubles((double[]) array, 0, len);
            } else if (ccl == Short.TYPE) {
                bin.readShorts((short[]) array, 0, len);
            } else if (ccl == Character.TYPE) {
                bin.readChars((char[]) array, 0, len);
            } else if (ccl == Boolean.TYPE) {
                bin.readBooleans((boolean[]) array, 0, len);
            } else {
                throw new InternalError();
            }
        } else {
            Object[] oa = (Object[]) array;
            for (int i = 0; i < len; i++) {
                oa[i] = readObject0(false);
                handles.markDependency(arrayHandle, passHandle);
            }
        }

        handles.finish(arrayHandle);
        passHandle = arrayHandle;
        return array;
    }
```

readHandle方法从缓存中寻找该对象

```java
    private Object readHandle(boolean unshared) throws IOException {
        if (bin.readByte() != TC_REFERENCE) {
            throw new InternalError();
        }
        passHandle = bin.readInt() - baseWireHandle;//因为写的时候handle值加上了baseWireHandle
        if (passHandle < 0 || passHandle >= handles.size()) {
            throw new StreamCorruptedException(
                String.format("invalid handle value: %08X", passHandle +
                baseWireHandle));
        }
        if (unshared) {
            // REMIND: what type of exception to throw here?
            throw new InvalidObjectException(
                "cannot read back reference as unshared");
        }

        Object obj = handles.lookupObject(passHandle);//缓存中寻找该对象
        if (obj == unsharedMarker) {
            // REMIND: what type of exception to throw here?
            throw new InvalidObjectException(
                "cannot read back reference to unshared object");
        }
        filterCheck(null, -1);       // just a check for number of references, depth, no class检查引用数量，递归深度，是否有这个类
        return obj;
    }
```

## BlockDataInputStream

跟输出时相同，对象输入流也使用的是块输入流，输入流有两个模式：默认模式下，输入数据写入的格式和DataOutputStream相同；在块数据模式，输入的数据被块数据标记归为一类。逻辑上来说，读和写应该是相对应的，主要是要判断读取到的头部信息，然后根据不同的信息来读取实际信息。有些跟输出部分非常重复的就跳过了

内部变量主要需要注意两个不同的输入流，din是块输入流，in是取数输入流，in中的下层输入流是最初在构造ObjectInputStream中传入的变量，所以通常是FileInputStream。

```java
        /** maximum data block length最大数据块长度1K */
        private static final int MAX_BLOCK_SIZE = 1024;
        /** maximum data block header length最大数据块头部长度 */
        private static final int MAX_HEADER_SIZE = 5;
        /** (tunable) length of char buffer (for reading strings)可调节的字符缓冲的长度作为供读取的字符串 */
        private static final int CHAR_BUF_SIZE = 256;
        /** readBlockHeader() return value indicating header read may block readBlockHeader()返回的值指示头部可能阻塞 */
        private static final int HEADER_BLOCKED = -2;

        /** buffer for reading general/block data读取一般/块数据的缓冲区 */
        private final byte[] buf = new byte[MAX_BLOCK_SIZE];
        /** buffer for reading block data headers读取块数据头部的缓冲区 */
        private final byte[] hbuf = new byte[MAX_HEADER_SIZE];
        /** char buffer for fast string reads快速字符串读取的字符缓冲区 */
        private final char[] cbuf = new char[CHAR_BUF_SIZE];

        /** block data mode块数据模式 */
        private boolean blkmode = false;

        // block data state fields; values meaningful only when blkmode true块数据状态字段，这些值仅在blkmode为true时有意义
        /** current offset into buf当前进入buf的偏移 */
        private int pos = 0;
        /** end offset of valid data in buf, or -1 if no more block data buf中有效值的结束偏移，没有更多块数据buf时为-1 */
        private int end = -1;
        /** number of bytes in current block yet to be read from stream 在当前block中还没有从流中读取的字节数 */
        private int unread = 0;

        /** underlying stream (wrapped in peekable filter stream) 下层流包装在可见过滤流中 */
        private final PeekInputStream in;
        /** loopback stream (for data reads that span data blocks) 回路流用于扩展数据块的数据读取 */
        private final DataInputStream din;
```

构造函数就是初始化两个流变量

```java
        BlockDataInputStream(InputStream in) {
            this.in = new PeekInputStream(in);
            din = new DataInputStream(this);
        }
```

setBlockDataMode将块数据模式设为给定的模式，true是打开，off是关闭，并返回之前的模式值。如果新的模式和旧模式一样，什么都不做。如果块数据模式从打开被改为关闭时有未消费的块数据还留在流中，会抛出IllegalStateException

```java
        boolean setBlockDataMode(boolean newmode) throws IOException {
            if (blkmode == newmode) {
                return blkmode;
            }
            if (newmode) {//从关闭到打开，重置块数据参数状态
                pos = 0;
                end = 0;
                unread = 0;
            } else if (pos < end) {//从打开到关闭且有未消费的块数据
                throw new IllegalStateException("unread block data");
            }
            blkmode = newmode;
            return !blkmode;
        }
```

skipBlockData如果在块数据模式，跳跃到数据块的当前组的末尾但不会取消块数据模式。如果不在块数据模式，抛出IllegalStateException。这个方法调用了refill，refill用块数据再装满内部缓冲区。任何buf中的数据在调用这个方法时被认为是已经被消费了。设置pos、end、unread字段值来反映有效块数据的数量。如果流中的下一个元素不是一个数据块，设置pos=0和unread=0，end=-1

```java
        void skipBlockData() throws IOException {
            if (!blkmode) {
                throw new IllegalStateException("not in block data mode");
            }
            while (end >= 0) {
                refill();
            }
        }

        private void refill() throws IOException {
            try {
                do {
                    pos = 0;
                    if (unread > 0) {//当前block中还有未读取的字节
                        int n =
                            in.read(buf, 0, Math.min(unread, MAX_BLOCK_SIZE));//读取unread和最大数据块长度中的较小值的数据到buf中
                        if (n >= 0) {
                            end = n;//end为读取到的字节数
                            unread -= n;//当前block中还没读取的字节减少n
                        } else {
                            throw new StreamCorruptedException(
                                "unexpected EOF in middle of data block");
                        }
                    } else {
                        int n = readBlockHeader(true);
                        if (n >= 0) {
                            end = 0;
                            unread = n;
                        } else {
                            end = -1;
                            unread = 0;
                        }
                    }
                } while (pos == end);//成功读取到数据就会退出循环
            } catch (IOException ex) {//出现异常说明下一个元素不是数据块
                pos = 0;
                end = -1;
                unread = 0;
                throw ex;
            }
        }
```

readBlockHeader尝试读取下一个块数据头部。如果canBlock是false并且一个完整的头部没有阻塞时不能被读取，返回HEADER_BLOCKED。否则如果流中下一个元素是一个块数据头部，返回头部标识的这个块数据的长度，否则返回-1

```java
        private int readBlockHeader(boolean canBlock) throws IOException {
            if (defaultDataEnd) {
                /*
                 * 流当前在一个字段值块通过默认序列化写入的末尾。因为没有终点TC_ENDBLOCKDATA标志，
                 * 明确地模仿常规数据结束行为。
                 */
                return -1;
            }
            try {
                for (;;) {
                    int avail = canBlock ? Integer.MAX_VALUE : in.available();//可以阻塞时avail是最大整数，不能阻塞时是流中能读取的字节数
                    if (avail == 0) {
                        return HEADER_BLOCKED;//没有可读取的字节数时返回HEADER_BLOCKED
                    }

                    int tc = in.peek();//从流中取数
                    switch (tc) {
                        case TC_BLOCKDATA://第二个字节是块数据长度
                            if (avail < 2) {
                                return HEADER_BLOCKED;
                            }
                            in.readFully(hbuf, 0, 2);
                            return hbuf[1] & 0xFF;

                        case TC_BLOCKDATALONG://块字节长度需要4个字节来表示
                            if (avail < 5) {
                                return HEADER_BLOCKED;
                            }
                            in.readFully(hbuf, 0, 5);
                            int len = Bits.getInt(hbuf, 1);//位运算获取后4位的整数值
                            if (len < 0) {
                                throw new StreamCorruptedException(
                                    "illegal block data header length: " +
                                    len);
                            }
                            return len;

                        /*
                         * TC_RESET可能发生在数据块之间。不幸的是，这个状况必须在比其他类型标志更低的级别被解析，
                         * 因为原始数据读取可能跨越被TC_RESET分割的数据块
                         */
                        case TC_RESET:
                            in.read();//跳过这个头部
                            handleReset();//重置
                            break;

                        default:
                            if (tc >= 0 && (tc < TC_BASE || tc > TC_MAX)) {
                                throw new StreamCorruptedException(
                                    String.format("invalid type code: %02X",
                                    tc));
                            }
                            return -1;
                    }
                }
            } catch (EOFException ex) {
                throw new StreamCorruptedException(
                    "unexpected EOF while reading block data header");
            }
        }
```

单个字节的read和peek都是基于refill来完成的，而read多字节时，最大不能超过缓冲区内剩余的字节，如果copy为true，需要先将字节读取到一个内部缓冲区再复制到目标数组，又增加开销。

```java
        int read(byte[] b, int off, int len, boolean copy) throws IOException {
            if (len == 0) {
                return 0;
            } else if (blkmode) {
                if (pos == end) {
                    refill();
                }
                if (end < 0) {
                    return -1;
                }
                int nread = Math.min(len, end - pos);//最大不超过缓冲区内剩余的字节
                System.arraycopy(buf, pos, b, off, nread);
                pos += nread;
                return nread;
            } else if (copy) {//copy模式先将数据读取到一个缓冲区，再从缓冲区复制到目标数组
                int nread = in.read(buf, 0, Math.min(len, MAX_BLOCK_SIZE));
                if (nread > 0) {
                    System.arraycopy(buf, 0, b, off, nread);
                }
                return nread;
            } else {
                return in.read(b, off, len);
            }
        }
```

readFully循环调用read直到达到要读取的字节数，如果不足会抛出EOFException

```java
        public void readFully(byte[] b, int off, int len, boolean copy)
            throws IOException
        {
            while (len > 0) {
                int n = read(b, off, len, copy);
                if (n < 0) {
                    throw new EOFException();
                }
                off += n;
                len -= n;
            }
        }
```

## HandleTable

比起输入流来，输入流的缓存表多了一个HandleList[] deps用来存储依赖，HandleList可以理解为一个简易版的ArrayList，当数组被填满再次增加元素的时候，自动分配一个新的2倍大小的数组，把旧的元素复制过去再插入新的。另外两个数组，Object[] entries存储对象或异常，byte[] status存储对象的状态。并且这个缓存表并没有使用hash，它是依靠ObjectInputStream中的passHandle来获取位置的。

```java
        /** array mapping handle -> object status 句柄->对象状态的数组映射*/
        byte[] status;
        /** array mapping handle -> object/exception (depending on status) */
        Object[] entries;
        /** array mapping handle -> list of dependent handles (if any) */
        HandleList[] deps;
        /** lowest unresolved dependency 最低的未解决依赖*/
        int lowDep = -1;
        /** number of handles in table */
        int size = 0;
```

从assign我们可以看到，对象和它的状态（默认为UNKNOWN）被顺序存储到了数组中，并返回下标值

```java
        int assign(Object obj) {
            if (size >= entries.length) {
                grow();
            }
            status[size] = STATUS_UNKNOWN;
            entries[size] = obj;
            return size++;
        }
```

关于deps这个数组到底是干什么的，我们可以一步步来分析。首先，寻找deps在HandleTable中被修改的地方，除了构造时初始化以外，finish中deps对应状态为UNKNOWN状态的全部被设为NULL，clear当中deps全部为设为null，grow当中新建了一个大小是原本2倍加1的deps数组并复制了原有内容来取代之前的数组，这些都不足以分析它的用途，关键在markDependency和markException两个方法中。首先markException我们可以看到所有调用都是发生在出现ClassNotFoundException的时候，传入的参数是这个类的handle值和异常。markDependency关联一个ClassNotFoundException和当前活动句柄并传播它到其他合适的引用对象。这个特定的句柄必须是打开的。简单的来说，如果一个类解析失败，那么添加到entries缓存里的对象会是一个异常，它对应的status为STATUS_EXCEPTION

```java
        void markException(int handle, ClassNotFoundException ex) {
            switch (status[handle]) {
                case STATUS_UNKNOWN:
                    status[handle] = STATUS_EXCEPTION;//标记当前对象状态为异常
                    entries[handle] = ex;//修改缓存中的对象为异常

                    // propagate exception to dependents传播异常给依赖
                    HandleList dlist = deps[handle];
                    if (dlist != null) {
                        int ndeps = dlist.size();
                        for (int i = 0; i < ndeps; i++) {
                            markException(dlist.get(i), ex);//递归传播依赖列表内的所有对象
                        }
                        deps[handle] = null;//清除依赖表中的对象，因为只要再有对象关联到相同类它必然是会从缓存中获取异常，不再需要依赖表项
                    }
                    break;

                case STATUS_EXCEPTION:
                    break;

                default:
                    throw new InternalError();
            }
        }
```

再来看一下markDependency的调用，一处是在readObject调用readObject0之后，传入参数是上一层对象的句柄和本层对象的句柄；一处是在readArray中读取非基本数据类型调用readObject0之后，传入参数也是上一层对象的句柄和本层对象的句柄；一处是在defaultReadFields中递归读取非基本数据类中，传入参数也是上一层对象的句柄和本层对象的句柄。总之，根据上面这几个例子再结合方法的注释，我们可以推测出这里deps的作用是在一层层读取对象时，比如自定义类中又又自定义类，**记录下一层指向上一层的关系，用来传递异常状态**。很明显，从代码中可以看出，在存在依赖双方的情况下，如果上一层状态是UNKNOWN，下一层状态是EXCEPTION，则从依次向上将整个依赖表上所有的状态全部置为EXCEPTION，如果下一层也是UNKNOWN状态，则在下一层的依赖表中增加上一层的句柄，并将未知状态最底层设为target。如果上一层状态是OK，则出现InternalError，因为上一层的句柄已经被关闭，不能再增加对它依赖的对象。

```java
        void markDependency(int dependent, int target) {
            if (dependent == NULL_HANDLE || target == NULL_HANDLE) {
                return;
            }
            switch (status[dependent]) {

                case STATUS_UNKNOWN:
                    switch (status[target]) {
                        case STATUS_OK:
                            // ignore dependencies on objs with no exception忽略没有异常的对象上依赖
                            break;

                        case STATUS_EXCEPTION:
                            // eagerly propagate exception急切传播的异常
                            markException(dependent,
                                (ClassNotFoundException) entries[target]);//如果依赖是未知状态也需将它们改为异常状态
                            break;

                        case STATUS_UNKNOWN:
                            // add to dependency list of target增加到依赖目标列表
                            if (deps[target] == null) {
                                deps[target] = new HandleList();
                            }
                            deps[target].add(dependent);

                            // remember lowest unresolved target seen记录被看见最低的未解决目标
                            if (lowDep < 0 || lowDep > target) {
                                lowDep = target;
                            }
                            break;

                        default:
                            throw new InternalError();
                    }
                    break;

                case STATUS_EXCEPTION:
                    break;

                default:
                    throw new InternalError();
            }
        }
```

finish是关闭句柄，是唯一可以将状态设为STATUS_OK的方法，标记给出的句柄为结束状态，说明这个句柄不会有新的依赖标记。调用设置和结束方法必须以后进先出的顺序。必须要handle之前不存在无法确认的状态才能修改状态为OK

```java
        void finish(int handle) {
            int end;
            if (lowDep < 0) {
                // 没有还没处理的未知状态，只需要解决当前句柄
                end = handle + 1;
            } else if (lowDep >= handle) {
                // handle之后存在还没有处理的未知状态，但上层句柄都已经解决了
                end = size;
                lowDep = -1;
            } else {
                // handle之前还有没有处理的向前引用，还不能解决所有对象
                return;
            }

            // change STATUS_UNKNOWN -> STATUS_OK in selected span of handles
            for (int i = handle; i < end; i++) {
                switch (status[i]) {
                    case STATUS_UNKNOWN:
                        status[i] = STATUS_OK;
                        deps[i] = null;
                        break;

                    case STATUS_OK:
                    case STATUS_EXCEPTION:
                        break;

                    default:
                        throw new InternalError();
                }
            }
        }
```

