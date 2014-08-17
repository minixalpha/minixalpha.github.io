---
layout: default
title: 对 Java 内存模型的理解
category: 科研
comments: true
---


# 对 Java 内存模型的理解

## Java 内存模型

Java内存模型规定了在多线程程序中，什么样的行为是允许出现的，什么样的行为是禁止出现的。这样说可能有点抽象，我们换一个角度。将程序行为抽象成读操作和写操作，每个线程有自己的局部变量，同时线程之间还存在共享变量。那么一个多线程程序执行结束后，所有变量会有一个最终值。Java内存模型来决定什么样的值合法，什么样的值不合法。

内存模型不能要求的太严格，这样会阻碍很多优化方法，降低程序执行的效率，但也不能要求的太松，因为这样会导致一些执行结果违反我们的直觉。例如指令间的重排序问题，如果线程内部的指令完全按照程序中指明的次序执行，并且每次执行一条指令，执行的结果立即生效，那么就会阻碍很多优化方法，但这样对程序员是有好处的，因为程序员很容易推断程序的执行结果，这样写出的程序就容易与自己的意图一致。这种内存模型被称为顺序一致性模型(Sequential Consistency)。反之，如果为了优化程序执行效率，重排序的可能性有很多，那么程序的效率是提高了，但对程序员来说，就很难推断程序的执行结果。这一类的内存模型被称为Relaxed Memory Model。



这样，我们就遇到了一个两难的问题：

* 内存模型要求严格，那么程序效率低，但程序员容易写对
* 内存模型要求松，那么程序效率高，但程序员不容易写对

而程序的效率，与程序是否容易写对都很重要。为了解决这个问题，科学家提出了 Data Race Free 的概念，它是对多线程程序同步程度的一种描述，基本的思想是如果多线程程序进行了正确的同步，那么程序员就可以按照顺序一致性模型去推断程序的执行结果，而底层对内存操作的实现，可以按照 Relaxed Memory Model进行优化。

Java 内存模型包含了两方面的内容

* 对正确同步的多线程程序，保证其执行结果与在顺序内存模型下执行的结果一致
* 对没有正确同步要求的多线程程序，进行一定程度的限制，以保证安全性

其中第一方面是与 Data Race Free相关的，第二方面与后面介绍的 Causality Requirements 相关。

## Data Race Free

Java 内存模型其实定义了好几个概念来说明什么是正确的同步。

* 冲突访问(conflicting accesses)

如果存在多个线程，同时访问同一地址，并且至少有一个是写操作，那么这个程序存在冲突访问

* happen-before order

两个操作之间如果满足下面任意一个条件，就可以说这两个操作之间存在 happen-before order:

1. 同一个线程内，在程序中有先后次序的操作
2. 构造器的结尾的操作与 finalize 函数的开始的操作
3. unlock 操作与所有同一把锁上的 lock操作
4. volatile 变量的读操作与所有对它的写操作
5. 对变量默认值的写操作与线程启动后的第一个操作
6. 如果线程 T2 检测到线程 T1 终止执行，那么 T1 的最后一次操作与 T2任意操作
7. 启动一个线程的操作与此线程内第一个操作
8. 如果线程 T1 中断了线程 T2，那么此中断操作与其它任何看到 T2 被中断的操作之间。

其中有些我也不是很理解。。

* data race free

所有存在冲突访问的操作之间都有 happen-before order，那么此多线程程序满足 data race free

* 正确同步 

假如多线程程序在顺序一致性模型下执行，如果它满足 data race free，那么此程序进行了正确的同步。

正确同步的多线程程序，其执行结果与在顺序一致性模型下的执行结果一致。仔细体会下概念之间的关系。有点绕。

另一方面，如果程序没有正确同步，执行结果也不是任意的，必须对其进行限制，但限制又不能太强，因为太强会阻碍优化。所以 Java 内存模型使用了 Causality Requirements 的概念。



## Causality Requirements

为了精确定义内存模型，Java语言规范中，提出了 Causality Requirements 的概念。不知道是什么原因，这个概念很少被提及，但是我觉得它是很重要的，但同时，也是非常令人费解的。语言规范中，首先定义了 Well-Formed Executions 的概念，现在对内存模型的很多讨论，都是在这一层，它包括了对多线程程序执行中，与锁，volatile变量，执行次序等等相关的规定。如果一个多线程程序的执行满足这些规定，那么这个执行就是 Well-Formed Executions 的。国内有一个系列文章《[深入理解Java内存模型](http://ifeve.com/java-memory-model-0/)》，主要是在这方面描述Java内存模型。此外，在 Java 并发领域内著名的 Doug Lea 也给出了一个 [The JSR-133 Cookbook for Compiler Writers](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)，为编译器作者们提供参考,探讨的也是这方面的问题。但是，内存模型对多线程程序的执行是否合法，不仅仅要看它是否是 Well-Formed Executions，这次执行还需要满足 Causality Requirements。

语言规范中规定了一个构造过程，如果通过这个构造过程，可以构造出多线程程序最终的执行结果，那么这次执行就满足 Causality Requirements。构造过程从一个空集合C0开始，每次将其中添加若干操作，如果所有操作都能被添加，那么构造成功。即，

```
C0 -> C1 -> C2 -> ... -> C
```

其中 C_i 是 C_(i+1) 的子集。你可能注意到了，之前说的“操作能被添加”，什么叫操作能被添加呢？语言规范中规定了，每一个 Ci 都对应一个 Ei，所有 Ei 都要满足 Well-Formed Executions。也就是说，如果你添加了操作后，对应的 Ei 不满足 Well-Formed Executions，那么这个操作就不能被添加。如果最终，你的多线程程序无法构造出这样一个执行链，那么，它的执行结果是非法的。

另外，Java 内存模型最初论文作者维护了一个页面 [The Java Memory Model](http://www.cs.umd.edu/~pugh/java/memoryModel/)，其中有一个条目叫 [Causality Test Cases](http://www.cs.umd.edu/~pugh/java/memoryModel/CausalityTestCases.html)，给出了一些小例子，以便人们明白哪些行为是满足 Causality Requirements 的，哪些是不满足的。此外，在 Java 并发领域内著名的 Doug Lea 也给出了一个 [The JSR-133 Cookbook for Compiler Writers](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)，为编译器作者们提供参考。不过据说这份规范有些地方要求太严格了，开发者们还是根据Java语言规范和虚拟机规范来开发。



