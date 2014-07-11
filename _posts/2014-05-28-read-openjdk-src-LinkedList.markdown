---
layout: default
title: OpenJDK 源码阅读之 LinkedList
categories: [技术, Java, 源代码阅读]
comments: true
---

# OpenJDK 源码阅读之 LinkedList

---

## 概要

* 类继承关系

```
java.lang.Object
    java.util.AbstractCollection<E>
        java.util.AbstractList<E>
            java.util.AbstractSequentialList<E>
                java.util.LinkedList<E>
```


* 定义

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
}
```

* 要点

1. 双链表
2. 非同步
3. fail-fast: 即创建 iterator 后，对链表进行了添加，删除，但是却没有用 iterator 的方法，就会抛出 ConcurrentModificationException 异常。

## 实现

* Node

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

双链表每个结点由 `Node` 类表示, 由其成员可以看出，确实是一个双链表。

* add

```java
/**
 * Appends the specified element to the end of this list.
 *
 * <p>This method is equivalent to {@link #addLast}.
 *
 * @param e element to be appended to this list
 * @return {@code true} (as specified by {@link Collection#add})
 */
public boolean add(E e) {
    linkLast(e);
    return true;
}
```

`add` 操作会在表的最后添加一个元素，调用 `linkLast` 实现。


* LinkLast

```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

`linkLast` 先生成新的结点，再将原来的最后结点的 `next` 指向新结点，需要注意链表为空时的情况。另外，新结点的 `prev`, `next` 都是在其构造函数中设置的，所以需要将其相邻结点传入。构造函数为：

```java
   private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

* serialVersionUID

```java
    private static final long serialVersionUID = 876323262645176354L;
```

序列化版本号，如果前一版本序列化后，后一版本发生了很大改变，就使用这个号告诉虚拟机，不能反序列化了。

* writeObject

比例一下 `ArrayList` 与 `LinkedList` 中的 writeObject

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out array length
        s.writeInt(elementData.length);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++)
            s.writeObject(elementData[i]);

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }

    }
```

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out any hidden serialization magic
        s.defaultWriteObject();

        // Write out size
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (Node<E> x = first; x != null; x = x.next)
            s.writeObject(x.item);
    }
```

注意后者没有检查　`modCount`，这是为什么呢？之前看 `ArrayList`的时候觉得是为线程安全考虑的，可是现在为什么又不检查了呢？虽然两个文件的注释中都说到，如果有多个线程操作此数据结构，应该从外部进行同步。但是一个检查，一个不检查是几个意思呀？


* 维护数据结构一致性

```java
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;
        if (f == null)
            last = newNode;
        else
            f.prev = newNode;
        size++;
        modCount++;
    }
```

注意代码第 `5`行对 `f` 的检查，`6` 行对 `last` 的调整。一定要细心，保证操作后， `所有可能` 涉及的数据都得到相应更新。

* 隐藏实现

```java
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
```

注意返回的是数据，而不是`Node`，外部根本不需要知道 `Node` 的存在。
另外，为什么 `f == null` 要抛出异常而不是返回 `null`？


* 为什么要分成两个函数　

```java
    private E unlinkFirst(Node<E> f) {
        // assert f == first && f != null;
        final E element = f.item;
        final Node<E> next = f.next;
        f.item = null;
        f.next = null; // help GC
        first = next;
        if (next == null)
            last = null;
        else
            next.prev = null;
        size--;
        modCount++;
        return element;
    }
    
    public E removeFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return unlinkFirst(f);
    }
```

删除操作分成两个函数，这是为什么呢？还有其它的一些操作也是这样。能想到的是其它操作可能也需要用到 `unlinkFirst`。

* LinkedList 中以 index 检索

```java
   Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

可以看出这里的小技巧，以 `index` 在前半段还是后半段，来决定是从前向后搜索，还是从后向前。

* 代码重复问题

```java
    public E getFirst() {
        final Node<E> f = first;
        if (f == null)
            throw new NoSuchElementException();
        return f.item;
    }
    
    public E peek() {
        final Node<E> f = first;
        return (f == null) ? null : f.item;
    }
```

好奇这两个函数为什么会同时存在，Google到，原来是为了实现不同的接口，所以需要同时存在这两个函数，类似的情况还存在。

* DescendingIterator 

```java
   private class DescendingIterator implements Iterator<E> {
        private final ListItr itr = new ListItr(size());
        public boolean hasNext() {
            return itr.hasPrevious();
        }
        public E next() {
            return itr.previous();
        }
        public void remove() {
            itr.remove();
        }
    }
```

这个很有意思，直接把 `next`, `hasNext`　函数设置为 `previous` 就行了，很大程度上减少了代码。
