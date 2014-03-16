---
layout: default
title: Java 内存模型论文阅读
categories: [科研]
comments: true
---


# Java 内存模型论文阅读

## 引言
Java 的内存模型最早出现在1995年，但是自1997年起，这一内存模型被发现了许多严重的错误和缺陷，它阻碍了很多优化措施，对程序的安全性也没有足够的保证。2001年[JSR 133](https://www.jcp.org/en/jsr/detail?id=133)被确立下来，由William Pugh领导，专家组的成员包括了Adve，Doug Lea， William Pugh等。2004年，JSR 133最终版本发布。2005年，Manson  Jeremy, William Pugh, 和 Sarita V. Adve 一同发表了论文 *The Java memory model*，描述了最新的Java内存模型，这一内存模型在Java 5.0中引入，一直沿用至今。此后，科学家们对Java内存模型进行了进一步的研究的探索，但大的改动并没有出现。

## 论文介绍

### Pugh2000

* 论文名
    Pugh W. The Java memory model is fatally flawed[J]. Concurrency - Practice and Experience, 2000, 12(6): 445-455.

* 主要内容
   介绍了现有Java内存模型的不足之处：
        * 难以理解，不同的人有不同解读
        * 禁止了很多编译器优化，大多数JVM实现都违反Java内存模型
        * 一些常用编程范式违反Java内存模型
        * 没有考虑到在共享内存，弱一致性内存模型下实现Java所带来的一些问题

* 参考文献
    * Gontmakher 1997: 证明了Java内存模型需要Coherence

### **Manson 2005**

* 论文名
    Manson J, Pugh W, Adve S V. The Java memory model[M]. ACM, 2005.

* 主要内容
    JSR 133的成果，介绍了新的Java内存模型，由Java 5.0引入，沿用至今。
   基本思想包括：
        * 对满足 Data Race Free 的程序保证顺序一致性(sequential consistency)
        * 对没有正确同步的程序，使用causality的概念加以限制，以保证程序的安全性
        * 新的内存模型足够强，以保证安全性，又足够弱，以保证编译器可以使用足够的优化
        
        
        

* 参考文献

       * Lamport 1979: 提出 Sequential Consistency 概念
       * Relaxed models in academia and commercial hardware
                - Adve 1990
                - Adve 1993
                - Dubois 1986
                - Gharachorloo 1990
                - IBM 1983(System/370 Principles of Operation)
                - May 1994(The PowerPC Architecture)
                - Sites 1995(Alpha AXP Architecture Reference Manual)
                - Weaver 1994(The SPARC Architecture Manual)
       * SC for DRF
                - Adve 1990
                - Adve 1993
                - Gharachorloo 1990
       * Flaws in original Java Memory Model
                - Pugh 1999
       * Original Java Memory Model Research
                - Gosling 1996
                - Kotrajaras 2001
                - Saraswat 2004

### **Polyakov 2006**

* 论文名
 Polyakov S, Schuster A. 
*Verification of the Java causality requirements*
[M]//Hardware and Software, Verification and Testing. Springer Berlin Heidelberg, 2006: 224-246.

* 主要内容
        * 证明验证causality是NP-complete的
        * 跟踪每个线程实际运行时 read 操作的顺序可以简化验证
        * 对可简化的验证提出了多项式算法
        * 不能简化的提出非多项式算法（仅用于短的测试序列）
        * 使用了Post-mortem的方法，实际运行一个多线程程序，在JVM或者定制过的JVM或者模拟器上运行程序，拿到trace，分析trace，以验证内存是否有问题
        * 使用frontier graph验证

*  参考文献
       * Boehm 2005: 通过库实现多线程不能保证程序正确性
       * causal acyclicity 形式化定义(reach condition)：
              -   Suﬃcient System Requirements for  Supporting the PLpc Memory Model. 1993
              -  Specifying System Requirements for Memory Consistency Models. 1993
       * Gibbons 1993：使用frontier graph验证SC


### Aspinall 2007-1

* 论文名
Aspinall D, Ševčík J. 
*Formalising Java’s data race free guarantee*
[M]//Theorem Proving in Higher Order Logics. Springer Berlin Heidelberg, 2007: 22-37.

* 主要内容
    * 给出精确的DRF定义和证明
    * 发现要保证DRF，JMM中并非所有条件都要满足
    * 形式化定义为测试具体实例提供了基础
    * 证明了JMM中给定的条件可以保证DRF
* 参考文献


### Huisman 2007

* 论文名
Huisman M, Petri G.
*The Java memory model: a formal explanation*
[J]. VAMP, 2007, 7: 81-96.

* 主要内容
  * 使用Coq中形式化描述JMM
  * 证明DRF的条件

* 参考文献

### Cenciarelli 2007

*论文名
Cenciarelli P, Knapp A, Sibilio E. 
*The Java memory model: Operationally, denotationally, axiomatically*
[M]//Programming Languages and Systems. Springer Berlin Heidelberg, 2007: 331-346.

* 主要内容

   * 构建新的语义框架，由 operational step 构成 denotional model，并被 axioms 限制
   * 使用Configuration Theory 描述 Java 操作规则
   * 为 Java 提供一个基于事件的语义

### Aspinall 2007-2

* 论文名
Aspinall D, Sevcik J.
*Java memory model examples: Good, bad and ugly*
[J]. VAMP07 Proceedings 2007.

* 主要内容
    * Good Example: JMM 允许的行为,展示了非SC的行为和一些优化 
    * Bad Example:JMM禁止的行为
    * Ugly Example:JMM禁止,但却出现的行为
    * 通过这些例子展示 Aspinall 2007-1 中提出的形式化定义优于官方定义

### Botinan 2007 

* 论文名
Botincan M, Glavan P, Runje D. 
*Distributed Algorithms: A Case Study of the Java Memory Model[J].*
Proceedings of the ASM, 2007.

* 主要内容
    * 对 JMM 提供数学化的精确定义
    * 为其在ASM context下提供解释

### Sevcik 2008

* 论文名
Ševčík J, Aspinall D. 
*On validity of program transformations in the Java memory model[M]*
//ECOOP 2008–Object-Oriented Programming. Springer Berlin Heidelberg, 2008: 27-51.

* 主要内容
    * 分析了一些常见但在JMM中禁止的优化措施,揭示了  Hotspot JVM 中违背JMM的情况
    * 对data race程序的要求比设计者想的要强

### **arnabde 2008**

* 论文名
De A, Roychoudhury A, D'Souza D. 
*Java memory model aware software validation[C]*
//Proceedings of the 8th ACM SIGPLAN-SIGSOFT workshop on Program analysis for software tools and engineering. ACM, 2008: 8-14.

* 主要内容

提出一种近似JMM的内存模型OpMM,它可以与模型检测工具JPF结合,寻找软件中的bug.

### **Chen Chen 2009**

* 论文名
Chen C, Chen W, Sreedhar V, et al. 
*Formalizing Causality as a Desideratum for Memory Models and transformations of Parallel Programs[J].*
2009.

* 主要内容
    * 提出causally ordered,用以构造 causality graph 框架,以找环的方式分析内存模型是否违反 causality
    * 识别出代码转换中保持/违反 causality的措施
    * 提出CMM内存模型,是保证不违反causality的最弱的内存模型

* 参考文献

    * JMM 社区提出了20 causality test cases,用于编译器和虚拟机验证


### Botincan 2010

* 论文名
Botinčan M, Glavan P, Runje D. Verification of causality requirements in Java memory model is undecidable[M]//Parallel Processing and Applied Mathematics. Springer Berlin Heidelberg, 2010: 62-67.

* 主要内容
    证明验证任意有限次执行的多线程程序是否满足causality requirments是undecidable的.

* 参考文献
    * Polyakov 2006:在无同步操作,final作用域上,通过验证有限次数执行的多线程程序,来验证JMM的causality requirements.证明了在给定的一些假设上,此问题是NP-complete的.

### **Torlak 2010**

* 论文名
Torlak E, Vaziri M, Dolby J. 
*MemSAT: checking axiomatic specifications of memory models[J].*
ACM Sigplan Notices, 2010, 45(6): 341-350.

* 主要内容
    * 基于SAT solver的工具MEMSAT,用于调试推导内存模型,如果给出内存模型的公理化描述,包含断言的多线程程序,工具可以输出一个trace,保证内存模型及多线程程序的断言都得到满足.
    * 在Manson 的 JMM上,以及Sevcik的修复版本上测试过.
    * 第一个对 JMM 的公理化描述进行自动调试推理的工具

* 参考文献
    * litmus tests 用于对内存模型的形式化说明进行补充,方便人们理解内存模型,验证litmus tests的工作包括:
        - model checking
            arnabde 2008: Java memory model aware software validation
            yang 2001: Analyzing the CRF Java memory model
        - constrain solving
            Burckhardt 2007: CheckFence: checking consistency of concurrent data types on relaxed memory models
            Gopalakrishnan 2004: QB or Not QB: An efﬁcient execution veriﬁcation tool for memory orderings
            Yang 2003: Analyzing the Intel Itanium memory ordering rules using logic programming and SAT.
        - custome search
            Sarkar 2009:The semantics of x86–CC multiproces sor machine code
这些工作已经成功对 Intel Itanium x86-CC MM, **JMM** 进行验证,但Aspinall 2007指出了JMM中commiting semantics带来的巨大状态空间使得model checking难以适用.

    * Yang 2004:Nemos:a framework for axiomatic and executable speciﬁcations of memory consistency models指出公理语义在描述内存模型上优于操作语义,

    * Sevcik 2008: 对Manson的JMM进行修复.

    * JMM 发展历史,相关工作
        * J. Manson JMM simulator

### Lochbihler 2012

*  论文名
Lochbihler A. 
*Java and the Java memory model—A unified, machine-checked formalisation[M]*
//Programming Languages and Systems. Springer Berlin Heidelberg, 2012: 497-517.

* 主要内容
对 JMM 进行形式化,并通过机器检查,与Java源代码及字节码的操作语义联系在一起.
证明了语义保证了DRF

*  参考文献

    * Related Work 介绍的挺全
    

### Jin 2012

* 论文名
Jin H, Yavuz-Kahveci T, Sanders B A. Java memory model-aware model checking[M]. Springer Berlin Heidelberg, 2012.

* 主要内容
       * 扩展JPF,产生包含data race的执行
       * 提供工具 Java PathRelaxer(JPR),用于推导包含data race的程序,验证它的性质


* 参考文献

    * Manson 2002:  The Java memory model simulator


### Demange 2013

* 论文名
Demange D, Laporte V, Zhao L, et al. 
*Plan B: A buffered memory model for Java[C]*
//ACM SIGPLAN Notices. ACM, 2013, 48(1): 329-342.

* 主要内容

* 提出了一种新的Java 内存模型BMM,给出公理化定义,刻画了内存事件的次序
* 给出BMM'的形式化定义,对Java程序的语义来说,这是一个可操作的定义,容易用于x86体系结构.
* 证明BMM和BMM'是一样的
* 给出BMM性能测试结果
* 相关工作介绍了JMM发展,验证等等.

* 参考文献 

    * Sevcik 2011: Relaxed-memory Concurrency and Veriﬁed Compilation

    * 在当前情况下,证明一个编译器是否符合JMM中的定义仍然是一个open的问题.

    * Sevcik2008: 现存的JVM是不符合JMM的

### **LOCHBIHLER 2013**

* 论文名
Lochbihler A.
Making the Java memory model safe[J]. 
ACM Transactions on Programming Languages and Systems (TOPLAS), 2013, 35(4): 12.

* 主要内容 
        * 基于之前的JinJiaThread语义,为公理语义的JMM提出了一个 *unified formalization*,将Java与JMM结合在一起
        * 澄清了现有的JMM标准,修复了一些不合适的地方
        * 证明了DRF需要满足的条件

* 参考文献
