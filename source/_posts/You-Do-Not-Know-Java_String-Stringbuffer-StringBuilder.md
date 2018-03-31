---
title: 你不知道的Java-之String-StringBuffer-StringBuilder
date: 2018-03-27 16:24:56
tags: Java
categories: Java源码
---

#### 简介

StringBuffer，由名字可以看出，是一个String的缓冲区，也就是说一个类似于String的字符串缓冲区，和String不同的是，它可以被修改，而且是线程安全的。StringBuffer在任意时刻都有一个特定的字符串序列，不过这个序列和它的长度可以通过一些函数调用进行修改。

<!-- more -->

它的结构层次如下图：

StringBuffer是线程安全的，因此如果有几个线程同时操作StringBuffer，对它来说也只是一个操作序列，所有操作串行发生。

每一个StringBuffer都有一个容量，如果内容的大小不超过容量，StringBuffer就不会分配更大容量的缓冲区。如果需要更大的容量，StringBuffer会自动增加容量。和StringBuffer类似的有StringBuilder，两者之间的操作相同，不过StringBuilder不是线程安全的。虽然如此，由于StringBuilder没有同步，所以它的速度更快一些。

#### StringBuffer的原理

StringBuffer继承了抽象类AbstractStringBuilder，在AbstractStringBuilder类中，有两个字段分别是char[]类型的value和int类型的count，也就是说，StringBuffer本质上是一个字符数组：

```
/** 
 * The value is used for character storage. 
 */  
char[] value;  
  
/** 
 * The count is the number of characters used. 
 */  
int count; 
```

value用来存储字符，而count表示数组中已有内容的大小，也就是长度。StringBuffer的主要操作有append、insert等，这些操作都是在value上进行的，而不是像String一样每次操作都要new一个String，因此，StringBuffer在效率上要高于String。有了append、insert等操作，value的大小就会改变，那么StringBuffer是如何操作容量的改变的呢？
首先StringBuffer有个继承自AbstractStringBuilder类的ensureCapacity的方法：
```
@Override  
public synchronized void ensureCapacity(int minimumCapacity) {  
    if (minimumCapacity > value.length) {  
        expandCapacity(minimumCapacity);  
    }  
}
```

StringBuffer对其进行了重写，直接调用父类的expandCapacity方法。这个方法用来保证value的长度大于给定的参数minimumCapacity，在父类的expandCapacity方法中这样操作：

```
/** 
 * This implements the expansion semantics of ensureCapacity with no 
 * size check or synchronization. 
 */  
void expandCapacity(int minimumCapacity) {  
    int newCapacity = value.length * 2 + 2;  
    if (newCapacity - minimumCapacity < 0)  
        newCapacity = minimumCapacity;  
    if (newCapacity < 0) {  
        if (minimumCapacity < 0) // overflow  
            throw new OutOfMemoryError();  
        newCapacity = Integer.MAX_VALUE;  
    }  
    value = Arrays.copyOf(value, newCapacity);  
}  
```
也就是说，value新的大小是max(value.length*2+2,minimumCapacity)，上面代码中的第二个判断是为了防止newCapacity溢出。
同时，StringBuffer还有一个直接设置count大小的函数setLength：

```
@Override  
public synchronized void setLength(int newLength) {  
    toStringCache = null;  
    super.setLength(newLength);  
}  
```
函数直接调用父类的函数，父类函数如下：

```
public void setLength(int newLength) {  
    if (newLength < 0)  
        throw new StringIndexOutOfBoundsException(newLength);  
    ensureCapacityInternal(newLength);  
  
    if (count < newLength) {  
        Arrays.fill(value, count, newLength, '\0');  
    }  
  
    count = newLength;  
}  
```
从代码中可以看出，如果newLength大于count，那么就会在后面添加'\0'补充。如果小于count，就直接使count=newLength。
StringBuffer中的每一个append和insert函数都会调用父类的函数：

```
@Override  
public synchronized StringBuffer append(Object obj) {  
    toStringCache = null;  
    super.append(String.valueOf(obj));  
    return this;  
}  
  
@Override  
public synchronized StringBuffer insert(int index, char[] str, int offset,  
                                        int len)  
{  
    toStringCache = null;  
    super.insert(index, str, offset, len);  
    return this;  
}  
```
而在父类中，这些函数都会首先保证value的大小够存储要添加的内容：

```
public AbstractStringBuilder append(String str) {  
    if (str == null)  
        return appendNull();  
    int len = str.length();  
    ensureCapacityInternal(count + len);  
    str.getChars(0, len, value, count);  
    count += len;  
    return this;  
}  
  
public AbstractStringBuilder insert(int index, char[] str, int offset,  
                                    int len)  
{  
    if ((index < 0) || (index > length()))  
        throw new StringIndexOutOfBoundsException(index);  
    if ((offset < 0) || (len < 0) || (offset > str.length - len))  
        throw new StringIndexOutOfBoundsException(  
            "offset " + offset + ", len " + len + ", str.length "  
            + str.length);  
    ensureCapacityInternal(count + len);  
    System.arraycopy(value, index, value, index + len, count - index);  
    System.arraycopy(str, offset, value, index, len);  
    count += len;  
    return this;  
}  
```
父类中通过ensureCapacityInternal的函数保证大小，函数如下：

```
private void ensureCapacityInternal(int minimumCapacity) {  
    // overflow-conscious code  
    if (minimumCapacity - value.length > 0)  
        expandCapacity(minimumCapacity);  
} 
```
如果空间不够，最终也是调用expandCapacity方法。这样就保证了随着操作value的空间够用。
还有就是，StringBuffer可以通过toString方法转换为String，它有一个私有的字段toStringCache：

```
/** 
 * A cache of the last value returned by toString. Cleared 
 * whenever the StringBuffer is modified. 
 */  
private transient char[] toStringCache; 
```
也就是一个缓存，用来保存上一次调用toString的结果，如果value的字符串序列发生改变，就会将它清空。toString代码如下：
```
@Override  
public synchronized String toString() {  
    if (toStringCache == null) {  
        toStringCache = Arrays.copyOfRange(value, 0, count);  
    }  
    return new String(toStringCache, true);  
}  
```
首先判断toStringCache是否为null，如果是先将value复制到缓存里，然后使用toStringCache new一个String。

#### 常用方法

##### 构造函数

StringBuffer有四个构造函数：
```
StringBuffer() value内容为空，并设置容量为16个字节。
StringBuffer(CharSequece seq)  使用seq初始化，容量在此基础上加16。
StringBuffer(int capacity) 设置特定容量。
StringBuffer(String str)  使用str初始化，容量str大小的基础上加16。
```
##### append方法

由于继承了Appendable接口，所以要实现append方法，StringBuffer类对几乎所有的基本类型都重载了append方法：
```
append(boolean b)
append(char c)
append(char[] str)
append(char[] str,int offset,int len)
append(CharSequence s)
append(CharSequence s,int start,int end)
append(double d)
append(float f)
append(int i)
append(long lng)
append(Object obj)
append(String str)
append(StringBuffer sb)
```
##### insert方法

insert方法可以控制插入的起始位置，也几乎对所有的基本类型都重载了insert方法：
```
insert(int offser,boolean b)
insert(int offset,char c)
insert(int offset,char[] str)
insert(int index,char[] str,int offset,int len)
insert(int dsfOffset,CharSequence s)
insert(int dsfOffset,CharSequence s,int start,int end)
insert(int offset,double d)
insert(int offset,float f)
insert(int offset,int i)
insert(int offset,long l)
insert(int offset,Object obj)
insert(int offset,String str)
```
##### 其它会改变内容的方法

上面的那些方法会增加StringBuffer的内容，还有一些方法可以改变StringBuffer的内容：

StringBuffer delete(int start,int end) 删除从start到end（不包含）之间的内容。
StringBuffer deleteCharAt(int index) 删除index位置的字符。
StringBuffer replace(int start,int end,String str) 用str中的字符替换value中从start到end位置的子序列。
StringBuffer reverse() 反转。
void setCharAt(int index,char ch) 使用ch替换位置index处的字符。
void setLength(int newLength) 可能会改变内容（添加'\0'）。

##### 其它常用方法

下面这些方法不会改变内容
```
/** 返回value的大小即容量。 **/
int capacity() 

/** 返回内容的大小，即count。 **/
int length() 

/** 返回位置index处的字符。 **/
char charAt(int index)
 
/** 确保容量至少是minimumCapacity。 **/
void ensureCapacity(int minimumCapacity) 

/** 返回srcBegin到srcEnd的字符到dst。 **/
void getChars(int srcBegin, int srcEnd,char[] dst, int dstBegin) 

/** 返回str第一次出现的位置。 **/
int indexOf(String str) 

/** 返回从fromIndex开始str第一次出现的位置。 **/
int indexOf(String str, int fromIndex) 

/** 返回str最后出现的位置。 **/
int lastIndexOf(String str) 

/** 返回从fromIndex开始最后一次出现str的位置。 **/
int lastIndexOf(String str, int fromIndex) 

/** 返回字符子序列。 **/
CharSequence subSequence(int start, int end) 

/** 返回子串。 **/
String substring(int start) 

/** 返回子串。 **/
String substring(int start, int end) 

/** 返回value形成的字符串。 **/
String toString() 

/** 缩小value的容量到真实内容大小。 **/
void trimToSize() 
```
#### 与String和StringBuilder的区别

三者都是处理字符串常用的类，不同在于：

速度上：String < StringBuffer < StringBuilder。

安全上：StringBuffer线程安全，StringBuilder线程不安全。

原因在于，String的每次改变都会涉及到字符数组的复制，而StringBuffer和StringBuilder直接在字符数组上改变。同时，StringBuffer是线程安全的，而StringBuilder线程不安全，没有StringBuffer的同步，所以StringBuilder快于StringBuffer。

#### 总结

如果对字符串的改变少，使用String。

如果对字符串修改的较多，需要线程安全就用StringBuffer，不需要就使用StringBuilder。