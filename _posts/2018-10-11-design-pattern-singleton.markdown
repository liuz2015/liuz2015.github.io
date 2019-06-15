---
layout:     post
title:      "【转】设计模式之单例模式"
subtitle:   ""
date:       2018-09-29
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 设计模式
    - 单例模式
---

> 转自：https://blog.csdn.net/learningcoding/article/details/80471475 

## 目录
- [简介](#简介)
- [实现](#实现)
- [多线程下实现](#多线程下实现)
- [指令重排](#指令重排)
- [单例的最简单实现Enum](#单例的最简单实现Enum)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

单例模式在网上已经是被写烂的一种设计模式了，笔者也看了不少的有关单例模式的文章，但是在实际生产中使用的并不是很多，如果一个知识点，你看过100遍，但是一次也没实践过，那么它终究不是属于你的。因此我借助这篇文章来复习下设计模式中的单例模式。

单例模式的作用在于保证整个程序在一次运行的过程中，被单例模式声明的类的对象要有且只有一个。针对不同的应用场景，单例模式的实现要求也不同。下文将描述几种单例模式的实现方案，从性能和实现上将有所差异，他们在一定程度上都能保证单例的存在，但是要在生产环境的角度来看待哪一种实现才是最合适的。

## 实现

单例模式的从实现步骤上来讲，分为三步：

- 构造方法私有，保证无法从外部通过 new 的方式创建对象。
- 对外提供获取该类实例的静态方法。
- 类的内部创建该类的对象，通过第2步的静态方法返回。

通过上述三点要求我们可以几乎就可以写出一个最最基本的单例实现方案，也就是各种资料中所描述的「饿汉式」。

### 饿汉式
```java
public class BasicSingleTon {

    //创建唯一实例
    private static final BasicSingleTon instance = new BasicSingleTon();

    //第二部暴露静态方法返回唯一实例
    public static BasicSingleTon getInstance() {
        return instance;
    }

    //第一步构造方法私有
    private BasicSingleTon() {
    }
}
```

该方法实现简单，也是最常用的一种，在不考虑线程安全的角度来说此实现也算是较为科学的，但是存在一个很大缺点就是，在虚拟机加载改类的时候，将会在初始化阶段为类静态变量赋值，也就是在虚拟机加载该类的时候（此时可能并没有调用 getInstance 方法）就已经调用了 new BasicSingleTon(); 创建了改对象的实例。但是如果追求代码的效率那么就需要采用下面这种方式，即延迟加载的方式。

也许这里看过看多例子的读者可能对 Instance 变量的声明为 static final 有所疑问，因为有的文章里之声明为 static，其实笔者认为在此单例模式的基本应用场景下，二者没有很大的区别，声明为 final 只是为了保证对象在方法区中的地址无法改变。而对对象的初始化时机没有影响。

### 延迟加载的单例模式
延迟加载的方式，是在我们编码过程中尽可能晚的实例化话对象，也就是避免在类的加载过程中，让虚拟机去创建这个实例对象。这种实现也就是我们所说的「懒汉式」。他的实现也很简单，将对象的创建操作后置到 getInstance 方法内部，最初的静态变量赋予 null ，而 在第一次调用 getInstance 的时候创建对象。
```java
public class LazyBasicSingleTon {

    private static LazyBasicSingleTon singleTon = null;

    public static LazyBasicSingleTon getInstance() {
        //延迟初始化 在第一次调用 getInstance 的时候创建对象
        if (singleTon == null) {
            singleTon = new LazyBasicSingleTon();
        }

        return singleTon;
    }

    private LazyBasicSingleTon() {
    }
}
```

## 多线程下实现
对于单线程模式上述的延迟加载已经算的上是很好的单例实践方式了。一方面Java 是一个多线程的内存模型。而静态变量存在于虚拟机的方法区中，该内存空间被线程共享，上述实现无法保证对单例对象的修改保证内存的可见性，原子性。而另一方面，newInstance 方法本身就不是一个原子类操作（分为两步第一步判空，第二步调用 new 来创建对象），所以结论是上述两种实现方式不适合多线程的引用场景。

那么对于多线程环境下单例实现模式，存在的问题，我们可以举个简单的例子，假设有两个线程都需要这个单例的对象，线程 A 率先进入语句 if (singleTon == null) 得到的结果为 true，此时 CPU 切换线程 B 去执行，由于 A 线程并没有进行 new LazyBasicSingleTon();的操作，那么 B 线程在执行语句 singleTon == null的结果认为 true，紧接着 B 线程创建了改类的实例对象，当 CPU 重新回到 A 线程去执行的时候，又会创建一个类的实例，这就导致了，所谓的单例并不真正的唯一，也就会产生错误。

为了解决这个缺点，我们能想到方法首先就是加锁，使用 synchronized 关键字来保证，在执行 getInstance 的时候不会发生线程的切换。
```java
public class SyncSingleTon {

    private static SyncSingleTon singleTon = null;

    /** 使用 synchronized 保证线程在创建对象的时候让其他线程阻塞*/
    public static synchronized SyncSingleTon getInstance() {
        if (singleTon == null) {
            singleTon = new SyncSingleTon();
        }

        return singleTon;
    }

    private SyncSingleTon() {
    }
}
```

其实 synchronized关键字也可以加在判空操作上,这样本质上并没有区别，只是别的资料中有这种实现方式，因此在这里给出实现：

```java
public static SyncSingleTon getInstance() {

   synchronized(SyncSingleTon.class）{
       if (singleTon == null) {
           singleTon = new SyncSingleTon();
       }
   }
   return singleTon;
}
```

### 双重判空操作的多线程单例实现
上面的例子给出的多线程下的单例实现，也可以保证在大多数情况下。可以保证单例的唯一性，但是对于效率会产生影响，因为如果我们可预料的线程切换场景并不是那么频繁，那么synchronized为getInstance方法加锁，将会带来很大效率丢失，比如单线程的模式下。

我们继续深入思考一下，可以想到，是因为在第一次获取该实例的时候，如果刚好发生了线程的切换将会早上我们所描述的单例不唯一的结果，在之后的调用过程中将会不会造成这样的结果。所以我们可以在 synchronized 语句之前，额外添加一次判空操作，来优化上述方案带来的效率损失。
```java
public class SyncSingleTon {

    private static SyncSingleTon singleTon = null;

    public static SyncSingleTon getInstance() {

        //这次判空是避免了，保证的多线程只有第一次调用getInstance 的时候才会加锁初始化
        if (singleTon == null) {
            synchronized (SyncSingleTon.class) {
                if (singleTon == null) {
                    singleTon = new SyncSingleTon();
                }
            }
        }
        return singleTon;
    }

    private SyncSingleTon() {
    }
}
```

上述方案很好的解决了，最开始的实现在效率上的损失，比如在多个线程场景中，即使在第一次if (singleTon == null) 判空操作中让出 CPU 去执行，那么在另一个线程中也会在同步代码中初始化改单例对象，待 CPU 切换回来的时候，也会在第二次判空的时候得到正确结果。

## 指令重排
当我们都认为这一切的看上去很完美的时候，JVM 又给我提出了个难题，那就是指令重排。

什么是指令重排，指令重排的用大白话来简单的描述，就是说在我们的代码运行时，JVM 并不一定总是按照我们想让它按照编码顺序去执行我们所想象的语义，它会在 “不改变” 原有代码语句含义的前提下进行代码，指令的重排序。

对于指令重排Java 语言规范给出来了下面的定义：
```
根据《The Java Language Specification, Java SE 7 Edition》（简称为java语言规范），所有线程在执行java程序时必须要遵守 intra-thread semantics(译为 线程内语义是一个单线程程序的基本语义)。intra-thread semantics 保证重排序不会改变单线程内的程序执行结果。换句话来说，intra-thread semantics 允许那些在单线程内，不会改变单线程程序执行结果的重排序。
```

那么我们上述双重检验锁的单例实现问题主要出在哪里呢？问题出在 singleTon = new SyncSingleTon();这句话在执行的过程。首先应该进行对象的创建操作大体可以分为三步：

- 分配内存空间。
- 初始化对象即执行构造方法。
- 设置 Instance 引用指向该内存空间。 
　 
那么如果有指令重排的前提下，这三部的执行顺序将有可能发生变化： 
　 
- 分配内存空间。 
- 设置 Instance 引用指向该内存空间。 
- 初始化对象即执行构造方法。 
　 
上面类初始化描述的步骤 2 和 3 之间虽然被重排序了， 但是这个重排序在没有改变单线程程序的执行结果。那么再多线程的前提下这将会造成什么样的后果呢？我们假设有两个线程同时想要初始化这个类，如果线程A中的操作指令重排了，就会造成的问题，B线程将返回一个空的Instance，可怕的是我们认为这一切是正常执行的。

为了解决上述问题我们可以从两个方面去考虑：

### 避免指令重排
让 A 线程完成对象初始化后，B 再去判断 instance == null；

通过 Volatile 避免指令重排序；

对于 Volatile 关键字，这里不做详细的描述，读者需要了解的是，volatile 作用有以下两点：

- 可以保证多线程条件下，内存区域的可见性,即使用 volatile 声明的变量，将对在一个线程从内主内存（线程共享的内存区域）读取变量，并写入后，通知其他线程，改变量被我改变了，别的线程在使用的时候，将会重新从主内存中去读改变量的最新值。

- 可以保证再多线程的情况下，指令重排这个操作将会被禁止。 
　
那么改造完成的双重检锁的单例将会是这样的：
```java
public class VolatileSingleTon {

    //使用 Volatile 保证了指令重排序在这个对象创建的时候不可用
    private volatile static  VolatileSingleTon singleTon = null;

    public static VolatileSingleTon getInstance() { 
        if (singleTon == null) {
            synchronized (VolatileSingleTon.class) {
                if (singleTon == null) {
                    singleTon = new VolatileSingleTon();
                }
            }
        }
        return singleTon;
    }
    private VolatileSingleTon() {}
}
```

由于 volatile 关键字是在 JDK 1.5 之后被明确了有禁止指令重排的语义的，那么有没有可能不用 volatile 就能解决我们上述描述的指令重排造成的问题呢，答案是肯定的。

### 静态内部类方式的单例实现
上述我们使用 Volatile 关键字去解决指令重排的方法是从避免指令重排的思路出发来解决问题的。那么对于第二种 让 A 线程完成对象初始化后，B 再去判断 instance == null 思路听起来好像有一定的加锁韵味，那么我们怎么去给一个对象的初始化过程去加锁呢，看起来好像没思路。

这里我们需要补充一个知识点，是有关 JVM 在类的初始化阶段期间，将会去获取一个锁，这个锁的作用是可以同步多个线程对同一个类的初始化操作。JVM 在类初始化期间会获得一个称做初始化锁的东西，并且每个线程至少获取一次锁来确保这个类已经被初始化过了。

我们可以理解为：如果一个线程在初始化一个类的时候，将会为这个初始化过程上锁，当此时有其他的线程尝试初始化这个类的时候，将会查看这个锁的状态，如果这个锁没有被释放，那么将会处于等待锁释放的状态。这和我们用的 synchronized 机制很相似，只是被用在类的初始化阶段。

对于静态内部类，相信读者一定清除它不依靠外部类的存在而存在。在编译阶段将作为独立的一个类，生成自己的 .class 文件。并且在初始化阶段也是独立的，也就是说拥有上述所说的初始化锁。

那么我们可以有如下思路：

返回该类的对象依赖于一个静态内部类的初始化操作。
在这个静态内部类初始化的时候，生成外部类的对象，然后在 getInstance 中返回
注意这里的初始化是指在JVM 类加载过程中 加载->链接（验证，准备，解析）->初始化 中的初始化。这个初始化过程将为类的静态变量付具体的值。

对于一个类的初始化时机有一下几种情况：

1）使用new关键字实例化对象的时候、读取或设置一个类的静态字段（被final修饰、已在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候。

2）使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化。

3）当初始化一个类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。

我们先来看下这里的具体实现：
```java
public class StaticInnerSingleTon {

    private static class InnerStaticClass{
        private static StaticInnerSingleTon singleTon  = new StaticInnerSingleTon();
    }

    public StaticInnerSingleTon getInstance(){
        //  //引用一个类的静态成员，将会触发该类的初始化 符合1）规则
        return InnerStaticClass.singleTon;
    }

    private StaticInnerSingleTon() {
    }
}
```

## 单例的最简单实现Enum
上述讲了这么多实现方法，也讲了各个实现的缺点。直到我们说了静态内部类的实现单例的思路后我们仿佛打开了新世界的大门。

为什么说枚举实现单例的方法最简单，这是因为 Enum 类的创建本身是就是线程安全的，这一点和静态内部类相似，因此我们不必去关心什么 DCL 问题，而是拿拿起键盘直接干：
```java
public enum  EnumSingleTon {
    INSTANCE
}

public class SingleTon {
    public static void main(String[] args) {
        EnumSingleTon instance = EnumSingleTon.INSTANCE;
        EnumSingleTon instance1 = EnumSingleTon.INSTANCE;

        System.out.println("instance1 == instance = " + (instance1 == instance));//输出结果为 true
    }
}
```
枚举的思想其实是通过共有的静态 final 与为每个枚举常量导出实例的类，由于没有可访问的构造器，所以不能调用枚举常量的构造方法去生成对应的对象，因此在《Effective Java》 中，枚举类型为类型安全的枚举模式，枚举也被称为单例的泛型化。

## 总结
一篇行文下来，对于单例模式的理解变的更加深刻了，尤其是 DSL（double checked locking）） 的问题的解决思路上，更是涉及到，指令重排和类的加载机制的方面的知识。面试的时候，面试官也经常由此引出更深的只是，比如JVM 类加载的相关知识点，volatile 关键字的作用，以及多线程方面的知识点。其实对于面试者来说这也许是个好事，毕竟有迹可循了。


## 参考资料
- https://blog.csdn.net/learningcoding/article/details/80471475 

