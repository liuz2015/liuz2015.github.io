---
layout:     post
title:      "【转】Java8 concurrentHashMap源码解读"
subtitle:   ""
date:       2018-09-24
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Java8
    - concurrentHashMap
    - 并发
---

> 转自：https://blog.csdn.net/u010412719/article/details/52145145 

## 引言

我们都知道HashMap是线程不安全的。Hashtable是线程安全的。看过Hashtable源码的我们都知道Hashtable的线程安全是采用在每个方法来添加了synchronized关键字来修饰，即Hashtable是针对整个table的锁定，这样就导致HashTable容器在竞争激烈的并发环境下表现出效率低下。

效率低下的原因说的更详细点：是因为所有访问HashTable的线程都必须竞争同一把锁。当一个线程访问HashTable的同步方法时，其他线程访问HashTable的同步方法时，可能会进入阻塞或轮询状态。如线程1使用put进行添加元素，线程2不但不能使用put方法添加元素，并且也不能使用get方法来获取元素，所以竞争越激烈效率越低。

基于Hashtable的缺点，人们就开始思考，假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率呢？？这就是我们的“锁分离”技术，这也是ConcurrentHashMap实现的基础。

ConcurrentHashMap使用的就是锁分段技术，ConcurrentHashMap由多个Segment组成(Segment下包含很多Node，也就是我们的键值对了)，每个Segment都有把锁来实现线程安全，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

因此，关于ConcurrentHashMap就转化为了对Segment的研究。这是因为，ConcurrentHashMap的get、put操作是直接委托给Segment的get、put方法，但是自己上手上的JDK1.8的具体实现确不想网上这些博文所介绍的。因此，就有了本篇博文的介绍。

## 目录
- [简介](#简介)
- [相关属性](#ConcurrentHashMap类中相关属性的介绍)
- [构造函数](#ConcurrentHashMap的构造函数)
- [相关节点](#ConcurrentHashMap类中相关节点类)
- [put分析](#ConcurrentHashMap类的put方法分析)
- [get分析](#ConcurrentHashMap类的get方法分析)
- [size分析](#分析ConcurrentHashMap类的size()方法)
- [clear，remove分析](#clear，remove方法)
- [参考资料](#参考资料)

## 简介

首先要说明的几点：

1、JDK1.8的ConcurrentHashMap中Segment虽保留，但已经简化属性，仅仅是为了兼容旧版本。

2、ConcurrentHashMap的底层与Java1.8的HashMap有相通之处，底层依然由“数组”+链表+红黑树来实现的，底层结构存放的是TreeBin对象，而不是TreeNode对象；

3、ConcurrentHashMap实现中借用了较多的CAS算法，unsafe.compareAndSwapInt(this, valueOffset, expect, update); CAS(Compare And Swap)，意思是如果valueOffset位置包含的值与expect值相同，则更新valueOffset位置的值为update，并返回true，否则不更新，返回false。

ConcurrentHashMap既然借助了CAS来实现非阻塞的无锁实现线程安全，那么是不是就没有用锁了呢？？答案：还是使用了synchronized关键字进行同步了的，在哪里使用了呢？在操作hash值相同的链表的头结点还是会synchronized上锁，这样才能保证线程安全。

看完ConcurrentHashMap整个类的源码，给自己的感觉就是和HashMap的实现基本一模一样，当有修改操作时借助了synchronized来对table[i]进行锁定保证了线程安全以及使用了CAS来保证原子性操作，其它的基本一致，例如：ConcurrentHashMap的get(int key)方法的实现思路为：根据key的hash值找到其在table所对应的位置i,然后在table[i]位置所存储的链表(或者是树)进行查找是否有键为key的节点，如果有，则返回节点对应的value，否则返回null。思路是不是很熟悉，是不是和HashMap中该方法的思路一样。所以，如果你也在看ConcurrentHashMap的源码，不要害怕，思路还是原来的思路，只是多了些许东西罢了。

## ConcurrentHashMap类中相关属性的介绍
为了方便介绍此类后面的实现，这里需要先将此类中的一些属性给介绍下。

sizeCtl最重要的属性之一，看源码之前，这个属性表示什么意思，一定要记住。

0、private transient volatile int sizeCtl;//控制标识符

此属性在源码中给出的注释如下：
```
/**
* Table initialization and resizing control.  When negative, the
* table is being initialized or resized: -1 for initialization,
* else -(1 + the number of active resizing threads).  Otherwise,
* when table is null, holds the initial table size to use upon
* creation, or 0 for default. After initialization, holds the
* next element count value upon which to resize the table.
*/
```
翻译如下：
```
sizeCtl是控制标识符，不同的值表示不同的意义。
负数代表正在进行初始化或扩容操作 ,其中-1代表正在初始化 ,-N 表示有N-1个线程正在进行扩容操作，
正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，类似于扩容阈值。
它的值始终是当前ConcurrentHashMap容量的0.75倍，这与loadfactor是对应的。实际容量>=sizeCtl，则扩容。
```
1、 transient volatile Node<K,V>[] table;是一个容器数组，第一次插入数据的时候初始化，大小是2的幂次方。这就是我们所说的底层结构：”数组+链表（或树）”

2、private static final int MAXIMUM_CAPACITY = 1 << 30; // 最大容量

3、private static final intDEFAULT_CAPACITY = 16;

4、static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8; // MAX_VALUE=2^31-1=2147483647

5、private static finalint DEFAULT_CONCURRENCY_LEVEL = 16;

6、private static final float LOAD_FACTOR = 0.75f;

7、static final int TREEIFY_THRESHOLD = 8; // 链表转树的阀值，如果table[i]下面的链表长度大于8时就转化为数

8、static final int UNTREEIFY_THRESHOLD = 6; //树转链表的阀值，小于等于6是转为链表，仅在扩容tranfer时才可能树转链表

9、static final int MIN_TREEIFY_CAPACITY = 64;

10、private static final int MIN_TRANSFER_STRIDE = 16;

11、private static int RESIZE_STAMP_BITS = 16;

12、private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1; // help resize的最大线程数

13、private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;

14、static final int MOVED = -1; // hash for forwarding nodes（forwarding nodes的hash值）、标示位

15、static final int TREEBIN = -2; // hash for roots of trees（树根节点的hash值）

16、static final int RESERVED = -3; // hash for transient reservations（ReservationNode的hash值）

## ConcurrentHashMap的构造函数
和往常一样，我们还是从类的构造函数开始说起。
```java
/**
* Creates a new, empty map with the default initial table size (16).
*/
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
            MAXIMUM_CAPACITY :
            tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

/*
* Creates a new map with the same mappings as the given map.
*
*/
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}
public ConcurrentHashMap(int initialCapacity,
                        float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```
有过HashMap和Hashtable源码经历，看这些构造函数是不是相当easy哈。

上面的构造函数主要干了两件事：

1、参数的有效性检查

2、table初始化的长度(如果不指定默认情况下为16)。

这里要说一个参数：concurrencyLevel，表示能够同时更新ConccurentHashMap且不产生锁竞争的最大线程数。默认值为16，(即允许16个线程并发可能不会产生竞争)。为了保证并发的性能，我们要很好的估计出concurrencyLevel值，不然要么竞争相当厉害，从而导致线程试图写入当前锁定的段时阻塞。

## ConcurrentHashMap类中相关节点类
主要包括：Node/TreeNode/TreeBin
### Node类
Node类是table数组中的存储元素，即一个Node对象就代表一个键值对(key,value)存储在table中。

Node类是没有提供修改入口的(唯一的setValue方法抛异常)，因此只能用于只读遍历。

此类的具体代码如下：
```java
/*
    *Node类是没有提供修改入口的(setValue方法抛异常，供子类实现)，
    即是可读的。只能用于只读遍历。
    */
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;//volatile，保证可见性
    volatile Node<K,V> next;

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    /*
        HashMap中Node类的hashCode()方法中的代码为：Objects.hashCode(key) ^ Objects.hashCode(value)
        而Objects.hashCode(key)最终也是调用了 key.hashCode()，因此，效果一样。写法不一样罢了
    */;
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    public final V setValue(V value) { // 不允许修改value值，HashMap允许
        throw new UnsupportedOperationException();
    }
    /*
            HashMap使用if (o == this)，且嵌套if；ConcurrentHashMap使用&& 
            个人觉得HashMap格式的代码更好阅读和理解
    */
    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /*
        * Virtualized support for map.get(); overridden in subclasses.
        *增加find方法辅助get方法  ，HashMap中的Node类中没有此方法
        */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```
我们在看这个类时，可以与HashMap中的Node类的具体代码进行比较，发现在具体的实现上，有一定的细微的区别。

例如：在ConcurrentHashMap.Node的hashCode的代码是这样的：
```java
public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
```
而HashMap.Node的hashCode的代码是这样的：
```java
public final int hashCode() {
    return Objects.hashCode(key) ^ Objects.hashCode(value);
}
```
而Objects.hashCode(key)最终也是调用了 key.hashCode()，因此，两者的效果一样，写法不一样罢了。

除了hashCode方法有一点差别，Node类中的find方法在两个类的实现中的写法也不一样。

### TreeNode
```java
/*
    * Nodes for use in TreeBins 
    */
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    TreeNode(int hash, K key, V val, Node<K,V> next,
                TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }

    Node<K,V> find(int h, Object k) {
        return findTreeNode(h, k, null);
    }

    /*
        * Returns the TreeNode (or null if not found) for the given key
        * starting at given root.
        *根据给定的key值从root节点出发找出节点
        *  
        */
    final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
        if (k != null) {//HashMap没有非空判断
            TreeNode<K,V> p = this;
            do  {
                int ph, dir; K pk; TreeNode<K,V> q;
                TreeNode<K,V> pl = p.left, pr = p.right;
                if ((ph = p.hash) > h)
                    p = pl;
                else if (ph < h)
                    p = pr;
                else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                    return p;
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                            (kc = comparableClassFor(k)) != null) &&
                            (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.findTreeNode(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
        }
        return null;
    }
}
```

和HashMap相比，这里的TreeNode相当简洁；ConcurrentHashMap链表转树时，并不会直接转， 

正如注释（Nodes for use in TreeBins）所说，只是把这些节点包装成TreeNode放到TreeBin中， 

再由TreeBin来转化红黑树。红黑树不理解没关系，并不影响看ConcurrentHashMap的内部实现

### TreeBins
TreeBin用于封装维护TreeNode，包含putTreeVal、lookRoot、UNlookRoot、remove、balanceInsetion、balanceDeletion等方法，当链表转树时，用于封装TreeNode，也就是说，ConcurrentHashMap的红黑树存放的时TreeBin，而不是treeNode。

TreeBins类代码太长，截取部分代码如下：
```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K,V> root;
    volatile TreeNode<K,V> first;
    volatile Thread waiter;
    volatile int lockState;
    // values for lockState
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock

    /**
        * Creates bin with initial set of nodes headed by b.
        */
    TreeBin(TreeNode<K,V> b) {
        super(TREEBIN, null, null, null);
        this.first = b;
        TreeNode<K,V> r = null;
        for (TreeNode<K,V> x = b, next; x != null; x = next) {
            next = (TreeNode<K,V>)x.next;
            x.left = x.right = null;
            if (r == null) {
                x.parent = null;
                x.red = false;
                r = x;
            }
            else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K,V> p = r;;) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                                (kc = comparableClassFor(k)) == null) ||
                                (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);
                        TreeNode<K,V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        r = balanceInsertion(r, x);
                        break;
                    }
                }
            }
        }
        this.root = r;
        assert checkInvariants(root);
    }
    //........other methods
}
```

### ForwardingNode
在transfer操作中，将一个节点插入到桶中
```java
/*
    * A node inserted at head of bins during transfer operations.
    *在transfer操作中，一个节点插入到bins中
    */
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        //Node(int hash, K key, V val, Node<K,V> next)是Node类的构造函数
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

## ConcurrentHashMap类的put方法分析
我们对Node、TreeNode、TreeBin有一点认识后，我们就可以看下ConcurrentHashMap类的put方法是如何来实现的了，这里给出一个建议，关于容器我们用的最多的就是put、get方法了，我们看源码的实现，我们核心要关注的就是put、get方法的实现，只要我们弄懂这两个方法实现，这个类的大概实现思想我们也就知道了哈

基于此，我们就先来看ConcurrentHashMap类的put方法

put(K key, V value)方法的功能：将制定的键值对映射到table中，key/value均不能为null

put方法的代码如下：
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
```
由于直接是调用了putVal(key, value, false)方法，那就我们就继续看。

putVal(key, value, false)方法的代码如下：
```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());//计算hash值，两次hash操作
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {//类似于while(true)，死循环，直到插入成功 
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)//检查是否初始化了，如果没有，则初始化
            tab = initTable();
            /*
                i=(n-1)&hash 等价于i=hash%n(前提是n为2的幂次方).即取出table中位置的节点用f表示。
                有如下两种情况：
                1、如果table[i]==null(即该位置的节点为空，没有发生碰撞)，则利用CAS操作直接存储在该位置，
                    如果CAS操作成功则退出死循环。
                2、如果table[i]!=null(即该位置已经有其它节点，发生碰撞)
            */
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null,
                            new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)//检查table[i]的节点的hash是否等于MOVED，如果等于，则检测到正在扩容，则帮助其扩容
            tab = helpTransfer(tab, f);//帮助其扩容
        else {//运行到这里，说明table[i]的节点的hash值不等于MOVED。
            V oldVal = null;
            synchronized (f) {//锁定,（hash值相同的链表的头节点）
                if (tabAt(tab, i) == f) {//避免多线程，需要重新检查
                    if (fh >= 0) {//链表节点
                        binCount = 1;
                        /*
                        下面的代码就是先查找链表中是否出现了此key，如果出现，则更新value，并跳出循环，
                        否则将节点加入到里阿尼报末尾并跳出循环
                        */
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)//仅putIfAbsent()方法中onlyIfAbsent为true
                                    e.val = value;//putIfAbsent()包含key则返回get，否则put并返回  
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {//插入到链表末尾并跳出循环
                                pred.next = new Node<K,V>(hash, key,
                                                            value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { //树节点，
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                        value)) != null) {//插入到树中
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            //插入成功后，如果插入的是链表节点，则要判断下该桶位是否要转化为树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)//实则是>8,执行else,说明该桶位本就有Node
                    treeifyBin(tab, i);//若length<64,直接tryPresize,两倍table.length;不转树 
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

代码比较长哈，但是不要怕，我刚开始看的时候，也被长度给吓住了，怎么可以有这么长的方法呢，HashMap中put方法的长度就很短的么。

虽然很长，但是思路相当的简单。代码详细流程如下,在上面代码中也有详细的注释
```
/*
putVal(K key, V value, boolean onlyIfAbsent)方法干的工作如下：
1、检查key/value是否为空，如果为空，则抛异常，否则进行2
2、进入for死循环，进行3
3、检查table是否初始化了，如果没有，则调用initTable()进行初始化然后进行 2，否则进行4
4、根据key的hash值计算出其应该在table中储存的位置i，取出table[i]的节点用f表示。
根据f的不同有如下三种情况：1）如果table[i]==null(即该位置的节点为空，没有发生碰撞)，
则利用CAS操作直接存储在该位置，如果CAS操作成功则退出死循环。
2）如果table[i]!=null(即该位置已经有其它节点，发生碰撞)，碰撞处理也有两种情况
2.1）检查table[i]的节点的hash是否等于MOVED，如果等于，则检测到正在扩容，则帮助其扩容
2.2）说明table[i]的节点的hash值不等于MOVED，如果table[i]为链表节点，则将此节点插入链表中即可
如果table[i]为树节点，则将此节点插入树中即可。插入成功后，进行 5
5、如果table[i]的节点是链表节点，则检查table的第i个位置的链表是否需要转化为数，如果需要则调用treeifyBin函数进行转化
*/
```
可能你觉得上面的详细流程也比较多哈，但是不要怕，用两句话来总结的话，是如下的两步：

1、第一步根据给定的key的hash值找到其在table中的位置index。

2、找到位置index后，存储进行就好了。

只是这里的存储有三种情况罢了，第一种：table[index]中没有任何其他元素，即此元素没有发生碰撞，这种情况直接存储就好了哈。第二种，table[i]存储的是一个链表，如果链表不存在key则直接加入到链表尾部即可，如果存在key则更新其对应的value。第三种，table[i]存储的是一个树，则按照树添加节点的方法添加就好。

在putVal函数，出现了如下几个函数

1、casTabAt tabAt 等CAS操作

2、initTable 作用是初始化table数组

3、treeifyBin 作用是将table[i]的链表转化为树

下面将分别进行介绍。

这里给出第二个建议，当一个类的代码量相当大且复杂时，从我们感兴趣的方法出发，然后是遇到哪个方法就才解决哪个方法

3个用的比较多的CAS操作：casTabAt tabAt setTabAt
```java
/*
    3个用的比较多的CAS操作
*/

@SuppressWarnings("unchecked") // ASHIFT等均为private static final  
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) { // 获取索引i处Node  
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);  
}  
// 利用CAS算法设置i位置上的Node节点（将c和table[i]比较，相同则插入v）。  
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,  
                                    Node<K,V> c, Node<K,V> v) {  
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);  
}  
// 设置节点位置的值，仅在上锁区被调用  
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {  
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);  
}
```

### initTable() terrifyBin方法

在putVal方法中遇到的第一个扩容函数为：initTable，即初始化

代码如下，注释相当详细，这里就不再解释。
```java
/**
    * Initializes table, using the size recorded in sizeCtl.
    */
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)//如果sizeCtl为负数，则说明已经有其它线程正在进行扩容，即正在初始化或初始化完成
            Thread.yield(); // lost initialization race; just spin
            //如果CAS成功，则表示正在初始化，设置为 -1，否则说明其它线程已经对其正在初始化或是已经初始化完毕
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {//再一次检查确认是否还没有初始化
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);//即sc = 0.75n。
                }
            } finally {
                sizeCtl = sc;//sizeCtl = 0.75*Capacity,为扩容门限
            }
            break;
        }
    }
    return tab;
}
```

treeifyBin方法：将数组tab的第index位置的链表转化为 树
```java
/*
    *链表转树：将将数组tab的第index位置的链表转化为 树
    */
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)// 容量<64，则table两倍扩容，不转树了
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) { // 读写锁  
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                                null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

treeifyBin方法的思想也相当的简单，如下：

1、检查下table的长度是否大于等于MIN_TREEIFY_CAPACITY（64），如果不大于，则调用tryPresize方法将table两倍扩容就可以了，就不降链表转化为树了。如果大于，则就将table[i]的链表转化为树。

tryPresize方法
在putVal方法中遇到的第二个扩容函数为：tryPresize
```java
/*
    扩容相关
    tryPresize在putAll以及treeifyBin中调用
*/
private final void tryPresize(int size) {
    // 给定的容量若>=MAXIMUM_CAPACITY的一半，直接扩容到允许的最大值，否则调用tableSizeFor函数扩容 
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);//tableSizeFor(count)的作用是找到大于等于count的最小的2的幂次方
    int sc;
    while ((sc = sizeCtl) >= 0) {//只有大于等于0才表示该线程可以扩容，具体看sizeCtl的含义
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) {//没有被初始化
            n = (sc > c) ? sc : c;
            // 期间没有其他线程对表操作，则CAS将SIZECTL状态置为-1，表示正在进行初始化  
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {//再一次检查
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);//无符号右移2位，此即0.75*n 
                    }
                } finally {
                    sizeCtl = sc;// 更新扩容阀值  
                }
            }
        }
        // 若欲扩容值不大于原阀值，或现有容量>=最值，什么都不用做了 
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) { // table不为空，且在此期间其他线程未修改table  
            int rs = resizeStamp(n);
            if (sc < 0) {//这里的sc可能小于零么？？？不明白为什么会有此判断
                Node<K,V>[] nt;//RESIZE_STAMP_SHIFT=16,MAX_RESIZERS=2^15-1  
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                            (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}

/*
    Returns the stamp bits for resizing a table of size n.当扩容到n时，调用该函数返回一个标志位
    Must be negative when shifted left by RESIZE_STAMP_SHIFT.
    numberOfLeadingZeros返回n对应32位二进制数左侧0的个数，如9（1001）返回28  
    RESIZE_STAMP_BITS=16,
    因此返回值为：(参数n的左侧0的个数)|(2^15)  
    */
static final int resizeStamp(int n) {
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

既然是扩容，思路就比较简单哈，注释的相当详细，就不介绍了哈，在这个函数中调用transfer函数，transfer方法的代码太长，这里不贴出。

在transfer方法中，用到了如下的属性
```java
private transient volatile Node<K,V>[] nextTable; 
```
仅仅在扩容使用，并且此时非空。

在扩容的过程中，还有一个辅助方法：helpTransfer方法。

代码如下：
```java
/*
    * Helps transfer if a resize is in progress.
    *在多线程情况下，如果发现其它线程正在扩容，则帮助转移元素。
    （只有这种情况会被调用）从某种程度上说，其“优先级”很高，只要检测到扩容，就会放下其他工作，先扩容。
    */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {// 调用之前，nextTable一定已存在。
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);//标志位
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);//调用扩容方法，直接进入复制阶段  
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

以上就把跟putVal相关的函数都看了一篇哈，可能细节我们没有看懂，但是各个方法的思路我们都清楚了，继续往下面来看

## ConcurrentHashMap类的get方法分析
看完了ConcurrentHashMap类的put(int key ,int value)方法的内部实现，接着看此类的get(int key)方法。
```java
/*
    功能：根据key在Map中找出其对应的value，如果不存在key，则返回null，
    其中key不允许为null，否则抛异常
    */
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());//两次hash计算出hash值
    if ((tab = table) != null && (n = tab.length) > 0 &&//table不能为null，是吧
        (e = tabAt(tab, (n - 1) & h)) != null) {//table[i]不能为空，是吧
        if ((eh = e.hash) == h) {//检查头结点
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)//table[i]为一颗树
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {//链表，遍历寻找即可
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

get(int key)方法代码实现流程如下：

1、根据key调用spread计算hash值；并根据计算出来的hash值计算出该key在table出现的位置i.

2、检查table是否为空；如果为空，返回null，否则进行3

3、检查table[i]处桶位不为空；如果为空，则返回null，否则进行4

4、先检查table[i]的头结点的key是否满足条件，是则返回头结点的value；否则分别根据树、链表查询。

get方法的思想是不是也很简单哈，与HashMap的get方法一模一样，分析到这里，ConcurrentHashMap类的源码的大概实现思路我们就基本清晰了哈，本着学习的精神，我们还是稍微看下其他的方法哈，例如：containsKey、remove、size等等

分析ConcurrentHashMap类的containsKey/containsValue方法
看下containsKey/containsValue方法
```java
/*
    * Tests if the specified object is a key in this table.
    */
public boolean containsKey(Object key) {
    return get(key) != null;//直接调用get(int key)方法即可，如果有返回值，则说明是包含key的
}

/*
    *功能，检查在所有映射(k,v)中只要出现一次及以上的v==value，返回true
    *注意：这个方法可能需要一个完全遍历Map，因此比containsKey要慢的多
    */
public boolean containsValue(Object value) {
    if (value == null)
        throw new NullPointerException();
    Node<K,V>[] t;
    if ((t = table) != null) {
        Traverser<K,V> it = new Traverser<K,V>(t, t.length, 0, t.length);
        for (Node<K,V> p; (p = it.advance()) != null; ) {
            V v;
            if ((v = p.val) == value || (v != null && value.equals(v)))
                return true;
        }
    }
    return false;
}
```

containsKey/containsValue方法的内部实现也比较简单哈。这里也不再详细介绍。

## 分析ConcurrentHashMap类的size()方法
```java
// Original (since JDK1.2) Map methods
public int size() {// 旧版本方法，和推荐的mappingCount返回的值基本无区别
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
```
这个方法是从JDK1.2版本开始就有的方法了。而ConcurrentHashMap在JDK1.8版本中还提供了另外一种方法可以获取大小，这个方法就是mappingCount。

代码如下：
```java
// ConcurrentHashMap-only methods

/**
    * Returns the number of mappings. This method should be used
    * instead of {@link #size} because a ConcurrentHashMap may
    * contain more mappings than can be represented as an int. The
    * value returned is an estimate(估计); the actual count may differ if
    * there are concurrent insertions or removals.
    *
    * @return the number of mappings
    * @since 1.8
    */
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n; // ignore transient negative values
}
```

根据mappingCount()方法头上的注释，我们可以得到如下的信息：

1、这个应该用来代替size()方法被使用。这是因为ConcurrentHashMap可能比包含更多的映射结果，即超过int类型的最大值。

2、这个方法返回值是一个估计值，由于存在并发的插入和删除，因此返回值可能与实际值会有出入。

虽然注释这么才说使用mappingCount来代替size()方法，但是我们比较两个方法的源码你会发现这两个方法的源码基本一致。

在size()方法和mappingCount方法中都出现了sumCount()方法，因此，我们也顺便看一下。
```java
/* ---------------- Counter support -------------- */

/**
    * A padded cell for distributing counts.  Adapted from LongAdder
    * and Striped64.  See their internal docs for explanation.
    */
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
// Table of counter cells. When non-null, size is a power of 2
private transient volatile CounterCell[] counterCells;
//ConcurrentHashMap中元素个数,基于CAS无锁更新,但返回的不一定是当前Map的真实元素个数。
private transient volatile long baseCount; 

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

## clear，remove方法
remove方法的代码如下;
```java
/*
    * Removes the key (and its corresponding value) from this map.
    * This method does nothing if the key is not in the map.
    */
public V remove(Object key) {
    return replaceNode(key, null, null);
}

/*
    *如果Map中存在(key,value)节点，则用对象cd来代替，
    *如果value为空，则删除此节点。
    */
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode());//计算hash值
    for (Node<K,V>[] tab = table;;) {//死循环，直到找到
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null)//如果为空，则立即返回
            break;
        else if ((fh = f.hash) == MOVED)//如果检测到其它线程正在扩容，则先帮助扩容，然后再来寻找，可见扩容的优先级之高
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            boolean validated = false;
            synchronized (f) {  //开始锁住这个桶，然后进行比对寻找满足(key,value)的节点
                if (tabAt(tab, i) == f) { //重新检查，避免由于多线程的原因table[i]已经被修改
                    if (fh >= 0) {//链表节点
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                    (ek != null && key.equals(ek)))) {//满足条件就是找到key出现的节点位置
                                V ev = e.val;
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) {
                                    oldVal = ev;
                                    if (value != null)//value不为空，则更新值
                                        e.val = value;
                                    //value为空，则删除此节点
                                    else if (pred != null)
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);//符合条件的节点e为头结点的情况
                                }
                                break;
                            }
                            //更改指向，继续向后循环
                            pred = e;
                            if ((e = e.next) == null)//如果为到链表末尾了，则直接退出即可
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) {//树节点
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val;
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) {
                    if (value == null)//如果删除了节点，则要减1
                        addCount(-1L, -1);
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

remove方法的实现思路也比较简单。如下；

1、先根据key的hash值计算书其在table的位置 i。

2、检查table[i]是否为空，如果为空，则返回null，否则进行3

3、在table[i]存储的链表(或树)中开始遍历比对寻找，如果找到节点符合key的，则判断value是否为null来决定是否是更新oldValue还是删除该节点。

clear()方法的源码如下，这里就不再进行分析了哈。
```java
/**
    * Removes all of the mappings from this map.
    */
public void clear() {
    long delta = 0L; // negative number of deletions
    int i = 0;
    Node<K,V>[] tab = table;
    while (tab != null && i < tab.length) {
        int fh;
        Node<K,V> f = tabAt(tab, i);
        if (f == null)
            ++i;
        else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
            i = 0; // restart
        }
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> p = (fh >= 0 ? f :
                                    (f instanceof TreeBin) ?
                                    ((TreeBin<K,V>)f).first : null);
                    while (p != null) {
                        --delta;
                        p = p.next;
                    }
                    setTabAt(tab, i++, null);
                }
            }
        }
    }
    if (delta != 0L)
        addCount(delta, -1);
}
```

## 参考资料
- https://blog.csdn.net/u010412719/article/details/52145145

