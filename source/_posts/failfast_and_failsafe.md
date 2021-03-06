---
title: 快速失败（fail-fast）和安全失败（fail-safe）
categories:
    - '技术'
    - 'Java'
tags:
    - Java
---

快速失败（fail-fast）和安全失败（fail-safe）
<!--more-->

# 快速失败（fail-fast）

在用迭代器遍历一个集合对象时，如果遍历过程中对集合对象的内容进行了修改（增加、删除、修改），则会抛出ConcurrentModificationException。


- 原理：迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个modCount变量。集合在被遍历期间如果内容发生变化，就会改变modCount的值。每当迭代器使用hasNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedModCount值，是的话就返回遍历；否则抛出异常，终止遍历。

- 注意：这里异常的抛出条件是检测到modCount != expectedModCount这个条件。如果集合发生变化时修改modCount值刚好又设置为了expectedModCount值，则异常不会抛出。因此，不能依赖于这个异常是否抛出而进行并发操作的变成，这个异常只建议用于检测并发修改的bug。
- 场景：java.utl包下的集合类都是快速失败的，不能再多线程下发生并发修改（迭代过程中被修改）。



# 安全失败（fail-safe）

采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问，而是先复制原有集合内容，在拷贝的集合上进行遍历。

- 原理：由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发ConcurrentModificationException。
- 缺点：
  - 基于拷贝内容的优点是避免了ConcurrentModificationException，但同样地，迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的。
  - 需要复制集合，产生大量的无效对象，开销大。
- 场景：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。



快速失败和安全失败是对迭代器而言的。快速失败：挡在迭代一个集合的时候，如果有另外一个线程在修改这个集合，就会抛出ConcurrentModificationException，java.utl下都是快速失败。安全失败：在迭代时候会在集合二层做一个拷贝，所以在修改集合上层元素不会影响下层。在java.util.concurrent下都是安全的。