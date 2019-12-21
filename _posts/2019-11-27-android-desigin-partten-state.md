---
layout: post
title:  "设计模式之状态模式"
date:   2019-11-27 00:00:00
catalog:  true
tags:
    - java
    - 设计模式
    - 状态模式
---



## 1 概述

状态模式的定义是当一个对象的内在状态改变时允许改变其行为，这个对象看起来像是改变了其类。状态模式中的行为是由状态来决定的，不同的状态下有不同的行为。状态模式和策略模式的结构几乎完全一样，但它们的目的、本质却完全不一样。状态模式的行为是平行的、不可替换的，策略模式的行为是彼此独立、可相互替换的。用一句话来表述，状态模式把对象的行为包装在不同的状态对象中，每一个状态对象都有一个共同的抽象状态基类。状态模式的意图是让一个对象在其内部状态改变的时候，其行为也随之改变。

## 2 状态模式的使用场景

-  一个对象的行为取决于它的状态，并且它必须在运行时根据状态改变它的行为
-  代码中包含大量与对象状态有关的条件语句

## 3 UML类图

![uml-state](/images/design-partten/uml-state.png)

- Context：环境类，定义客户感兴趣的接口，维护一个State子类的实例，这个实例定义了对象的当前状态
- State：抽象状态类或者状态接口，定义一个或者一组接口，表示该状态下的行为
- ConcreteStateA、ConcreteStateB：具体状态类，每一个具体的状态类实现抽象State中定义的接口，从而达到不同状态下的不同行为

## 4 状态模式的简单示例

模拟红绿灯的行为，红灯禁止通行，绿灯允许通行，黄灯等待通行。

```java
public interface State {
	void doAction();
}
```

```java
public class RedState implements State {

	@Override
	public void doAction() {
		System.out.println("红灯，禁止通行！");
	}
}
```

```java
public class GreenState implements State {

	@Override
	public void doAction() {
		System.out.println("绿灯，允许通行！");
	}
}
```

```java
public class YellowState implements State {

	@Override
	public void doAction() {
		System.out.println("黄灯，等待通行！");
	}
}
```

```java
public class PeopleContext {
	private State state;
	public void setState(State s){
		state = s;
		state.doAction();
	}
}
```

```java
public class Client {
	public static void main(String[] args) {
		PeopleContext ctx = new PeopleContext();
		ctx.setState(new RedState());
		ctx.setState(new YellowState());
        ctx.setState(new GreenState());
	}
}
```

结果打印如下：

```logcat
红灯，禁止通行！
黄灯，等待通行！
绿灯，允许通行！
```

## 5 Android源码中的状态模式

Android系统源码中应用状态模式比较经典的是Wifi的状态机机制，因为Wifi有多种状态，二每种状态做的操作又不一样，所以非常适合使用状态模式来组织Wifi的各种状态。

在WifiStateMachine.java类中，有很多Wifi的状态类，以内部类表示，如DefaultState、InitialState、SupplicantStartingState、SupplicantStartedState和DriverStartingState等等，不同的状态类都集成State类，有enter()、processMessage()和exit()方法。状态类是有层级关系，呈现树形结构，状态之间并不是跨越式转换的，当前状态只能转换到上一状态或下一状态，例如，当Wifi的状态为mDefault时，它不能直接跳转到mDriverStartingState，需要先转换到mSupplicantStartedState状态，然后才能转换到mDriverStartingState。

## 6 总结

状态模式的关键点在于不同的状态下对同一行为有不同的响应。状态模式将所有与一个特定的状态相关的行为都放入一个状态对象中，它提供了一个更好的方法来组织与特定状态相关的代码，将繁琐的状态判断转换成结构清晰的状态类族，在避免代码膨胀的同时也保证了可扩展性与可维护性。