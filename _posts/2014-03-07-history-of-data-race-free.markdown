---
layout: default
title: Data Race Free 的前世今生
categories: [科研]
comments: true
---


# Data Race Free 的前世今生

Data Race Free 是多线程程序是非常重要的概念，因为Java 和 C++的内存模型都是基于 Data Race Free 的，这篇文章将介绍这个概念的由来，另一篇文章《Data Race Free的理解》介绍它的主要思想。

事情要追溯到遥远的1979年， Lamport 在他的著名论文  *How to make a multiprocessor computer that correctly executes multiprocess programs* 中提出了今后在内存模型领域被广泛使用的概念 ：*sequential consistency*，即顺序一致性。这篇文章告诉我们，你要做一台多处理器的计算机，需要满足什么条件，才能保证程序的正确性。当然，这里的程序跑在不同处理器上，共享同一块内存。虽然现在不说多处理器了，都说多核，多线程，但是问题的本质是没有变的。就是多个执行单元一起完成一个任务，并且通过共享存储单元的方式通信，在这种情况下，底层的系统需要提供什么样的支持，才能保证计算的结果和程序员的预期是一样的。


7年后的1986年，Dubois,  Scheurich , Briggs 三人在论文 *Memory access buffering in multiprocessors* 中对 Lamport 的工作进行了扩展，他们提出了一种框架，用于分析共享内存的多处理器系统中的一致性问题。文中引入了三个 states，以此为基础提出了strong ordering的概念，并说明了他与Lamport提出的sequential consistency是一致的。但是，strong ordering对内存操作的限制太强了，对系统性能是一个阻碍，所以，他们又提出了 weak ordering的概念，以此提高系统性能。 满足weak ordering的系统并不是 sequential consistency的，程序员们需要自己去声明同步变量，以保证程序的正确性。




又过了4年，到了1990年，Adve大妈出手了，大妈现在是内存模型这个领域的权威，Java和C++内存模型的确立都有大妈的功劳，而Java和C++内存模型中相当重要的Data Race Free 概念就是Adve大妈在这一年提出的。

在这篇名为 *Weak ordering—a new definition*的文章中，Adve对Dubois等人提出的Weak Ordering进行了新的定义，并做出了一些修改以便进一步提高系统的性能。
新的定义出于这样一种想法，程序员习惯使用 sequential consistency来推断程序的运行结果，而底层的系统要想取得更高的性能，又不能使用sequential consistency内存模型来运行程序。那么

> 如何使得程序员可以使用sequential consistency推断程序结果，底层的实现又可以进行种种优化呢？

解决方案是：对程序本身进行足够的同步。

这种内存模型保证：如果你的程序进行了足够的同步，那么在我的weak oerding内存模型上运行，我可以保证结果和你在sequential consistency模型下运行的结果一样。

这样一来，程序员保证程序正确同步，就可以使用sequential consistency推断程序结果，而底层又可以灵活地进行各种优化，提高系统性能。

这里有一个关键问题：

> 什么叫“足够的同步”

Adve提出了Data Race Free的概念，也就是说，你的程序要是满足Data Race Free的条件了，你的同步就足够了，“足够”的意思就是说，这程序在weak ordering上跑和在sequential consistency上跑，结果是一样一样的～

Adve对weak ordering给出的新定义是：

> Hardware is weakly ordered with respect to a synchronization model if
and only if it appears sequentially consistent to all software that obey the synchronization model.

这里的synchronization model的**一种实现方式**，就是 Data Race Free。

Data Race Free 后来成为了 Java 和 C++ 内存模型的基础。


Java 的内存模型最早出现在1995年，但是自1997年起，这一内存模型被发现了许多严重的错误和缺陷，它阻碍了很多优化措施，对程序的安全性也没有足够的保证。2001年[JSR 133](https://www.jcp.org/en/jsr/detail?id=133)被确立下来，由William Pugh领导，专家组的成员包括了Adve，Doug Lea， William Pugh等。2004年，JSR 133最终版本发布。2005年，Manson  Jeremy, William Pugh, 和 Sarita V. Adve 一同发表了论文 *The Java memory model*，描述了最新的Java内存模型，这一内存模型在Java 5.0中引入，一直沿用至今。

Java 内存模型的关键是：如果多线程程序满足Data Race Free，那么内存模型保证程序执行结果和sequentially consistent模型下一样。另外，Java 内存模型的复杂之处还在于，为了保证程序的安全性，即使多线程程序不满足Data Race Free，我们也要对它进行一定程度的限制，这种限制必须恰到好处，太强会阻碍合理的优化，太弱保证不了程序的安全性。


三年后的2008年，Hans-J. Boehm 和 Sarita V. Adve 一同发表了文章 *Foundations of the C++ concurrency memory model*，描述了C++内存模型的基础，这一内存模型为C++ 11标准中的线程提供了明确的语义。

C++内存模型与的关键在于：如果多线程程序满足Data Race Free，那么内存模型保证程序执行结果和sequentially consistent模型下一样。与Java内存模型不同，对于那些不满足 Data Race Free的多线程程序，C++内存模型不对其结果提供任何保证。另外，C++内存模型提供了一些特性用以实现不同的内存模型。
