---
layout: article
title: Java设计模式 -- 单例模式
key: 2019-01-08-singleton
tags: 设计模式
---

> 单例模式是平时用到最多的设计模式，也是相对最简单的一个，但具体细节仍有许多需要注意的地方，本文通过单例模式的三种写法来讨论其中的细节。

<!--more-->

## 饿汉模式
最简单的写法，代码如下：
```java
public class Singleton {
  // 声明私有静态的实例变量，这里在单例类第一次使用的时候就会被创建
  private static Singleton instance = new Singleton();

  // 声明构造方法为私有
  private Singleton() {}

  // 公有的获取实例的方法
  public static Singleton getInstance() {
    return instance;
  }

  // 其他静态方法，调用该方法时，也会导致实例被创建
  public static void doSomething() {
    // do something
  }
}
```
饿汉模式的缺点就是如果我们调用单例类中的其他静态方法而不需要创建单例类实例，而这个时候实例已经被创建，占用了内存。不过实际生产环境应该很少遇到这种情况。

## 懒汉模式
最容易出错的写法，正确代码如下：
```java
public class Singleton {
  // 声明私有静态实例变量，赋值为null，注意要使用volatile关键字修饰
  private volatitle Singleton instance = null;

  // 声明构造方法为私有
  private Singleton() {}

  // 获取实例方法，双重检查
  public static Singleton getInstance() {
    if (instance == null) { // 1. 第一次检查
      synchronized (Single.class) { // 2. 加锁
        if (instance == null) { // 3. 第二次检查
          instance = new Singleton(); // 4. 创建实例
        }
      }
    }
    return instance;
  }
}
```
双重检查指的是加锁前和加锁后两次检查instance是否为null，在多线程的情况下，避免重复创建实例。

接下来重点说明一下为什么需要对instance使用volatile关键字修饰，首先看一下不加volatile的代码：
```java
public class Singleton {
  
  private Singleton instance = null;

  private int value;

  private Singleton() {
    this.value = 100;
  }

  public static Singleton getInstance() {
    if (instance == null) { // 1. 第一次检查
      synchronized (Single.class) { // 2. 加锁
        if (instance == null) { // 3. 第二次检查
          instance = new Singleton(); // 4. 创建实例
        }
      }
    }
    return instance;
  }
}
```
假设有两个线程A和B，线程A进入getInstance方法，执行到第4步创建实例，这个步骤实际又分为三步：(1)在堆内存中申请一段空间，将各个属性初始化为零值(0/null) (2)通过构造方法对各个属性赋值 (3)把instance对象指向堆内存空间。由于指令重排序，(3)可能会先于(2)执行，如果此时线程B进入方法，判断instance不为空，就会直接返回instance实例。这种情况下拿到的instance还没有经过构造方法复制，不是完整的实例。另外，即使没有发生重排序，线程A对于属性的赋值是先写入到线程的工作内存中，此时线程B仍然看不到属性的值，在退出synchronized块后，才会将刚才的赋值操作刷入到主内存中。

通过volatile关键字修饰，可以保证内存可见性并禁止重排序，这里的内存可见性是指对volatile属性的写操作会立刻刷入到主内存，避免了上面情况的出现。

## 静态嵌套类模式
推荐这种方式实现单例模式：
```java
public class Singleton {
  // 私有构造方法
  private Singleton() {}

  // 在嵌套类中创建实例
  private static class InstanceHolder {
    private static Singleton instance = new Singleton();
  }

  public static getInstance() {
    return InstanceHolder.instance;
  }
}
```
这里注意一下嵌套类(nested class)和内部类(inner class)的区别，嵌套类分为静态嵌套类和非静态嵌套类，其中非静态嵌套类也被称为内部类。

静态嵌套类模式的优点：
1. 延迟创建实例，避免饿汉模式的缺点。外部类加载时，并不需要加载嵌套类。在上面的例子中，只有调用getInstance方法时，才会加载InstanceHolder类。
2. 保证线程安全：JVM会保证一个类的`<clinit>()`方法在多线程环境中被正确地加锁、同步。同懒汉模式相比，避免了使用synchronized锁，提高了效率。

综上，静态嵌套类模式的单例避免了饿汉和懒汉模式的缺陷，在实际生产中更加推荐使用。当然这种模式也有其局限性，如果创建实例的时候需要传入外部类的非静态属性时，静态嵌套类是无法访问到这种属性。

**参考文档：**

<https://javadoop.com/post/design-pattern#%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F>

<https://blog.csdn.net/mnb65482/article/details/80458571>

<https://blog.csdn.net/hguisu/article/details/7270086>

<https://javadoop.com/post/java-memory-model#%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F%E4%B8%AD%E7%9A%84%E5%8F%8C%E9%87%8D%E6%A3%80%E6%9F%A5>

<http://literatejava.com/jvm/fastest-threadsafe-singleton-jvm/>