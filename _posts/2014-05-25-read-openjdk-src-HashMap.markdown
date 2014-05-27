---
layout: default
title: OpenJDK 源码阅读之 HashMap
categories: [技术, Java, 源代码阅读]
comments: true
---

# OpenJDK 源代码阅读之 HashMap

---

## 概要

* 类继承关系

```
java.lang.Object
    java.util.AbstractMap<K,V>
        java.util.TreeMap<K,V>
```

* 定义 

```
public class TreeMap<K,V>
extends AbstractMap<K,V>
implements NavigableMap<K,V>, Cloneable, Serializable
```

* 要点

1) 与 Hashtable 区别在于：非同步，允许 `null`
2) 不保证次序，甚至不保证次序随时间不变
3) 基本操作 put, get 常量时间
4) 遍历操作 与 capacity+size 成正比
5) HashMap 性能与 `capacity` 和 `load factor` 相关，`load factor` 是当前元素个数与 `capacity` 的比值，通常设定为 `0.75`，如果此值过大，空间利用率高，但是冲突的可能性增加，因而可能导致查找时间增加，如果过小，反之。当元素个数大于 `capacity * load_factor` 时，`HashMap` 会重新安排 Hash 表。因此高效地使用 `HashMap` 需要预估元素个数，设置最佳的 `capacity` 和 `load factor` ，使得重新安排 Hash 表的次数下降。


## 实现

* capacity

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);

    // Find a power of 2 >= initialCapacity
    int capacity = 1;
    while (capacity < initialCapacity)
        capacity <<= 1;

    this.loadFactor = loadFactor;
    threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    useAltHashing = sun.misc.VM.isBooted() &&
            (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
    init();
}
```

注意，`HashMap` 并不会按照你指定的 `initialCapacity` 来确定 `capacity` 大小，而是会找到一个比它大的数，并且是 `2的n次方`。

> 为什么要是 2 的n次方呢？



* hash 

```java
/**
 * Retrieve object hash code and applies a supplemental hash function to the
 * result hash, which defends against poor quality hash functions.  This is
 * critical because HashMap uses power-of-two length hash tables, that
 * otherwise encounter collisions for hashCodes that do not differ
 * in lower bits. Note: Null keys always map to hash 0, thus index 0.
 */
final int hash(Object k) {
    int h = 0;
    if (useAltHashing) {
        if (k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }
        h = hashSeed;
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```


如果 `k` 是 `String` 类型，使用了特别的 `hash` 函数，否则首先得到 `hashCode`，然后又对 `h` 作了移位，异或操作，问题：

> 为什么这里要作移位，异或操作呢？

```
at 22: 
h = abcdefgh
h1 = h >>> 20 = 00000abc
h2 = h >>> 12 = 000abcde
h3 = h1 ^ h2 = [0][0][0][a][b][a^c][b^d][c^e]
h4 = h ^ h3 = [a][b][c][a^d][b^e][a^c^f][b^d^g][c^e^h]
h5 = h4 >>> 4 = [0][a][b][c][a^d][b^e][a^c^f][b^d^g]
h6 = h4 >>> 7 = ([0][:3])[0][0][a][b][c][a^d][b^e][a^c^f]([a^c^f][0])
h7 = h4 ^ h6 = 太凶残了。。。
```

* put

```java
/**
 * Associates the specified value with the specified key in this map.
 * If the map previously contained a mapping for the key, the old
 * value is replaced.
 *
 * @param key key with which the specified value is to be associated
 * @param value value to be associated with the specified key
 * @return the previous value associated with <tt>key</tt>, or
 *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
 *         (A <tt>null</tt> return can also indicate that the map
 *         previously associated <tt>null</tt> with <tt>key</tt>.)
 */
public V put(K key, V value) {
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```

从 `put` 其实可以看出各个 `hash` 表是如何实现的，首先取得 `hash` 值，然后由 `indexFor` 找到链表头的 `index`，然后开始遍历链表，如果链表里的一个元素 `hash` 值与当前 `key` 的 `hash` 值相同，或者元素 `key` 的引用与当前 `key` 相同，或者 `equals` 相同，就说明当前 `key` 已经在 `hash` 表里了，那么修改它的值，返回旧值。

如果不在表里，会调用 `addEntry`，将这一 `(key, value)` 对添加进去。

```java
/**
 * Adds a new entry with the specified key, value and hash code to
 * the specified bucket.  It is the responsibility of this
 * method to resize the table if appropriate.
 *
 * Subclass overrides this to alter the behavior of put method.
 */
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

/**
 * Like addEntry except that this version is used when creating entries
 * as part of Map construction or "pseudo-construction" (cloning,
 * deserialization).  This version needn't worry about resizing the table.
 *
 * Subclass overrides this to alter the behavior of HashMap(Map),
 * clone, and readObject.
 */
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

可以看出，新增加元素时，可能会调整 `hash` 表的大小，原因之前已经讨论过。直接的添加在 `createEntry` 中完成，但是这里并没有体现出如何处理冲突。


```java
Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k;
    hash = h;
}
```

注意这里，将 `n` 赋值给了 `next`，这其实就是将新添加的项指向了当前链表头。这一操作在 `Entry` 的构造函数中完成。

`put` 操作的基本思路在到这里已经很清楚了，有了这个思路，不难想象 `get` 是如何动作的。

```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}

final Entry<K,V> getEntry(Object key) {
    int hash = (key == null) ? 0 : hash(key);
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

和 `put` 差不多，只是找到了就会返回相应的 `value` ，找不到就返回 `null`。
