---
layout: default
title: OpenJDK 源码阅读之 TreeMap
categories: [技术, Java, 源代码阅读]
comments: true
---

# OpenJDK 源代码阅读之 TreeMap

---

## 概要

* 类继承关系

```
java.lang.Object
    java.util.AbstractMap<K,V>
        java.util.HashMap<K,V>
```

* 定义 

```
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable
```

* 要点

1) 基于 NavigableMap 实现的红黑树
2) 按 `natrual ordering` 或者 `Comparator` 定义的次序排序。
3) 基本操作 `containsKey`,`get`,`put` 有 `log(n)` 的时间复杂度。
4) 非线程安全


## 实现

* Comparator

```java
private final Comparator<? super K> comparator;
```

这说明提供的 `Comparator` 参数类型是 `K` 的基类就行。这似乎意味着基类的 `Comparator` 与导出类的要一致。

* Entry

```java
private transient Entry<K,V> root = null;
```

`root` 是这棵红黑树的根，那么从 `Entry` 的定义可以体现树的结构：

```java
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left = null;
    Entry<K,V> right = null;
    Entry<K,V> parent;
    boolean color = BLACK;
    
    ...
}
```

注意这是个 `static final` 类，`left`,`right`,`parent` 分别指向左子树，右子树，父结点, `color` 颜色默认为黑。

* containsKey

```java
public boolean containsKey(Object key) {
    return getEntry(key) != null;
}
```

```java
final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if (comparator != null)
        return getEntryUsingComparator(key);
    if (key == null)
        throw new NullPointerException();
    Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = k.compareTo(p.key);
        if (cmp < 0)
            p = p.left;
        else if (cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

关键操作 `containsKey` 是通过调用 `getEntry` 完成其功能的。可以看出，这是通过在红黑树上进行查找完成的，每次比较都会下降到树的下一层，由于红黑树的平衡性，时间复杂度为 `log(n)`。

* containsValue

```java
public boolean containsValue(Object value) {
    for (Entry<K,V> e = getFirstEntry(); e != null; e = successor(e))
        if (valEquals(value, e.value))
            return true;
    return false;
}
```

可以看出，对 `value` 的查找，与对 `key` 是不同的。是通过 `getFirstEntry` 取得第一个结点，再通过 `successor` 遍历实现。

```java
final Entry<K,V> getFirstEntry() {
    Entry<K,V> p = root;
    if (p != null)
        while (p.left != null)
            p = p.left;
    return p;
}
```

可以看出，这是找到了树中最左边的结点，如果左子树中的值小于右子树，这就意味是第一个比较的结点是最小的结点。

```java
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if (t == null)
        return null;
    else if (t.right != null) {
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else {
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while (p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```

`successor` 其实就是一个结点的 `下一个结点`，所谓 `下一个`，是按次序排序后的下一个结点。从代码中可以看出，如果右子树不为空，就返回右子树中最小结点。如果右子树为空，就要向上回溯了。在这种情况下，`t` 是以其为根的树的最后一个结点。如果它是其父结点的左孩子，那么父结点就是它的下一个结点，否则，`t` 就是以其父结点为根的树的最后一个结点，需要再次向上回溯。一直到 `ch` 是 `p` 的左孩子为止。


* getCeilingEntry

```java
/**
 * Gets the entry corresponding to the specified key; if no such entry
 * exists, returns the entry for the least key greater than the specified
 * key; if no such entry exists (i.e., the greatest key in the Tree is less
 * than the specified key), returns {@code null}.
 */
final Entry<K,V> getCeilingEntry(K key) {
    Entry<K,V> p = root;
    while (p != null) {
        int cmp = compare(key, p.key);
        if (cmp < 0) {
            if (p.left != null)
                p = p.left;
            else
                return p;
        } else if (cmp > 0) {
            if (p.right != null) {
                p = p.right;
            } else {
                Entry<K,V> parent = p.parent;
                Entry<K,V> ch = p;
                while (parent != null && ch == parent.right) {
                    ch = parent;
                    parent = parent.parent;
                }
                return parent;
            }
        } else
            return p;
    }
    return null;
}
```

这个函数看起来有点奇怪，可以从 `return` 语句，猜想一下这是在做什么，我觉得是在找一个 `x.key >= key` 的元素，并且 `x` 是满足条件的元素中最小的。从 `23` 行看，找到的 `p`，`key` 值与参数 `key` 相等，从 `8` 行看，又有 `key <= p.key`，并且 `p.left == null`。

`put`

```java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 *
 * @return the previous value associated with {@code key}, or
 *         {@code null} if there was no mapping for {@code key}.
 *         (A {@code null} return can also indicate that the map
 *         previously associated {@code null} with {@code key}.)
 * @throws ClassCastException if the specified key cannot be compared
 *         with the keys currently in the map
 * @throws NullPointerException if the specified key is null
 *         and this map uses natural ordering, or its comparator
 *         does not permit null keys
 */
public V put(K key, V value) {
    Entry<K,V> t = root;
    if (t == null) {
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

`put` 的过程，其实是将 `(key, value)` 加入到红黑树中的过程。如果树是空的，那么创建根结点。否则就要在树中插入结点。这个过程根据 `comparator` 是否存在设置成了两种方式，其实没什么区别，就是比较方式的不同，都是在树中查找一个合适的位置，如果 `key` 在树中，就 `setValue` 设置新值，否则，就在 `60-64` 行插入新结点。这里有个重要的地方是 `65` 行的 `fixAfterInsertion` ，这个很重要，因为这是一棵红黑树，红黑树关键的思想是要保持它的平衡性，插入结点后，平衡性可能被破坏。所以需要 `fix`。

```java
/** From CLR */
private void fixAfterInsertion(Entry<K,V> x) {
    x.color = RED;

    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            Entry<K,V> y = rightOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == rightOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateLeft(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK;
}
```

这是 `fixAfterInsertion` 的源代码，我不具体解释了，这是一个根据不同情况进行旋转，调整结点颜色的过程，可以参考《算法导论》中的解释。


* remove

```java
public V remove(Object key) {
    Entry<K,V> p = getEntry(key);
    if (p == null)
        return null;

    V oldValue = p.value;
    deleteEntry(p);
    return oldValue;
}
```

删除的过程主要调用 `delteEntry` 完成：


```java
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;

    // If strictly internal, copy successor's element to p and then make p
    // point to successor.
    if (p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } // p has 2 children

    // Start fixup at replacement node, if it exists.
    Entry<K,V> replacement = (p.left != null ? p.left : p.right);

    if (replacement != null) {
        // Link replacement to parent
        replacement.parent = p.parent;
        if (p.parent == null)
            root = replacement;
        else if (p == p.parent.left)
            p.parent.left  = replacement;
        else
            p.parent.right = replacement;

        // Null out links so they are OK to use by fixAfterDeletion.
        p.left = p.right = p.parent = null;

        // Fix replacement
        if (p.color == BLACK)
            fixAfterDeletion(replacement);
    } else if (p.parent == null) { // return if we are the only node.
        root = null;
    } else { //  No children. Use self as phantom replacement and unlink.
        if (p.color == BLACK)
            fixAfterDeletion(p);

        if (p.parent != null) {
            if (p == p.parent.left)
                p.parent.left = null;
            else if (p == p.parent.right)
                p.parent.right = null;
            p.parent = null;
        }
    }
}
```

这个过程同样是与红黑树的性质相关的。在树中删除一个结点，那他的孩子怎么办啊，红黑树不平衡了怎么办啊，树空了怎么办啊，都需要考虑到。

* clear

```java
public void clear() {
    modCount++;
    size = 0;
    root = null;
}
```

居然就是把值都清空了。。

* clone

```java
/**
 * Returns a shallow copy of this {@code TreeMap} instance. (The keys and
 * values themselves are not cloned.)
 *
 * @return a shallow copy of this map
 */
public Object clone() {
    TreeMap<K,V> clone = null;
    try {
        clone = (TreeMap<K,V>) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new InternalError();
    }

    // Put clone into "virgin" state (except for comparator)
    clone.root = null;
    clone.size = 0;
    clone.modCount = 0;
    clone.entrySet = null;
    clone.navigableKeySet = null;
    clone.descendingMap = null;

    // Initialize clone with our mappings
    try {
        clone.buildFromSorted(size, entrySet().iterator(), null, null);
    } catch (java.io.IOException cannotHappen) {
    } catch (ClassNotFoundException cannotHappen) {
    }

    return clone;
}
```

`clone` 复制了一份新的元素，使用了 `super.clone()` 得的一个新对象，而不是使用 `new`，这是为啥？然后把各个域值清空，然后使用 `buildFromSorted` 插入数据。之所以使用这个函数，是因为红黑树插入操作时间复杂度为 `O(lgn)`，n个元素插入就是 `O(n*lgn)`，太不划算，更何况我们现在插入的是一个红黑树，所以用一个线性时间复杂度的算法来实现复制数据的操作。

```java
/**
 * Linear time tree building algorithm from sorted data.  Can accept keys
 * and/or values from iterator or stream. This leads to too many
 * parameters, but seems better than alternatives.  The four formats
 * that this method accepts are:
 *
 *    1) An iterator of Map.Entries.  (it != null, defaultVal == null).
 *    2) An iterator of keys.         (it != null, defaultVal != null).
 *    3) A stream of alternating serialized keys and values.
 *                                   (it == null, defaultVal == null).
 *    4) A stream of serialized keys. (it == null, defaultVal != null).
 *
 * It is assumed that the comparator of the TreeMap is already set prior
 * to calling this method.
 *
 * @param size the number of keys (or key-value pairs) to be read from
 *        the iterator or stream
 * @param it If non-null, new entries are created from entries
 *        or keys read from this iterator.
 * @param str If non-null, new entries are created from keys and
 *        possibly values read from this stream in serialized form.
 *        Exactly one of it and str should be non-null.
 * @param defaultVal if non-null, this default value is used for
 *        each value in the map.  If null, each value is read from
 *        iterator or stream, as described above.
 * @throws IOException propagated from stream reads. This cannot
 *         occur if str is null.
 * @throws ClassNotFoundException propagated from readObject.
 *         This cannot occur if str is null.
 */
private void buildFromSorted(int size, Iterator it,
                             java.io.ObjectInputStream str,
                             V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    this.size = size;
    root = buildFromSorted(0, 0, size-1, computeRedLevel(size),
                           it, str, defaultVal);
}
```

好吧，我们继续。


```java
private final Entry<K,V> buildFromSorted(int level, int lo, int hi,
                                         int redLevel,
                                         Iterator it,
                                         java.io.ObjectInputStream str,
                                         V defaultVal)
    throws  java.io.IOException, ClassNotFoundException {
    /*
     * Strategy: The root is the middlemost element. To get to it, we
     * have to first recursively construct the entire left subtree,
     * so as to grab all of its elements. We can then proceed with right
     * subtree.
     *
     * The lo and hi arguments are the minimum and maximum
     * indices to pull out of the iterator or stream for current subtree.
     * They are not actually indexed, we just proceed sequentially,
     * ensuring that items are extracted in corresponding order.
     */

    if (hi < lo) return null;

    int mid = (lo + hi) >>> 1;

    Entry<K,V> left  = null;
    if (lo < mid)
        left = buildFromSorted(level+1, lo, mid - 1, redLevel,
                               it, str, defaultVal);

    // extract key and/or value from iterator or stream
    K key;
    V value;
    if (it != null) {
        if (defaultVal==null) {
            Map.Entry<K,V> entry = (Map.Entry<K,V>)it.next();
            key = entry.getKey();
            value = entry.getValue();
        } else {
            key = (K)it.next();
            value = defaultVal;
        }
    } else { // use stream
        key = (K) str.readObject();
        value = (defaultVal != null ? defaultVal : (V) str.readObject());
    }

    Entry<K,V> middle =  new Entry<>(key, value, null);

    // color nodes in non-full bottommost level red
    if (level == redLevel)
        middle.color = RED;

    if (left != null) {
        middle.left = left;
        left.parent = middle;
    }

    if (mid < hi) {
        Entry<K,V> right = buildFromSorted(level+1, mid+1, hi, redLevel,
                                           it, str, defaultVal);
        middle.right = right;
        right.parent = middle;
    }

    return middle;
}
```

可以看出这是一个递归的算法，找出中间结点，构造左右子树，并将他们连接在一起。时间复杂度分析可以根据递归式：

> T(n) = 2 * T(n/2) + C

如果看不出来的话，可以使用《算法导论》第4章中的主定理，可以得到时间复杂度为 `O(n)` 。
