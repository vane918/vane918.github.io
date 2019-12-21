---
layout: post
title:  "设计模式之工厂模式"
date:   2019-11-21 00:00:01
catalog:  true
tags:
    - java
    - 设计模式
    - 工厂模式
---



## 1 概述

工厂模式是创建型设计模式之一。工厂模式是一种结构简单的模式， 此模式提供了创建对象的最佳方法之一。  在工厂模式中，我们没有创建逻辑暴露给客户端创建对象，并使用一个通用的接口引用新创建的对象。 工厂模式又有以下几种细分的模式：

- 简单工厂模式
-  工厂方法模式 
-  抽象工厂模式 

## 2 简单工厂模式

 简单工厂模式就是建立一个工厂类，对实现了同一接口的一些类进行实例的创建。简单工厂模式的实质是由一个工厂类根据传入的参数，动态决定应该创建哪一个产品类（这些产品类继承自一个父类或接口）的实例。 

### 2.1 UML类图

![uml-simple-factory](/images/design-partten/uml-simple-factory.png)

### 2.2 简单工厂模式实现

```java
public interface Animal {
    public void eat();
}
```

```java
public class Tiger implements Animal {

    @Override
    public void eat() {
        System.out.println("Tiger eat");
    }

}
```

```java
public class Monkey implements Animal {

    @Override
    public void eat() {
        System.out.println("Monkey eat");
    }

}
```

```java
public class SimpleFactory {
    public static Animal makeAnimal(String type){
        if(type.equals("tiger")){
            Tiger tiger = new Tiger();
            return tiger;
        }else if(type.equals("monkey")){
            Animal monkey = new Monkey();
            return monkey;
        }else{
            System.out.println("make nothing");
            return null;
        }            
    }
}
```

还有一种更加优雅的方式：利用反射机制，如下：

```java
public class SimpleFactory1 {
    public static Animal makeAnimal(Class clzz){
        Animal animal = null;
        try {
            animal = (Animal) Class.forName(clzz.getName()).newInstance();
        } catch (InstantiationException e) {
            System.out.println("不支持抽象类或接口");
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
            System.out.println("没有足够权限，即不能访问私有对象");
        } catch (ClassNotFoundException e) {
            System.out.println("类不存在");
            e.printStackTrace();
        }    
        return animal;
    }
}
```

测试代码如下：

```java
public class Client {
    public static void main(String[] args) {
        Animal tiger = SimpleFactory.makeAnimal("tiger");
        tiger.eat();
        Animal monkey = SimpleFactory.makeAnimal("monkey");
        monkey.eat();
       
        Animal tiger = SimpleFactory1.makeAnimal(Tiger.class);
        tiger.eat();
        Animal monkey = SimpleFactory1.makeAnimal(Monkey.class);
        monkey.eat();
    }
}
```

### 2.3 小结 

在简单的工厂模式里，我们创建了一个类似工具的类来创建相应的具体类对象 由于工厂类集中了所有实例的创建逻辑，违反了高内聚责任分配原则，将全部创建逻辑集中到了一个工厂类中。它所能创建的类只能是事先考虑到的，如果需要添加新的类，则就需要改变工厂类了。
当系统中的具体产品类不断增多时候，可能会出现要求工厂类根据不同条件创建不同实例的需求。这种对条件的判断和对具体产品类型的判断交错在一起，很难避免模块功能的蔓延，对系统的维护和扩展非常不利。

## 3 工厂方法模式

 工厂方法模式的定义是定义一个创建对象的接口，但由子类决定要实例化的类是哪一个。工厂方法模式把类实例化的过程推迟到子类。 

### 3.1 UML类图

![uml-method-factory](/images/design-partten/uml-method-factory.png)

### 3.2 工厂方法模式实现

还是以上面简单工厂模式的例子举例。

```java
public interface Animal {
    public void eat();
}
```

```java
public class Tiger implements Animal {

    @Override
    public void eat() {
        System.out.println("Tiger eat");
    }

}
```

```java
public class Monkey implements Animal {

    @Override
    public void eat() {
        System.out.println("Monkey eat");
    }

}
```

```java
public abstract class Factory {
    public abstract Animal makeAnimal();
}
```

```java
public class TigerFactory extends Factory {
    public Animal makeAnimal() {
        return new Tiger();
    }
}
```

```java
public class MonkeyFactory extends Factory {
    public Animal makeAnimal() {
        return new Monkey();
    }
}
```

测试代码：

```java
public class Client {
    public static void main(String[] args) {
        Factory tigerFactory = new TigerFactory();
        Animal tiger = tigerFactory.makeAnimal();
        tiger.eat();
        
        Factory monkeyFactory = new MonkeyFactory();
        Animal monkey = monkeyFactory.makeAnimal();
        monkey.eat();
    }
}
```

这几个角色都很简单，Tiger和Monkey对象分别由TigerFactory和MonkeyFactory创建，这两个是具体工厂，都继承抽象的工厂类Factory。

### 3.3 小结

总的来说，工厂方法模式是一个很好的设计模式，但具体工厂A和B是两个完全独立的，两者除了都是抽象工厂子类，没有任何其他的交集。 每次为工厂方法模式添加新的产品时就要编写一个相对应的具体工厂和具体的产品，导致类结构的复杂化。

## 4 抽象工厂模式

抽象工厂模式的定义是为创建一组相关或者相互依赖的对象提供一个接口，而不需要指定他们的具体类。
抽象工厂模式是一个超级工厂，用来创建其他工厂。 这个工厂也被称为工厂的工厂。 这种类型的设计模式属于创建模式，因为此模式提供了创建对象的最佳方法之一。
在抽象工厂模式中，接口负责创建相关对象的工厂，而不明确指定它们的类。 每个生成的工厂可以按照工厂模式提供对象。

### 4.1 UML类图

![uml-abstract-factory](/images/design-partten/uml-abstract-factory.png)

### 4.2 抽象工厂模式实现

我们将创建一个Shape和Color接口并实现这些接口的具体类。

结构图如下：

![abstract-factory-struct](/images/design-partten/abstract-factory-struct.png)

代码如下：

```java
public interface Shape {
   void draw();
}
```

```java
public class Rectangle implements Shape {

   @Override
   public void draw() {
      System.out.println("Inside Rectangle::draw() method.");
   }
}
```

```java
public class Square implements Shape {

   @Override
   public void draw() {
      System.out.println("Inside Square::draw() method.");
   }
}
```

```java
public class Circle implements Shape {

   @Override
   public void draw() {
      System.out.println("Inside Circle::draw() method.");
   }
}
```

```java
public interface Color {
   void fill();
}
```

```java
public class Red implements Color {

   @Override
   public void fill() {
      System.out.println("Inside Red::fill() method.");
   }
}
```

```java
public class Green implements Color {

   @Override
   public void fill() {
      System.out.println("Inside Green::fill() method.");
   }
}
```

```java
public class Blue implements Color {

   @Override
   public void fill() {
      System.out.println("Inside Blue::fill() method.");
   }
}
```

```java
public abstract class AbstractFactory {
   abstract Color getColor(String color);
   abstract Shape getShape(String shape) ;
}
```

```java
public class ShapeFactory extends AbstractFactory {

   @Override
   public Shape getShape(String shapeType){

      if(shapeType == null){
         return null;
      }

      if(shapeType.equalsIgnoreCase("CIRCLE")){
         return new Circle();

      }else if(shapeType.equalsIgnoreCase("RECTANGLE")){
         return new Rectangle();

      }else if(shapeType.equalsIgnoreCase("SQUARE")){
         return new Square();
      }

      return null;
   }

   @Override
   Color getColor(String color) {
      return null;
   }
}
```

```java
public class ColorFactory extends AbstractFactory {

   @Override
   public Shape getShape(String shapeType){
      return null;
   }

   @Override
   Color getColor(String color) {

      if(color == null){
         return null;
      }        

      if(color.equalsIgnoreCase("RED")){
         return new Red();

      }else if(color.equalsIgnoreCase("GREEN")){
         return new Green();

      }else if(color.equalsIgnoreCase("BLUE")){
         return new Blue();
      }

      return null;
   }
}
```

```java
public class FactoryProducer {
   public static AbstractFactory getFactory(String choice){

      if(choice.equalsIgnoreCase("SHAPE")){
         return new ShapeFactory();

      }else if(choice.equalsIgnoreCase("COLOR")){
         return new ColorFactory();
      }

      return null;
   }
}
```

测试代码， 使用`FactoryProducer`来获取`AbstractFactory`，以便通过传递类型等信息来获取具体类的工厂。 

```java
public class AbstractFactoryPatternDemo {
   public static void main(String[] args) {

      //get shape factory
      AbstractFactory shapeFactory = FactoryProducer.getFactory("SHAPE");

      //get an object of Shape Circle
      Shape shape1 = shapeFactory.getShape("CIRCLE");

      //call draw method of Shape Circle
      shape1.draw();

      //get an object of Shape Rectangle
      Shape shape2 = shapeFactory.getShape("RECTANGLE");

      //call draw method of Shape Rectangle
      shape2.draw();

      //get an object of Shape Square 
      Shape shape3 = shapeFactory.getShape("SQUARE");

      //call draw method of Shape Square
      shape3.draw();

      //get color factory
      AbstractFactory colorFactory = FactoryProducer.getFactory("COLOR");

      //get an object of Color Red
      Color color1 = colorFactory.getColor("RED");

      //call fill method of Red
      color1.fill();

      //get an object of Color Green
      Color color2 = colorFactory.getColor("Green");

      //call fill method of Green
      color2.fill();

      //get an object of Color Blue
      Color color3 = colorFactory.getColor("BLUE");

      //call fill method of Color Blue
      color3.fill();
   }
}
```

结果如下：

```logcat
Inside Circle::draw() method.
Inside Rectangle::draw() method.
Inside Square::draw() method.
Inside Red::fill() method.
Inside Green::fill() method.
Inside Blue::fill() method.
```

### 4.3 小结

抽象工厂模式的一个显著优点是分离接口与实现，客户端使用抽象工厂模式创建对象，客户端不知道具体的实现是谁，这样能够使其从具体的产品实现中解耦，同时基于接口与实现的分离，使抽象工厂模式在切换产品类时更加灵活。缺点就是增加了很多类文件，不太容易扩展新的产品类。

## 5 总结

工厂模式对于创建不同的产品对象是一个很好的选择，简单工厂模式就是建立一个工厂类，对实现了同一接口的一些类进行实例的创建。工厂方法模式的定义是定义一个创建对象的接口，但由子类决定要实例化的类是哪一个。工厂方法模式把类实例化的过程推迟到子类。抽象工厂模式的定义是为创建一组相关或者相互依赖的对象提供一个接口，而不需要指定他们的具体类。