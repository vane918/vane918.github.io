---
layout: post
title:  "设计模式之命令模式"
date:   2019-12-17 00:00:00
catalog:  true
tags:
    - java
    - 设计模式
    - 命令模式
---



## 1 概述

命令模式是行为型模式之一。其定义为：将一个请求封装为一个对象，从而使你可用不同的请求对客户进行参数化； 对请求排队或记录请求日志，以及支持可撤销的操作。简单来说，命令是实例化的方法调用。 命令模式是一种回调的面向对象实现，这是一种对命令模式更好的解释。举个例子，司令员下令让士兵去干件事情，从整个事情的角度来考虑，司令员的作用是，发出口令，口令经过传递，传到了士兵耳朵里，士兵去执行。这个过程好在，三者相互解耦，任何一方都不用去依赖其他人，只需要做好自己的事儿就行，司令员要的是结果，不会去关注到底士兵是怎么实现的。

## 2 责任链模式的使用场景

-  多个对象可以处理同意请求，但具体由哪个对象处理则在运行时动态决定
-  在控件相互嵌套的界面程序中，当传递来了一个事件消息，这个消息在何处被传递，何处被消费，这就是一个典型的责任链模式

## 3 UML类图

![uml-command](/images/design-partten/uml-command.png)

- Handler： 抽象处理者角色 ， 定义出一个处理请求的接口。如果需要，接口可以定义 出一个方法以设定和返回对下家的引用。这个角色通常由一个Java抽象类或者Java接口实现。上图中Handler类的聚合关系给出了具体子类对下家的引用，抽象方法handleRequest()规范了子类处理请求的操作。 
-  ConcreteHandler ： 具体处理者角色，具体处理者接到请求后，可以选择将请求处理掉，或者将请求传给下家。由于具体处理者持有对下家的引用，因此，如果需要，具体处理者可以访问下家。 

## 4 责任链模式的简单示例

下面模拟报账的流程。

```java
public abstract class Leader {
	protected Leader nextHandler;//上一级领导处理者
    //处理报账请求
    public final void handleRequest(int money) {
        if(money <- limit()) {
            handle(money);
        } else {
            if(null != nextHandler) {
                nextHandler.handleRequest(money);
            }
        }
    }
    //自身能批复的额度权限
    public abstract int limit();
    //处理报账行为
    public abstract void handle(int money);
}
```

```java
public class GroupLeader extends Leader {

	@Override
	public int limit() {
		return 1000;
	}
    
    public void handle(int money) {
        System.out.println("组长批复报销" + money + "元");
    }
}
```

```java
public class Director extends Leader {

	@Override
	public int limit() {
		return 5000;
	}
    
    public void handle(int money) {
        System.out.println("主管批复报销" + money + "元");
    }
}
```

```java
public class Manager extends Leader {

	@Override
	public int limit() {
		return 10000;
	}
    
    public void handle(int money) {
        System.out.println("经理批复报销" + money + "元");
    }
}
```

```java
public class Boss extends Leader {

	@Override
	public int limit() {
		return Integer.MAX_VALUE;
	}
    
    public void handle(int money) {
        System.out.println("老板批复报销" + money + "元");
    }
}
```

```java
public class Client {
	public static void main(String[] args) {
		GroupLeader groupLeader = new GroupLeader();
		Director director = new Director();
        Manager manager = new Manager();
        Boss boss = new Boss();
        //设置上一级领导处理者对象
        groupLeader.nextHandler = director;
        director.nextHandler = manager;
        manager.nextHandler = boss;
        //发起报账申请
        groupLeader.handleRequest(50000);
	}
}
```

结果打印如下：

```logcat
老板批复报销50000元
```

等于责任链中的一个处理者对象，其只有两个行为，一是处理请求，二是将请求转送给下一个节点，不允许某个处理者在处理了请求后又将请求转送给上一个节点的情况。

## 5 Android源码中的责任链模式

Android系统源码中应用责任链模式比较经典的是对事件分发的处理，每当用户接触屏幕时，Android都会将对应的事件包装成一个事件对象从ViewTree的顶部至上而下地分发传递，这里我们看下ViewGroup中是如何将事件派发到子View的。ViewGroup中执行事件派发的方法是dispatchTouchEvent，在该方法中其对事件进行了统一的分发。 下图是ViewGroup的事件分发流程图：

![viewgroup-dispatchtouchevent](/images/design-partten/viewgroup-dispatchtouchevent.png)

ViewGroup事件分发的递归调用就类似于一条责任链，一旦其寻找到责任者，那么将由责任者持有并消费掉该事件，具体体现在View的onTouchEvent方法中返回值的设置，如果onTouchEvent返回false，那么意味着当前View不会是该次事件的责任者，将不会对其持有；如果为true，说明此View会持有该事件，并且不再向外传递。

## 6 总结

职责链模式的主要优点

- 对象仅需知道该请求会被处理即可，且链中的对象不需要知道链的结构，由客户端负责链的创建，降低了系统的耦合度
- 请求处理对象仅需维持一个指向其后继者的引用，而不需要维持它对所有的候选处理者的引用，可简化对象的相互连接
- 在给对象分派职责时，职责链可以给我们更多的灵活性，可以在运行时对该链进行动态的增删改，改变处理一个请求的职责
- 新增一个新的具体请求处理者时无须修改原有代码，只需要在客户端重新建链即可，符合 "开闭原则"

职责链模式的主要缺点

- 一个请求可能因职责链没有被正确配置而得不到处理
- 对于比较长的职责链，请求的处理可能涉及到多个处理对象，系统性能将受到一定影响，且不方便调试
- 可能因为职责链创建不当，造成循环调用，导致系统陷入死循环