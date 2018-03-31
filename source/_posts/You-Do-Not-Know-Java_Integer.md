---
title: 你不知道的Java - 之Integer类
date: 2018-03-26 23:48:09
tags: Java
categories: Java源码
---


Java用了这么久，今天突然发现一个很有意思的事：
> 为什么在Java中1000 == 1000为false，而100 == 100为true?

这是一个挺有意思的讨论话题。
如果你运行下面的代码

```
Integer a = 1000, b = 1000;
Integer c = 100, d = 100;
System.out.println(a == b);
System.out.println(c == d);
```
你会得到：
false
true

<!-- more -->

基本知识：我们知道，如果两个引用指向同一个对象，用 == 表示它们是相等的。如果两个引用指向不同的对象，用 == 表示它们是不相等的，即使它们的内容相同。

因此，后面一条语句也应该是false 。

这就是它有趣的地方了。如果你看去看Integer.Java类，你会发现有一个内部私有类，IntegerCache.java，它缓存了从-128到127之间的所有的整数对象。

所以事情就成了，所有的小整数在内部缓存，然后当我们声明类似——
```
Integer c = 100;
``` 
的时候，它实际上在内部做的是：
```
Integer i = Integer.valueOf(100); 
```
现在，如果我们去看valueOf()方法，我们可以看到：
```
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i
    return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```
如果值的范围在-128到127之间，它就从高速缓存返回实例。

所以…
```
Integer c = 100, d = 100; 
```
指向了同一个对象。

这就是为什么我们写：
```
System.out.println(c == d); 
```
我们可以得到true。

> 现在你可能会问，为什么这里需要缓存？

合乎逻辑的理由是，在此范围内的“小”整数使用率比大整数要高，因此，使用相同的底层对象是有价值的，可以减少潜在的内存占用。然而，通过反射API你会误用此功能。

运行下面的代码，享受它的魅力吧

```
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
    Class cache = Integer.class.getDeclaredClasses()[0];
    Field myCache = cache.getDeclaredField("cache");
    myCache.setAccessible(true);
    Integer[] newCache = (Integer[]) myCache.get(cache);
    newCache[132] = newCache[133];
    int a = 2;
    int b = a + a;
    System.out.printf("%d + %d = %d", a, a, b);
} 
```