---
layout:     post
title:      "设计模式之访问者模式"
subtitle:   ""
date:       2018-11-05
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 设计模式
    - 访问者模式
---

## 目录
- [简介](#简介)
- [代码实现](#代码实现)
- [准备过程时序图](#准备过程时序图)
- [访问过程时序图](#访问过程时序图)
- [访问者模式优缺点](#访问者模式优缺点)
- [参考资料](#参考资料)

## 简介
访问者模式适用于数据结构相对未定的系统，它把数据结构和作用于结构上的操作之间的耦合解脱开，使得操作集合可以相对自由地演化。访问者模式的简略图如下所示：

![简略图](../img/in-post/DP-visitor/pic1.png)

数据结构的每一个节点都可以接受一个访问者的调用，此节点向访问者对象传入节点对象，而访问者对象则反过来执行节点对象的操作。这样的过程叫做“双重分派”。节点调用访问者，将它自己传入，访问者则将某算法针对此节点执行。访问者模式的示意性类图如下所示：
![示意图](../img/in-post/DP-visitor/pic2.png)

访问者模式涉及到的角色如下：
- 抽象访问者(Visitor)角色：声明了一个或者多个方法操作，形成所有的具体访问者角色必须实现的接口。
- 具体访问者(ConcreteVisitor)角色：实现抽象访问者所声明的接口，也就是抽象访问者所声明的各个访问操作。
- 抽象节点(Node)角色：声明一个接受操作，接受一个访问者对象作为一个参数。
- 具体节点(ConcreteNode)角色：实现了抽象节点所规定的接受操作。
- 结构对象(ObjectStructure)角色：有如下的责任，可以遍历结构中的所有元素；如果需要，提供一个高层次的接口让访问者对象可以访问每一个元素；如果需要，可以设计成一个复合对象或者一个聚集，如List或Set。
## 代码实现
可以看到，抽象访问者角色为每一个具体节点都准备了一个访问操作。由于有两个节点，因此，对应就有两个访问操作。
```java
public interface Visitor {
    /**
     * 对应于NodeA的访问操作
     */
    public void visit(NodeA node);
    /**
     * 对应于NodeB的访问操作
     */
    public void visit(NodeB node);
}
　　//具体访问者VisitorA类
public class VisitorA implements Visitor {
    /**
     * 对应于NodeA的访问操作
     */
    @Override
    public void visit(NodeA node) {
        System.out.println(node.operationA());
    }
    /**
     * 对应于NodeB的访问操作
     */
    @Override
    public void visit(NodeB node) {
        System.out.println(node.operationB());
    }

}
　　//具体访问者VisitorB类
public class VisitorB implements Visitor {
    /**
     * 对应于NodeA的访问操作
     */
    @Override
    public void visit(NodeA node) {
        System.out.println(node.operationA());
    }
    /**
     * 对应于NodeB的访问操作
     */
    @Override
    public void visit(NodeB node) {
        System.out.println(node.operationB());
    }

}
　　//抽象节点类
public abstract class Node {
    /**
     * 接受操作
     */
    public abstract void accept(Visitor visitor);
}
　　//具体节点类NodeA
public class NodeA extends Node{
    /**
     * 接受操作
     */
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
    /**
     * NodeA特有的方法
     */
    public String operationA(){
        return "NodeA";
    }

}
　　//具体节点类NodeB
public class NodeB extends Node{
    /**
     * 接受方法
     */
    @Override
    public void accept(Visitor visitor) {
        visitor.visit(this);
    }
    /**
     * NodeB特有的方法
     */
    public String operationB(){
        return "NodeB";
    }
}
```
　　结构对象角色类，这个结构对象角色持有一个聚集，并向外界提供add()方法作为对聚集的管理操作。通过调用这个方法，可以动态地增加一个新的节点。
```java
public class ObjectStructure {
    
    private List<Node> nodes = new ArrayList<Node>();
    
    /**
     * 执行方法操作
     */
    public void action(Visitor visitor){
        
        for(Node node : nodes)
        {
            node.accept(visitor);
        }
        
    }
    /**
     * 添加一个新元素
     */
    public void add(Node node){
        nodes.add(node);
    }
}
　　//客户端类
public class Client {

    public static void main(String[] args) {
        //创建一个结构对象
        ObjectStructure os = new ObjectStructure();
        //给结构增加一个节点
        os.add(new NodeA());
        //给结构增加一个节点
        os.add(new NodeB());
        //创建一个访问者
        Visitor visitor = new VisitorA();
        os.action(visitor);
    }
}
```
虽然在这个示意性的实现里并没有出现一个复杂的具有多个树枝节点的对象树结构，但是，在实际系统中访问者模式通常是用来处理复杂的对象树结构的，而且访问者模式可以用来处理跨越多个等级结构的树结构问题。这正是访问者模式的功能强大之处。
## 准备过程时序图
![准备过程](../img/in-post/DP-visitor/pic4.png)
首先，这个示意性的客户端创建了一个结构对象，然后将一个新的NodeA对象和一个新的NodeB对象传入。

其次，客户端创建了一个VisitorA对象，并将此对象传给结构对象。

然后，客户端调用结构对象聚集管理方法，将NodeA和NodeB节点加入到结构对象中去。

最后，客户端调用结构对象的行动方法action()，启动访问过程。
　　
## 访问过程时序图
![访问过程](../img/in-post/DP-visitor/pic4.png)
结构对象会遍历它自己所保存的聚集中的所有节点，在本系统中就是节点NodeA和NodeB。首先NodeA会被访问到，这个访问是由以下的操作组成的：

（1）NodeA对象的接受方法accept()被调用，并将VisitorA对象本身传入；

（2）NodeA对象反过来调用VisitorA对象的访问方法，并将NodeA对象本身传入；

（3）VisitorA对象调用NodeA对象的特有方法operationA()。

从而就完成了双重分派过程，接着，NodeB会被访问，这个访问的过程和NodeA被访问的过程是一样的，这里不再叙述。

## 访问者模式优缺点
### 优点：
- 好的扩展性：能够在不修改对象结构中的元素的情况下，为对象结构中的元素添加新的功能。
- 好的复用性：可以通过访问者来定义整个对象结构通用的功能，从而提高复用程度。
- 分离无关行为：可以通过访问者来分离无关的行为，把相关的行为封装在一起，构成一个访问者，这样每一个访问者的功能都比较单一。
### 缺点：
- 对象结构变化很困难：不适用于对象结构中的类经常变化的情况，因为对象结构发生了改变，访问者的接口和访问者的实现都要发生相应的改变，代价太高。
- 破坏封装：访问者模式通常需要对象结构开放内部数据给访问者和ObjectStructrue，这破坏了对象的封装性。
## 参考资料


