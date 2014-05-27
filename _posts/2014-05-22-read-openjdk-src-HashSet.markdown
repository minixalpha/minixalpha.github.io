---
layout: default
title: OpenJDK 源码阅读之 HashSet
categories: [技术, Java, 源代码阅读]
comments: true
---

# OpenJDK 源代码阅读之 HashSet

---

## 概要

* 类继承关系

```
java.lang.Object
    java.util.AbstractCollection<E>
        java.util.AbstractSet<E>
            java.util.HashSet<E>
```

* 定义

```
public class HashSet<E>
extends AbstractSet<E>
implements Set<E>, Cloneable, Serializable
```

* 要点

1. 不保证元素次序，甚至不保证次序不随时间变化
2. 基本操作(add, remove, contains, size)常量时间
3. 迭代操作与当前元素个数加底层容量大小成正比
4. 不保证同步

## 思考

* 总体实现

底层是用 `HashMap` 实现的，`Set` 中的数据是 `HashMap` 的 `key`，所有的 `key` 指向同一个 `value`, 此 `value` 定义为：

```java
// Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
```

再看一下 `add`，大概就能明白了

```java
/**
 * Adds the specified element to this set if it is not already present.
 * More formally, adds the specified element <tt>e</tt> to this set if
 * this set contains no element <tt>e2</tt> such that
 * <tt>(e==null&nbsp;?&nbsp;e2==null&nbsp;:&nbsp;e.equals(e2))</tt>.
 * If this set already contains the element, the call leaves the set
 * unchanged and returns <tt>false</tt>.
 *
 * @param e element to be added to this set
 * @return <tt>true</tt> if this set did not already contain the specified
 * element
 */
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

* load factor

```java
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
```

初始化中，注意使用的 `HashMap` 的 load factor 设置为 0.75，如果太小，就设置成 16. 为什么要 0.75 呢？ 有什么依据吗？


`HashSet` 并没有什么特别之处，几乎没有自己特有的实现，都是调用 `HashMap` 的方法实现相应的功能。
