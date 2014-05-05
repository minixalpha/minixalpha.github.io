---
layout: default
title: 使用JVMTI获取Java多线程程序指令执行次序
categories: [技术, Java, 虚拟机]
comments: true
---

# 使用JVMTI获取Java多线程程序指令执行次序

---

在Java多线程程序中，由于线程调度，指令间的次序在每次运行时都可能不相同，有时候，我们需要得到指令次序，用来分析程序的行为。这样细粒度的底层行为用一般方法很难完成，我们需要借助 [JVM Tool Interface](http://docs.oracle.com/javase/7/docs/platform/jvmti/jvmti.html)，即JVMTI，来帮助我们获取Java虚拟机执行时的信息。本文先介绍编写JVMTI程序的基本框架，然后介绍如何使用JVMTI来获取多线程程序中指令之间的次序。

## JVMTI简介
JVMTI是用于编写开发与监视工具的编程接口，使用它可以检查并控制运行于Java虚拟机上的程序。使用它可以完成性能分析，调试，监视(monitoring)，线程分析，覆盖分析(coverage analysis)等工具。

使用JVMTI可以编写出一个agent。在运行Java程序时，指定这个agent，那么当虚拟机运行程序时，如果agent中指定的一些事件发生，虚拟机就会调用agent中相应的回调函数。JVMTI提供了一系列可以指定的事件，以及获取虚拟机中信息的函数接口。


## JVMTI基本编程方法

### 编写agent

* 头文件
agent程序中，需要包含 `jvmti.h`头文件，才能使用JVMTI中提供的接口。

```c
#include <jvmti.h>
```

* 基本事件
和agent有关的两个基本事件是agent的启动与关闭，我们需要自己编写与启动与关闭相关的函数，这样，虚拟机才知道启动与关闭agent时，都需要做些什么。

 与启动相关的函数有两个，如果你的agent在虚拟机处于`OnLoad`阶段时启动，会调用`Agent_OnLoad`函数，如果你的agent在虚拟机处于`Live`阶段时启动，会调用`Agent_OnAttach `函数。

 我的理解是，如果你的agent想要全程监视一个程序的运行，就编写`Agent_OnLoad`，并在启动虚拟机时指定agent。如果你的agent想获取一个已经在运行的虚拟机中程序的信息，就编写`Agent_OnAttach`。

 两个函数的原型如下：

```c
JNIEXPORT jint JNICALL 
Agent_OnLoad(JavaVM *vm, char *options, void *reserved)
```

```c
JNIEXPORT jint JNICALL 
Agent_OnAttach(JavaVM* vm, char *options, void *reserved)
```

 与agent关闭相关的函数是`Agent_OnUnload`，当agent要被关闭时，虚拟机会调用这个函数，函数原型为：

```c
JNIEXPORT void JNICALL 
Agent_OnUnload(JavaVM *vm)
```

* 程序基本框架

主要的内容框架在`Agent_OnLoad`中编写：

 1 获取jvm环境

```c
/* get env */
jvmtiEnv *jvmti = NULL;
jvmtiError error;

error = (*jvm)->GetEnv(jvm, (void **)&jvmti, JVMTI_VERSION);
if (error != JNI_OK) {
    fprintf(stderr, "Couldn't get JVMTI environment");
    return JNI_ERR;
}
```

可以为同一个虚拟机指定多个agent，每个agent都有自己的环境，在指定agent行为前，首先要获取的就是环境信息，后面的操作都是针对这个环境的。另外，JVMTI中的函数都会返回错误代码，在调用函数后，需要检查返回值，以确定函数调用是否成功。不同的函数会返回不同类型的错误码，可自行参阅JVMTI的API。

另外，需要注意，JVMTI程序可以使用C/C++编写，两者在调用函数时略有不同，上面的例子是用C编写，gcc编译。如果你使用C++编写，`GetEnv`需要这样调用：

```c++
error = (jvm)->GetEnv(reinterpret_cast<void**>(&jvmti), JVMTI_VERSION_1_1);
```

其它函数依次类推。 

2 添加capabilities

JVMTI中有很多事件，每个事件都对对应一些Capabilities，如果你想为此事件编写函数，就要开启相应的Capabilities，例如，我们想对 `JVMTI_EVENT_SINGLE_STEP` 事件编写函数，可以查到，需要开启`can_generate_single_step_events`：

```c
/* add capabilities */]
jvmtiCapabilities capa;
memset(&capa, 0, sizeof(jvmtiCapabilities));
capa.can_generate_single_step_events = 1;
error = (*jvmti)->AddCapabilities(jvmti, &capa);
check_jvmti_error(jvmti, error, \
        "Unable to get necessary JVMTI capabilities.");
```

如果开启的Capabilities多于一个，不用声明多个`jvmtiCapabilities`变量，只需要使用类似

```c
capa.can_generate_single_step_events = 1;
```
的方式指定就行。

3 指定事件

JVMTI编写的目的是，当虚拟机中一个事件发生时，调用我们为此事件编写的函数。所以我们需要指定哪个事件发生时，通知agent：

```c
/* set events */
error = (*jvmti)->SetEventNotificationMode \
        (jvmti, JVMTI_ENABLE, JVMTI_EVENT_SINGLE_STEP, NULL);
check_jvmti_error(jvmti, error, "Cannot set event notification");
```

其中 `JVMTI_EVENT_SINGLE_STEP` 就是事件代码。

> 需要特别注意的是，要先开启相关capabilities，然后才能指定事件。

4 设置回调函数

我们还需要为事件指定回调函数，并自行编写回调函数，事件回调函数的接口是由JVMTI指定的，例如`JVMTI_EVENT_SINGLE_STEP`事件的回调函数原型：

```
void JNICALL
SingleStep(jvmtiEnv *jvmti_env,
            JNIEnv* jni_env,
            jthread thread,
            jmethodID method,
            jlocation location)
```

为事件指定回调函数的方法是：

```c
jvmtiEventCallbacks callbacks;

/* add callbacks */
memset(&callbacks, 0, sizeof(callbacks));
callbacks.SingleStep = &callbackSingleStep;
error = (*jvmti)->SetEventCallbacks \
        (jvmti, &callbacks, (jint)sizeof(callbacks)); 
check_jvmti_error(jvmti, error, "Canot set jvmti callbacks");

```

之后，我们需要自己编写 `callbackSingleStep`函数：

```c
void JNICALL
callbackSingleStep(
    jvmtiEnv *jvmti, 
    JNIEnv* jni, 
    jthread thread,
    jmethodID method,
    jlocation location) {

}
```

### 运行agent

运行agent，通过指定虚拟机参数来设定，例如运行`PossibleReordering`时：

```sh
java -classpath . \
    -agentpath:`pwd`/jvmagent/TraceAgent.so PossibleReordering
```
其中`TraceAgent.so`就是编译后生成的agent。

## 使用JVMTI获取多线程程序指令执行次序

我们知道，在Java虚拟机中的运行时数据区中，每个线程都有它的私有区域，每个线程有自己的PC寄存器，PC寄存器表示线程当前执行的指令在内存中的地址。其实我最初的目的是想得到这个PC的值，但是找了很久都没有找到，然后在JVMTI中找到了类似的概念。

在JVMTI中，介绍单步事件(Single Step Event)时说，当一个线程到达一个新的位置(location)时，单步事件就会产生。单步事件使agent以虚拟机允许的最细粒度，跟踪线程执行。

我们回到单步事件回调函数的原型：

```c
void JNICALL
SingleStep(jvmtiEnv *jvmti_env,
            JNIEnv* jni_env,
            jthread thread,
            jmethodID method,
            jlocation location)
```

其中的 `location` 就是新指令的位置。

我们首先来编写一个Java多线程程序，这个程序是 《Java并发编程实战》(Java Concurrency in Practice) 中的一个例子，我做了一点变形：

```java
import java.lang.Thread;

public class PossibleReordering {
	static int x = 0, y = 0;
	static int a = 0, b = 0;
	
	public static void main(String[] args) throws InterruptedException {
		
		Thread one = new Thread(new Runnable() {
			public void run() {
                a = 1;
                x = b;
			}
		});
		
		Thread other = new Thread(new Runnable() {
			public void run() {
                b = 1;
                y = a;
                a = 1;
                x = b;
                a = 1;
                x = b;
                a = 1;
                x = b;
                a = 1;
                x = b;
                a = 1;
                x = b;
                a = 1;
                x = b;
                a = 1;
                x = b;
			}
		});
		
		
		one.start();
		other.start();
		one.join();
		other.join();
	}
}
```

给Thread other多加了一些语句，用以区分两个线程。

这里有一个问题是，我们关心的其实只是两个线程的 `run`函数中指令的次序，而单步事件会在任何指令执行时，都调用回调函数，这就需要我们在回调函数中，只保留源代码中的两个线程的`run`函数中的指令的位置，其它的都过滤掉。

我们可以使用JVMTI提供的 `GetMethodName` 来得到函数名，使用 `GetMethodDeclaringClass`得到类名，然后通过比较类名和函数名，只保留 `run`中的指令：

```c
error = (*jvmti)->GetMethodName( \
        jvmti, method, &method_name, &method_signature, SKIP_GENERIC);

error = (*jvmti)->GetMethodDeclaringClass( \
         jvmti, method, &declaring_class);
       

if (strncmp(method_name, "run", 4) == 0   && \
      strstr(class_signature, "PossibleReordering") != NULL) {
      printf("%s\t", thread_info.name);
      printf("%s\t", class_signature);
      printf("%lld\t", location);
      printf("%s %lld:%lld\t", method_name, s_location, e_location);
      printf("\n");
}
```

执行下列命令：

```sh
java -classpath . -agentpath:`pwd`/jvmagent/TraceAgent.so=log.txt PossibleReordering
```

即可得到指令次序信息：

```
Thread-0	LPossibleReordering$1;	0	run 0:10	
Thread-0	LPossibleReordering$1;	1	run 0:10	
Thread-1	LPossibleReordering$2;	0	run 0:80	
Thread-0	LPossibleReordering$1;	4	run 0:10	
Thread-0	LPossibleReordering$1;	7	run 0:10	
Thread-0	LPossibleReordering$1;	10	run 0:10	
Thread-1	LPossibleReordering$2;	1	run 0:80	
Thread-1	LPossibleReordering$2;	4	run 0:80	
Thread-1	LPossibleReordering$2;	7	run 0:80	
Thread-1	LPossibleReordering$2;	10	run 0:80	
Thread-1	LPossibleReordering$2;	11	run 0:80	
Thread-1	LPossibleReordering$2;	14	run 0:80	
Thread-1	LPossibleReordering$2;	17	run 0:80	
Thread-1	LPossibleReordering$2;	20	run 0:80	
Thread-1	LPossibleReordering$2;	21	run 0:80	
Thread-1	LPossibleReordering$2;	24	run 0:80	
Thread-1	LPossibleReordering$2;	27	run 0:80	
Thread-1	LPossibleReordering$2;	30	run 0:80	
Thread-1	LPossibleReordering$2;	31	run 0:80	
Thread-1	LPossibleReordering$2;	34	run 0:80	
Thread-1	LPossibleReordering$2;	37	run 0:80	
Thread-1	LPossibleReordering$2;	40	run 0:80	
Thread-1	LPossibleReordering$2;	41	run 0:80	
Thread-1	LPossibleReordering$2;	44	run 0:80	
Thread-1	LPossibleReordering$2;	47	run 0:80	
Thread-1	LPossibleReordering$2;	50	run 0:80	
Thread-1	LPossibleReordering$2;	51	run 0:80	
Thread-1	LPossibleReordering$2;	54	run 0:80	
Thread-1	LPossibleReordering$2;	57	run 0:80	
Thread-1	LPossibleReordering$2;	60	run 0:80	
Thread-1	LPossibleReordering$2;	61	run 0:80	
Thread-1	LPossibleReordering$2;	64	run 0:80	
Thread-1	LPossibleReordering$2;	67	run 0:80	
Thread-1	LPossibleReordering$2;	70	run 0:80	
Thread-1	LPossibleReordering$2;	71	run 0:80	
Thread-1	LPossibleReordering$2;	74	run 0:80	
Thread-1	LPossibleReordering$2;	77	run 0:80	
Thread-1	LPossibleReordering$2;	80	run 0:80		
```

最终的源代码中，我还输出了线程名，和方法的指令地址范围。

我们可以反编译 `PossibleReordering$1` 和 `PossibleReordering$2`，看看相应的指令范围是否可以对应上。

```sh
$ javap -c PossibleReordering\$1.class
Compiled from "PossibleReordering.java"
final class PossibleReordering$1 implements java.lang.Runnable {
  PossibleReordering$1();
    Code:
       0: aload_0       
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return        

  public void run();
    Code:
       0: iconst_1      
       1: putstatic     #2                  // Field PossibleReordering.a:I
       4: getstatic     #3                  // Field PossibleReordering.b:I
       7: putstatic     #4                  // Field PossibleReordering.x:I
      10: return        
}
```

```sh
$ javap -c PossibleReordering\$2.class 
Compiled from "PossibleReordering.java"
final class PossibleReordering$2 implements java.lang.Runnable {
  PossibleReordering$2();
    Code:
       0: aload_0       
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return        

  public void run();
    Code:
       0: iconst_1      
       1: putstatic     #2                  // Field PossibleReordering.b:I
       4: getstatic     #3                  // Field PossibleReordering.a:I
       7: putstatic     #4                  // Field PossibleReordering.y:I
      10: iconst_1      
      11: putstatic     #3                  // Field PossibleReordering.a:I
      14: getstatic     #2                  // Field PossibleReordering.b:I
      17: putstatic     #5                  // Field PossibleReordering.x:I
      20: iconst_1      
      21: putstatic     #3                  // Field PossibleReordering.a:I
      24: getstatic     #2                  // Field PossibleReordering.b:I
      27: putstatic     #5                  // Field PossibleReordering.x:I
      30: iconst_1      
      31: putstatic     #3                  // Field PossibleReordering.a:I
      34: getstatic     #2                  // Field PossibleReordering.b:I
      37: putstatic     #5                  // Field PossibleReordering.x:I
      40: iconst_1      
      41: putstatic     #3                  // Field PossibleReordering.a:I
      44: getstatic     #2                  // Field PossibleReordering.b:I
      47: putstatic     #5                  // Field PossibleReordering.x:I
      50: iconst_1      
      51: putstatic     #3                  // Field PossibleReordering.a:I
      54: getstatic     #2                  // Field PossibleReordering.b:I
      57: putstatic     #5                  // Field PossibleReordering.x:I
      60: iconst_1      
      61: putstatic     #3                  // Field PossibleReordering.a:I
      64: getstatic     #2                  // Field PossibleReordering.b:I
      67: putstatic     #5                  // Field PossibleReordering.x:I
      70: iconst_1      
      71: putstatic     #3                  // Field PossibleReordering.a:I
      74: getstatic     #2                  // Field PossibleReordering.b:I
      77: putstatic     #5                  // Field PossibleReordering.x:I
      80: return        
}
```

可以看出，确实是一个线程的run方法指令范围是 `0:10` ，另一个是 `0:80`，说明我们正确获取了相应指令。

完整的源代码，包含如何编译，运行，可以在我的GitHub中找到：[AgentDemo](https://github.com/minixalpha/Demo/tree/master/AgentDemo)
