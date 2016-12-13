---
layout:     post
title:      "关于线程同步的一些小记"
subtitle:   "Keep Learning"
date: 2016-12-12 11:00:01 +0800
author:     "Roger"
header-img: "img/home-bg-o.jpg"
tags:
    - Java
---
关于线程同步的一些小记
---

多线程同步作为基础还是很重要的，在面试中基本作为必备问题，然而在平时的 Android 开发中使用的频率却不是很高，因为一个 *synchronized* 关键字即可帮我们解决绝大部分情况，但是如果在面试中仅仅回答 *synchronized* 就略显单薄了，这里记录一下关于多线程同步的一些点。

1. *synchronized* 最强大最方便的线程同步方法

*synchronized* 具有 **原子性** 以及 **可见性** ：

    * 原子性

    原子性是指这个操作时不可分割的，例如 int a=0; 那么我们说这个操作具有原子性。非原子性的操作都会存在线程安全问题。当一系列操作具有原子性时，别的线程只能等待这一系列的操作完成之后，才能修改这一操作相关的变量。

    * 可见性

    可见性指的是线程之间的可见性，一个线程修改的状态对另一个线程是可见的。也就是说一个线程修改变量的结果，另一个线程马上就能看到。

*synchronized* 可以修饰变量或是方法，还可以定义同步代码块。由于java的每个对象都有一个内置锁，当用此关键字修饰方法时，内置锁会保护整个方法。在调用该方法前，需要获得内置锁，否则就处于阻塞状态。synchronized关键字也可以修饰静态方法，此时如果调用该静态方法，将会锁住整个类。

在同步控制方法或同步控制块中可以调用 *wait()* 和 *notifyAll()* 来实现控制多线程同步的问题。例如生产者消费者问题等。

2. 使用特殊域变量(*volatile*)实现线程同步

*volatile* 具有可见性而不具有原子性。所以不能使用该关键字实现程序计数器。

    * volatile关键字为域变量的访问提供了一种免锁机制，
    * 使用volatile修饰域相当于告诉虚拟机该域可能会被其他线程更新，
    * 因此每次使用该域就要重新计算，而不是使用寄存器中的值
    * volatile不会提供任何原子操作，它也不能用来修饰final类型的变量
    * 要始终牢记使用 volatile 的限制 —— 只有在状态真正独立于程序内其他内容时才能使用 volatile  

这里插一个知识点，在使用双重锁定的单例模式时，单例变量 *mInstance* 必须申明为 *volatile* 变量，这是因为 java 平台内存模型是无序写入的，一个类在 new 的过程中可能会返回一个没有充分初始化的引用，这个引用如果恰好被其他线程通过 getInstance 获取就会出现问题，所以将 *mInstance* 申明为 *volatile* ，使得获取的引用都是最后充分初始化后的类，才能保证线程安全问题。

3. 使用 *Lock* 实现同步

*Lock* 是一种比 *synchronized* 更强大的锁的方式。

*java.util.concurrent.lock* 中的 *Lock* 框架是锁定的一个抽象，它允许把锁定的实现作为 Java 类，而不是作为语言的特性来实现。这就为 *Lock* 的多种实现留下了空间，各种实现可能有不同的调度算法、性能特性或者锁定语义。

*ReentrantLock* 类实现了 *Lock* ，它拥有与 *synchronized* 相同的并发性和内存语义，但是添加了类似锁投票、定时锁等候和可中断锁等候的一些特性。此外，它还提供了在激烈争用情况下更佳的性能。（换句话说，当许多线程都想访问共享资源时，JVM 可以花更少的时候来调度线程，把更多时间用在执行线程上）

需要注意的是，用 *sychronized* 修饰的方法或者语句块在代码执行完之后锁自动释放，而是用 *Lock* 需要我们手动释放锁，所以为了保证锁最终被释放(发生异常情况)，要把互斥区放在 *try* 内，释放锁放在 *finally* 内！！

*ReadWriteLock* 读写锁，与互斥锁相比，读写锁允许对共享数据进行更高级别的并发访问。虽然一次只有一个线程（*writer* 线程）可以修改共享数据，但在许多情况下，任何数量的线程可以同时读取共享数据（*reader* 线程）。从理论上讲，与互斥锁定相比，使用读-写锁定所允许的并发性增强将带来更大的性能提高。

*Lock* 与 *Condition* 的配合使用，Condition 可以替代传统的线程间通信，用 await() 替换 wait() ，用 signal() 替换 notify() ，用 signalAll() 替换 notifyAll()。Condition 的强大之处在于它可以为多个线程间建立不同的 Condition 。具体代码可参考 [Link](http://blog.csdn.net/vking_wang/article/details/9952063)

4. 使用 *ThreadLocal*

如果使用 *ThreadLocal* 管理变量，则每一个使用该变量的线程都获得该变量的副本，副本之间相互独立，这样每一个线程都可以随意修改自己的变量副本，而不会对其他线程产生影响。例如 *Android* 中著名的 *Looper* 就是一个线程的局部变量。关于 *ThreadLocal* 的工作原理请参考 [Link](http://blog.csdn.net/imzoer/article/details/8262101)

5. 线程安全的一些集合

JDK 1.2 中引入的 *Collection* 框架是一种表示对象集合的高度灵活的框架，它使用基本接口 *List*、*Set* 和 *Map*。通过 JDK 提供每个集合的多次实现（*HashMap*、*Hashtable*、*TreeMap*、*WeakHashMap*、*HashSet*、*TreeSet*、*Vector*、*ArrayList*、*LinkedList* 等等）。其中一些集合已经是线程安全的（ *Hashtable* 和 *Vector* ），通过同步的封装工厂（ *Collections.synchronizedMap()*、*synchronizedList()* 和 *synchronizedSet()*），其余的集合均可表现为线程安全的。

*java.util.concurrent* 包添加了多个新的线程安全集合类（*ConcurrentHashMap*、*CopyOnWriteArrayList* 和 *CopyOnWriteArraySet*）。这些类的目的是提供高性能、高度可伸缩性、线程安全的基本集合类型版本。

java.util 中的线程集合仍有一些缺点。例如，在迭代锁定时，通常需要将该锁定保留在集合中，否则，会有抛出 *ConcurrentModificationException* 的危险。（这个特性有时称为条件线程安全；有关的更多说明，请参阅参考资料。）此外，如果从多个线程频繁地访问集合，则常常不能很好地执行这些类。java.util.concurrent 中的新集合类允许通过在语义中的少量更改来获得更高的并发。

JDK 5.0 还提供了两个新集合接口 --*Queue* 和 *BlockingQueue*。Queue 接口与 List 类似，但它只允许从后面插入，从前面删除。通过消除 List 的随机访问要求，可以创建比现有 ArrayList 和 LinkedList 实现性能更好的 Queue 实现。因为 List 的许多应用程序实际上不需要随机访问，所以Queue 通常可以替代 List，来获得更好的性能。

6. 生产者消费者模式的五种写法

[Link](http://huachao1001.github.io/article.html?QhSkxKKX)

搞定这些，面试中关于 java 线程同步的问题都可以扯上一二了。
