---
title: 详解CopyOnWrite容器及其源码
categories:
    - '技术'
    - 'Java基础'
tags:
    - CopyOnWrite
---

在`jave.util.concurrent`包下有这样两个类：`CopyOnWriteArrayList`和`CopyOnWriteArraySet`。
其中利用到了CopyOnWrite机制，本篇就来聊聊CopyOnWrite技术与Java中的CopyOnWrite容器。
主要包扩以下内容：
- 什么是CopyOnWrite
- CopyOnWriteArrayList
- CopyOnWriteArraySet
- CopyOnWrite适用场景

<!--more-->

### 什么是CopyOnWrite
对于一般的容器，比如ArrayList，在进行并发操作时，如果一个线程读，一个线程写，会抛出java.util.ConcurrentModificationException异常。而CopyOnWrite容器则避免了这种情况。
CopyOnWrite，顾名思义，写时复制，在修改集合中数据的时候，不直接修改当前容器，而是先将当前容器进行拷贝，复制出一个新的容器，然后在新的容器里完成修改，再将原容器的引用指向新的容器。
这样做的好处是，可以不通过加锁，实现对CopyOnWrite容器的并发读写。需要注意的是，CopyOnWrite技术并不保证实时一致性，因为在读写并行时，有可能会读到过期的数据。CopyOnWrite技术保证的是最终一致性。

### CopyOnWriteArrayList
CopyOnWriteArrayList的底层是通过数组来实现的，其包含两个属性：
```java:n
final transient ReentrantLock lock = new ReentrantLock();
private transient volatile Object[] array;
```
前者用于在对CopyOnWriteArrayList进行修改时加锁，后者用于保存容器中的元素（允许null元素），对array加了volatile关键字，保证每次修改容器的时候对其他线程都是可见的。
在其各种接口的实现中，用的最多的是如下两个方法：
```java:n
final Object[] getArray() {
    return array;
}
final void setArray(Object[] a) {
    array = a;
}
```
`getArray()`方法返回当前数组，而`setArray()`方法用于在CopyOnWriteArrayList变化时，将array执行修改后的数组内存地址。
CopyOnWriteArrayList提供了三种构造函数：
```java:n
CopyOnWriteArrayList(); // 创建一个array长度为0的CopyOnWriteArrayList
CopyOnWriteArrayList(Collection<? extends E> c); // 以一个特定容器为参数创建CopyOnWriteArrayList
CopyOnWriteArrayList(E[] toCopyIn); // 以一个数组为参数创建CopyOnWriteArrayList
```
根据实际的需要创建即可。
CopyOnWriteArrayList提供的读方法与数组的读方法并无什么大的不同，因为CopyOnWriteArrayList本身解决的问题就不是读读并发的问题，所以重一下其写方法。
首先看一下`set()`方法，`set()`方法为指定位置设置特定值，如下是其实现：
```java:n
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 1
    try {
        Object[] elements = getArray(); // 2
        E oldValue = get(elements, index);
        if (oldValue != element) { // 3
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len); // 4
            newElements[index] = element; // 5
            setArray(newElements); // 6
        } else {
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock(); // 7
    }
}
```
分别看一下以上代码中的关键几步：
1. 可以看到CopyOnWriteArrayList为了保证在复制原容器时是加了一个可重入锁的，在set完成后释放该锁；
2. 获取当前的数组；
3. 判断要设置的位置的旧值与新值是否相同，如果相同则免去容器的拷贝工作；
4. 将原容器复制一份；
5. 修改该处的值为新值；
6. 重新创建CopyOnWriteArrayList容器，将旧容器的内存地址改为新容器所在内存地址；
7. 完成set，释放锁。
再看一下`add()`方法，向容器中添加一个元素，如下是其源码：
```java:n
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 1
    try {
        Object[] elements = getArray(); // 2
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1); // 3
        newElements[len] = e; // 4
        setArray(newElements); // 5
        return true;
    } finally {
        lock.unlock();
    }
}
```
add操作与set操作的大体过程都是相同的，多了两步的是，给新元素新增空间，即3~4步。
上面的`add()`方法是在容器最后添加一个元素，如果是在指定位置添加一个元素呢，源码如下：
```java:n
public void add(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 1
    try {
        Object[] elements = getArray(); // 2
        int len = elements.length; // 3
        if (index > len || index < 0) // 4
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + len);
        Object[] newElements;
        int numMoved = len - index; // 5
        if (numMoved == 0) // 6
            newElements = Arrays.copyOf(elements, len + 1); // 7
        else {
            newElements = new Object[len + 1]; // 8
            System.arraycopy(elements, 0, newElements, 0, index); // 9
            System.arraycopy(elements, index, newElements, index + 1, numMoved); // 10
        }
        newElements[index] = element; // 11
        setArray(newElements); // 12
    } finally {
        lock.unlock(); // 13
    }
}
```
add(int index, E element)相比于add(E e)又多了数组元素移位的过程，即3~10步。移位的时候用到了System.arraycopy()方法，以第9步为例，其意为将elements数组从0开始的index个元素拷贝到newElements数组的从0开始的位置上。System.arraycopy()是一个native方法，用于保证每次add操作对数组移位时的性能不至于太差。
那么从容器中移除一个元素呢，请看其`remove()`方法源码：
```java:n
public E remove(int index) {
    final ReentrantLock lock = this.lock;
    lock.lock();  // 1
    try {
        Object[] elements = getArray(); // 2
        int len = elements.length; // 3
        E oldValue = get(elements, index);
        int numMoved = len - index - 1; // 4
        if (numMoved == 0)
            setArray(Arrays.copyOf(elements, len - 1)); // 5
        else {
            Object[] newElements = new Object[len - 1]; // 6
            System.arraycopy(elements, 0, newElements, 0, index); //7
            System.arraycopy(elements, index + 1, newElements, index, numMoved); // 8
            setArray(newElements); // 9
        }
        return oldValue;
    } finally {
        lock.unlock(); // 10
    }
}
```
remove(index)方法的实现与add(index, element) 方法的实现是类似的，区别在于前者是数组缩小。
除此之外，还提供了remove(Object o)方法用于移除特定值，remove(Object o, Object[] snapshot, int index)方法用于移除特定版本数组下的特定值，且前者的实现是以后者为基础的，故而前者在自己的实现中没有加锁。这里仅拿出第三个remove方法的源码来作分析：
```java:n
private boolean remove(Object o, Object[] snapshot, int index) {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 1
    try {
        Object[] current = getArray(); // 2
        int len = current.length;
        if (snapshot != current) findIndex: { // 3
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                if (current[i] != snapshot[i] && eq(o, current[i])) { // 4
                    index = i;
                    break findIndex;
                }
            }
            if (index >= len) // 不存在要删除的元素
                return false;
            if (current[index] == o)
                break findIndex;
            index = indexOf(o, current, index, len); // 获取current数组中从index开始到len的值为o的第一个元素的位置
            if (index < 0) // 不存在要删除的元素
                return false;
        }
        Object[] newElements = new Object[len - 1]; // 5
        System.arraycopy(current, 0, newElements, 0, index); // 6
        System.arraycopy(current, index + 1, newElements, index, len - index - 1); // 7
        setArray(newElements); // 8
        return true;
    } finally {
        lock.unlock(); // 9
    }
}
```
这个remove方法给定要删除的值和一个数组，以及结束的位置。这个snapshot数组可以认为是某一个版本的array数组，当二者相同时，remove方法和remove(o)就几乎一样了；当二者不同时，即3、4步，则是记录当前数组中与要删除的值相同的那个元素的位置，此时说明snapshot数组已经修改过了，所以相同位置的那个元素已经不同了。
以上是CopyOnWriteArrayList的源码中重要属性和函数的实现剖析。

### CopyOnWriteArraySet
了解了CopyOnWriteArrayList的实现之后，CopyOnWriteArraySet的实现就比较简单了，看一眼CopyOnWriteArraySet保存元素的结构就知道为何了：
```java:n
private final CopyOnWriteArrayList<E> al;
public CopyOnWriteArraySet() {
    al = new CopyOnWriteArrayList<E>();
}
```
可以看到CopyOnWriteArraySet的实现是基于CopyOnWriteArrayList来做的，CopyOnWriteArraySet提供的各种方法也都是通过CopyOnWriteArrayList来实现，原理基本相同，就不单独详细说明了。

### CopyOnWrite适用场景
考虑到每次CopyOnWrite容器进行修改的时候都需要加锁和对容器进行拷贝，写的性能开销较大，所以更适合使用在读操作远远大于写操作的场景里，比如缓存、搜索引擎对某些关键词过滤使用的黑名单等。发生修改时候做copy，新老版本分离，保证读的高性能，适用于以读为主的情况。
因为CopyOnWrite容器只能保证最终一致性，所以不适用于对数据实时性要求较高的场景中，因为一个线程修改了数据，其他线程并不一定能够马上读取到新的数据。

### 关注我的公众号，获取更多关于面试、技术的文章及福利资源。

![Dali王的技术博客公众号](/pictures/Dali王的技术博客公众号.jpg)