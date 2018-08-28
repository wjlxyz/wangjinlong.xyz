---
title: 深度解析Java单例模式
---

　　单件模式，也称单例模式，用以创建独一无二的、只能有一个实例的对象。

　　单件模式的类图是所有模式的类图中最简单的——只有一个类。尽管从类设计的视角来看单件模式很简单，但是实现上还是会遇到一些问题，本文着重对这一点来进行分析解决。

<!--more-->

　　最简单的单件模式的实现，代码如下：

```java
 /**
  * Created by McBye King on 2016/10/23.
  */
 public class Singleton {
     private static Singleton singleton;
     private Singleton(){}
     public static Singleton getSingleton(){
         if(singleton == null){
             singleton = new Singleton();
         }
         return singleton;
     }
 }
```

　　结合以上的代码，对单件模式进行简单的阐述。

　　单件模式中，利用一个静态变量来记录`Singleton`类的唯一实例。把构造器声明为私有的，只有自`Singleton`类内才可以调用构造器。为了实例化这个类，于是调用`getSingleton`方法，在其中实例化并返回这个实例。这里还有一个“延迟实例化”的思想，在此处，如果我们不需要这个实例，它就永远不会产生。当然在实际中可以给这个类添加其他行为。

　　看起来这已经是单件模式的全部了，因为单件模式太简单了，但是如果细细追究，还有很多问题。

　　想一个问题，如果有两个或者更多的线程调用使用上述的单例的类，会怎么样呢？

　　因为没有对这个单例的类对外的接口`getSingleton()`方法进行保护，每一个线程都可以同时调用到这个函数，有可能若干个线程同时访问到这个方法，同时进行了`if(singleton == null)`的判断，因为是同时的，所以大家看到的都是未曾实例化的`singleton`，于是紧接着就有若干个`Singleton`实例对象出现——这完全违反了单件模式的本意。——如果你说这也太偶然了吧，确实是的，但是确实实际存在的问题，有很大几率出现重大`bug`的问题

　　接下来我们考虑怎么解决这个问题。

　　1、**只要把getSingleton()变成同步（synchronized）方法**，多线程灾难几乎就可以解决了，如下示例：

```
/**
 * Created by McBye King on 2016/10/23.
 */
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    public static synchronized Singleton getSingleton(){
        if(singleton == null){
            singleton = new Singleton();
        }
        return singleton;
    }
}
```

　　通过添加`synchronized`关键字到`getSingleton()`方法中，我们迫使每个线程在进入这个方法之前，要先等候别的线程离开该方法。也就是说不允许两个线程可以同时进入这个方法。

　　2、**添加synchronized方法无疑可以解决同步问题**，但是很明显会降低性能，这又导致了另一个问题。如果可以牺牲性能，也即`getSingleton()`方法的性能对应用程序影响不大的时候，就用上面的方法没有错。否则，就把“延迟实例化”变成“急切”创建实例把。

```
/**
 * Created by McBye King on 2016/10/23.
 */
public class Singleton {
    private static Singleton singleton = new Singleton();
    private Singleton(){}
    public static synchronized Singleton getSingleton(){
        return singleton;
    }
}
```

　　在静态初始化器中创建单件，就保证了线程安全。当然了，这种办法适用于应用程序总是创建并使用单件实例，或者在创建和运行时方面的负担不会太重。

　　3、相对更好一点的办法是：**用“双重检查加锁”，在getSingleton()中减少使用同步**。

　　来看看代码：

```
/**
 * Created by McBye King on 2016/10/23.
 */
public class Singleton {
    private volatile static Singleton singleton;
    private Singleton(){}
    public static Singleton getSingleton(){
        if(singleton == null){
            synchronized (Singleton.class){
                if(singleton == null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

　　在上述代码的`getSingleton()`实例化方法中，先检查实例，如果不存在，就进入同步区块；且只有第一次才会彻底执行同步区块中的代码。

　　其中的`volatile`关键字确保：当`singleton`变量被初始化成`Singleton`实例时，多个线程正确地处理`singleton`变量。

　　如果性能是考虑的重点的话，上述办法可以帮助大大减少`getSingleton()`的时间耗费。——前提是在`Java 5`以及之后的`Java`版本中。

　　4,、今天再更新一种方法，结合以上的三种方法的优点，既能拥有单件模式延迟实例化的优点，又能保证性能的要求，同时也避免了多线程情况下出错。

　　具体代码如下：

```
/**
 * Created by McBye King on 2016/10/23.
 */
public class Singleton {
    private static Singleton singleton;
    private Singleton(){}
    private static synchorized void init(){
        if(singleton == null){
            singleton = new Singleton();
        }          
    }  
    //延迟实例化
    public static Singleton getSingleton(){
        if(singleton == null){
            init();
        }
        return singleton;
    }
}
```

　　如上代码所示，只有第一次实例化`Singleton`的时候才会调用`init()`方法。

​        5、今天再更新一种方法，使用内部类的形式，只有在第一次需要单例实例的时候才会初始化该内部类，从而实现只加载一次该实例，同时也保证线程安全。

```java
public class Singleton {
// 使用内部类实现延迟加载
    private static class SingletonHolder {
        private static Singleton singleton = new Singleton();
    }

    public static Singleton getSingleton() {
        return SingletonHolder.singleton;
    }
}
```

　　这几天在蚂蚁金服的技术面中考察到了这种方法，很巧妙的实现了以上几种方法的优点，也避免了其缺点。

　　以上代码的github地址：[AntiTechInterview](https://github.com/wjlxyz/AntiTechInterview)。



 