---
title: Java集合源码系列-ArrayList
date: 2019-01-22 23:02:45
tags:
    - Java
    - Java源码
---

## 前言

这是Java集合源码系列的第二篇，第一篇分析了HashMap，本来想把TreeMap和HashMap这几个Map实现分析一下，但是一个是红黑树的一个是不常用的类，想想就不浪费这个时间了。后面有时间有能力可以尝试分析下更有意义的`ConcurrentHashMap`。不过这篇文章将要分析的是经常用到的`ArrayList`。

## 概览

`ArrayList`是一个列表，底层是一个大小可以改变的数组类型。而数组类型在内存中的地址是连续的，可以通过数组下标直接计算出地址，所以可以以O(1)的时间复杂度进行**get**操作，但是**add**操作，需要对插入下标之后的数据进行向后移动，所以平均时间复杂度将会是O(n)。而列表的另外一种实现`LinkedList`则是采用双向链表，靠指针将前后数据连接起来，但数据在内存中并不是连续存放的，这种方式对插入和删除的操作友好，但是**get**的性能稍差。

下面还是按照之前的思路，从使用`ArrayList`的整个生命周期来作为行文路线，即创建-->添加-->获取-->删除。

<!--more-->

### 创建

使用`ArrayList`的第一步就是创建一个该类的对象，那么先来看下他的构造函数吧。一共有三个构造函数。

```java
// 无参构造函数是最常用的构造函数
// elementData即为底层存储数据的数组，无参构造函数将一个空数组赋值它，它的大小将在第一个元素被添加时指定，后面会提到
transient Object[] elementData;
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
// 指定数组大小的构造函数，这里的EMPTY_ELEMENTDATA与上面的DEFAULTCAPACITY_EMPTY_ELEMENTDATA虽然值一样，但是名字不一样，先记住这一点，后面在添加元素的时候会介绍到
private static final Object[] EMPTY_ELEMENTDATA = {};
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
// 通过已经存在的collection创建ArrayList
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

### 添加元素

构造函数还是比较简单的，但是也留了一个问题，`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`和`EMPTY_ELEMENTDATA`值明明是一样的，为什么需要两个呢？？将在这里解开谜底。

```java
// 最常用的是不带index的add方法，将元素追加到末尾,size是实例变量，初始化时为0
private int size; // size变量记录元素的数量
public boolean add(E e) {
    ensureCapacityInternal(size + 1);
// 确保容量后，元素添加到size++的位置，即末尾添加
    elementData[size++] = e;

    return true;
}
// add()调用ensureCapacityInternal()
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
// 这里就解答了上面的提到的问题，如果elementData指向的是ensureExplicitCapacity，即ArrayList是通过无参构造函数生成的，那么在第一次添加元素的时候，数组的默认大小设为10。因为第一次添加元素时minCapacity=size+1=1 < 10.
private static final int DEFAULT_CAPACITY = 10;
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
// modCount变量的作用是记录修改的次数，包括添加，删除等操作，修改值不记录，用于使用迭代器时的fail-fast
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
// 数组容量不够时进行扩容
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
// 扩容1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
// 第一次扩容时，elementData.length为0，minCapacity为10，所以数组大小最终会被设置为minCapacity
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
// MAX_ARRAY_SIZE需要减8的原因在注释中提到了，有些虚拟机的实现需要将一些头部字段存到数组中，因此留出一些空间，避免出现OutOfMemoryError错误
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
// 如果扩容后的数组大小大于MAX_ARRAY_SIZE,进行hugeCapacity调整
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0)
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

上面的那部分是以追加的方式在末尾添加元素，现在看下在指定下标处添加元素的代码

```java
public void add(int index, E element) {
    rangeCheckForAdd(index); 
    ensureCapacityInternal(size + 1);
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
// 调用系统的数组复制方法将原数组中index及之后的所有元素，向后移动一格，所以ArrayList在插入元素时的性能逊于LinkedList
    elementData[index] = element;
    size++;
}
// add方法首先校验index的合法性，对于不合法的index抛出异常，给出index和size信息
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
private String outOfBoundsMsg(int index) {
    return "Index: "+index+", Size: "+size;
}
// add方法然后调用ensureCapacityInternal进行保证内部容量的合适的验证，该方法前面介绍到，不再次介绍
```

除了上面介绍的两个add方法，还有两个个addAll方法，其实实现方式是一样的，就简单的贴下代码，大家稍微看下就能明白跟前面的方法几乎是一样的。

```java
 public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);
    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```

### 获取元素

前面的几个代码片段已经把添加元素都分析完了，还是不难的。现在看一下如何取出刚才添加的元素呢？

```java
// 因为是底层是数组，所以取出元素的代码更加简单，直接通过数组下标获取，然后进行类型转换后返回
public E get(int index) {
    rangeCheck(index);
    return elementData(index);
}
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
// 将元素类型转换后返回
E elementData(int index) {
    return (E) elementData[index];
}
```

前面介绍的是根据指定下标找元素，其实还可以通过指定元素找元素对应的下标

```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
// 从前向后遍历，找到第一个equal的元素后返回其下标，可以查找null对象，找不到时返回-1
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
// 从后向前遍历，逻辑一样
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

#### for-each遍历

```java
@Override
public void forEach(Consumer<? super E> action) {
    Objects.requireNonNull(action);
// action 为 null时抛出NullPointerException
    final int expectedModCount = modCount;
// 记录遍历时modCount的值，遍历时不允许对数组进行添加，删除等修改结构的操作
    @SuppressWarnings("unchecked")
    final E[] elementData = (E[]) this.elementData;
    final int size = this.size;
// 遍历数组，对每个元素进行action方法相关的操作，并判断modCount没有修改过
    for (int i=0; modCount == expectedModCount && i < size; i++) {
        action.accept(elementData[i]);
    }
// 遍历过程中发现modCount被修改过，抛出异常
    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}
```

### 删除元素

介绍完了获取元素，现在来看下如何删除元素

```java
// 根据下标删除元素
public E remove(int index) {
    rangeCheck(index); // 确保index<size
    modCount++; // 进行修改操作时modCount++
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
// 将index+1到最后一个元素全都向前移动一格
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}
// 根据对象删除
public boolean remove(Object o) {
// 如果找到该元素返回true，反之返回false，删除逻辑都是调用了fastRomeve
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
// fastRemove的取名还挺形象，跟remove(index)相比，省去了校验index合法性的步骤，直接进行移位
private void fastRemove(int index) {
    modCount++; // 修改次数+1
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
// 清除所有元素
public void clear() {
    modCount++;
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    size = 0;
}
// 把修改值的方法也放入删除的分类里吧，可以看出修改值的操作不会对modCount进行修改
public E set(int index, E element) {
    rangeCheck(index); // 验证index合法性
    E oldValue = elementData(index);
    elementData[index] = element; // 直接替换旧值
    return oldValue;
}
// 移除collection c中所有的元素
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        for (; r < size; r++)
// 遍历elementData，如果元素不存在c中，赋值给elementData[w]，w++,这里巧妙的是w<=r，所以不会丢失值
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
// 按理说r是等于size的，源码注释中解释这个finally语句的目的是兼容AbstractCollection，因为c.contains可能会抛出异常
        if (r != size) {
            System.arraycopy(elementData, r,
                             elementData, w,
                             size - r);
            w += size - r;
        }
        if (w != size) {
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w;
            size = w;
            modified = true;
        }
    }
    return modified;
}
```

### 总结

`ArrayList`源码中除了迭代器的部分，已经差不多都分析完了，相比于`HashMap`来说简单了很多，因为其底层的数据结构是数组，没有任何其他东西，所以实现较为简单。那么需要注意的就是什么场景使用`ArrayList`，什么场景使用`LinkedList`.`ArrayList`适合按序插入及频繁的通过index的查找的场景，而`LinkedList`适合随机的插入与删除元素的场景。

### 往期回顾

* [Java集合源码系列-HashMap](https://cfk1996.github.io/2019/01/12/Java%E9%9B%86%E5%90%88%E6%BA%90%E7%A0%81%E7%B3%BB%E5%88%97-HashMap/)

### 欢迎关注我的公众号

欢迎关注我的公众号，会经常分享一些技术文章和生活随笔～

![技术旅途](https://raw.githubusercontent.com/cfk1996/cfk1996.github.io/source/photos/wechat.jpg)
