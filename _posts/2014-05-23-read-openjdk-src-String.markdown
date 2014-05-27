---
layout: default
title: OpenJDK 源码阅读之 String
categories: [技术, Java, 源代码阅读]
comments: true
---

# OpenJDK 源代码阅读之 String

---

## 概要

* 类继承关系

```
java.lang.Object
    java.lang.String
```

* 定义

```java
public final class String
extends Object
implements Serializable, Comparable<String>, CharSequence
```

* 要点

一旦创建就不可改变

## 实现

* storage

```java
/** The value is used for character storage. */
private final char value[];
```

可以看出 `String` 中的数据是如何存储的。

* 初始化

```java
    public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
```

可以看出使用 `String` 类型初始化，新 `String` 实际上与原来的 `String` 指向同一块内存。

```java
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}
```

如果用 `char[]` 初始化，可以看出，新分配了内存，并复制，保证了两者相互独立，只是内容相同。

```java
    public String(StringBuffer buffer) {
        synchronized(buffer) {
            this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
        }
    }
```

注意用 `StringBuffer` 初始化时，对同一 `buffer` 是线程安全的，即初始化 `String` 的过程中，其它线程不会改变 `buffer` 的内容。

另外，能告诉我下面这段代码是怎么回事么？

```java
   public String(StringBuilder builder) {
        this.value = Arrays.copyOf(builder.getValue(), builder.length());
    }
```

为啥这次不同步了呢？


* equals 

```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String anotherString = (String) anObject;
        int n = value.length;
        if (n == anotherString.value.length) {
            char v1[] = value;
            char v2[] = anotherString.value;
            int i = 0;
            while (n-- != 0) {
                if (v1[i] != v2[i])
                        return false;
                i++;
            }
            return true;
        }
    }
    return false;
}
```

注意：

1) 检查类型
2) `value` 直接通过点访问了，`value` 是 `private` 的啊，怎么能这样？


* hashCode 

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && value.length > 0) {
        char val[] = value;

        for (int i = 0; i < value.length; i++) {
            h = 31 * h + val[i];
        }
        hash = h;
    }
    return h;
}
```

`String` 的 `hashCode` 公式：

> s[0]\*31^(n-1) + s[1]\*31^(n-2) + ... + s[n-1]

* replace

```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

        while (++i < len) {
            if (val[i] == oldChar) {
                break;
            }
        }
        if (i < len) {
            char buf[] = new char[len];
            for (int j = 0; j < i; j++) {
                buf[j] = val[j];
            }
            while (i < len) {
                char c = val[i];
                buf[i] = (c == oldChar) ? newChar : c;
                i++;
            }
            return new String(buf, true);
        }
    }
    return this;
}
```

从中可以看出，虽然说是 `replace`，但是实际上还是新生成了 `buf` ，然后再生成新的 `String`，而不是在原来的 `value` 上修改。如果有大量的替换，还是自己实现比较好诶～


* indexOf

```java
/**
 * Code shared by String and StringBuffer to do searches. The
 * source is the character array being searched, and the target
 * is the string being searched for.
 *
 * @param   source       the characters being searched.
 * @param   sourceOffset offset of the source string.
 * @param   sourceCount  count of the source string.
 * @param   target       the characters being searched for.
 * @param   targetOffset offset of the target string.
 * @param   targetCount  count of the target string.
 * @param   fromIndex    the index to begin searching from.
 */
static int indexOf(char[] source, int sourceOffset, int sourceCount,
        char[] target, int targetOffset, int targetCount,
        int fromIndex) {
    if (fromIndex >= sourceCount) {
        return (targetCount == 0 ? sourceCount : -1);
    }
    if (fromIndex < 0) {
        fromIndex = 0;
    }
    if (targetCount == 0) {
        return fromIndex;
    }

    char first = target[targetOffset];
    int max = sourceOffset + (sourceCount - targetCount);

    for (int i = sourceOffset + fromIndex; i <= max; i++) {
        /* Look for first character. */
        if (source[i] != first) {
            while (++i <= max && source[i] != first);
        }

        /* Found first character, now look at the rest of v2 */
        if (i <= max) {
            int j = i + 1;
            int end = j + targetCount - 1;
            for (int k = targetOffset + 1; j < end && source[j]
                    == target[k]; j++, k++);

            if (j == end) {
                /* Found whole string. */
                return i - sourceOffset;
            }
        }
    }
    return -1;
}
```

这段代码从 `source` 中寻找 `target` 第一次出现的位置，`for` 循环每次都先让 `i` 停留在一个位置，此位置上内容与 `target` 首字符相同，然后开始遍历。可以看出这是一个 `O(n^2)` 的算法，所以，标准库也不一定是最高效的，要是要高效，还是需要自己实现，或者找其它库的。

* matches

```java
public boolean matches(String regex) {
    return Pattern.matches(regex, this);
}
```

正则表达式匹配函数。可以看出，是直接调用了 `Pattern` 中的相应函数。

```java
public String[] split(String regex, int limit) {
    /* fastpath if the regex is a
     (1)one-char String and this character is not one of the
        RegEx's meta characters ".$|()[{^?*+\\", or
     (2)two-char String and the first char is the backslash and
        the second is not the ascii digit or ascii letter.
     */
    char ch = 0;
    if (((regex.value.length == 1 &&
         ".$|()[{^?*+\\".indexOf(ch = regex.charAt(0)) == -1) ||
         (regex.length() == 2 &&
          regex.charAt(0) == '\\' &&
          (((ch = regex.charAt(1))-'0')|('9'-ch)) < 0 &&
          ((ch-'a')|('z'-ch)) < 0 &&
          ((ch-'A')|('Z'-ch)) < 0)) &&
        (ch < Character.MIN_HIGH_SURROGATE ||
         ch > Character.MAX_LOW_SURROGATE))
    {
        int off = 0;
        int next = 0;
        boolean limited = limit > 0;
        ArrayList<String> list = new ArrayList<>();
        while ((next = indexOf(ch, off)) != -1) {
            if (!limited || list.size() < limit - 1) {
                list.add(substring(off, next));
                off = next + 1;
            } else {    // last one
                //assert (list.size() == limit - 1);
                list.add(substring(off, value.length));
                off = value.length;
                break;
            }
        }
        // If no match was found, return this
        if (off == 0)
            return new String[]{this};

        // Add remaining segment
        if (!limited || list.size() < limit)
            list.add(substring(off, value.length));

        // Construct result
        int resultSize = list.size();
        if (limit == 0)
            while (resultSize > 0 && list.get(resultSize - 1).length() == 0)
                resultSize--;
        String[] result = new String[resultSize];
        return list.subList(0, resultSize).toArray(result);
    }
    return Pattern.compile(regex).split(this, limit);
}
```

按 `regex` 将字符串分割，思路是如果是单个字符，或者转义字符，就手工分割，否则就直接调用 `Pattern.comile(regex).split` 函数。手工分割时每次都将 `[off, next]` 之间的内容加入 `list`，最后将 剩余的 `[off, ]`  加入。另外注意 `limit` 对分割次数的限制。
