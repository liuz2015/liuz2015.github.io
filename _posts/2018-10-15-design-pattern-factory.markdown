---
layout:     post
title:      "设计模式之工厂模式"
subtitle:   ""
date:       2018-10-15
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 设计模式
    - 工厂模式
---

## 目录
- [简介](#简介)
- [简单工厂](#简单工厂)
- [工厂方法](#工厂方法)
- [抽象工厂](#抽象工厂)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介

简单工厂，工厂方法，抽象工厂都属于设计模式中的创建型模式。其主要功能都是帮助我们把对象的实例化部分抽取了出来，优化了系统的架构，并且增强了系统的扩展性。

最常使用到的就是与数据库的连接中，SQLSessionFactory根据你传入的不同数据库驱动，以及用户名、密码等，返回它加工好的SqlSession给你使用。

 
## 简单工厂
简单工厂模式的工厂类一般是使用静态方法，通过接收的参数的不同来返回不同的对象实例。
![简单工厂](../img/in-post/DP-factory/simpleFactory.jpeg)
只有一个工厂，生产扩展/继承自同一类的多种产品。

不修改代码的话，是无法扩展的。

必须要修改工厂类的代码，同时增加新的产品类，才能获得新种类的对象。

代码实现：
```java
abstract class Car {
 
    public abstract void run();
 
}
//实现接口
class Audi extends Car {
    @Override
    public void run() {
        System.out.println("奥迪跑的快");
    }
}
class Byd extends Car {
    @Override
    public void run() {
        System.out.println("比亚迪跑的慢");
    }
}
 
//接下来需要解决的就是对象的创建问既如何根据不同的情况创建不同的对象：我们正好可以通过简单工厂模式实现
class CarFactory {
 
    public static Car createCar(String type){
        switch (type){
            case "audi" :  return new Audi();
            case "byd" :  return new Byd();
            default:break;
        }
        return null;
    }
}
//调用：
public class SimpleFactory {
    public static void main(String[] args) {
        Car c1 =CarFactory.createCar("audi");
        Car c2 = CarFactory.createCar("byd");
 
        c1.run();
        c2.run();
 
    }
}
```
 
## 工厂方法
工厂方法是针对每一种产品提供一个工厂类。通过不同的工厂实例来创建不同的产品实例。
![工厂方法](../img/in-post/DP-factory/factoryMethod.jpeg)
在同一等级结构中，支持增加任意产品。

可以通过实现工厂方法接口来创建新的工厂方法类，从而达到增加新产品类的目的。

代码实现：
```java
interface CarFactory{
   Car createCar();
}
interface Car{
    void run();
}
//实现工厂类
class AudiFactory implements CarFactory {
    @Override
    public Car createCar() {
        return new Audi();
    }
}
class BydFactory implements CarFactory {
    @Override
    public Car createCar() {
        return new Byd();
    }
}
//实现产品类
class Audi implements Car {
    @Override
    public void run() {
        System.out.println("奥迪再跑！");
    }
}
class Byd implements Car {
    @Override
    public void run() {
        System.out.println("比亚迪再跑！");
    }
}
 
public class FactoryMethod {
    public static void main(String[] args) {
        Car c1 = new AudiFactory().createCar();
        Car c2 = new BydFactory().createCar();
 
        c1.run();
        c2.run();
    }
 
}
```
 
## 抽象工厂
抽象工厂是应对产品族（若一个族只有一个产品，则与工厂方法无异了）概念的。
![抽象工厂](../img/in-post/DP-factory/abstractFactory.jpeg)
比如说，每个汽车公司可能要同时生产轿车，货车，客车，那么每一个工厂都要有创建轿车，货车和客车的方法。

应对产品族概念而生，增加新的产品线很容易，但是无法增加新的产品。

实现抽象工厂接口创建新的工厂类，从而增加新的产品线。

代码实现：
```java
interface CarFactory {
    Engine createEngine();
    Seat createSeat();
}
interface Engine {
    void run();
    void start();
}
interface Seat {
    void massage();
}
//低配车
class LowCarFactory implements CarFactory {
    @Override
    public Engine createEngine() {
        return new LowEngine();
    }
    @Override
    public Seat createSeat() {
        return new LowSeat();
    }
}
//高配车
class LuxuryCarFactory implements CarFactory {
    @Override
    public Engine createEngine() {
        return new LuxuryEngine();
    }
    @Override
    public Seat createSeat() {
        return new LuxurySeat();
    }
}
 
//配件
class LuxuryEngine implements Engine{
    @Override
    public void run() {
        System.out.println("转的快！");
    }
    @Override
    public void start() {
        System.out.println("启动快!可以自动启停！");
    }
}
 
class LowEngine implements Engine{
    @Override
    public void run() {
        System.out.println("转的慢！");
    }
    @Override
    public void start() {
        System.out.println("启动慢!");
    }
}
class LuxurySeat implements Seat {
    @Override
    public void massage() {
        System.out.println("可以自动按摩！");
    }
}
class LowSeat implements Seat {
    @Override
    public void massage() {
        System.out.println("不能按摩！");
    }
}
public class AbstractFactory {
    public static void main(String[] args) {
        CarFactory  factory = new LuxuryCarFactory();
        Engine e = factory.createEngine();
        e.run();
        e.start();
    }
}
```

## 总结
- 工厂模式中，重要的是工厂类，而不是产品类。产品类可以是多种形式，多层继承或者是单个类都是可以的。但要明确的，工厂模式的接口只会返回一种类型的实例，这是在设计产品类的时候需要注意的，最好是有父类或者共同实现的接口。
- 使用工厂模式，返回的实例一定是工厂创建的，而不是从其他对象中获取的。
- 工厂模式返回的实例可以不是新创建的，返回由工厂创建好的实例也是可以的。
 
### 区别
简单工厂：用来生产同一等级结构中的任意产品。（对于增加新的产品，无能为力）

工厂方法：用来生产同一等级结构中的固定产品。（支持增加任意产品）

抽象工厂：用来生产不同产品族的全部产品。（对于增加新的产品，无能为力；支持增加产品族）  
 
以上三种工厂 方法在等级结构和产品族这两个方向上的支持程度不同。所以要根据情况考虑应该使用哪种方法。  


## 参考资料
- https://blog.csdn.net/superbeck/article/details/4446177
- https://blog.csdn.net/zz_15127160921/article/details/81221685

