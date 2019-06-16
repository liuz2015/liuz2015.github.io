---
layout:     post
title:      "设计模式之观察者模式"
subtitle:   ""
date:       2018-10-30
author:     "Liuz2015"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 设计模式
    - 观察者模式
---

## 目录
- [简介](#简介)
- [代码实现](#代码实现)
- [参考资料](#参考资料)

## 简介
观察者模式（有时又被称为发布（publish）-订阅（Subscribe）模式、模型-视图（View）模式、源-收听者(Listener)模式或从属者模式）是软件设计模式的一种。在此种模式中，一个目标物件管理所有相依于它的观察者物件，并且在它本身的状态改变时主动发出通知。这通常透过呼叫各观察者所提供的方法来实现。此种模式通常被用来实现事件处理系统。

观察者设计模式定义了对象间的一种一对多的依赖关系，以便一个对象的状态发生变化时，所有依赖于它的对象都得到通知并自动刷新。

实现观察者模式的时候要注意，观察者和被观察对象之间的互动关系不能体现成类之间的直接调用，否则就将使观察者和被观察对象之间紧密的耦合起来，从根本上违反面向对象的设计的原则。无论是观察者“观察”观察对象，还是被观察者将自己的改变“通知”观察者，都不应该直接调用。

常常使用到的地方是在安卓开发中，将一个listener注册到界面的某个按钮上，当该按钮被点击，就会通知这个listener做出相应行为。

## 代码实现
```java
//观察者，需要用到观察者模式的类需实现此接口
public interface Observer{
    void update(Object...objs);
}
 
//被观察者（一个抽象类，方便扩展）
public abstract class Observable{
 
    public final ArrayList<Class<?>> obserList = new ArrayList<Class<?>>();
 
    /**AttachObserver（通过实例注册观察者）
    *<b>Notice:</b>obcan'tbenull,oritwillthrowNullPointerException
    **/
    public<T> void registerObserver(T ob){
        if(ob==null) throw new NullPointerException();
        this.registerObserver(ob.getClass());
    }
 
    /**
    *AttachObserver（通过Class注册观察者）
    *@paramcls
    */
    public void registerObserver(Class<?> cls){
        if(cls==null) throw new NullPointerException();
        synchronized(obserList){
            if(!obserList.contains(cls)){
                obserList.add(cls);
            }
        }
    }
 
    /**UnattachObserver（注销观察者）
    *<b>Notice:</b>
    *<b>ItreverseswithattachObserver()method</b>
    **/
    public<T>void unRegisterObserver(Tob){
        if(ob==null) throw new NullPointerException();
        this.unRegisterObserver(ob.getClass());
    }
 
    /**UnattachObserver（注销观察者，有时候在未获取到实例使用）
    *<b>Notice:</b>
    *<b>ItreverseswithattachObserver()method</b>
    **/
    public void unRegisterObserver(Class<?>cls){
        if(cls==null) throw new NullPointerException();
        synchronized(obserList){
            Iterator<Class<?>>iterator=obserList.iterator();
            while(iterator.hasNext()){
                if(iterator.next().getName().equals(cls.getName())){
                    iterator.remove();
                    break;
                }
            }
        }
    }
 
    /**detachallobservers*/
    public void unRegisterAll(){
        synchronized(obserList){
            obserList.clear();
        }
    }
 
    /**Ruturnthesizeofobservers*/
    public int countObservers(){
        synchronized(obserList){
            returnobserList.size();
        }
    }
 
    /**
    *notify all observer（通知所有观察者，在子类中实现）
    *@paramobjs
    */
    public abstract void notifyObservers(Object... objs);
 
    /**
    *notify one certain observer（通知某一个确定的观察者）
    *@paramcls
    *@paramobjs
    */
    public abstract void notifyObserver(Class<?> cls, Object... objs);
 
    /**
    *notifyonecertainobserver
    *@paramcls
    *@paramobjs
    */
    public abstract<T> void notifyObserver(T t, Object... objs);
}
 
//目标被观察者
public class ConcreteObservable extends Observable{
 
    private static ConcreteObservableinstance = null;
    private ConcreteObservable(){};
    public static synchronized ConcreteObservablegetInstance(){
        if(instance == null){
            instance=newConcreteObservable();
        }
        returninstance;
    }
 
    @Override
    public <T> void notifyObserver(T t, Object... objs){
        if(t == null) throw new NullPointerException();
        this.notifyObserver(t.getClass(), objs);
    }
 
    @Override
    public void notifyObservers(Object... objs){
        for(Class<?>cls : obserList){
            this.notifyObserver(cls, objs);
        }
    }
 
 
    //通过java反射机制实现调用
    @Override
    public void notifyObserver(Class<?>cls, Object...objs){
        if(cls == null) throw new NullPointerException();
        Method[] methods = cls.getDeclaredMethods();
        for(Method method : methods){
            if(method.getName().equals("update")){
                try{
                    method.invoke(cls,objs);
                    break;
                }catch(IllegalArgumentException e){
                    e.printStackTrace();
                }catch(IllegalAccessException e){
                    e.printStackTrace();
                }catch(InvocationTargetException e){
                    e.printStackTrace();
                }
            }
        }
    }
}
 
//使用(实现Observer接口）
public class Text extends Activity implements Observer{
    publicvoidonCreate(...){
        ConcreteObservable.getInstance().registerObserver(Text.class);
        ....
    }
 
    //实现接口处理
    publicvoidupdate(Object...objs){
        //做操作，比如更新数据，更新UI等
    }
}
```
## 参考资料


