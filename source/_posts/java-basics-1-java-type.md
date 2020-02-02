---
title: Java基础系列1：深入理解Java数据类型
categories:
    - '技术'
    - 'Java基础'
tags:
    - Java基础类型
    - Java封装类型
---

当初学习计算机的时候，教科书中对程序的定义是：程序=数据结构+算法，Java基础系列第一篇就聊聊Java中的数据类型。

本篇聊Java数据类型主要包括四个内容：

- Java基本类型
- Java封装类型
- 自动装箱和拆箱
- 封装类型缓存机制

<!--more-->

### Java基本类型

#### Java基本类型分类、大小及表示范围

Java的基本数据类型总共有8种，包括三类：数值型，字符型，布尔型，其中

-  数值型：
   - 整数类型：byte、short、int、long
   - 浮点类型：float、double
-  字符型：char
-  布尔型：boolean


字符类型在内存中占有2个字节，可以用来保存英文字母等字符。计算机处理字符类型时，是把这些字符当成不同的整数来看待，即ASKII码，因此，严格来说，字符类型也算是整数类型的一种。

Java的这8种基本类型的大小，即所占用的存储字节数，以及可以表示的数据范围如下表所示：

![img](/pictures/java-basics/java-basic-type.png)

#### Java基本类型之间的转换

Java是强类型的编程语言，其数据类型在定义时就已经确定了，因此不能随意转换成其他的数据类型，但是Java允许将一种类型赋值给另一种类型。

在Java中，**boolean类型与其他7种类型的数据都不能进行转换**，这一点很明确。

但对于其他7种数据类型，它们之间都可以进行转换，只是可能会存在精度损失或其他一些变化。转换分为自动转换和强制转换：

- 自动类型转换（隐式）：无需任何操作
- 强制类型转换（显式）：需使用转换操作符

自动类型转换需要满足如下两个条件：

1. 转换前的数据类型与转换后的数据类型兼容；
2. 转换后的数据类型的表示范围比转换前的类型大。

如果将6种数值类型作如下排序：

```
double > float > long > int > short > byte
```

那么从小转换到大，那么可以直接转换，而从大到小，或char或其他6种数据类型转换，则必须使用强制转换，且可能会发生精度损失。

#### Java基本数据类型的默认值

在某些场景下，比如在Restful API接口中，如果在dto中使用了基本类型的参数，那么即使请求体中没有传该参数，服务器在做反序列化的时候也会将该参数以默认值来处理。所以在实际开发的dto中务必不要使用基本类型。

以下是Java基本数据类型的默认值：

![img](pictures/java-basics/java-basic-type-default-value.png)

### Java封装类型

对于上面的8种基本类型，Java都有对应的封装类型：

| 基本类型 | 封装类型  |
| -------- | --------- |
| byte     | Byte      |
| int      | Integer   |
| short    | Short     |
| float    | Float     |
| double   | Double    |
| long     | Long      |
| boolean  | Boolean   |
| char     | Character |

#### 基本类型 vs 封装类型

Java封装类型与基本类型相比，有如下区别：

1. 从参数传递上来说，基本类型只能按值传递，而每个封装类都是按引用传递的；
2. 从存储的位置上来说，基本类型是存储在栈中的，而所有的对象都是在堆上创建和存储的，所以基本类型的存取速度要快于在堆中的封装类型的实例对象；JDK5.0开始可以自动封包了 ，也就是基本数据可以自动封装成封装类,基本数据类型的好处就是速度快（不涉及到对象的构造和回收），封装类的目的主要是更好的处理数据之间的转换，方法很多，用起来也方便。
3. 基本类型的优势是：数据存储相对简单，运算效率比较高；
4. 封装类型的优势是：类型转换的api更好用了，比如Integer.parseInt(*)等的，每个封装类型都提供了parseXXX方法和toString方法。而且在集合当中，也只能使用封装类型。封装类型满足了Java中一切皆对象的原则。

### 自动装箱和拆箱

#### 什么是自动装箱和拆箱

```java
// 自动装箱
Integer numInteger = 66;

// 自动拆箱
int numInt = numInteger;
```

简单地说，装箱就是自动将基本数据类型转换为封装类型；拆箱就是自动将封装类型转换为基本类型。

#### 自动装箱和拆箱的执行过程

我们就以上面的Integer的简单例子来研究执行过程，具体代码如下：

```java
public class Main {
    public static void main(String[] args) {
    		// 自动装箱
				Integer numInteger = 66;

    		// 自动拆箱
				int numInt = numInteger;
    }
}
```

先编译，执行：`javac Main.java `，

再反编译，执行：`javap -c Main`，

执行后得到如下内容：

![img](pictures/java-basics/compile-box.png)

可以看到，

在执行`Integer numInteger = 66;`的时候，系统为我们执行了`Integer numInteger = Integer.valueOf(66)`；

在执行`int numInt = numInteger;`的时候，系统为我们执行了`int numInt = numInteger.intValue();`

我们再来看一下Integer中valueOf方法的源码：

```java
public static Integer valueOf(int i) {
		if (i >= IntegerCache.low && i <= IntegerCache.high)
    		return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

其中IntegerCache.low=-128，IntegerCache.high=127。

也即，在执行Integer.valueOf(num)方法时，会先判断num的大小，如果小于-128或者大于127，就创建一个Integer对象，否则就从IntegerCache中来获取。这里涉及到了Integer的缓存机制，下一小节详细讨论。

```java
private final int value;

public Integer(int value) {
    this.value = value;
}
public Integer(String s) throws NumberFormatException {
		this.value = parseInt(s, 10);
}
```

这是Integer的构造函数，里面定义了一个value变量，创建一个Integer对象，就会给这个变量初始化。

再来简单看看IntegerCache是什么东西，IntegerCache类时Integer类的一个内部类，其包含了三个属性，如下：

```java
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
        		……
        }

        private IntegerCache() {}
    }
```

在valueOf方法中用到的cache数组，是一个静态的Integer数组对象，而这个数组对象在Integer第一次使用的时候就会创建好。

总之，valueOf返回的都是一个Integer对象。所以我们这里可以总结一点：装箱的过程会创建对应的对象，这个会消耗内存，所以装箱的过程会增加内存的消耗，影响性能。

### 封装类型缓存机制

#### Integer缓存机制源码分析

我们仍旧以Integer的例子来说明封装类型的缓存机制，看一下完整的IntegerCache类的代码：

```java
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

代码很简单，JVM在初始化的时候可以配置值`java.lang.Integer.IntegerCache.high`，默认为127，然后在第一次使用Integer的时候，不是只创建需要的那一个Integer对象，而是创建值在-128到`java.lang.Integer.IntegerCache.high`范围内的所有的Integer对象，然后将其放入到cache数组中。

然后在每次自动装箱的时候，如果值落在该范围内，则自动从cache数组中去拿出已经实例化的对象来用，而不用再次去实例化这样一个Integer对象。

每一个整数类型和字符类型、bool类型的封装类型都有类似的缓存机制，这也是为了减轻封装类型相比于基本类型的性能消耗。

#### Integer缓存机制实例

我们再举一个例子来说明缓存机制。

```java
public class Main {
    public static void main(String[] args) {

        Integer i1 = 100;
        Integer i2 = 100;
        Integer i3 = 200;
        Integer i4 = 200;

        System.out.println(i1 == i2);  //true
        System.out.println(i3 == i4);  //false
    }
}
```

后面的执行结果大家可能会很吃惊，原因是什么呢，结合Integer缓存机制的说明，可以明白这个过程如下：

1. i1和i2会进行自动装箱，执行了valueOf方法，它们的值落在[-128,128)，所以它们取到的IntegerCache.cache中的是同一个对象，所以它们是相等的；
2. i3和i4也会进行自动装箱，执行valueOf方法时，它们的值都大于128，所以会执行new Integer(200)，也即它们分别创建了两个不同的对象，所以它们肯定不相等。

#### 浮点类型无缓存机制

上面介绍的缓存机制仅针对整数类型、字符类型、布尔类型，因为这几种数据类型在一定区间的值的数量是固定，但是浮点类型如Float和Double却在任意区间都有无数个值。

来看看Double.valueOf的源码就知道了：

```java
public static Double valueOf(String s) throws NumberFormatException {
    return new Double(parseDouble(s));
}
```

可以看到Double.valueOf是直接返回一个新的Double对象，并没有缓存机制。

#### 使用缓存机制的封装类型

进行一个归类，使用了缓存机制的封装类型有这样几种：

| 类型      | 默认缓存对象范围 |
| --------- | ---------------- |
| Integer   | [-128,127]       |
| Short     | [-128,127]       |
| Long      | [-128,127)       |
| Character | [0,127]          |

#### 总结

1. 当一个基本数据类型与封装类型进行==、+、-、*、/运算时，会将封装类进行拆箱，对基本数据类型进行运算；
2. 拆箱完成运算之后，如果返回的结果需要是封装类型，则需要进行自动装箱，返回封装对象；
3. equals(Object o) 因为原equals方法中的参数类型是封装类型，所传入的参数类型（a）是原始数据类型，所以会自动对其装箱，反之，会对其进行拆箱；
4. 当两种不同类型用==比较时，包装器类的需要拆箱， 当同种类型用==比较时，会自动拆箱或者装箱。

### 关注我的公众号，获取更多关于面试、技术的文章及福利资源。

![Dali王的技术博客公众号](/pictures/Dali王的技术博客公众号.jpg)
