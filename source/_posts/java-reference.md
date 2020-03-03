---
title: 探究Java中的引用
categories:
    - '技术'
    - 'Java基础'
tags:
    - Java引用
---


从JDK1.2版本开始，Java把对象的引用分为四种级别，从而使程序能更加灵活的控制对象的生命周期。这四种级别由高到低依次为：强引用、软引用、弱引用和虚引用。本篇就来详细探究一下这四种引用的机制：

- 强引用

- 软引用

- 弱引用

- 虚引用

- 详解ReferenceQueue与Reference

<!--more-->

### 强引用

强引用是最普遍的引用，一般通过new关键字来创建出来的对象引用都属于强引用，比如Object o = new Object()。

如果一个对象具有强引用，它就不会被垃圾回收器回收。即使当前内存空间不足，JVM也不会回收它，而是抛出 OutOfMemoryError 错误，使程序异常终止。如果想中断强引用和某个对象之间的关联，可以显式地将引用赋值为null，这样一来的话，JVM在合适的时间就会回收该对象。



### 软引用

在使用软引用时，如果内存的空间足够，软引用就能继续被使用，而不会被垃圾回收器回收；只有在内存空间不足时，软引用才会被垃圾回收器回收。软引用最长被用作内存敏感型的内存缓存。

创建一个软引用的代码示例：

```java:n

SoftReference<String> soft = new SoftReference<>("World");

```

SoftReference类的结构如下：

```text:n

java.lang.ref.SoftReference#SoftReference(T)

java.lang.ref.SoftReference#SoftReference(T, java.lang.ref.ReferenceQueue<? super T>)

java.lang.ref.SoftReference#get

java.lang.ref.SoftReference#clock

java.lang.ref.SoftReference#timestamp

```

- 前两个是构造函数，后面会详细介绍ReferenceQueue；

- get方法用于获取这个软引用所指向的对象，如果这个对象已经清除或者被GC收集，那么就返回null；

- clock属性是一个static的long型时间戳，由GC线程进行GC的时候更新；

- timestamp属性也是一个时间戳，每次在调用get方法的时候会对其进行更新，JVM用这个属性用于帮助收集和清理软引用。



### 弱引用

如果一个对象只具有弱引用，当 JVM 进行垃圾回收时，只要GC线程检测到了，无论当前内存空间是否充足，都会将其回收。不过由于垃圾回收器是一个优先级较低的线程，所以并不一定能迅速发现弱引用对象。弱引用通常用于实现规范化映射。

创建一个弱引用的代码示例：

```java:n

WeakReference<String> weakName = new WeakReference<String>("hello");

```

WeakReference类的结构如下：

```text:n

java.lang.ref.WeakReference#WeakReference(T)

java.lang.ref.WeakReference#WeakReference(T, java.lang.ref.ReferenceQueue<? super T>)

```

只是提供了两个构造函数，后一个构造函数传入了一个ReferenceQueue对象。



### 虚引用

顾名思义，就是形同虚设的引用，如果一个对象仅持有虚引用，那么它相当于没有引用，在任何时候都可能被垃圾回收器回收。

创建虚引用的代码示例：

```java:n

ReferenceQueue<String> queue = new ReferenceQueue<String>();

PhantomReference<String> pr = new PhantomReference<String>(new String("hello"), queue);

```

PhantomReference类的结构如下：

```text:n

java.lang.ref.PhantomReference#PhantomReference(T, ReferenceQueue<? super T>)

java.lang.ref.PhantomReference#get

```

- 只有一个带有ReferenceQueue参数的构造函数，也就是虚引用一定要和引用队列一起使用；

- 同时其get方法也有点特殊，因为虚引用的引用对象相当于没有引用，所以其get方法总是返回null。



### 详解ReferenceQueue与Reference

引用队列可以与软引用、弱引用以及虚引用一起配合使用，当垃圾回收器准备回收一个对象时，如果发现它还有引用，那么就会在回收对象之前，把这个引用加入到与之关联的引用队列中去。程序可以通过判断引用队列中是否已经加入了引用，来判断被引用的对象是否将要被垃圾回收，这样就可以在对象被回收之前采取一些必要的措施。

与软引用、弱引用不同，虚引用必须和引用队列一起使用。

ReferenceQueue实现了一个队列的入队（enqueue）和出队（poll还有remove）操作，内部元素就是Reference。ReferenceQueue名义上是一个队列，但内部并没有实际的存储结构，它的存储是依赖于内部节点之间的关系来实现的。看一下ReferenceQueue实现中用到的属性：

```java:n

  static ReferenceQueue<Object> NULL = new Null<>();

  static ReferenceQueue<Object> ENQUEUED = new Null<>();



  static private class Lock { };

  private Lock lock = new Lock();

  private volatile Reference<? extends T> head = null;

  private long queueLength = 0;

```

其实就是一个增加了同步操作的链表的设计，通过head属性来找到链表头，每个链表节点，即Reference对象，都有一个next属性来找到下一个节点。

刚才分析四种引用的时候看到，java.lang.ref.Reference 为 软（soft）引用、弱（weak）引用、虚（phantom）引用的父类。那我们再来看一下Reference类的实现中用到的属性：

```java:n

  private T referent;     /* Treated specially by GC */

  volatile ReferenceQueue<? super T> queue;

  volatile Reference next;

  transient private Reference<T> discovered; /* used by VM */

  static private class Lock { }

  private static Lock lock = new Lock();

  private static Reference<Object> pending = null;

```

Reference作为ReferenceQueue中的节点，定义了next属性来指向下一个节点，referent为实际指向的对象，pending存储等待被放入ReferenceQueue的引用对象；discovered表示要处理的下一个对象。

Reference类还定义了一个ReferenceHandler线程。

```text:n

java.lang.ref.Reference.ReferenceHandler#ReferenceHandler

java.lang.ref.Reference.ReferenceHandler#ensureClassInitialized

java.lang.ref.Reference.ReferenceHandler#run

```

这个线程在Reference类的static的构造块中启动，并且被设置为最高优先级和daemon状态。此线程要做的事情就是不断的检查pending属性是否为null，如果pending不为null，则将pending进行enqueue，否则线程进入wait状态。

由此可见，pending是由jvm来赋值的，当Reference内部的referent对象的可达状态改变时，jvm会将Reference对象放入pending链表。并且这里enqueue的队列是我们在初始化（构造函数）Reference对象时传进来的queue，如果传入了null( 实际使用的是ReferenceQueue.NULL )，则ReferenceHandler则不进行enqueue操作，所以只有非RefernceQueue.NULL的queue才会将Reference进行enqueue。

ReferenceQueue作为 JVM GC与上层Reference对象管理之间的一个消息传递方式，它使得我们可以对所监听的对象引用可达发生变化时做一些处理。


### 关注我的公众号，获取更多关于面试、技术的文章及福利资源。

![Dali王的技术博客公众号](/pictures/Dali王的技术博客公众号.jpg)