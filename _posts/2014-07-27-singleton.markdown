---
layout: default
title: 单例模式
categories: [技术, 设计模式]
comments: true
---

# 单例模式

---

* 定义 

单例模式确保一个类只有一个实例，并提供一个全局访问点

* 解释

从定义可以看出，特点是这个类只有一个实例。那么，为什么要这么做呢？原因在于，有些时候，这个类只有一个实例会节约资源，或者只有一个实例才能保证整个程序运行正确，一致。例如：线程池，缓存，对话框，日志对象等等 。


* 示例

```java
class Singleton {
	private static Singleton singleton;

	private Singleton() {
	}

	public static Singleton getInstance() {
		if (singleton == null) {
			singleton = new Singleton();
		}
		return singleton;
	}
}
```

这是单例的经典使用方式：

* 一个 private static 对象
* 构造器设置为 private
* 一个 public static 方法提供全局访问点 

开始的时候，其实我比较困惑为什么不在 singleton 声明处直接实例化对象，后来明白了，这是一种延迟实例化的手段，保证只在需要时才实例化。如果直接在声明时实例化，那么只要类加载了，即使不需要对象，也会对它进行实例化。

另外，这个经典使用方式其实是有问题的，对比后面 Tomcat 中的应用场景，你可能会发现问题所在。

在 Tomcat 中，就有一个单例模式，它是 `org.apache.catalina.tribes.util.StringManager` 类。在 Tomcat 中，会有许多地方需要对错误消息进行处理。我们使用 `StringManager` 类来管理这些错误消息。

错误消息首先不能硬编码到代码中，否则需要提供国际化支持时，就会很痛苦。错误消息需要定义在配置文件中。Tomcat 为每个包 (package) 都提供了三种语言的错误消息配置文件。每个包内都有很多类，我们没必要为每个类都生成一个 `StringManager` 类，因为它们共享同一个配置文件。所以，一个包只需要一个 `StringManager` 对象就好了。我们怎么为一个包生成一个 `StringManager` 呢？ Tomcat 在 `StringManager` 内部保存着所有包的 `StringManager` 实例，你需要一个实例时，只需要提供包名，调用 `StringManager` 的相应方法，就会返回与此包名对应的 `StringManager` 实例。下面是相关的代码，一目了然。

```java

public class StringManager { 

private StringManager(String packageName) {
    ...
}

private static final Hashtable<String, StringManager> managers =
new Hashtable<>();

/**
* Get the StringManager for a particular package. If a manager for
* a package already exists, it will be reused, else a new
* StringManager will be created and returned.
*
* @param packageName The package name
*/
public static final synchronized StringManager getManager(String packageName) {
    StringManager mgr = managers.get(packageName);
    if (mgr == null) {
        mgr = new StringManager(packageName);
        managers.put(packageName, mgr);
    }
    return mgr;
}
```

使用了一个 `Hashtable` 来保存 `managers`，每次通过 `getManager` 方法，通过包名访问，如果访问不到，就为此包新生成一个 `StringManager` 实例。

有没有注意到 getManager 方法被  `synchronized` 修饰？这就是之前我们举的经典示例时说的问题。在使用单例时，只有一个对象，这个对象可能是被多个线程共享的。如果不同步，就可能会出现数据不一致的情况。例如，两个线程同时调用了 `getManager`，访问同一个包名。正确的执行是，其中一个线程第一次调用时，mgr 为空，此时生成一个 StringManager。第二个线程调用时，mgr 就不为空了。但是，如果不同步，那么当第一个线程通过了 `if(mgr == null)` 时，此时线程被切换了，这时，第二个线程也会通过 `if(mgr == null)` ，这样就导致同一个包，生成了两个 `StringManager`。

* 扩展阅读 

关于单例模式与线程安全，建议阅读一下这篇文章：

[深入浅出单实例Singleton设计模式](http://blog.csdn.net/haoel/article/details/4028232)

关于单例模式的其它实现方式，可以阅读 

[单例模式的七种写法](http://cantellow.iteye.com/blog/838473)
