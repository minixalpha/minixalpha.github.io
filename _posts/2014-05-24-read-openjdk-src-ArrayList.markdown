---
layout: default
title: OpenJDK 源码阅读之 ArrayList
categories: [技术, Java, 源代码阅读]
comments: true
---

# OpenJDK 源码阅读之 ArrayList

---

## 概要

* 类继承关系 

```java
java.lang.Object
    java.util.AbstractCollection<E>
        java.util.AbstractList<E>
            java.util.ArrayList<E>
```

* 定义

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{
}
```


## 实现

* transient

```java
 private transient Object[] elementData;
```

声明为 `transient`后，这个字段不会被序列化。

* toArray

```java
    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        size = elementData.length;
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    }
```

注意对 `elementData` 的检查，[Bug 6260652](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6260652)中对此有详细描述。主要原因是 `c.toArray()` 不一定会返回　`Object[]` 类型的值。


* SuppressWarnings

```java
 @SuppressWarnings("unchecked")
                ArrayList<E> v = (ArrayList<E>) super.clone();
```

告诉编译器，对特定类型的 `warning` 保持静默。


* 参数检查

可以看出标准库中的程序，在很多地方都需要对参数进行检查，以保证程序的健壮性。

检查 `null`

```java
public int indexOf(Object o) {
    if (o == null) {
    } else {
    }
    return -1;
```

检查参数上界，下界

```java
 private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

* ArrayList 的 index 检查

```java
 @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }
```

```java
 public E get(int index) {
        rangeCheck(index);

        return elementData(index);
    }
```

```java
private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

注意　`rangeCheck` 只检查了上界，但是如果将　`index` 设置成负数，也会抛出异常，异常是在 `elementData[index]` 中抛出的，猜想是在数组的实现中，对负数进行检查，因为任何一个数组，`index` 都不可能为负数，但是在实现数组时，不知道数组的元素个数，所以上界检查在此时发生。

* 元素访问

```java
 @SuppressWarnings("unchecked")
    E elementData(int index) {
        return (E) elementData[index];
    }
```

专门写了一个函数用来访问元素，而不是直接使用 `elementData[index]`，只因为需要向上转型么？还是 `SuppressWarning`　会重复。

* private

对于仅仅在类内部使用的函数，要声明为 `private`。

* add 参数检查

```java
   public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```

```java
   private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

可以看出这里对　`index` 的上界和下界都检查了，虽然　`add` 的`7` 行会进行检查，但在 `add`　的 `4`, `5` 行中就已经可能出错。

* 强制垃圾回收

```java
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // Let gc do its work

        return oldValue;
    }
```

注意第 `11`行把最后一个元素设置为`null`，这可以使得`gc`工作。好奇如何用实验验证这一点。

* remove(Object o)

```java
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
```

整个框架与 `indexOf` 函数是相似的，注意那个 `fastRemove` 函数，它与 `remove(index)` 的不同在于它：

1. 是 `private`
2. 无参数检查，因为传给它的参数一定是合法的
3. 不返回值

由此细节可见，标准库中函数的精益求精。(不知道是不是我过度揣测了，有经过性能测试么？)


* batchRemove

```java
   private boolean batchRemove(Collection<?> c, boolean complement) {
        final Object[] elementData = this.elementData;
        int r = 0, w = 0;
        boolean modified = false;
        try {
            for (; r < size; r++)
                if (c.contains(elementData[r]) == complement)
                    elementData[w++] = elementData[r];
        } finally {
            // Preserve behavioral compatibility with AbstractCollection,
            // even if c.contains() throws.
            if (r != size) {
                System.arraycopy(elementData, r,
                                 elementData, w,
                                 size - r);
                w += size - r;
            }
            if (w != size) {
                for (int i = w; i < size; i++)
                    elementData[i] = null;
                modCount += size - w;
                size = w;
                modified = true;
            }
        }
        return modified;
    }
```

注意 `finally` 里的代码，这段代码保证，即使 `try` 中的代码出了问题，也会最大程度上保证数据的一致性。如果 `r` 没有遍历完，那么后面没有检查过的数据都要保留下来。

* 线程安全

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

注意那个　`modCount` 的检查，这是为了确定在 `5-12`　行代码执行过程中，`List` 没有改变。改变的原因可能是由于多线程并发执行，在这期间另一个线程执行，改变了 `List` 的状态。

* 容量扩充

容量扩充会在任何可能引起　`ArrayList` 大小改变的情况下发生，如何扩充呢，代码在 `grow` 函数中。

```java
   private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

可以看出，`oldCapacity` 新增的容量是它的一半。另外，还有一个 `hugeCapacity`，如果需要扩充的容量比　`MAX_ARRAY_SIZE` 还大，会调用这个函数，重新调整大小。但再大也大不过　`Integer.MAX_VALUE`。

* 元素位置调整

```java
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
    
        public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // Let gc do its work

        return oldValue;
    }
```

无论是增加元素还是删除元素，都可能使得很多元素的位置发生改变，这里就是用 `System.arraycopy` 来把大量元素放在其它位置，如果元素很多，经常需要调整，是很浪费时间的。
