---
layout: post
title:  "设计模式之策略模式"
date:   2019-11-25 00:00:00
catalog:  true
tags:
    - java
    - 设计模式
    - 策略模式
---



## 1 概述

策略模式定义了一系列的算法，并将每一个算法封装起来，而且使它们还可以相互替换。策略模式让算法独立于使用它的客户而独立变化， 这种类型的设计模式属于行为模式。比如之前笔者做的一款app，里面集成了百度地图和谷歌地图的SDK，那么就可以使用策略模式来实现地图类，根据是在国内还是国外来动态选择百度地图还是谷歌地图。

## 2 策略模式的使用场景

-  针对同一类型问题的多种处理方式，仅仅是具体行为有差别时
- 需要安全地封装多种同一类型的操作时
- 出现同一抽象类有多个子类，而又需要使用if-else或者switch-case来选择具体子类时

## 3 UML类图

![uml-stragety](/images/design-partten/uml-stragety.png)

- Context：用来操作策略的上下文环境
- Stragety：策略的抽象
- ConcreteStragetyA、ConcreteStragetyA：具体的策略实现

## 4 策略模式的实现

拿之前做的项目的地图类型来说明策略模式的使用。

```java
public interface MapStrategy {
   public int doLocation();
}
```

```java
public interface BaiDuMapStrategy {
   public int doLocation() {
       System.out.println("BaiDuMap location");
   }
}
```

```java
public interface GoogleMapStrategy {
   public int doLocation() {
       System.out.println("GoogleMap location");
   }
}
```

```java
public class Context {
   private MapStrategy strategy;

   public Context(MapStrategy strategy){
      this.strategy = strategy;
   }

   public int executeStrategy(){
      return strategy.doLocation();
   }
}
```

```java
public class StrategyPatternDemo {
   public static void main(String[] args) {
      Context context = new Context(new BaiDuMapStrategy());
      System.out.println(context.executeStrategy());

      context = new Context(new GoogleMapStrategy());
      System.out.println(context.executeStrategy());
   }
}
```

输出结果：

```logcat
BaiDuMap location
GoogleMap location
```

## 5 Android源码中的策略模式实现

ListView 是一个很重要的组件，我们通常在布局里写个 ListView 组件，然后在代码中 setAdapter，把 View 与 Model 结合的任务交给了 Adapter。
当更换 Adapter 的具体实现时，仍然调用的是 ListView.setAdapter(…) 方法，传入的是ArrayAdapter或BaseAdapter等，查看 ListView 源码，发现 setAdapter 方法的参数是一个 ListAdapter，如下：

```java
@Override
public void setAdapter(ListAdapter adapter) {
    ...
}
public interface ListAdapter extends Adapter{
    ...
} 
```
可以看到 ListAdapter 是一个接口，ArrayAdapter 和 BaseAdapter 是它的一个实现类。
可以发现 ListAdapter 就是 strategy 接口，ArrayAdpater 等就是具体的实现类，而在 ListView 中引用的是接口 ListAdapter，可以证实这就是一个 策略模式 的使用。

## 6 总结

策略模式主要是用来分离算法，在相同的行为抽象下有不同的具体实现策略，其优点如下：

- 结构清晰、使用简单
- 耦合度相对较低，扩展方便
- 操作封装更彻底，数据更安全

其缺点也是一般模式的通病，随着策略的增加，子类也会变得繁多。