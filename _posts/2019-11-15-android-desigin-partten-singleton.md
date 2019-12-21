---
layout: post
title:  "设计模式之单例模式"
date:   2019-11-15 00:00:01
catalog:  true
tags:
    - java
    - 设计模式
    - 设计原则
---



## 1 概述

单例模式是应用最广泛的模式之一，该模式确保某各类只有一个实例对象，而且自行实例化并向整个系统提供这个实例。

## 2 使用场景

确保各类有且只有一个对象的场景，避免产生多个对象小豪过多的资源，或者某种类型的对象只应该有且只有一个。例如，创建一个对象需要消耗的资源过多，如要访问的IO和数据库等资源，这时就要考虑使用单例模式。

## 3 UML类图

![uml-singleton](/images/design-partten/uml-singleton.png)

角色介绍：

- Client:：高层客户端
- Singleton：单例类

实现单例模式主要关键点：

- 构造函数不对外开放，一般为private
- 通过一个静态方法或枚举返回单例类对象
- 确保单例类对象在反序列化时不会重新构建对象

通过将单例类的构造函数私有化，使得客户端代码不能通过new的形式手动构造单例类的对象。

## 4 单例模式类型

单例模式有以下几种：

- 饿汉单例模式
- 懒汉单例模式
- DCL实现单例模式
- 静态内部类单例模式
- 枚举单例模式
- 使用容器实现单例模式

下面一一具体介绍每个单例模式的实现和优缺点。

### 4.1 饿汉单例模式

```java
public class Role {
    private static final Role sInstance = new Role();
    private Role() {}
    public static Role getInstance() {
        return sInstance;
    }
}
```

由于Role的构造函数私有化，因此Role的实例不能通过new创建，只能通过Role.getInstance()获取，而且该实例是在类加载的时候就已经创建了，因此得名饿汉单例模式，非常形象。那么什么时候类被加载呢？以下三种情况：

-  生成该类对象的时候，会加载该类及该类的所有父类 
-  访问该类的静态成员的时候 
-  class.forName("类名")

通过这种方式创建的单例实例的好处是不用担心线程安全的问题，因为在类加载的时候就已经创建了。缺点就是因为还没调用该类就已经创建了，所以如果没调用该类，就会造成资源浪费，但这点资源在如今的硬件中可忽略不计。

### 4.2 懒汉单例模式

```java
public class Role {
    private static Role sInstance = null;
    private Role() {}
    public static synchronized Role getInstance() {
        if(sInstance == null) {
            sInstance = new Role();
        }
        return sInstance;
    }
}
```

该方式是在该类被调用时才创建实例，getInstance()函数添加了synchronized关键字是为了应对线程安全问题，这样就能保证多线程开发中当且仅有一个实例创建，但是一旦该实例创建成功，后续继续调用getInstance()获取实例，都会进行同步，synchronized是一个很耗资源的机制，这样虽然保证了线程安全，但是会造成不必要的同步开销，因此不建议使用该方式创建单例实例。

### 4.3 DCL实现单例模式

上小节的懒汉单例模式虽然能创建单例实例，并且线程安全，但是多次调用getInstance()会造成资源浪费，DCL的意思是双重检查锁懒汉单例模式，该方式的优点是既能够在需要时才初始化单例，又能够保证线程安全，且单例对象初始化后调用getInstance()不进行同步锁，代码如下：

```java
public class Role {
    private static Role sInstance = null;
    private Role() {}
    public static Role getInstance() {
        if(sInstance == null) {
            synchronized(Role.class) {
                if(sInstance == null) {
                    sInstance = new Role();
                }
            }
        }
        return sInstance;
    }
}
```

getInstance()方法中对sInstance单例实例进行了两次判空，目的是：第一层判断是为了避免不必要的同步，第二层的判断是为了在null的情况下创建实例。

但是该种方式创建单例实例在高并发的情况下有可能失效，下面具体来分析下原因。

sInstance = new Role()这个方法实际上并不是一个原子操作，这句代码最终被翻译成多条汇编指令，大概做了3件事情：

1. 给Role的实例分配内存；
2. 调用Role()的构造函数，初始化成员字段；
3. 将sInstance 对象指向分配的内存空间(此刻sInstance 对象就不为null了)

但是，由于Java编译器允许处理器乱序执行，以及JDK1.5之前的内存模型中Cache、寄存器到主内存回写顺序的规定，上面的步骤2和步骤3就无法保证，即执行顺序有可能是1-2-3，也有可能是1-3-2。如果是后者，线程A在3执行完毕，2未执行的情况下，线程被切换到B，这时候线程B仅限第一层的判空，由于线程A已经执行了步骤3，此刻sInstance已经不为null了，因此线程B直接取走了sInstance，但是sInstance还没调用上述的步骤2，也就是说sInstance对象还没完全创建成功，如果使用就会出错，这就是该方式创建单例失效的原因。

好在JDK1.5或之后的版本已经修复了该问题，方案是用将sInstance定义为volatile，即private volatile static Role sInstance = null就可以保证sInstance 对象每次都是从主内存中读取，就可以解决该模式失效的问题。

DCL模式是使用最多的单例实现方式，它能够在需要时才实例化单例对象，并且能够在绝大多数场景下保证对象的唯一性，除非你的代码在并发场景比较复杂或版本低于JDK1.5版本下使用，否则该方式一般能够满足需求。

### 4.4 静态内部类单例模式

在《Java并发编程实践》该书中，建议使用静态内部类单例模式，代码如下：

```java
public class Role {
    private Role() {}
    public static Role getInstance() {
        return RoleHolder.sInstance;
    }
    private static class RoleHolder {
        private static final Role sInstance = new Role();
    }
}
```

由于类加载时不会执行静态内部类，因此第一次加载Role类时并不会初始化sInstance，只有在第一次调用Role的getInstance()方法时才会导致sInstance被初始化。因此，第一次调用getInstance()方法会导致虚拟机加载RoleHolder类，这种方式不仅能够确保线程安全，也能够保证单例对象的唯一性，同时也延迟了单例的实例化。

### 4.5 枚举单例模式

在《Effect Java》 中说道，最佳的单例实现模式就是枚举单例模式。利用枚举的特性，让JVM来帮我们保证线程安全和单一实例的问题。除此之外，写法还特别简单。 

```java
public enum RoleEnum {
    INSTANCE;
    public void doSomething() {
        System.out.println("doSomething");
    }
}
```

 调用方法： 

```java
RoleEnum.INSTANCE.doSomething();
```

 这里有几个原因关于为什么在Java中宁愿使用一个枚举量来实现单例模式： 

-  自由序列化； 
-  保证只有一个实例（即使使用反射机制和反序列化也无法多次实例化一个枚举量）； 
-  线程安全； 

## 5 Java反射技术破坏单例模式

 Java中的反射技术可以获取类的所有方法、成员变量、还能访问private的构造方法，这样一来，单例模式中用的私有构造函数被调用就会产生多个实例，拿静态内部类创建单例模式来测试一下。 

```java
public class Singleton {

    private static class SingletonHolder {
        private static Singleton instance = new Singleton();
    }

    private Singleton() {
        
    }

    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
}
```

 通过静态内部类的方式实现单例模式是线程安全的，同时静态内部类不会在Singleton类加载时就加载，而是在调用getInstance()方法时才进行加载，达到了懒加载的效果。 

似乎静态内部类看起来已经是最完美的方法了，其实不是，可能还存在反射攻击或者反序列化攻击。且看如下代码： 

```java
public static void main(String[] args) throws Exception {
    Singleton singleton = Singleton.getInstance();
    Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
    constructor.setAccessible(true);
    Singleton newSingleton = constructor.newInstance();
    System.out.println(singleton == newSingleton);
}
```

除了反射攻击之外，还可能存在反序列化攻击的情况。

总的来说，一般情况下没必要考虑反射技术和反序列化攻击破坏单例的，上面几种方式创建单例，通过枚举创建单例是最理想的方式了，枚举特性是JDK1.5引入的，因此，JDK1.5及之后的版本，推荐首选枚举创建单例。

## 6 Android源码中的单例模式

frameworks/base/core/ava/android/app/ResourcesManager.java

```java
public class ResourcesManager {
    private static ResourcesManager sResourcesManager;
    ...
    public static ResourcesManager getInstance() {
        synchronized (ResourcesManager.class) {
            if (sResourcesManager == null) {
                sResourcesManager = new ResourcesManager();
            }
            return sResourcesManager;
        }
    }
}
```

但在该类中没有将其构造函数私有化，也就是说是可以通过new ResourcesManager()的方式来创建ResourcesManager实例的，但是由于ResourcesManager.java是系统的类，而且该类标注了@hide声明，也就是说普通的app是不能访问ResourcesManager.java的，因此，不担心开发者创建多个ResourcesManager实例。但系统层面的模块还是可以通过new ResourcesManager()的方式来创建多个ResourcesManager实例的，只不过可能goole的工程师编写Android源码时约定指定通过ResourcesManager.getInstance()的方式取得ResourcesManager的实例吧。

再举一个Android源码应用单例模式的例子。

> frameworks/base/core/java/android/app/ActivityManager.java
>
> frameworks/base/core/java/android/uti/Singleton.java

创建ActivityManager.java实例的方式：

[->ActivityManager.java]

```java
public class ActivityManager {
    ...
    /*package*/ ActivityManager(Context context, Handler handler) {
        mContext = context;
    }
    ...
    /**
     * @hide
     */
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
    };
    ...
}
```

创建AMS的远端代理实例IActivityManager，是通过getService()方法创建的，调用Singleton实例方法的get方法取得IActivityManager实例。

事实上类加载时就已经初始化了IActivityManagerSingleton实例，这正是饿汉单例模式的方式。执行new Singleton的create()方法就是通过ServiceManager.getService(Context.ACTIVITY_SERVICE)的方式取得IActivityManager实例，然后调用Singleton的get()方法就可取得创建的IActivityManager实例。

[->Singleton.java]

```java
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

## 7 总结

由于在客户端通常没有高并发，因此选择哪种实现方式来创建单例模式都并不会有太大的影响，处于效率考虑，推荐使用DCL实现单例模式和静态内部类单例模式。
另外需要注意的是，单例对象如果持有Context，是很容易引起内存泄露的，需要注意传递给单例对象的Context对好是Application Context。