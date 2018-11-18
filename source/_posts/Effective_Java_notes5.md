---
title: 《Effective Java》学习笔记五——枚举和注解
categories:
    - '技术'
    - 'Java'
tags:
    - Java
    - 读书笔记
---

《Effective Java》学习笔记
<!--more-->




# 用enum代替int常量

枚举类型是指由一组固定的常量组成合法值的类型，例如一年中的季节、太阳系中的行星或者一副牌中的花色。

int枚举模式、String枚举模式都是不可取的。

Java的枚举本质上是int值。

Java枚举类型背后的基本想法非常简单：它们就是通过公有的静态final域为每个枚举常量导出实例的类。因为没有可以访问的构造器，枚举类型是真正的final。因为客户端既不能创建枚举类型的实例，也不能对它进行扩展，因此很可能没有实例，而只有声明过的枚举常量。换句话说，枚举类型是实例受控的。它们是单例的泛型化，本质上是单元素的枚举。枚举类型为类型安全的枚举模式提供了语言方面的支持。

为了将数据与枚举常量关联起来，得声明实例域，并编写一个带有数据并将数据保存在域中的构造器。枚举天生就是不可变的，因此所有的域都应该为final的。

与枚举常量关联的有些行为，可能只需要用在定义了枚举的类或者包中。这种行为最好被实现成私有的或者包级私有的方法。

如果一个枚举具有普遍适用性，它就应该成为一个顶层类；如果它只是被用在一个特定的顶层类中，它就应该成为该顶层类的一个成员类。

有时需要将本质上不同的行为与每个常量关联起来。有一种方法是通过启用枚举的值来实现；一种更好的方法可以将不同的行为与每个枚举常量关联起来：在枚举类型中声明一个抽象的apply方法，并在特定于常量的类主体中，用具体的方法覆盖每个常量的抽象apply方法。这种方法被称作特定于常量的方法实现。

特定于常量的方法实现可以与特定于常量的数据结合起来。

枚举类型有一个自动产生的valueOf(String)方法，它将常量的名字转变成常量本身。



# 用实例域代替序数

许多枚举天生就与一个单独的int值相关联。所有的枚举都有一个ordinal方法，它返回每个枚举常量在类型中的数字位置。

永远不要根据枚举的序数导出与它关联的值，而是要将它保存在一个实例域中。



# 用EnumSet代替位域

如果一个枚举类型的元素主要用在集合中，一般就使用int枚举模式，将2的不同倍数赋予每个常量。这种表示法让你用OR位运算将几个常量合并到一个集合中，称作位域。

位域是指信息在存储时，并不需要占用一个完整的字节，而只需占几个或一个二进制位。
例如在存放一个开关量时，只有0和1 两种状态， 用一位二进位即可。

java.util包提供了EnumSet类来有效地表示从单个枚举类型中提取的多个值的多个集合。这个类实现Set接口,提供了丰富的功能，类型安全性，以及可以从任何其他Set实现中得到的互用性。但是在内部具体的实现上，每个EnumSet内容都表示为位矢量。如果底层的枚举类型有64个或者更少的元素——大多数如此。整个EnumSet就用单个long来表示，因此它的性能比的上位域的性能。



# 用EnumMap代替序数索引

有一种非常快速的Map实现专门用于枚举键，称作java.util.EnumMap。EnumMap在运行速度方面之所以能与通过序数索引的数组相媲美，是因为EnumMap在内部使用了这种数组。

最好不要用序数来索引数组，而要使用EnumMap。如果你所表示的这种关系是多维的，就使用EnumMap<..., EnumMap<...>>。



# 用接口模拟可伸缩的枚举

虽然无法编写可扩展的枚举类型，却可以通过编写接口以及实现该接口的基础枚举类型，对它进行模拟。这样允许客户端编写自己的枚举来实现接口。如果API是根据接口编写的，那么在可以使用基础枚举类型的任何地方，也都可以使用这些枚举。



# 注解优先于命名模式

Java1.5 发行版本之前，一般使用命名模式表明有些程序元素需要通过某种工具或者框架进行特殊处理。例如，JUnit测试框架原本要求它的用户一定要用test作为测试方法名称的开头。这种方法有几个缺点：

1. 文字拼写错误会导致失败，且没有任何提示；
2. 无法确保它们只用于相应的程序元素上；
3. 它们没有提供将参数值与程序元素关联起来的好方法。

所有的程序员都应该使用Java平台所提供的预定义的注解类型。还要考虑使用IDE或者静态分析工具所提供的任何注解。这种注解可以提升由这些工具所提供的诊断信息的质量。但是要注意这些注解还没有标准化。



# 坚持使用Override注解

这个注解只能用在方法声明中，它表示被注解的方法声明覆盖了超类型中的一个声明。如果坚持使用这个注解，可以防止一大类的非法错误。

应该在你想要覆盖超类声明的每个方法声明中使用Override注解。

有一个例外，在具体的类中，不必标注你确信覆盖了抽象方法声明的方法（虽然这么做也没有什么坏处）。



# 用标记接口定义类型

标记接口是没有包含方法声明的接口，而只是指明或者标明一个类实现了具有某种属性的接口。例如，考虑Serializable接口，通过实现这个接口，类表明它的实例可以被写到ObjectOutputStream（或者“被序列化”）。

标记接口定义的类型是由被标记类的实例实现的；标记注解则没有定义这样的类型。这个类型允许你在编译时捕捉在使用标记注解的情况下要到运行时才能捕捉到的错误。

标记接口胜过标记注解的另一个优点是，它们可以被更加精确地进行锁定。

标记注解胜过标机接口的最大优点在与，它可以通过默认的方式添加一个或者多个注解类型元素，给已被使用的注解类型添加更多的信息。标记注解的另一个优点在于，它们是更大的注解机制的一部分。因此，标记注解在那些支持注解作为编程元素之一的框架中同样具有一致性。

如果想要定义一个任何新方法都不会与之关联的类型，标记接口就是最好的选择。如果想要标记程序元素而非类和接口，考虑到未来可能要给标记添加更多的信息，或者标记要适合于已经广泛使用了注解类型的框架，那么标记注解就是正确的选择。如果你发现自己在编写的目标为ElementType.TYPE的标记注解类型，就要花点时间考虑清楚，它是否真的应该为注解类型，想想标记接口是否会更加合适呢。


