---
title: 单例模式的八种写法
categories:
    - '技术'
    - '设计模式'
tags:
    - 设计模式
    - 单例模式
---

单例模式作为日常开发中最常用的设计模式之一，是最基础的设计模式，也是最需要熟练掌握的设计模式。
单例模式的定义是：保证一个类仅有一个实例，并提供一个访问它的全局访问点。
那么你知道单例模式有多少种实现方式吗？以及每种实现方式的利弊呢？
- 饿汉模式
- 懒汉模式（线程不安全）
- 懒汉模式（线程安全）
- 双重检查模式（DCL）
- 静态内部类单例模式
- 枚举类单例模式
- 使用容器实现单例模式
- CAS实现单例模式 

<!--more-->

### 饿汉模式
代码如下：
```java
public class Singleton {
     private static Singleton instance = new Singleton();
     private Singleton () {
     }
     public static Singleton getInstance() {
         return instance;
     }
 }
```
这种方式在类加载时就完成了实例化，会影响类的加载速度，但获取对象的速度快。
这种方式基于类加载机制保证实例仅有一个，避免了多线程的同步问题，是线程安全的。 
### 懒汉模式（线程不安全）
绝大多数时候，类加载的时机和对象使用的时机都是分开的，所以没有必要在类加载的时候就去实例化单例对象。为了消除单例对象实例化对类加载的影响，引入了延迟加载，就有了懒汉模式的实现方式。
代码如下：
```java
public class Singleton {
    private static Singleton instance;
    private Singleton () {
    }
    public static Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```
懒汉模式声明了一个静态对象，在用户第一次调用时完成实例化，属于延迟加载方式。
而且这种方式不是线程安全。 
### 懒汉模式（线程安全）
针对线程不安全的懒汉模式，对其中的获取单例对象的方法增加同步关键字。
代码如下：
```java
public class Singleton {
      private static Singleton instance;
      private Singleton () {
      }
      public static synchronized Singleton getInstance() {
          if (instance == null) {
              instance = new Singleton();
          }
          return instance;
      }
}
```
这种写法保证了线程安全，但是每次调用getInstance方法获取单例时都需要进行同步，造成不必要的同步开销，但实际上除了第一次实例化需要同步，其他时候都是不需要同步的。

### 双重检查模式（DCL）
既然懒汉模式中的实例化只需要在第一次的时候保证同步，那何不只在实例为空的时候加同步关键字呢。
代码如下：
```java
public class Singleton {
      private volatile static Singleton singleton;  // 1
      private Singleton () {
      }
      public static Singleton getInstance() {
          if (instance== null) {  // 2
              synchronized (Singleton.class) {  // 3
                  if (instance== null) {  // 4
                      instance= new Singleton();  // 5
                  }
             }
         }
         return singleton;
    }
}
```
双重检查写法，主要关注以上代码中的5点：
1. 声明单例对象时加上volatile关键字，保证多线程的内存可见性，也即当在一个线程中单例对象实例化完成之后，其他线程也同时能够看到。同时，还有更为重要的一点，下面会说。
2. 第一次检查单例对象是否为空，判断是否已经完成了实例化。
3. 如果第一次检查发现单例对象为空，那么该线程就要对此单例类进行加锁，准备进行实例化，加锁是为了保证该线程进行实例化的时候没有其他线程也同时进行实例化。
4. 第二次检查单例对象是否为空，则是为了避免这种情况：此时单例对象为空，两个线程，A线程在第2步，B线程在第5步，A线程发现单例对象为空，紧接着B线程就完成了实例化，然后就会导致A线程又会走一次第5步的实例化过程，即重复实例化。那么加上了第二次检查后，当A线程到第4步的时候就会发现单例对象已经实例化完成，自然不会到第5步。
5. 真正的实例化操作就发生在第5步，且只发生一次。
#### DCL思考
在以上代码的第一步中，我们提到volatile关键字，volatile关键字除了保证内存可见性，还有一点是禁止指令重排序。
那么问题出在哪里呢？对，第5步。
实际上，实例化对象的动作并不是一个原子操作，` instance= new Singleton(); `可以分为以下三步完成：
```
memory = allocate(); // 5.1:分配对象的内存空间
ctorInstance(memory); // 5.2:初始化对象
instance = memory; // 5.3: 设置instance指向刚分配的内存地址
```
而上面三行代码，5.2和5.3可能发生重排序。跟着上面代码中的第二次检查的位置进行分析。当线程B执行到5.3之后，5.2之前时，这时候线程A首次判断单例对象是否为空。这时候当然单例对象是不为空的，但是却不能使用，因为单例对象还没有被初始化呢。
这既是DCL的缺陷所在，也是为什么要对单例对象家volatile关键字的原因。禁止了指令重排序，自然不会出现线程A拿到一个不可用的单例对象。

### 静态内部类单例模式
```java
public class Singleton {
    private Singleton() {
    }
    public static Singleton getInstance() {
        return SingletonHolder.sInstance;
    }
    private static class SingletonHolder {
        private static final Singleton sInstance = new Singleton();
    }
}
```
第一次加载Singleton类时并不会初始化sInstance，只有第一次调用getInstance方法时虚拟机加载SingletonHolder 并初始化sInstance ，这样不仅能确保线程安全也能保证Singleton类的唯一性，所以推荐使用静态内部类单例模式。

### 枚举类单例模式
```java
public enum Singleton {
     INSTANCE;
     public void doSomeThing() {
     }
 }
```
那这个单例如何来填充属性呢，增加构造函数和属性即可啦，请看代码：
```java
public enum Singleton {
    INSTANCE("name", 18);
    private String name;
    private int age;
    Singleton(String name, int age) {
        this.name = name;
        this.age = age;
    }
     public void doSomeThing() {
     }
 }
```
默认枚举实例的创建是线程安全的，并且在任何情况下都是单例，上述讲的几种单例模式实现中，有一种情况下他们会重新创建对象，那就是反序列化，将一个单例实例对象写到磁盘再读回来，从而获得了一个实例。反序列化操作提供了readResolve方法，这个方法可以让开发人员控制对象的反序列化。在上述的几个方法示例中如果要杜绝单例对象被反序列化是重新生成对象，就必须加入如下方法：
```java
private Object readResolve() throws ObjectStreamException{
    return singleton;
}
```

### 使用容器实现单例模式
代码如下：
```java
public class SingletonManager {
　　private static Map<String, Object> objMap = new HashMap<String,Object>();
　　private Singleton() {
　　}
　　public static void registerService(String key, Objectinstance) {
　　　　if (!objMap.containsKey(key) ) {
　　　　　　objMap.put(key, instance) ;
　　　　}
　　}
　　public static ObjectgetService(String key) {
　　　　return objMap.get(key) ;
　　}
}
```
在程序的初始化，将多个单例类型注入到一个统一管理的类中，使用时通过key来获取对应类型的对象，这种方式使得我们可以管理多种类型的单例，并且在使用时可以通过统一的接口进行操作。这种方式是利用了Map的key唯一性来保证单例。

### CAS实现单例模式
以上实现主要用到了两点来保证单例，一是JVM的类加载机制，另一个就是加锁了。那么有没有不加锁的线程安全的单例实现吗？
有点，那就是使用CAS。
CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。
代码如下：
```java
public class Singleton {
    private static final AtomicReference<Singleton> INSTANCE = new AtomicReference<Singleton>();
    private Singleton() {}
    public static Singleton getInstance() {
        for (;;) {
            Singleton singleton = INSTANCE.get();
            if (null != singleton) {
                return singleton;
            }
            singleton = new Singleton();
            if (INSTANCE.compareAndSet(null, singleton)) {
                return singleton;
            }
        }
    }
}
```
用CAS的好处在于不需要使用传统的锁机制来保证线程安全，CAS是一种基于忙等待的算法，依赖底层硬件的实现，相对于锁它没有线程切换和阻塞的额外消耗，可以支持较大的并行度。
CAS的一个重要缺点在于如果忙等待一直执行不成功(一直在死循环中),会对CPU造成较大的执行开销。
