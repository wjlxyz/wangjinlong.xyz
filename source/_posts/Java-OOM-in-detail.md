---
title: 剖析Java OutOfMemoryError异常
categories:
    - '技术'
    - 'Java基础'
tags:
    - OutOfMemory异常
---


在JVM中，除了程序计数器外，虚拟机内存中的其他几个运行时区域都有发生OutOfMemoryError异常的可能，本篇就来深入剖析一下各个区域出现OOM异常的情形，以及如何解决各个区域的OOM问题。

本篇主要包括如下内容：
- Java堆溢出
- 运行时常量池和方法区溢出
- 本地内存溢出

<!--more-->

### Java堆溢出

Java堆用于存储对象实例，只要不断地创建对象，并且保证GC Roots到对象之间有可达路径来避免JVM清除这些对象，那么在对象数量到达最大堆的容量限制后就会产生溢出异常。

#### 堆溢出复现

要复现这种情况也很简单：将Java堆的大小限制为固定值，且不可扩展（将堆的最小值-Xms参数与最大值-Xmx参数设置为一样即可避免堆自动扩展）；当使用一个 while(true) 循环来不断创建对象就会发生 OutOfMemory，还可以使用 -XX:+HeapDumpOutofMemoryErorr 当发生 OOM 时会自动 dump 堆栈到文件中。

测试代码：

```java:n
    public static void main(String[] args) {
        List<String> list = new ArrayList<>() ;
        while (true){
            list.add("1") ;
        }
    }
```

运行结果：

```text:n

Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:265)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:239)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:231)
	at java.util.ArrayList.add(ArrayList.java:462)
	at Main.main(Main.java:13)

Process finished with exit code 1
```
`Exception in thread "main" java.lang.OutOfMemoryError: Java heap space`即是说发生了堆溢出。

#### 原因

1. 代码中可能存在大对象分配 ；
2. 可能存在内存泄露，导致在多次GC之后，还是无法找到一块足够大的内存容纳当前对象；
3. 如果不是以上两种情况，也就是说内存中的对象都必须存活，就应当检查虚拟机的堆参数（-Xmx与-Xms），是否设置的堆内存空间太小，以及检查代码中是否存在某些对象声明周期过长、持有状态时间过长的情况。

上面复现代码产生堆溢出的原因主要是第三点。

#### 解决方法

1. 检查是否存在大对象的分配，最有可能的是大数组分配；
2. 通过jmap命令，把堆内存dump下来，使用mat工具分析一下，检查是否存在内存泄露的问题
3. 如果没有找到明显的内存泄露，使用 -Xmx 加大堆内存；
4. 还有一点容易被忽略，检查是否有大量的自定义的 Finalizable 对象，也有可能是框架内部提供的，考虑其存在的必要性。

### 运行时常量池和方法区溢出

运行时常量池是方法区的一部分，我们先对运行时常量池溢出进行测试。

#### 运行时常量池溢出复现

最典型的使用运行时常量池的方法是String的intern()方法，该方法是一个Native方法，它的作用是：如果字符串常量池中已经包含一个等于此String对象的字符串，则返回代表池中这个字符串的String对象；否则，将此String包含的字符串添加到常量池中，并且返回此String对象的引用。

在JDK1.6及以前的版本中，由于常量池分配在永久代中，可以通过-XX:PermSize和-XX:MaxPermSIze限制方法区大小，从而限制其中常量池的容量

测试代码：

```java:n
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();
        int i = 0;
        while (true) {
            list.add(String.valueOf(i++).intern());
        }
    }
```

笔者所用为JDK1.8，已经去除了对这两个JVM参数的支持，程序执行的结果如下：

```text:n
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=10m; support was removed in 8.0

Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=10m; support was removed in 8.0

```
暂不做深究。

#### 方法区溢出复现

方法区用于存放class的相关信息，包括类名、访问修饰符、常量池、字段描述、方法描述等。可以通过借助CGLib直接操作字节码运行时生成大量的动态类，来填满方法区。

PermSize 和 MaxPermSize 已经不能使用了，那在JDK1.8中怎么设置方法区大小呢？

JDK 8 中将类信息移到了本地堆内存(Native Heap)中，将原有的永久代移动到了本地堆中成为 MetaSpace ,如果不指定该区域的大小，JVM 将会动态的调整。

可以使用 -XX:MaxMetaspaceSize=10M 来限制最大元空间。这样当不停的创建类时将会占满该区域并出现 OOM。

测试代码：

```java:n
    public static void main(String[] args) {
        while (true){
            Enhancer  enhancer = new Enhancer() ;
            enhancer.setSuperclass(Main.class);
            enhancer.setUseCache(false) ;
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                    return methodProxy.invoke(o,objects) ;
                }
            });
            enhancer.create() ;
        }
    }
```
设置好JVM参数后，执行上述代码，得到下面的额结果：

```text:n

Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
	at org.springframework.cglib.core.ReflectUtils.defineClass(ReflectUtils.java:530)
	at org.springframework.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:363)
	at org.springframework.cglib.proxy.Enhancer.generate(Enhancer.java:582)
	at org.springframework.cglib.core.AbstractClassGenerator$ClassLoaderData.get(AbstractClassGenerator.java:131)
	at org.springframework.cglib.core.AbstractClassGenerator.create(AbstractClassGenerator.java:319)
	at org.springframework.cglib.proxy.Enhancer.createHelper(Enhancer.java:569)
	at org.springframework.cglib.proxy.Enhancer.create(Enhancer.java:384)
	at com.etekcity.cloud.Main.main(Main.java:27)

Process finished with exit code 1
```

这里的 OOM 伴随的是 `Exception in thread "main" java.lang.OutOfMemoryError: Metaspace` 也就是元空间溢出。

方法区溢出在应用中是比较常见的OOM异常，Spring、Hibernate等框架在对类进行增强时，都会使用到CGLib技术来增强类，增强的类越多，对方法区的容量要求就越大，就越可能出现方法区的OOM异常。

#### 解决方法

因为该OOM原因比较简单，解决方法有如下几种：

1. 检查是否永久代空间或者元空间设置的过小；
2. 检查代码中是否存在大量的反射操作；
3. dump之后通过mat检查是否存在大量由于反射生成的代理类；
4. 重启JVM。

### 本机内存溢出

以上OOM异常都是出现于JVM内部，那么如果是机器本身分给JVM的内存不够导致溢出呢。

机器本身分给JVM的内存容量可以通过-XX:MaxDirectMemorySize指定，如果不指定，则默认与Java堆最大值（-Xmx指定一样）。

可以通过反射获取Unsafe实例进行内存分配，测试代码如下：

```java:n
    public static void main(String[] args) throws IllegalAccessException {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(1024 * 1024);
        }
    }
```

运行结果如下：

```text:n
Exception in thread "main" java.lang.OutOfMemoryError
    at sun.misc.Unsafe.allocateMemory(Native Method)
    at Main.main(Main.19)
```
有DirectMemory导致的内存溢出，在Heap Dump文件中不会看到明显的异常，如果发现OOM之后的dump文件很小，可以考虑一下是否是这方面的原因。


### 关注我的公众号，获取更多关于面试、技术的文章及福利资源。

![Dali王的技术博客公众号](/pictures/Dali王的技术博客公众号.jpg)
