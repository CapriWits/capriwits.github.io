---
title: "Iterator remove 时出现 ConcurrentModificationException"
slug: "Iterator-ConcurrentModificationException-problem-when-remove"
date: 2021-03-15T20:10:31+08:00
author: CapriWits
image: java-programming-cover.png
tags:
  - Java
categories:
  - Backend
hidden: false
math: true
comments: true
draft: false
---
# Iterator remove 时出现 ConcurrentModificationException

![](https://img.shields.io/badge/JDK-1.8-red)

## 前言

```java
import java.util.*;

public class Solution {
    public static void main(String[] args) {
        List<String> arrayList = new ArrayList<String>();
        arrayList.add("a");
        arrayList.add("b");
        arrayList.add("c");
        arrayList.add("d");

        Iterator<String> iterator = arrayList.iterator();
        while (iterator.hasNext()) {
            String cur = iterator.next();
            if ("b".equals(cur)) {
                arrayList.remove(cur);
            } else {
                System.out.println(cur + " ");
            }
        }
        /*for (String s : arrayList) {
            if ("b".equals(s)) {
                arrayList.remove(s);
            } else {
                System.out.println(s + " ");
            }
        }*/
        System.out.println(arrayList);
    }
}
```

- for-each 实际就是隐式使用 iterator 遍历集合，上面的例子会抛出异常，并删除失败。

> a  
> Exception in thread "main" java.util.ConcurrentModificationException  
> at java.base/java.util.ArrayList$Itr.checkForComodification(ArrayList.java:937)  
> at java.base/java.util.ArrayList$Itr.next(ArrayList.java:891)  
> at Solution.main(Solution.java:14)  

- 然而删除 **倒数第二个** 元素却不会报错

```java
import java.util.*;

public class Solution {
    public static void main(String[] args) {
        List<String> arrayList = new ArrayList<>();
        arrayList.add("a");
        arrayList.add("b");
        arrayList.add("c");
        arrayList.add("d");

        Iterator<String> iterator = arrayList.iterator();
        while (iterator.hasNext()) {
            String cur = iterator.next();
            if ("c".equals(cur)) {
                arrayList.remove(cur);
            } else {
                System.out.println(cur + " ");
            }
        }
        /*for (String s : arrayList) {
            if ("c".equals(s)) {
                arrayList.remove(s);
            } else {
                System.out.println(s + " ");
            }
        }*/
        System.out.println(arrayList);
    }
}
```

> a  
> b  
> [a, b, d]  

## 分析

- 首先先观察 ArrayList 的 `iterator()`，看迭代器怎么构造。
- ArrayList 的 父类 `AbstractList` 中

```java
public Iterator<E> iterator() {
    return new Itr();
}
```

- `Itr` 是里面的内部类

```java
private class Itr implements Iterator<E> {
    /**
     * Index of element to be returned by subsequent call to next.
     */
    int cursor = 0;

    /**
     * Index of element returned by most recent call to next or
     * previous.  Reset to -1 if this element is deleted by a call
     * to remove.
     */
    int lastRet = -1;

    /**
     * The modCount value that the iterator believes that the backing
     * List should have.  If this expectation is violated, the iterator
     * has detected concurrent modification.
     */
    int expectedModCount = modCount;

    public boolean hasNext() {
        return cursor != size();
    }

    public E next() {
        checkForComodification();
        try {
            int i = cursor;
            E next = get(i);
            lastRet = i;
            cursor = i + 1;
            return next;
        } catch (IndexOutOfBoundsException e) {
            checkForComodification();
            throw new NoSuchElementException();
        }
    }

    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            AbstractList.this.remove(lastRet);
            if (lastRet < cursor)
                cursor--;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException e) {
            throw new ConcurrentModificationException();
        }
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

- `cursor`：下一个要访问的元素的索引
- `lastRet`：上一个访问的元素的索引
- `expectedModCount` 是期望的该 List 被修改的次数，初始化为 `modCount`
- `modCount` 是 `AbstractList` 的一个成员变量。

> The number of times this list has been structurally modified. Structural modifications are those that change the size of the list, or otherwise perturb it in such a fashion that iterations in progress may yield incorrect results.  
> This field is used by the iterator and list iterator implementation returned by the iterator and listIterator methods. If the value of this field changes unexpectedly, the iterator (or list iterator) will throw a ConcurrentModificationException in response to the next, remove, previous, set or add operations. This provides fail-fast behavior, rather than non-deterministic behavior in the face of concurrent modification during iteration.  
> Use of this field by subclasses is optional. If a subclass wishes to provide fail-fast iterators (and list iterators), then it merely has to increment this field in its add(int, E) and remove(int) methods (and any other methods that it overrides that result in structural modifications to the list). A single call to add(int, E) or remove(int) must add no more than one to this field, or the iterators (and list iterators) will throw bogus ConcurrentModificationExceptions. If an implementation does not wish to provide fail-fast iterators, this field may be ignored.  

```java
protected transient int modCount = 0;
```

- 结构修改是指那些改变列表大小的修改，或者以某种方式扰乱列表，使得正在进行的迭代可能产生不正确的结果。
- 此字段由迭代器和 `listIterator`方法返回的迭代器和列表迭代器实现使用。如果此字段的值意外更改，迭代器（或列表迭代器）将抛出 `ConcurrentModificationException`以响应 `next`、`remove`、`previous`、`set` 或 `add` 操作。这提供了 **快速失败** 的行为。
- 深入 `ArrayList` 里观察 `next()`

```java
public E next() {
    checkForComodification();
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```

```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

- 抛出的 `ConcurrentModificationException` 异常是 `checkForComodification()` 抛出的。
- 条件是：`modCount != expectedModCount`

所以在 `add` `remove` 的过程中 `modCount` 会自增自减。如果用集合的 `remove`则 `List` 的 `modCount`减少一，而 Iterator 的 `expectedModCount`不变，就会抛出异常。

至于为什么倒数第二个元素删除不会报错，我们要先了解 Iterator 遍历的特点。

`while + iterator` 的组合是需要先判空 `hasNext()`，然后再 `next()`，最后才 `remove()`，否则会报错，可以自行实验，调换 `next` 和 `remove`。

因为要先 `next`，将游标 **越过** 当前的元素，然后再决定要怎么操作当前的（游标前面的）这个元素，即游标是插在 **当前元素** 和 **下一个元素** 的中间（可以这么理解）。

删除倒数第二个元素的时候，cursor 指向 `最后一个元素`，而此时删掉了倒数第二个元素后，cursor 和 `size()` 正好相等了，所以 `hasNext()` 返回 false，遍历结束，成功的删除了倒数第二个元素。

## 建议用法

- 一个原则是，尽量在遍历的过程中不要对原集合进行增删，容易改变原结构，可以用 `immutable` 的思想，重新封装一个集合。
- 要 remove() ，则要在 `iterator()` 上面来进行 `remove()`，因为 Iterator 迭代，就把操作权交给了 Iterator，就不要再用原集合进行操作了。

- 正确用法：

```java
import java.util.*;

public class Solution {
    public static void main(String[] args) {
        List<String> arrayList = new ArrayList<>();
        arrayList.add("a");
        arrayList.add("b");
        arrayList.add("c");
        arrayList.add("d");

        Iterator<String> iterator = arrayList.iterator();
        while (iterator.hasNext()) {
            String cur = iterator.next();
            if ("a".equals(cur)) {
                iterator.remove();
            } else {
                System.out.println(cur + " ");
            }
        }
        System.out.println(arrayList);
    }
}
```

> b  
> c  
> d  
> [b, c, d]

以上分析是基于 `ArrayList`，基于链表的 `LinkedList` 道理大同小异，思想不变，测试的结果也是不变的。