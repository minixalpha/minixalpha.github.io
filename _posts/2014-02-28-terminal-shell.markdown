---
layout: default
title: 控制台，终端，tty，shell等概念的区别
category: 工具
comments: true
---

# 控制台，终端，tty，shell等概念的区别

使用linux已经有一段时间，却一直弄不明白这几个概念之间的区别。虽然一直在用，但是很多概念都感觉模糊不清，这样不上不下的状态实在令人不爽。下面就澄清一下这些概念。

这些概念本身有着非常浓厚的历史气息，随着时代的发展，他们的含义也在发生改变，它们有些已经失去了最初的含义，但是它们的名字却被保留了下来。


## 控制台(Console)
控制台([Console](http://en.wikipedia.org/wiki/System_console))是物理设备，用于输入输出，它直接连接在计算机上，是计算机系统的一部分。计算机输出的信息会显示在控制台上，例如BIOS的输出，内核的输出。

## 终端(Terminal)
终端([Terminal](http://en.wikipedia.org/wiki/Computer_terminal))也是一台物理设备，只用于输入输出，本身没有强大的计算能力。一台计算机只有一个控制台，在计算资源紧张的时代，人们想共享一台计算机，可以通过终端连接到计算机上，将指令输入终端，终端传送给计算机，计算机完成指令后，将输出传送给终端，终端将结果显示给用户。

## 虚拟控制台(Virtual Console)，虚拟终端(Virtual Terminal)
虚拟控制台([Virtual Console](http://en.wikipedia.org/wiki/Virtual_console))和虚拟终端是一样的。我们只有一台终端（物理设备），这是我们与计算机之间的用户接口。假如有一天，我们想拥有多个用户接口，那么，一方面我们可以增加终端数目（物理设备），另一方面，还可以在同一台终端（物理设备）上虚拟出多个终端，它们之间互相不影响，至少看起来互相不影响。这些终端就是虚拟终端。

在Ubuntu中，我们按下Ctrl+Alt+Fx时，会进入第x个虚拟终端，一共有七个虚拟终端，其中第七个虚拟终端，就是我们默认使用的图形用户界面。

## 终端模拟器(Terminal Emulator)
我们知道，终端是一种物理设备，而终端模拟器([Terminal Emulator](http://en.wikipedia.org/wiki/Terminal_emulator))，是一个程序，这些程序用来模拟物理终端。图形用户界面中的终端模拟器一般称为终端窗口(Terminal Window)，我们在Ubuntu下打开的gnome-terminal就属于此类。

## tty
tty的全称是[TeleTYpewriter](http://en.wikipedia.org/wiki/Teletypewriter)，这就是早期的终端（物理设备），它们用于向计算机发送数据，并将计算机的返回结果打印出来。显示器出现后，终端不再将结果打印出来，而是显示在显示器上。但是tty的名字还是保留了下来。

在Ubuntu中，我们按下Ctrl+Alt+F1时，会进入第1个虚拟终端，你可以看到屏幕上方显示的tty1。

## shell
[shell](http://en.wikipedia.org/wiki/Shell_(computing)) 和之前说的几个概念截然不同，之前的几个概念都是与计算机的输入输出相关的，而shell是和内核相关的。内核为上层的应用提供了很多服务，shell在内核的上层，在应用程序的下层。例如，你写了一个 hello world 程序，你并不用显式地创建一个进程来运行你的程序，你把写好的程序交给shell就行了，由shell负责为你的程序创建进程。

我们在终端模拟器中输入命令时，终端模拟器本身并不解释执行这些命令，它只负责输入输出，真正解释执行这些命令的，是shell。

我们平时使用的sh, bash, csh是shell的不同实现。

* sh
sh这个概念本身就有岐义，它可以指shell程序的名字，也代表了shell的实现。

    [Thompson shell](http://en.wikipedia.org/wiki/Thompson_shell)是第一个Unix shell，由 Ken Thompso于1971年在Unix第一版本中引入，shell的程序名即为sh。[Bourne shell](http://en.wikipedia.org/wiki/Bourne_shell)作为Thompson shell的替代，由 Stephen Bourne于1977年在Unix第七版中引入，它的程序名也是sh。Bourne shell不仅仅是一个命令解释器，更作为一种编程语言，提供了Thompson shell不具备的程序控制功能，并随着 Brian W. Kernighan 和 Rob Pike 的 *The UNIX Programming Environment*的出版而名声大噪。

* csh
[csh](http://en.wikipedia.org/wiki/C_shell)全称为 C Shell，由 Bill Joy在70年代晚期完成，那时候他还是加州伯克利大学的研究生。tcsh是csh的升级版。与sh不同，csh的shell脚本，语法接近于C语言。

* bash
[bash](http://en.wikipedia.org/wiki/Bash_(Unix_shell))是由 Brian Fox为GNU项目开发的自由软件，作为Bourne shell的替代品，于1989年发布。是Linux和Mac OS X的默认shell。bash的命令语法是Bourne shell命令语法的超集，从ksh和csh借鉴了一些思想。


好了，就写到这里，上面的内容是我参考维基百科后写下的，**不保证完全正确**，
下面还提供了一些资料，如果有兴趣可以阅读一下。

## 扩展阅读

1. [What is the exact difference between a 'terminal', a 'shell', a 'tty' and a 'console'?](http://unix.stackexchange.com/questions/4126/what-is-the-exact-difference-between-a-terminal-a-shell-a-tty-and-a-con)

2. [shell，bash,zsh,console,terminal到底是什么意思，它们之间又是什么关系？](http://www.linuxsir.org/bbs/thread362001.html?pageon=1#2059206)

3. [shell、控制台、终端的区别](http://blog.csdn.net/caomiao2006/article/details/8791775)

4. [Why is a virtual terminal “virtual”, and what/why/where is the “real” terminal?](http://askubuntu.com/questions/14284/why-is-a-virtual-terminal-virtual-and-what-why-where-is-the-real-terminal)
