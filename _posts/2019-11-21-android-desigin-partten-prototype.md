---
layout: post
title:  "设计模式之原型模式"
date:   2019-11-21 00:00:00
catalog:  true
tags:
    - java
    - 设计模式
    - 原型模式
---



## 1 概述

原型模式：使用原型实例指定待创建对象的类型，并且通过复制这个原型来创建新的对象。原型模式是为了解决一些不必要的对象创建过程。 原型表明了该模式应该有一个样板实例，用户从这个样板对象中复制出一个内部属性一致性的对象，这个过程就是所谓的“克隆”。原型模式多用于创建复杂的或者构造耗时的实例。

## 2 使用场景

- 类初始化需要消耗非常多的资源，如数据、硬件资源等
- 通过个过new产生一个对象需要非常繁琐的书库准备或访问权限
- 一个对象需要提供给其他对象访问时，为了防止该对象被其他对象修改，可以使用原型模式拷贝对象供其他对象调用，即保护性拷贝

## 3 UML类图

![uml-prototype](/images/design-partten/uml-prototype.png)

角色介绍：

- Client：客户端用户
- Prototype：抽象类或接口，声明具备clone能力
- ConcretePrototype：具体的原型类

## 4 原型模式实现

```java
public class Person implements Cloneable {

	private String name;
	private int age;
	private ArrayList<String> childs = new ArrayList<>();
	
	public Person() {
		System.out.println("Person的构造函数");
	}

	
	public String getName() {
		return name;
	}


	public void setName(String name) {
		this.name = name;
	}


	public int getAge() {
		return age;
	}


	public void setAge(int age) {
		this.age = age;
	}

    
	public ArrayList<String> getChilds() {
		return childs;
	}


	public void setChilds(String child) {
		this.childs.add(child);
	}


	@Override
	protected Person clone() {
		try {
			Person person = (Person) super.clone();
			person.name = this.name;
			person.age = this.age;
			person.childs = this.childs;
			return person;
		} catch (CloneNotSupportedException e) {
		}
		return null;
	}
	
	@Override
	public String toString() {
		return "Person[name=" + name + ", age=" + age + ", childs=" + childs + "]";
	}
	
}
```

```java
public class Client {

	public static void main(String[] args) {
        Person person = new Person();
        person.setName("张三");
        person.setAge(28);
        person.setChilds("张一");
        person.setChilds("张二");
        System.out.println(person.toString());
        
        Person person2 = person.clone();
        System.out.println(person2.toString());
        
        person2.setName("李四");
        person2.setAge(30);
        person2.setChilds("李一");
        person2.setChilds("李二");
        System.out.println(person2.toString());
        
        System.out.println(person.toString());
    }

}
```

输出结果如下：

```
Person的构造函数
Person[name=张三, age=28, childs=[张一, 张二]]
Person[name=张三, age=28, childs=[张一, 张二]]
Person[name=李四, age=30, childs=[张一, 张二, 李一, 李二]]
Person[name=张三, age=28, childs=[张一, 张二, 李一, 李二]]
```

可以看到，person2是通过person.clone()创建的，并且person2第一次输出的时候和person一样，即person2是person的一份拷贝，他们的内容是一样的。但person2修改内容后，person的name和age还是和原来一样，没有被person2的修改而改变，但是，person2修改了childs的值，childs是一个ArrayList<String>对象，person的childs被person2修改了，这其实是浅拷贝和深拷贝有关。

## 5 浅拷贝与深拷贝

上面的原型模式的实现实际上只是一个浅拷贝，即影子拷贝，只是将原始Person，即张三的所有字段引用拷贝了一份而已，由于childs是一个对象，因此person和person2的childs都指向同一个对象，哪一个Person修改了childs值，都能影响另外一个Person的childs值。解决这个问题需要采用深拷贝。
深拷贝在拷贝对象时，对于引用型的字段也采用拷贝的形式，而不是单纯引用的形式。

上面例子的clone方法修改如下：

```java
@Override
	protected Person clone() {
		try {
			Person person = (Person) super.clone();
			person.name = this.name;
			person.age = this.age;
			person.childs = (ArrayList<String>) this.childs.clone();
			return person;
		} catch (CloneNotSupportedException e) {
		}
		return null;
	}
```

打印结果如下：

```
Person的构造函数
Person[name=张三, age=28, childs=[张一, 张二]]
Person[name=张三, age=28, childs=[张一, 张二]]
Person[name=李四, age=30, childs=[张一, 张二, 李一, 李二]]
Person[name=张三, age=28, childs=[张一, 张二]]
```

可以看到person2的修改也影响不了person的childs值了。

## 6 Android源码中的原型模式实现

在Android中，Intent、Bundle类都使用了原型模式，实现了clone方法。 我们看下Intent的clone方法。

frameworks/base/core/java/android/content/Intent.java

```java
@Override
    public Object clone() {
        return new Intent(this);
    }
```

## 7 总结

原型模式本质上就是对象的拷贝，有浅拷贝和深拷贝之分。使用原型模式可以解决构建复杂对象的资源消耗问题，能够在某些场景下提升创建对象的效率。另一个用途就是保护性拷贝，也就是某个对象对外可能是只读的，为了防止外部对这个只读对象修改，通常可以通过返回一个对象拷贝的形式实现只读的限制。需要注意的是，拷贝对象时，不会再次执行原型对象的构造函数。