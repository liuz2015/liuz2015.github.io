---
layout:     post
title:      "Java8 HashMap源码解读"
subtitle:   ""
date:       2018-08-24
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Java8
    - hashMap
---


## 目录
- [简介](#简介)
- [重要参数](#重要参数)
- [方法实现](#方法实现)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

基于哈希表的 Map 接口的实现。此实现提供所有可选的映射操作，并允许使用 null 值和 null 键。（除了非同步和允许使用 null 之外，HashMap 类与 Hashtable 大致相同。）此类不保证映射的顺序，特别是它不保证该顺序恒久不变。 此实现假定哈希函数将元素适当地分布在各桶之间，可为基本操作（get 和 put）提供稳定的性能。迭代 collection 视图所需的时间与 HashMap 实例的“容量”（桶的数量）及其大小（键-值映射关系数）成比例。所以，如果迭代性能很重要，则不要将初始容量设置得太高（或将加载因子设置得太低）。

## 重要参数

static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //初始容量

static final float DEFAULT_LOAD_FACTOR = 0.75f;//默认加载因子

transient Node<K,V>[] table;//容器（存放数据的桶）

transient int size;//元素个数，不等于数组的长度。

transient int modCount;//每次扩容和更改容器结构的计数器

int threshold;//阈值 当实际节点个数超过(容量*填充因子)时进行扩容

final float loadFactor;//加载因子

### Node类
```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```
这个类用来存储数据，通过这个类构建数组，就是map中的核心容器，而在Java8中，又对这个类进一步扩展，实现红黑树节点TreeNode，来应对数组的某个位置中元素过多的情况。

## 方法实现

### 基础：哈希值的计算

HashMap是以hash操作作为散列依据。但是又与传统的hash存在着少许的优化。
```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
其hash值是key的hashcode与其hashcode右移16位的异或结果。在put方法中，将取出的hash值与当前的hashmap容量-1进行与运算。得到的就是位桶的下标。那么为何需要使用key.hashCode() ^ h>>>16的方式来计算hash值呢。其实从微观的角度来看，这种方法与直接去key的哈希值返回在功能实现上没有差别。但是由于最终获取下表是对二进制数组最后几位的与操作。所以直接取hash值会丢失高位的数据，从而增大冲突引起的可能。由于hash值是32位的二进制数。将高位的16位于低位的16位进行异或操作，即可将高位的信息存储到低位。因此该函数也叫做扰乱函数。目的就是减少冲突出现的可能性。而官方给出的测试报告也验证了这一点。直接使用key的hash算法与扰乱函数的hash算法冲突概率相差10%左右。

### 初始化

在初始化方法中，只设置loadFactor和threshold，并不进行table的初始化。

### put

```java
/**
* Implements Map.put and related methods.
*
* @param hash hash for key
* @param key the key
* @param value the value to put
* @param onlyIfAbsent if true, don't change existing value
* @param evict if false, the table is in creation mode.
* @return previous value, or null if none
*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

- 判断table是否初始化，若没有则执行初始化。
- 计算key的哈希值，找到table中该位置是否有Node，若无，则创建新的Node并存入；若有Node，则进入下一步。
- 如果Node为TreeNode，则在红黑树中插入节点。
- 如果Node为普通节点，则在链表中插入节点；如果链表过长，需要转换为红黑树。
- 如果原先已经存在相同的key，则实际上是替换value。
- 如果是插入新节点，则size加一，并判断是否需要扩容。
- afterNodeInsertion在hashmap中没有功能，不过在linkedHashMap中进行了实现，主要目的是为了维护容器存储的数据的顺序。

### get

```java
/**
* Implements Map.get and related methods.
*
* @param hash hash for key
* @param key the key
* @return the node, or null if none
*/
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

- 判断table是否非空，若为空，则返回null。
- 使用传入的hash值计算桶的位置，判断该位置上是否有值，若没有，返回null。
- 找到对应桶后，首先判断第一个节点是否是要取的值。
- 第一个节点不是的话，继续判断，此时要注意，桶中的节点Node可能是链表也可能是红黑树。
- 节点为链表时，遍历搜索，时间复杂度为O(n)；节点为红黑树，则调用getTreeNode方法进行搜索，与二叉树的搜索同理，因此时间复杂度为O(logn)。

### remove

```java
/**
* Implements Map.remove and related methods.
*
* @param hash hash for key
* @param key the key
* @param value the value to match if matchValue, else ignored
* @param matchValue if true only remove if value is equal
* @param movable if false do not move other nodes while removing
* @return the node, or null if none
*/
final Node<K,V> removeNode(int hash, Object key, Object value,
                            boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                            (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                                (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

- 使用与get相同的方法将相应的节点node找出。
- 若node为红黑树结构，调用removeTreeNode方法删除，即红黑树中删除节点的方法，详见红黑树讲解，在此不做深入分析。
- 若node为链表，则进行链表节点的删除。
- afterNodeRemoval在hashmap中没有功能，不过在linkedHashMap中进行了实现，主要目的是为了维护容器存储的数据的顺序。

### resize

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                    oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                    (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

该方法用于对map进行扩容。
- 判断原容量和阈值的大小，为0则进行初始化，过大则直接返回。
- 将原容量oldCap和阈值oldThr乘以二得到newCap和newThr，代码使用了移位操作，更高效。
- 创建一个新的容器（新数组newTab），长度设置为newCap。
- 对原容器（数组）进行遍历，对遍历到的节点，先判断其类型。
- 若为红黑树节点，调用split方法。
- 若为链表节点，则继续进行遍历，重新计算位置并放入。
- 计算位置的方法是，将同一桶中的元素根据(e.hash & oldCap)是否为0进行分割成两个不同的链表，为0就直接使用原索引，不为0说明hash值高位必不为0，在新数组中要向后移动oldCap的距离。


## 总结

利用散列表的特性，hashMap在存取数据时可以做到O(1)的时间复杂度，不过当出现哈希碰撞，在一个桶里面，之前的实现方法都是使用链表，这就导致了最坏的情况下，存取数据的时间复杂度会是O(n)，而在Java8实现中，当链表长度过长，会将其调整为红黑树，使得时间复杂度可以控制在O(logn)，从而优化了hashMap的性能。

Java7 与 Java8 中HashMap的对比：
- Java8为**红黑树**+链表+数组的形式
- hash值的计算方式不同
- Java7table在创建hashmap时分配空间，而1.8在put的时候分配，如果table为空，则为table分配空间。
- 在发生冲突，插入链中时，7是头插法，8是尾插法。
- 在resize操作中，7需要重新进行index的计算，而8不需要，通过判断相应的位是0还是1，要么依旧是原index，要么是oldCap + 原index


## 参考资料
- java源码

