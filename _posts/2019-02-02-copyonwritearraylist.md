---
layout: article
title: Java集合之CopyOnWriteArrayList
key: 2019-02-02-copyonwritearraylist
tags: 
    - Java
    - Collection
---

> JDK1.8版本，CopyOnWriteArrayList核心源码分析

<!--more-->

CopyOnWriteArrayList利用读写分离的思想，在副本上执行写入操作，写入完成后，将原来的引用指向副本。这样一来，并发读的时候无需加锁，提高了集合容器的效率。

## 成员变量
```java
    /** 所有修改操作使用的锁 */
    final transient ReentrantLock lock = new ReentrantLock();

    /** 用volatile关键字修饰存储数据元素的数组，保证了内存可见性 */
    private transient volatile Object[] array;
```

## 核心方法
```java
    public boolean add(E e) {
        // 使用ReentrantLock加锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            // 将array拷贝至新的副本，长度加1
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            // 添加元素
            newElements[len] = e;
            // 将当前副本赋值给array
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }

    public E remove(int index) {
        // remove操作同样需要加锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            E oldValue = get(elements, index);
            int numMoved = len - index - 1;
            if (numMoved == 0)
                // 如果移除的是最后一个元素，直接将原来前len - 1 个元素拷贝至array即可
                setArray(Arrays.copyOf(elements, len - 1));
            else {
                // 将移除元素所在位置的前后两段数据拷贝至新副本，并赋值给array
                Object[] newElements = new Object[len - 1];
                System.arraycopy(elements, 0, newElements, 0, index);
                System.arraycopy(elements, index + 1, newElements, index,
                                 numMoved);
                setArray(newElements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }

    // get操作无需加锁
    public E get(int index) {
        return get(getArray(), index);
    }
```

## set方法中的happen-before问题
```java
    public E set(int index, E element) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            // 得到index位置上的旧元素
            E oldValue = get(elements, index);

            // 如果旧元素与要写入的数据不相等，写入新的数据
            if (oldValue != element) {
                int len = elements.length;
                Object[] newElements = Arrays.copyOf(elements, len);
                newElements[index] = element;
                setArray(newElements);
            } else {
                // Not quite a no-op; ensures volatile write semantics
                setArray(elements);
            }
            return oldValue;
        } finally {
            lock.unlock();
        }
    }
```
上面的set方法的中有一个细节值得注意，在else分支中，并没有对数据做实际的修改，只是将读取的elements又重新写回了array中。要理解这个看似毫无意义的操作，先看下面这个例子：
```java
// 线程1            // 线程2
a = 1;              list.get(0);
list.set(1,"t");    int b = a;
```
在上面的程序中，由于指令的重排序，不能保证b的取值最后一定是1。然而在[Oracle中的Java文档](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html)中有这样的描述：
> Actions in a thread prior to placing an object into any concurrent collection happen-before actions subsequent to the access or removal of that element from the collection in another thread.

翻译过来就是，线程中将一个对象放入并发集合之前的操作`happen-before`另一线程中的访问或移除该集合元素的后续操作。具体对于上面的例子，保证了set `happen-before` get，就保证了b最后的值是1。

对于CopyOnWriteArrayList，如何保证set `happen-before` get。对lock的解锁与加锁是符合`happen-before`原则的，但是get方法中并没有加锁操作。除此之外，volatile变量的写操作`happen-before`读操作。setArray方法中有对volatile变量array的写操作，只要保证if else分支中都有setArray方法，就保证了set happen-before get。这里比较巧妙地运用了volatile变量的特性，在get方法不加锁的情况下，保证了`happen-before`原则。

总结来说，else分支中的操作不是为了CopyOnWriteArrayList本身的可见性，而是为了保证list外部非volatile变量操作的内存可见性。副本写入和无锁的get方法等特性使得CopyOnWriteArrayList适用于读多写少的应用场景，但同时对内存的占用较高，不能保证读取数据的实时性，有可能会读到旧的数据。

**参考文档**

<http://ifeve.com/easy-happens-before/>
<http://ifeve.com/copyonwritearraylist-set/>