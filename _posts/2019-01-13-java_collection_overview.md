---
layout: article
title: Java集合概览
key: 2019-01-13-java_collection_overview
tags: 
    - Java
    - Collection
---

> JDK1.8版本，包括层次关系图、fail-fast机制和Set的实现等介绍

<!--more-->

## 层次关系图
![image](https://ws3.sinaimg.cn/large/7dfacda1ly1fzq1e5q5khj21mq0q5q5t.jpg)

## fail-fast机制
A fail-fast system is nothing but immediately report any failure that is likely to lead to failure. When a problem occurs, a fail-fast system fails immediately.
In Java, we can find this behavior with iterators. In case, you have called iterator on a collection object, and another thread tries to modify the collection object, then concurrent modification exception will be thrown. This is called fail-fast.

fail-fast机制是指在迭代遍历集合的时候，另一线程对集合进行修改操作，就会抛出ConcurrentModificationException异常。

符合fail-fast机制的集合类包括：ArrayList, LinkedList, Vector, HashMap, LinkedHashMap, TreeMap, Hashtable, HashSet, LinkedHashSet, TreeSet。

下面以ArrayList为例，说明fail-fast机制的实现原理：

ArrayList继承自AbstractList，包含一个整型变量modCount，每次修改集合的操作(包括新增、删除等)，都会使modCount的值加1。在迭代前，会将当前modCount的值赋给expectedModCount，然后每迭代一个元素都会将expectedModCount与modCount做比较，如果两者不一致，说明有其他线程修改了集合，抛出ConcurrentModificationException异常。
```java
    private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        // 存储迭代前的modCount值
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            // 检查expectedModCount和modCount是否相等
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

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            // 检查expectedModCount和modCount是否相等
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

需要注意的是，fail-fast机制并不保证一定会发生，在多线程的环境下，只是会尽最大的努力去抛出ConcurrentModificationException异常。因此这种机制仅可用于检测bug，真正保证线程安全还需使用java.util.concurrent包下面的相关集合类。

## Set实现的本质
分别看下HashSet和TreeSet的代码：
```java
    private static final Object PRESENT = new Object();
    /** HashSet **/
    public HashSet() {
        map = new HashMap<>();
    }

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

    /** TreeSet **/
    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    private transient NavigableMap<E,Object> m;

    public boolean add(E e) {
        return m.put(e, PRESENT)==null;
    }
```
可以看到HashSet底层基于HashMap，TreeSet底层基于TreeMap，Map中的key用来存储Set中的数据，value传入的是一个空的Object对象。所以理解了Map的实现原理，也就能理解对应Set的底层实现。