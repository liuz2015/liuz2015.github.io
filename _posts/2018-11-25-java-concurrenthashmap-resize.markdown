---
layout:     post
title:      "【转】Java8 concurrentHashMap扩容深入分析"
subtitle:   ""
date:       2018-11-25
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Java8
    - concurrentHashMap
    - 并发
    - 扩容
---

> 转自：https://www.jianshu.com/p/f6730d5784ad 

## 目录
- [什么情况会触发扩容](#什么情况会触发扩容)
- [transfer实现](#transfer实现)
- [参考资料](#参考资料)

ConcurrentHashMap相关的文章写了不少，有个遗留问题一直没有分析，也被好多人请教过，被搁置在一旁，即如何在并发的情况下实现数组的扩容。

## 什么情况会触发扩容

当往hashMap中成功插入一个key/value节点时，有可能触发扩容动作：

1、如果新增节点之后，所在链表的元素个数达到了阈值 8，则会调用treeifyBin方法把链表转换成红黑树，不过在结构转换之前，会对数组长度进行判断，实现如下：

![pic1](../img/in-post/java-concurrenthashmap-resize/pic1.png)

如果数组长度n小于阈值MIN_TREEIFY_CAPACITY，默认是64，则会调用tryPresize方法把数组长度扩大到原来的两倍，并触发transfer方法，重新调整节点的位置。

![pic2](../img/in-post/java-concurrenthashmap-resize/pic2.png)

2、新增节点之后，会调用addCount方法记录元素个数，并检查是否需要进行扩容，当数组元素个数达到阈值时，会触发transfer方法，重新调整节点的位置。

![pic3](../img/in-post/java-concurrenthashmap-resize/pic3.png)

## transfer实现
transfer方法实现了在并发的情况下，高效的从原始组数往新数组中移动元素，假设扩容之前节点的分布如下，这里区分蓝色节点和红色节点，是为了后续更好的分析：

![pic4](../img/in-post/java-concurrenthashmap-resize/pic4.png)

在上图中，第14个槽位插入新节点之后，链表元素个数已经达到了8，且数组长度为16，优先通过扩容来缓解链表过长的问题，实现如下：
1、根据当前数组长度n，新建一个两倍长度的数组nextTable；

![pic5](../img/in-post/java-concurrenthashmap-resize/pic5.png)

2、初始化ForwardingNode节点，其中保存了新数组nextTable的引用，在处理完每个槽位的节点之后当做占位节点，表示该槽位已经处理过了；

![pic6](../img/in-post/java-concurrenthashmap-resize/pic6.png)

3、通过for自循环处理每个槽位中的链表元素，默认advace为真，通过CAS设置transferIndex属性值，并初始化i和bound值，i指当前处理的槽位序号，bound指需要处理的槽位边界，先处理槽位15的节点；

![pic7](../img/in-post/java-concurrenthashmap-resize/pic7.png)

4、在当前假设条件下，槽位15中没有节点，则通过CAS插入在第二步中初始化的ForwardingNode节点，用于告诉其它线程该槽位已经处理过了；

![pic8](../img/in-post/java-concurrenthashmap-resize/pic8.png)

5、如果槽位15已经被线程A处理了，那么线程B处理到这个节点时，取到该节点的hash值应该为MOVED，值为-1，则直接跳过，继续处理下一个槽位14的节点；

![pic14](../img/in-post/java-concurrenthashmap-resize/pic14.png)

6、处理槽位14的节点，是一个链表结构，先定义两个变量节点ln和hn，按我的理解应该是lowNode和highNode，分别保存hash值的第X位为0和1的节点，具体实现如下：

![pic9](../img/in-post/java-concurrenthashmap-resize/pic9.png)

使用fn&n可以快速把链表中的元素区分成两类，A类是hash值的第X位为0，B类是hash值的第X位为1，并通过lastRun记录最后需要处理的节点，A类和B类节点可以分散到新数组的槽位14和30中，在原数组的槽位14中，蓝色节点第X为0，红色节点第X为1，把链表拉平显示如下：

![pic10](../img/in-post/java-concurrenthashmap-resize/pic10.png)

1、通过遍历链表，记录runBit和lastRun，分别为1和节点6，所以设置hn为节点6，ln为null；
2、重新遍历链表，以lastRun节点为终止条件，根据第X位的值分别构造ln链表和hn链表：
ln链：和原来链表相比，顺序已经不一样了

![pic11](../img/in-post/java-concurrenthashmap-resize/pic11.png)

hn链：

![pic12](../img/in-post/java-concurrenthashmap-resize/pic12.png)

通过CAS把ln链表设置到新数组的i位置，hn链表设置到i+n的位置；
7、如果该槽位是红黑树结构，则构造树节点lo和hi，遍历红黑树中的节点，同样根据hash&n算法，把节点分为两类，分别插入到lo和hi为头的链表中，根据lo和hi链表中的元素个数分别生成ln和hn节点，其中ln节点的生成逻辑如下：
（1）如果lo链表的元素个数小于等于UNTREEIFY_THRESHOLD，默认为6，则通过untreeify方法把树节点链表转化成普通节点链表；
（2）否则判断hi链表中的元素个数是否等于0：如果等于0，表示lo链表中包含了所有原始节点，则设置原始红黑树给ln，否则根据lo链表重新构造红黑树。

![pic13](../img/in-post/java-concurrenthashmap-resize/pic13.png)

最后，同样的通过CAS把ln设置到新数组的i位置，hn设置到i+n位置。

## 参考资料
- https://www.jianshu.com/p/f6730d5784ad

