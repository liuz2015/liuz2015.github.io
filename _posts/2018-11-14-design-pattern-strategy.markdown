---
layout:     post
title:      "【转】设计模式之策略模式"
subtitle:   ""
date:       2018-11-14
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 设计模式
    - 策略模式
---

> 转自：https://www.cnblogs.com/lewis0077/p/5133812.html 

## 目录
- [简介](#简介)
  - [定义](#定义)
  - [结构](#结构)
  - [UML类图](#UML类图)
  - [UML时序图](#UML时序图)
  - [代码实现](#代码实现)
- [深入理解](#深入理解)
- [策略模式在JDK中的应用](#策略模式在JDK中的应用)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介
在讲策略模式之前，我们先看一个日常生活中的小例子：

现实生活中我们到商场买东西的时候，卖场往往根据不同的客户制定不同的报价策略，比如针对新客户不打折扣，针对老客户打9折，针对VIP客户打8折。

现在我们要做一个报价管理的模块，简要点就是要针对不同的客户，提供不同的折扣报价。

如果是有你来做，你会怎么做？

我们很有可能写出下面的代码：
```java
package strategy.examp02;

import java.math.BigDecimal;

public class QuoteManager {

    public BigDecimal quote(BigDecimal originalPrice,String customType){
        if ("新客户".equals(customType)) {
            System.out.println("抱歉！新客户没有折扣！");
            return originalPrice;
        }else if ("老客户".equals(customType)) {
            System.out.println("恭喜你！老客户打9折！");
            originalPrice = originalPrice.multiply(new BigDecimal(0.9)).setScale(2,BigDecimal.ROUND_HALF_UP);
            return originalPrice;
        }else if("VIP客户".equals(customType)){
            System.out.println("恭喜你！VIP客户打8折！");
            originalPrice = originalPrice.multiply(new BigDecimal(0.8)).setScale(2,BigDecimal.ROUND_HALF_UP);
            return originalPrice;
        }
        //其他人员都是原价
        return originalPrice;
    }
}
```

经过测试，上面的代码工作的很好，可是上面的代码是有问题的。上面存在的问题：把不同客户的报价的算法都放在了同一个方法里面，使得该方法很是庞大(现在是只是一个演示，所以看起来还不是很臃肿)。

下面看一下上面的改进，我们把不同客户的报价的算法都单独作为一个方法
 ```java
package strategy.examp02;

import java.math.BigDecimal;

public class QuoteManagerImprove {

    public BigDecimal quote(BigDecimal originalPrice, String customType){
        if ("新客户".equals(customType)) {
            return this.quoteNewCustomer(originalPrice);
        }else if ("老客户".equals(customType)) {
            return this.quoteOldCustomer(originalPrice);
        }else if("VIP客户".equals(customType)){
            return this.quoteVIPCustomer(originalPrice);
        }
        //其他人员都是原价
        return originalPrice;
    }

    /**
     * 对VIP客户的报价算法
     * @param originalPrice 原价
     * @return 折后价
     */
    private BigDecimal quoteVIPCustomer(BigDecimal originalPrice) {
        System.out.println("恭喜！VIP客户打8折");
        originalPrice = originalPrice.multiply(new BigDecimal(0.8)).setScale(2,BigDecimal.ROUND_HALF_UP);
        return originalPrice;
    }

    /**
     * 对老客户的报价算法
     * @param originalPrice 原价
     * @return 折后价
     */
    private BigDecimal quoteOldCustomer(BigDecimal originalPrice) {
        System.out.println("恭喜！老客户打9折");
        originalPrice = originalPrice.multiply(new BigDecimal(0.9)).setScale(2,BigDecimal.ROUND_HALF_UP);
        return originalPrice;
    }

    /**
     * 对新客户的报价算法
     * @param originalPrice 原价
     * @return 折后价
     */
    private BigDecimal quoteNewCustomer(BigDecimal originalPrice) {
        System.out.println("抱歉！新客户没有折扣！");
        return originalPrice;
    }

}
```
 
上面的代码比刚开始的时候要好一点，它把每个具体的算法都单独抽出来作为一个方法，当某一个具体的算法有了变动的时候，只需要修改响应的报价算法就可以了。

但是改进后的代码还是有问题的，那有什么问题呢？

1.当我们新增一个客户类型的时候，首先要添加一个该种客户类型的报价算法方法，然后再quote方法中再加一个else if的分支，是不是感觉很是麻烦呢？而且这也违反了设计原则之一的开闭原则（open-closed-principle）。

**开闭原则：**
对于扩展是开放的（Open for extension）。这意味着模块的行为是可以扩展的。当应用的需求改变时，我们可以对模块进行扩展，使其具有满足那些改变的新行为。也就是说，我们可以改变模块的功能。

对于修改是关闭的（Closed for modification）。对模块行为进行扩展时，不必改动模块的源代码或者二进制代码。

2.我们经常会面临这样的情况，不同的时期使用不同的报价规则，比如在各个节假日举行的各种促销活动时、商场店庆时往往都有普遍的折扣，但是促销时间一旦过去，报价就要回到正常价格上来。按照上面的代码我们就得修改if-else里面的代码很是麻烦。

那有没有什么办法使得我们的报价管理即可扩展、可维护，又可以方便的响应变化呢？当然有解决方案啦，就是我们下面要讲的策略模式。


### 定义
策略模式定义了一系列的算法，并将每一个算法封装起来，使每个算法可以相互替代，使算法本身和使用算法的客户端分割开来，相互独立。

### 结构
1.策略接口角色IStrategy:用来约束一系列具体的策略算法，策略上下文角色ConcreteStrategy使用此策略接口来调用具体的策略所实现的算法。

2.具体策略实现角色ConcreteStrategy:具体的策略实现，即具体的算法实现。

3.策略上下文角色StrategyContext:策略上下文，负责和具体的策略实现交互，通常策略上下文对象会持有一个真正的策略实现对象，策略上下文还可以让具体的策略实现从其中获取相关数据，回调策略上下文对象的方法。
### UML类图
![pic1](../img/in-post/DP-strategy/pic1.png)

### UML序列图
![pic2](../img/in-post/DP-strategy/pic2.png)

### 代码实现
策略接口：
```java
package strategy.examp01;

//策略接口
public interface IStrategy {
    //定义的抽象算法方法 来约束具体的算法实现方法
    public void algorithmMethod();
}
```

具体的策略：
```java
package strategy.examp01;

// 具体的策略实现
public class ConcreteStrategy implements IStrategy {
    //具体的算法实现
    @Override
    public void algorithmMethod() {
        System.out.println("this is ConcreteStrategy method...");
    }
}

package strategy.examp01;

 // 具体的策略实现2
public class ConcreteStrategy2 implements IStrategy {
     //具体的算法实现
    @Override
    public void algorithmMethod() {
        System.out.println("this is ConcreteStrategy2 method...");
    }
}
```
策略上下文：
```java
package strategy.examp01;

/**
 * 策略上下文
 */
public class StrategyContext {
    //持有一个策略实现的引用
    private IStrategy strategy;
    //使用构造器注入具体的策略类
    public StrategyContext(IStrategy strategy) {
        this.strategy = strategy;
    }

    public void contextMethod(){
        //调用策略实现的方法
        strategy.algorithmMethod();
    }
}
```
外部客户端：
```java
package strategy.examp01;

//外部客户端
public class Client {
    public static void main(String[] args) {
        //1.创建具体测策略实现
        IStrategy strategy = new ConcreteStrategy2();
        //2.在创建策略上下文的同时，将具体的策略实现对象注入到策略上下文当中
        StrategyContext ctx = new StrategyContext(strategy);
        //3.调用上下文对象的方法来完成对具体策略实现的回调
        ctx.contextMethod();
    }
}
```
 
针对我们一开始讲的报价管理的例子：我们可以应用策略模式对其进行改造，不同类型的客户有不同的折扣，我们可以将不同类型的客户的报价规则都封装为一个独立的算法，然后抽象出这些报价算法的公共接口。

公共报价策略接口：
```java
package strategy.examp02;

import java.math.BigDecimal;
//报价策略接口
public interface IQuoteStrategy {
    //获取折后价的价格
    BigDecimal getPrice(BigDecimal originalPrice);
}
```
新客户报价策略实现：
```java
package strategy.examp02;

import java.math.BigDecimal;
//新客户的报价策略实现类
public class NewCustomerQuoteStrategy implements IQuoteStrategy {
    @Override
    public BigDecimal getPrice(BigDecimal originalPrice) {
        System.out.println("抱歉！新客户没有折扣！");
        return originalPrice;
    }
}
```
老客户报价策略实现：
```java
package strategy.examp02;

import java.math.BigDecimal;
//老客户的报价策略实现
public class OldCustomerQuoteStrategy implements IQuoteStrategy {
    @Override
    public BigDecimal getPrice(BigDecimal originalPrice) {
        System.out.println("恭喜！老客户享有9折优惠！");
        originalPrice = originalPrice.multiply(new BigDecimal(0.9)).setScale(2,BigDecimal.ROUND_HALF_UP);
        return originalPrice;
    }
}
```
VIP客户报价策略实现：
```java
package strategy.examp02;

import java.math.BigDecimal;
//VIP客户的报价策略实现
public class VIPCustomerQuoteStrategy implements IQuoteStrategy {
    @Override
    public BigDecimal getPrice(BigDecimal originalPrice) {
        System.out.println("恭喜！VIP客户享有8折优惠！");
        originalPrice = originalPrice.multiply(new BigDecimal(0.8)).setScale(2,BigDecimal.ROUND_HALF_UP);
        return originalPrice;
    }
}
```
报价上下文：
```java
package strategy.examp02;

import java.math.BigDecimal;
//报价上下文角色
public class QuoteContext {
    //持有一个具体的报价策略
    private IQuoteStrategy quoteStrategy;

    //注入报价策略
    public QuoteContext(IQuoteStrategy quoteStrategy){
        this.quoteStrategy = quoteStrategy;
    }

    //回调具体报价策略的方法
    public BigDecimal getPrice(BigDecimal originalPrice){
        return quoteStrategy.getPrice(originalPrice);
    }
}
```
外部客户端：
```java
package strategy.examp02;

import java.math.BigDecimal;
//外部客户端
public class Client {
    public static void main(String[] args) {
        //1.创建老客户的报价策略
        IQuoteStrategy oldQuoteStrategy = new OldCustomerQuoteStrategy();

        //2.创建报价上下文对象，并设置具体的报价策略
        QuoteContext quoteContext = new QuoteContext(oldQuoteStrategy);

        //3.调用报价上下文的方法
        BigDecimal price = quoteContext.getPrice(new BigDecimal(100));

        System.out.println("折扣价为：" +price);
    }
}
```
控制台输出：

```
恭喜！老客户享有9折优惠！
折扣价为：90.00
```
 
这个时候，商场营销部新推出了一个客户类型--MVP用户（Most Valuable Person），可以享受折扣7折优惠，那该怎么办呢？

这个很容易，只要新增一个报价策略的实现，然后外部客户端调用的时候，创建这个新增的报价策略实现，并设置到策略上下文就可以了，对原来已经实现的代码没有任何的改动。

MVP用户的报价策略实现：
```java
package strategy.examp02;

import java.math.BigDecimal;
//MVP客户的报价策略实现
public class MVPCustomerQuoteStrategy implements IQuoteStrategy {
    @Override
    public BigDecimal getPrice(BigDecimal originalPrice) {
        System.out.println("哇偶！MVP客户享受7折优惠！！！");
        originalPrice = originalPrice.multiply(new BigDecimal(0.7)).setScale(2,BigDecimal.ROUND_HALF_UP);
        return originalPrice;
    }
}
```
外部客户端：
```java
package strategy.examp02;

import java.math.BigDecimal;
//外部客户端
public class Client {
    public static void main(String[] args) {
        //创建MVP客户的报价策略
        IQuoteStrategy mvpQuoteStrategy = new MVPCustomerQuoteStrategy();

        //创建报价上下文对象，并设置具体的报价策略
        QuoteContext quoteContext = new QuoteContext(mvpQuoteStrategy);

        //调用报价上下文的方法
        BigDecimal price = quoteContext.getPrice(new BigDecimal(100));

        System.out.println("折扣价为：" +price);
    }
}
```
控制台输出：
```
哇偶！MVP客户享受7折优惠！！！
折扣价为：70.00
 ```
## 深入理解

**策略模式的作用**：就是把具体的算法实现从业务逻辑中剥离出来，成为一系列独立算法类，使得它们可以相互替换。

**策略模式的着重点**：不是如何来实现算法，而是如何组织和调用这些算法，从而让我们的程序结构更加的灵活、可扩展。

我们前面的第一个报价管理的示例，发现每个策略算法实现对应的都是在QuoteManager 中quote方法中的if else语句里面，我们知道if else if语句里面的代码在执行的可能性方面可以说是平等的，你要么执行if,要么执行else,要么执行else if。

策略模式就是把各个平等的具体实现进行抽象、封装成为独立的算法类，然后通过上下文和具体的算法类来进行交互。各个策略算法都是平等的，地位是一样的，正是由于各个算法的平等性，所以它们才是可以相互替换的。虽然我们可以动态的切换各个策略，但是同一时刻只能使用一个策略。

在这个点上，我们举个历史上有名的故事作为示例：

三国刘备取西川时，谋士庞统给的上、中、下三个计策：

上策：挑选精兵，昼夜兼行直接偷袭成都，可以一举而定，此为上计计也。

中策：杨怀、高沛是蜀中名将，手下有精锐部队，而且据守关头，我们可以装作要回荆州，引他们轻骑来见，可就此将其擒杀，而后进兵成都，此为中计。

下策：退还白帝，连引荆州，慢慢进图益州，此为下计。

这三个计策都是取西川的计策，也就是攻取西川这个问题的具体的策略算法，刘备可以采用上策，可以采用中策，当然也可以采用下策，由此可见策略模式的各种具体的策略算法都是平等的，可以相互替换。

那谁来选择具体采用哪种计策（算法）？

在这个故事中当然是刘备选择了，也就是外部的客户端选择使用某个具体的算法，然后把该算法（计策）设置到上下文当中；

还有一种情况就是客户端不选择具体的算法，把这个事交给上下文，这相当于刘备说我不管有哪些攻取西川的计策，我只要结果（成功的拿下西川），具体怎么攻占（有哪些计策，怎么选择）由参谋部来决定（上下文）。

下面我们演示下这种情景：
```java
//攻取西川的策略
public interface IOccupationStrategyWestOfSiChuan {
    public void occupationWestOfSiChuan(String msg);
}

//攻取西川的上上计策
public class UpperStrategy implements IOccupationStrategyWestOfSiChuan {
    @Override
    public void occupationWestOfSiChuan(String msg) {
        if (msg == null || msg.length() < 5) {
            //故意设置障碍，导致上上计策失败
            System.out.println("由于计划泄露，上上计策失败！");
            int i = 100/0;
        }
        System.out.println("挑选精兵，昼夜兼行直接偷袭成都，可以一举而定,此为上计计也!");
    }
}


//攻取西川的中计策
public class MiddleStrategy implements IOccupationStrategyWestOfSiChuan {
    @Override
    public void occupationWestOfSiChuan(String msg) {
        System.out.println("杨怀、高沛是蜀中名将，手下有精锐部队，而且据守关头，我们可以装作要回荆州，引他们轻骑来见，可就此将其擒杀，而后进兵成都，此为中计。");
    }
}

//攻取西川的下计策
public class LowerStrategy implements IOccupationStrategyWestOfSiChuan {
    @Override
    public void occupationWestOfSiChuan(String msg) {
        System.out.println("退还白帝，连引荆州，慢慢进图益州，此为下计。");
    }
}


//攻取西川参谋部，就是上下文啦，由上下文来选择具体的策略
public class OccupationContext  {

    public void occupationWestOfSichuan(String msg){
        //先用上上计策
        IOccupationStrategyWestOfSiChuan strategy = new UpperStrategy();
        try {
            strategy.occupationWestOfSiChuan(msg);
        } catch (Exception e) {
            //上上计策有问题行不通之后，用中计策
            strategy = new MiddleStrategy();
            strategy.occupationWestOfSiChuan(msg);
        }
    }
}


//此时外部客户端相当于刘备了,不管具体采用什么计策，只要结果（成功的攻下西川）
public class Client {

    public static void main(String[] args) {
        OccupationContext context = new  OccupationContext();
        //这个给手下的人激励不够啊
        context.occupationWestOfSichuan("拿下西川");
        System.out.println("=========================");
        //这个人人有赏，让士兵有动力啊
        context.occupationWestOfSichuan("拿下西川之后，人人有赏！");
    }
}
```

控制台输出：
```
由于计划泄露，上上计策失败！
杨怀、高沛是蜀中名将，手下有精锐部队，而且据守关头，我们可以装作要回荆州，引他们轻骑来见，可就此将其擒杀，而后进兵成都，此为中计。
=========================
挑选精兵，昼夜兼行直接偷袭成都，可以一举而定,此为上计计也!
```

我们上面的策略接口采用的是接口的形式来定义的，其实这个策略接口，是广义上的接口，不是语言层面的interface,也可以是一个抽象类，如果多个算法具有公有的数据，则可以将策略接口设计为一个抽象类，把公共的东西放到抽象类里面去。
 

### 策略和上下文的关系：

在策略模式中，一般情况下都是上下文持有策略的引用，以进行对具体策略的调用。但具体的策略对象也可以从上下文中获取所需数据，可以将上下文当做参数传入到具体策略中，具体策略通过回调上下文中的方法来获取其所需要的数据。

下面我们演示这种情况：

在跨国公司中，一般都会在各个国家和地区设置分支机构，聘用当地人为员工，这样就有这样一个需要：每月发工资的时候，中国国籍的员工要发人民币，美国国籍的员工要发美元，英国国籍的要发英镑。
```java
//支付策略接口
public interface PayStrategy {
    //在支付策略接口的支付方法中含有支付上下文作为参数，以便在具体的支付策略中回调上下文中的方法获取数据
    public void pay(PayContext ctx);
}

//人民币支付策略
public class RMBPay implements PayStrategy {
    @Override
    public void pay(PayContext ctx) {
        System.out.println("现在给："+ctx.getUsername()+" 人民币支付 "+ctx.getMoney()+"元！");
    }
}

//美金支付策略
public class DollarPay implements PayStrategy {
    @Override
    public void pay(PayContext ctx) {
        System.out.println("现在给："+ctx.getUsername()+" 美金支付 "+ctx.getMoney()+"dollar !");
    }
}

//支付上下文,含有多个算法的公有数据
public class PayContext {
    //员工姓名
    private String username;
    //员工的工资
    private double money;
    //支付策略
    private PayStrategy payStrategy;

    public void pay(){
        //调用具体的支付策略来进行支付
        payStrategy.pay(this);
    }

    public PayContext(String username, double money, PayStrategy payStrategy) {
        this.username = username;
        this.money = money;
        this.payStrategy = payStrategy;
    }

    public String getUsername() {
        return username;
    }

    public double getMoney() {
        return money;
    }
}

//外部客户端
public class Client {
    public static void main(String[] args) {
        //创建具体的支付策略
        PayStrategy rmbStrategy = new RMBPay();
        PayStrategy dollarStrategy = new DollarPay();
        //准备小王的支付上下文
        PayContext ctx = new PayContext("小王",30000,rmbStrategy);
        //向小王支付工资
        ctx.pay();

        //准备Jack的支付上下文
        ctx = new PayContext("jack",10000,dollarStrategy);
        //向Jack支付工资
        ctx.pay();
    }
}
```

控制台输出：
```
现在给：小王 人民币支付 30000.0元！
现在给：jack 美金支付 10000.0dollar !
 ```
那现在我们要新增一个银行账户的支付策略，该怎么办呢？

显然我们应该新增一个支付找银行账户的策略实现，由于需要从上下文中获取数据，为了不修改已有的上下文，我们可以通过继承已有的上下文来扩展一个新的带有银行账户的上下文，然后再客户端中使用新的策略实现和带有银行账户的上下文，这样之前已有的实现完全不需要改动，遵守了开闭原则。
```java
//银行账户支付
public class AccountPay implements PayStrategy {
    @Override
    public void pay(PayContext ctx) {
        PayContextWithAccount ctxAccount = (PayContextWithAccount) ctx;
        System.out.println("现在给："+ctxAccount.getUsername()+"的账户："+ctxAccount.getAccount()+" 支付工资："+ctxAccount.getMoney()+" 元！");
    }
}

//带银行账户的支付上下文
public class PayContextWithAccount extends PayContext {
    //银行账户
    private String account;
    public PayContextWithAccount(String username, double money, PayStrategy payStrategy,String account) {
        super(username, money, payStrategy);
        this.account = account;
    }

    public String getAccount() {
        return account;
    }
}

//外部客户端
public class Client {
    public static void main(String[] args) {
        //创建具体的支付策略
        PayStrategy rmbStrategy = new RMBPay();
        PayStrategy dollarStrategy = new DollarPay();
        //准备小王的支付上下文
        PayContext ctx = new PayContext("小王",30000,rmbStrategy);
        //向小王支付工资
        ctx.pay();
        //准备Jack的支付上下文
        ctx = new PayContext("jack",10000,dollarStrategy);
        //向Jack支付工资
        ctx.pay();
        //创建支付到银行账户的支付策略
        PayStrategy accountStrategy = new AccountPay();
        //准备带有银行账户的上下文
        ctx = new PayContextWithAccount("小张",40000,accountStrategy,"1234567890");
        //向小张的账户支付
        ctx.pay();
    }
}
```
控制台输出：
```
现在给：小王 人民币支付 30000.0元！
现在给：jack 美金支付 10000.0dollar !
现在给：小张的账户：1234567890 支付工资：40000.0 元！
 ```

除了上面的方法，还有其他的实现方式吗？

当然有了，上面的实现方式是策略实现所需要的数据都是从上下文中获取，因此扩展了上下文；现在我们可以不扩展上下文，直接从策略实现内部来获取数据，看下面的实现：
```java
//支付到银行账户的策略
public class AccountPay2 implements PayStrategy {
    //银行账户
    private String account;
    public AccountPay2(String account) {
        this.account = account;
    }
    @Override
    public void pay(PayContext ctx) {
        System.out.println("现在给："+ctx.getUsername()+"的账户："+getAccount()+" 支付工资："+ctx.getMoney()+" 元！");
    }
    public String getAccount() {
        return account;
    }
    public void setAccount(String account) {
        this.account = account;
    }
}

//外部客户端
public class Client {
    public static void main(String[] args) {
        //创建具体的支付策略
        PayStrategy rmbStrategy = new RMBPay();
        PayStrategy dollarStrategy = new DollarPay();
        //准备小王的支付上下文
        PayContext ctx = new PayContext("小王",30000,rmbStrategy);
        //向小王支付工资
        ctx.pay();
        //准备Jack的支付上下文
        ctx = new PayContext("jack",10000,dollarStrategy);
        //向Jack支付工资
        ctx.pay();
        //创建支付到银行账户的支付策略
        PayStrategy accountStrategy = new AccountPay2("1234567890");
        //准备上下文
        ctx = new PayContext("小张",40000,accountStrategy);
        //向小张的账户支付
        ctx.pay();
    }
}
```
控制台输出：
```
现在给：小王 人民币支付 30000.0元！
现在给：jack 美金支付 10000.0dollar !
现在给：小张的账户：1234567890 支付工资：40000.0 元！
```
那我们来比较一下上面两种实现方式：

### 扩展上下文的实现：

优点：具体的策略实现风格很是统一，策略实现所需要的数据都是从上下文中获取的，在上下文中添加的数据，可以视为公共的数据，其他的策略实现也可以使用。

缺点：很明显如果某些数据只是特定的策略实现需要，大部分的策略实现不需要，那这些数据有“浪费”之嫌，另外如果每次添加算法数据都扩展上下文，很容易导致上下文的层级很是复杂。

### 在具体的策略实现上添加所需要的数据的实现：

优点：容易想到，实现简单。

缺点：与其他的策略实现风格不一致，其他的策略实现所需数据都是来自上下文，而这个策略实现一部分数据来自于自身，一部分数据来自于上下文；外部在使用这个策略实现的时候也和其他的策略实现不一致了，难以以一个统一的方式动态的切换策略实现。

## 策略模式在JDK中的应用

在多线程编程中，我们经常使用线程池来管理线程，以减缓线程频繁的创建和销毁带来的资源的浪费，在创建线程池的时候，经常使用一个工厂类来创建线程池Executors,实际上Executors的内部使用的是类ThreadPoolExecutor.它有一个最终的构造函数如下：
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
corePoolSize：线程池中的核心线程数量，即使这些线程没有任务干，也不会将其销毁。

maximumPoolSize：线程池中的最多能够创建的线程数量。

keepAliveTime：当线程池中的线程数量大于corePoolSize时，多余的线程等待新任务的最长时间。

unit：keepAliveTime的时间单位。

workQueue：在线程池中的线程还没有还得及执行任务之前，保存任务的队列（当线程池中的线程都有任务在执行的时候，仍然有任务不断的提交过来，这些任务保存在workQueue队列中）。

threadFactory：创建线程池中线程的工厂。

handler：当线程池中没有多余的线程来执行任务，并且保存任务的多列也满了（指的是有界队列），对仍在提交给线程池的任务的处理策略。

RejectedExecutionHandler 是一个策略接口，用在当线程池中没有多余的线程来执行任务，并且保存任务的多列也满了（指的是有界队列），对仍在提交给线程池的任务的处理策略。
```java
public interface RejectedExecutionHandler {

    /**
     *当ThreadPoolExecutor的execut方法调用时，并且ThreadPoolExecutor不能接受一个任务Task时，该方法就有可能被调用。
　　　* 不能接受一个任务Task的原因：有可能是没有多余的线程来处理，有可能是workqueue队列中没有多余的位置来存放该任务，该方法有可能抛出一个未受检的异常RejectedExecutionException
     * @param r the runnable task requested to be executed
     * @param executor the executor attempting to execute this task
     * @throws RejectedExecutionException if there is no remedy
     */
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```
该策略接口有四个实现类：

AbortPolicy:该策略是直接将提交的任务抛弃掉，并抛出RejectedExecutionException异常。
```java
/**
    * A handler for rejected tasks that throws a
    * <tt>RejectedExecutionException</tt>.
    */
public static class AbortPolicy implements RejectedExecutionHandler {
    /**
        * Creates an <tt>AbortPolicy</tt>.
        */
    public AbortPolicy() { }

    /**
        * Always throws RejectedExecutionException.
        * @param r the runnable task requested to be executed
        * @param e the executor attempting to execute this task
        * @throws RejectedExecutionException always.
        */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException();
    }
}
```
　　
DiscardPolicy:该策略也是将任务抛弃掉（对于提交的任务不管不问，什么也不做），不过并不抛出异常。
```java
/**
    * A handler for rejected tasks that silently discards the
    * rejected task.
    */
public static class DiscardPolicy implements RejectedExecutionHandler {
    /**
        * Creates a <tt>DiscardPolicy</tt>.
        */
    public DiscardPolicy() { }

    /**
        * Does nothing, which has the effect of discarding task r.
        * @param r the runnable task requested to be executed
        * @param e the executor attempting to execute this task
        */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}
```

DiscardOldestPolicy:该策略是当执行器未关闭时，从任务队列workQueue中取出第一个任务，并抛弃这第一个任务，进而有空间存储刚刚提交的任务。使用该策略要特别小心，因为它会直接抛弃之前的任务。

```java
/**
    * A handler for rejected tasks that discards the oldest unhandled
    * request and then retries <tt>execute</tt>, unless the executor
    * is shut down, in which case the task is discarded.
    */
public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    /**
        * Creates a <tt>DiscardOldestPolicy</tt> for the given executor.
        */
    public DiscardOldestPolicy() { }

    /**
        * Obtains and ignores the next task that the executor
        * would otherwise execute, if one is immediately available,
        * and then retries execution of task r, unless the executor
        * is shut down, in which case task r is instead discarded.
        * @param r the runnable task requested to be executed
        * @param e the executor attempting to execute this task
        */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```
CallerRunsPolicy:该策略并没有抛弃任何的任务，由于线程池中已经没有了多余的线程来分配该任务，该策略是在当前线程（调用者线程）中直接执行该任务。
```java
/**
    * A handler for rejected tasks that runs the rejected task
    * directly in the calling thread of the {@code execute} method,
    * unless the executor has been shut down, in which case the task
    * is discarded.
    */
public static class CallerRunsPolicy implements RejectedExecutionHandler {
    /**
        * Creates a {@code CallerRunsPolicy}.
        */
    public CallerRunsPolicy() { }

    /**
        * Executes task r in the caller's thread, unless the executor
        * has been shut down, in which case the task is discarded.
        *
        * @param r the runnable task requested to be executed
        * @param e the executor attempting to execute this task
        */
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}
```
类ThreadPoolExecutor中持有一个RejectedExecutionHandler接口的引用，以便在构造函数中可以由外部客户端自己制定具体的策略并注入。下面看一下其类图：
![pic3](../img/in-post/DP-strategy/pic3.png)

## 总结

### 策略模式的优点：

1.策略模式的功能就是通过抽象、封装来定义一系列的算法，使得这些算法可以相互替换，所以为这些算法定义一个公共的接口，以约束这些算法的功能实现。如果这些算法具有公共的功能，可以将接口变为抽象类，将公共功能放到抽象父类里面。

2.策略模式的一系列算法是可以相互替换的、是平等的，写在一起就是if-else组织结构，如果算法实现里又有条件语句，就构成了多重条件语句，可以用策略模式，避免这样的多重条件语句。

3.扩展性更好：在策略模式中扩展策略实现非常的容易，只要新增一个策略实现类，然后在使用策略实现的地方，使用这个新的策略实现就好了。
 
### 策略模式的缺点：

1.客户端必须了解所有的策略，清楚它们的不同：

如果由客户端来决定使用何种算法，那客户端必须知道所有的策略，清楚各个策略的功能和不同，这样才能做出正确的选择，但是这暴露了策略的具体实现。

2.增加了对象的数量：

由于策略模式将每个具体的算法都单独封装为一个策略类，如果可选的策略有很多的话，那对象的数量也会很多。

3.只适合偏平的算法结构：

由于策略模式的各个策略实现是平等的关系（可相互替换），实际上就构成了一个扁平的算法结构。即一个策略接口下面有多个平等的策略实现（多个策略实现是兄弟关系），并且运行时只能有一个算法被使用。这就限制了算法的使用层级，且不能被嵌套。
 
### 策略模式的本质：

分离算法，选择实现。

如果你仔细思考策略模式的结构和功能的话，就会发现：如果没有上下文，策略模式就回到了最基本的接口和实现了，只要是面向接口编程，就能够享受到面向接口编程带来的好处，通过一个统一的策略接口来封装和分离各个具体的策略实现，无需关系具体的策略实现。

貌似没有上下文什么事，但是如果没有上下文的话，客户端就必须直接和具体的策略实现进行交互了，尤其是需要提供一些公共功能或者是存储一些状态的时候，会大大增加客户端使用的难度；引入上下文之后，这部分工作可以由上下文来完成，客户端只需要和上下文进行交互就可以了。这样可以让策略模式更具有整体性，客户端也更加的简单。

策略模式体现了开闭原则：策略模式把一系列的可变算法进行封装，从而定义了良好的程序结构，在出现新的算法的时候，可以很容易的将新的算法实现加入到已有的系统中，而已有的实现不需要修改。

策略模式体现了里氏替换原则：策略模式是一个扁平的结构，各个策略实现都是兄弟关系，实现了同一个接口或者继承了同一个抽象类。这样只要使用策略的客户端保持面向抽象编程，就可以动态的切换不同的策略实现以进行替换。

## 参考资料
- https://www.cnblogs.com/lewis0077/p/5133812.html 

