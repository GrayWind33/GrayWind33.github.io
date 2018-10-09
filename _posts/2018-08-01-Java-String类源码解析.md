---
layout:     post
title:      Java String类源码解析
subtitle:   分析字符串String实现原理
date:       2018-08-01
author:     GrayWind
header-img: img/post-bg-debug.png
catalog: true
tags:
    - JDK源码解析
---

String直接继承Object

含有一个char[] value，还有一个int hash默认值为0

new String()的构造产生的是一个值为””的字符数组

String(**char** value[], **int** offset, **int** count)当count=0且offset<=value.length时构造一个值为””的字符串。offset>0且offset+count<=value.length时复制该部分子串。其余情况都会抛错。

字符数据类型是一个采用UTF-16编码表示Unicode代码点的代码单元。大多数的常用Unicode字符使用一个代码单元就可以表示，而辅助字符需要一对代码单元表示。而length返回的是UTF-16下的代码单元的数量，而codePointCount返回的是代码点的数量。对于大部分人工输入的字符，这两者是相等的，会出现length比codePointCount长的通常是某些数学或者机器符号，需要两个代码单元来表示一个代码点。

对于返回char[]的方法，底层调用的是System.arraycopy方法，这也是高效的数组拷贝函数。

getBytes方法会调用StringCoding.encode返回序列化后的byte[]

关于String a == String b的判断，是指a和b指向内存中的同一个对象，凡是new String初始化的对象，都不会产生a==b的情况，因为他会新开辟一个对象空间，然后复制value的值，仅当b=a初始化时a==b成立。

```java
public static void main(String args[]) {
        String a, b;
        a = "123";
        b = "123";
        System.out.println(a==b);//true
        a = "123";
        b = new String("123");
        System.out.println(a==b);//false
        a = new String("123");
        b = new String("123");
        System.out.println(a==b);//false
        a = "123";
        b = new String(a);
        System.out.println(a==b);//false
        a = new String("123");
        b = a;
        System.out.println(a==b);//true
    }
```

而a.equals(b)先判断a == b是否成立，再判断b是否是String类，然后逐个比较value数组的值是否相等。equalsIgnoreCase在此基础上忽略大小写的区别

a.compareTo(b)比较a和b第一个不相等字符的差值，若都相等则比较长度差值。compareToIgnoreCase多一个忽略大小写的区别。regionMatches(int toffset, String other, int ooffset, int len)则是比较a从toffset开始和other从ooffset开始长度为len的部分是否相等。

startsWith(String prefix, int toffset)字符串从tooffset位置开始和prefix是否相等。endsWith(String suffix)字符串结尾和suffix等长部分是否相等。

hashCode()调用时，若hash值为0且字符串长度不为0，则要计算hash值，方法是value数组化为31进制

indexOf是返回第一个出现的指定值的位置，可以通过fromIndex来指定开始查找的位置，而indexOfSupplementary是忽略大小写的该方法。lastIndexOf则是从尾部开始查找最后一个。

substring根据指定的位置返回一个新的子字符串，若指定位置不符合原字符串的长度，则抛错。

a.concat(String str)新建一个字符串内容是a+str并返回，不会修改a原本的值

```java
public static void main(String args[]) {
        String a, b;
        a = "123";
        b = "123";
        a.concat(b);
        System.out.println(a);//123
        System.out.println(a.concat(b));//123123
        a = a.concat(b);
        System.out.println(a);//123123
    }
```

replace(char oldChar, char newChar)生成一个新的字符串，将原字符串中的oldChar字符全部替换为newChar，不会改变原字符串的值。replaceAll(String regex, String replacement)和前一个方法相比，参数regex是正则表达式，其余相同。

```java
public static void main(String args[]) {
        String a;
        a = "12131";
        a.replace("1", "a");
        System.out.println(a);//12131
        a = a.replace("1", "a");
        System.out.println(a);//a2a3a
    }
```

split(String regex, int limit)将字符串按照给定的正则表达式分割为字符串组，limit是分割产生的数组最大数量，对于多余部分不做分割全部保留在最后一个字符串中。

```java
public static void main(String args[]) {
        String a;
        a = "1,2,3,4";
        String[] b = a.split(",");
        for(String t : b){
            System.out.print(t + " ");//1 2 3 4
        }
        System.out.println("");
        String[] c = a.split(",", 3);
        for(String t : c){
            System.out.print(t + " ");//1 2 3,4
        }
    }
```

toCharArray()复制出一个新的char[]而不是直接返回value

trim()生成一个新的字符串，去掉头部的所有空格

**public** **native** String intern()这个方法的作用是在常量池当中寻找是否已经存在该字符串，若已存在则返回该引用，若不存在则在常量池新建。从上面的源码分析中，我们可以看出String的所有操作都是返回一个新的字符串，对自身是没有修改的，String被设计为一个不可变的final对象，理由有以下几点：

1. 字符串常量池的需要。字符串常量池的诞生是为了提升效率和减少内存分配。、因为String的不可变性，常量池很容易被管理和优化。
2. 安全性考虑。正因为使用字符串的场景如此之多，所以设计成不可变可以有效的防止字符串被有意或者无意的篡改。（通过反射或者Unsafe直接操作内存的手段也可以实现对所谓不可变String的修改）。
3. 作为HashMap、HashTable等hash型数据key的必要。因为不可变的设计，jvm底层很容易在缓存String对象的时候缓存其hashcode，这样在执行效率上会大大提升。

```java
public static void main(String args[]) {
        String a, b;
        a = "123";
        b = new String(a).intern();
        System.out.println(a == b);//true
        a = "12" + "3";
        b = "123";
        System.out.println(a == b);//true
        a = "12" + "3";
        b = new String("123");
        System.out.println(a == b);//false
        a = "12" + "3";
        b = new String("123").intern();
        System.out.println(a == b);//true
        a = new String("123");
        b = a.intern();
        System.out.println(a == b);//false
    }
```

从上面一段代码的运行结果我们可以看到，intern()会从常量池寻找指定的字符串，指向同一个常量池对象的时候，a==b就是成立的。这里说明一下最后一个case，首先常量池存在了”123”，然后a获得的引用是另一个”123”（因为是new String得到的对象），而b得到的是常量池中第一个”123”的引用，所以a!=b。对于字符串相加的操作"12" + "3"，操作过后常量池内会有3个字符串，"12"  "3" “123”