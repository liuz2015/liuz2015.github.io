---
layout:     post
title:      "设计模式之职责链模式"
subtitle:   ""
date:       2018-10-22
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 设计模式
    - 职责链模式
---

## 目录
- [简介](#简介)
- [适用性](#适用性)
- [结构](#结构)
- [模式的组成](#模式的组成)
- [效果](#效果)
- [纯与不纯的职责链模式](#纯与不纯的职责链模式)
- [代码实现](#代码实现)
  - [servlet中的Filter](#servlet中的Filter)
  - [Dubbo中的Filter](#Dubbo中的Filter)
  - [Mybatis中的Plugin](#Mybatis中的Plugin)
- [总结](#总结)
- [参考资料](#参考资料)

## 简介
职责链模式(Chain of Responsibility)：使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止。（Avoid coupling the sender of a request to itsreceiver by giving morethan one objecta chance to handle the request.Chain the receiving objects andpassthe request along the chainuntil an object handles it. ）
- 在职责链模式里，很多对象由每一个对象对其下家的引用而连接起来形成一条链。
- 请求在这条链上传递，直到链上的某一个对象处理此请求为止。
- 发出这个请求的客户端并不知道链上的哪一个对象最终处理这个请求，这使得系统可以在不影响客户端的情况下动态地重新组织链和分配责任。
## 适用性
在以下条件下使用Responsibility 链：
- 有多个的对象可以处理一个请求，哪个对象处理该请求运行时刻自动确定。
- 你想在不明确指定接收者的情况下，向多个对象中的一个提交一个请求。
- 可动态指定一组对象处理请求。
## 结构
![结构1](../img/in-post/DP-COR/pic1.jpeg)
一个典型的对象结构可能如下图所示：
![结构2](../img/in-post/DP-COR/pic2.jpeg)

## 模式的组成
抽象处理者角色(Handler:Approver):定义一个处理请求的接口，和一个后继连接(可选)
具体处理者角色(ConcreteHandler:President):处理它所负责的请求，可以访问后继者，如果可以处理请求则处理，否则将该请求转给他的后继者。
客户类(Client):向一个链上的具体处理者ConcreteHandler对象提交请求。
## 效果
职责链有下列优点和缺点。

职责链模式的优点：
1. 降低耦合度 ：该模式使得一个对象无需知道是其他哪一个对象处理其请求。对象仅需知道该请求会被“正确”地处理。接收者和发送者都没有对方的明确的信息，且链中的对象不需知道链的结构。
2. 职责链可简化对象的相互连接 :    结果是，职责链可简化对象的相互连接。它们仅需保持一个指向其后继者的引用，而不需保持它所有的候选接受者的引用。
3. 增强了给对象指派职责( R e s p o n s i b i l i t y )的灵活性 ：当在对象中分派职责时，职责链给你更多的灵活性。你可以通过在运行时刻对该链进行动态的增加或修改来增加或改变处理一个请求的那些职责。你可以将这种机制与静态的特例化处理对象的继承机制结合起来使用。
4. 增加新的请求处理类很方便。

职责链模式的缺点:
1. 不能保证请求一定被接收。既然一个请求没有明确的接收者，那么就不能保证它一定会被处理 —该请求可能一直到链的末端都得不到处理。一个请求也可能因该链没有被正确配置而得不到处理。
2. 系统性能将受到一定影响，而且在进行代码调试时不太方便；可能会造成循环调用。
## 纯与不纯的职责链模式

纯的职责链模式：一个具体处理者角色处理只能对请求作出两种行为中的一个：一个是自己处理（承担责任），另一个是把责任推给下家。不允许出现某一个具体处理者对象在承担了一部分责任后又将责任向下传的情况。请求在责任链中必须被处理，不能出现无果而终的结局。

反之就是不纯的职责链模式。  

在一个纯的职责链模式里面，一个请求必须被某一个处理者对象所接收；在一个不纯的职责链模式里面，一个请求可以最终不被任何接收端对象所接收。

## 代码实现
这里主要来说说java中如何编写。主要从下面3个框架中的代码中介绍。

### servlet中的Filter
servlet中分别定义了一个 Filter和FilterChain的接口，核心代码如下：
```java
public final class ApplicationFilterChain implements FilterChain {
    private int pos = 0; //当前执行filter的offset
    private int n; //当前filter的数量
    private ApplicationFilterConfig[] filters;  //filter配置类，通过getFilter()方法获取Filter
    private Servlet servlet
  
    @Override
    public void doFilter(ServletRequest request, ServletResponse response) {
        if (pos < n) {
            ApplicationFilterConfig filterConfig = filters[pos++];
            Filter filter = filterConfig.getFilter();
            filter.doFilter(request, response, this);
        } else {
            // filter都处理完毕后，执行servlet
            servlet.service(request, response);
        }
    }
}
```
代码还算简单，结构也比较清晰，定义一个Chain，里面包含了Filter列表和servlet，达到在调用真正servlet之前进行各种filter逻辑。
![servlet](../img/in-post/DP-COR/pic3.png)

### Dubbo中的Filter
Dubbo在创建Filter的时候是另外一个方法，通过把Filter封装成 Invoker的匿名类，通过链表这样的数据结构来完成责任链，核心代码如下：
```java
private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
    Invoker<T> last = invoker;
    //只获取满足条件的Filter
    List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
    if (filters.size() > 0) {
        for (int i = filters.size() - 1; i >= 0; i --) {
            final Filter filter = filters.get(i);
            final Invoker<T> next = last;
            last = new Invoker<T>() {
                ...
                public Result invoke(Invocation invocation) throws RpcException {
                    return filter.invoke(next, invocation);
                }
                ...
            };
        }
    }
    return last;
}
```
Dubbo的责任链就没有类似FilterChain这样的类吧Filter和调用Invoker结合起来，而是通过创建一个链表，调用的时候我们只知道第一个节点，每个节点包含了下一个调用的节点信息。 这里的虽然Invoker封装Filter没有显示的指定next，但是通过java匿名类和final的机制达到同样的效果。
![dubbo](../img/in-post/DP-COR/pic4.png)

### Mybatis中的Plugin
Mybatis可以配置各种Plugin，无论是官方提供的还是自己定义的，Plugin和Filter类似，就在执行Sql语句的时候做一些操作。Mybatis的责任链则是通过动态代理的方式，使用Plugin代理实际的Executor类。（这里实际还使用了组合模式，因为Plugin可以嵌套代理），核心代码如下：
```java
public class Plugin implements InvocationHandler{
    private Object target;
    private Interceptor interceptor;
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {      
        if (满足代理条件) {
            return interceptor.intercept(new Invocation(target, method, args));
        }
        return method.invoke(target, args);     
    }
   
    //对传入的对象进行代理，可能是实际的Executor类，也可能是Plugin代理类
    public static Object wrap(Object target, Interceptor interceptor) {
  
        Class<?> type = target.getClass();
        Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
        if (interfaces.length > 0) {
            return Proxy.newProxyInstance(
                    type.getClassLoader(),
                    interfaces,
                    new Plugin(target, interceptor, signatureMap));
        }
        return target;
    }
} 
```
简单的示意图如下：
![mybatis](../img/in-post/DP-COR/pic5.png)

## 总结
这里简单介绍了Servlet、Dubbo、Mybatis对责任链模式的不同实现手段，其中Servlet是相对比较清晰，又易于实现的方式，而Dubbo和Mybatis则适合在原有代码基础上，增加责任链模式代码改动量最小的。

## 参考资料


