---
layout: default
title: 对C语言中的static关键字的深入理解
categories: [技术, C语言, 计算机系统]
comments: true
---



# 对C语言中的static关键字的深入理解

在阅读一些项目源代码时，我发现很多时候，会把函数和变量声明为static，所以，很好奇为什么要这样做，于是有了下面这篇文章。

# 基本概念

使用static有三种情况：

* 函数内部static变量
* 函数外部static变量
* static函数

函数内部的static变量，关键在于生命周期持久，他的值不会随着函数调用的结束而消失，下一次调用时，static变量的值，还保留着上次调用后的内容。



函数外部的static变量，以及static函数，关键在于私有性，它们只属于当前文件，其它文件看不到他们。例如：

```c
/* test_static1.c */
#include <stdio.h>

void foo() {
}

static void bar() {
}

int i = 3;
static int j = 4;

int main(void){
    printf ("%d \n", i);
    printf ("%d \n", j);
    return 0;
}


```

```c
/* test_static2.c */
void foo() {
}

static void bar() {
}

int i = 1;
static int j = 2;

```

将两个文件一起编译

> gcc test_static1.c test_static2.c -o test_static

编译器会提示：

    /tmp/ccuerF9V.o: In function `foo':
    test_static2.c:(.text+0x0): multiple definition of `foo'
    /tmp/cc9qncdw.o:test_static1.c:(.text+0x0): first defined here
    /tmp/ccuerF9V.o:(.data+0x0): multiple definition of `i'
    /tmp/cc9qncdw.o:(.data+0x0): first defined here
    collect2: ld returned 1 exit status


把与非static变量i相的语句注释掉就不会有此提示i重复定义了，原因就在于使用static声明后，变量私有化了，不同文件中的同名变量不会相互冲突。

static 函数也与此类似，将函数声明为static，说明我们只在当前文件中使用这个函数，其它文件看不到，即使重名，也不会相互冲突。


# 深入理解

> 从来就不应该仅仅满足于了解现象，还要了解现象的背后有什么

## 为什么函数内部的static变量和普通函数变量生命周期不一样
我们的程序，从源代码经过编译，链接生成了可执行文件，可执行文件被加载到存储器中，然后执行。以Unix程序为例，每个Unix程序都有一个运行时存储器映像。可以理解为程序运行时，存储器中的数据分布。

![linux_rtmi](/assets/blog-images/linux_run_time_memory_image.png)

图1 Linux运行时存储器映像

当程序运行时，操作系统会创建用户栈(User stack)，一个函数被调用时，它的参数，局部变量，返回地址等等，都会被压入栈中，当函数执行结束后，这些数据就会被其它函数使用，所以函数调用结束后，局部变量的值不会被保持。我们将此区域放大，可以看到用户栈中都有哪些内容。

![sfs](/assets/blog-images/stack_frame_structure.png)

图2 栈帧结构

而static变量与普通局部变量不同，它不是保留在栈中。注意图一中，有一块区域，"Loaded from executable file"，其中有一块 .data, .bss区，static变量会被存储在这里，所以函数调用结束后，static变量的值仍然会得到保留。而 .data, .bss区，executable file，与程序的编译，链接，相关。

首先，多个源代码会分别被编译成可重定位目标程序，然后链接器会最终生成可执行目标程序。可重定位目标程序的结构如图3所示，可以看出，此时，.data, .bss区，已经出现。

![re](/assets/blog-images/relocatable_elf.png)

图3 可重定位目标程序

.data 区存储已经初始化的全局C变量，.bss 区存储没有初始化的全局C变量，而编译器会为每个static变量在.data或者.bss中分配空间。

可执行目标程序的结构如图4所示

![ee](/assets/blog-images/executable_elf.png)

图4 可执行目标程序

将图4与图1比较，就会发现，可执行目标程序的一部分被加载到存储器中，这就是"Loaded from executable file"的来源。

另外，从图一中，也可以看出，使用malloc分配的内存空间，与函数局部变量，static变量的不同。

## 为什么函数外部的static变量及static函数只对文件内部可见

要解释这个问题，我们首先要理解问题本身。这个问题的本质其实是，当我们遇到一个变量或者函数时，我们去哪里寻找它，static变量/函数与普通变量/函数的寻找方式有什么不同。

我们回到刚才的例子，这一次，仔细地观察编译链接时的提示信息：

```c
/* test_static1.c */
#include <stdio.h>

void foo() {
}

static void bar() {
}

int i = 3;
static int j = 4;

int main(void){
    printf ("%d \n", i);
    printf ("%d \n", j);
    return 0;
}


```

```c
/* test_static2.c */
void foo() {
}

static void bar() {
}

int i = 1;
static int j = 2;

```

将两个文件一起编译

> gcc test_static1.c test_static2.c -o test_static

编译器会提示：

    /tmp/ccuerF9V.o: In function `foo':
    test_static2.c:(.text+0x0): multiple definition of `foo'
    /tmp/cc9qncdw.o:test_static1.c:(.text+0x0): first defined here
    /tmp/ccuerF9V.o:(.data+0x0): multiple definition of `i'
    /tmp/cc9qncdw.o:(.data+0x0): first defined here
    collect2: ld returned 1 exit status


你会发现，虽然我们只用了一条命令对两个文件进行编译链接，但是，实际上，两个源文件是被分别编译成/tmp/ccuerF9V.o及/tmp/cc9qncdw.o，并且，错误并不是出现在编译时，而是出现在链接时，链接器ld返回了1。链接是把两个可重新定位的目标程序，组合在一起，组合的时候，我们发现了变量i及函数foo的定义出现冲突。而声明为static的变量j及函数bar并没有提示冲突。

这说明，在ld进行链接时，需要进行某种检查，去发现冲突。ld的输入是每个源文件生成的可重定位目标文件，那么这些目标文件里一定会有一些信息，告诉ld它们有什么变量，然后ld才能检查是不是有冲突。

说起`可重定位目标文件`，我们一直都没有解释为什么要重定位。其实这很好理解，一个源文件编译后，如果生成的目标文件中，各个地址就是最终运行时的地址，那么这些地址很可能会和其它文件中的地址冲突。因为编译一个文件时，我们不会知道有其它文件的存在，所以编译时无法确定最终的地址。因此，编译单个文件时，生成的目标文件中的地址都是从0开始，链接时，链接器会将不同目标文件中的地址重新定位，最终生成可执行文件。注意这里的冲突和前面说的冲突不是一回事，这里的冲突是不同的可重定位目标文件中相同地址的冲突，前面一段讲的是同名变量之间的冲突。

此时，我们不得不回到可重定位目标文件的格式。

![re](/assets/blog-images/relocatable_elf.png)

图3 可重定位目标程序

注意 .symtab节，这个节存储符号表，假设当前可重定位目标模块为m, 符号表会告诉我们m中定义和引用的符号信息，主要分为：

* m定义，并可以被其它模块引用的全局符号：m中的非static函数，非static全局变量。
* 由其它模块定义，并被m引用的全局符号：m中使用extern声明的变量
* 只被m引用的本地符号：m中的static函数，static全局变量。

现在编译一下，然后用GNU READELF工具看一下符号表。

```
    $ gcc -c test_static1.c -o test_static1.o
    $ readelf -s test_static1.o

Symbol table '.symtab' contains 15 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 00000000     0 FILE    LOCAL  DEFAULT  ABS test_static1.c
     2: 00000000     0 SECTION LOCAL  DEFAULT    1 
     3: 00000000     0 SECTION LOCAL  DEFAULT    3 
     4: 00000000     0 SECTION LOCAL  DEFAULT    4 
     5: 00000005     5 FUNC    LOCAL  DEFAULT    1 bar
     6: 00000004     4 OBJECT  LOCAL  DEFAULT    3 j
     7: 00000000     0 SECTION LOCAL  DEFAULT    5 
     8: 00000000     0 SECTION LOCAL  DEFAULT    7 
     9: 00000000     0 SECTION LOCAL  DEFAULT    8 
    10: 00000000     0 SECTION LOCAL  DEFAULT    6 
    11: 00000000     5 FUNC    GLOBAL DEFAULT    1 foo
    12: 00000000     4 OBJECT  GLOBAL DEFAULT    3 i
    13: 0000000a    62 FUNC    GLOBAL DEFAULT    1 main
    14: 00000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
```

表的数据结构不解释，有兴趣，看扩展阅读部分。

现在，假如你是链接器ld，我给你2个可重定位目标程序，你从中得到两个符号表，这时候，你就可以检查出两个符号表是否存在冲突了。

由于全局符号可能会定义相同的名字，链接器会有一套规则，来确定选择哪个符号。符号分为强符号与弱符号。

* 强符号：函数和已经初始化的全局变量是强符号
* 弱符号：未初始化的全局变量是弱符号

处理相同名字的全局符号的规则是：

1. 不允许有多个强符号
2. 如果有一个强符号，多个弱符号，那么选择强符号
3. 如果有多个弱符号，那么从中任意选择一个

明白了这些规则，你其实可以明白很多事情，不仅仅包括什么时候，变量名，函数名会冲突，还包括为什么要尽量避免使用全局变量，为什么要使用static把数据私有化。看看规则3，“任意”两个字，有没有让你感觉有一丝不适。

这也是为什么我们要探索事物背后机理的原因，不仅仅是在出现错误时，我们知道哪里有问题，还帮助我们写出更健壮的程序。


# 扩展阅读
《深入理解计算机系统》(Computer Systems, A Programmer's Perspective): 第七章 链接
