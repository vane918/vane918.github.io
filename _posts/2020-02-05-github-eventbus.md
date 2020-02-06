---
layout: post
title:  "开源库学习之EventBus"
date:   2020-02-05 00:00:00
catalog:  true
tags:
    - 开源库
    - EventBus
---



## 1 常规事件传递

常用的事件传递方式有以下几种：

- Intent意图，跳转+传参(局限性很大)
- Handler，通常用来更新主线程UI，使用不当容易出现内存泄露
- Interface接口，仅限于同一线程中数据交互
- BroadCastReceiver，有序广播+无序广播
- AIDL跨进程通信，代码阅读性不好，维护成本偏高
- 其他方式，比如本地文件存储

上面的方式或多或少都有点缺陷，EventBus开源库应运而生。

## 2 EventBus简介

EventBus是一个Android优化的publish/subscribe消息时间总线。简化了应用程序内各个组件间、组件与后台线程间的通信。由GreenRobot开发，Gihub地址是：[EventBus](https://github.com/greenrobot/EventBus)

### 2.1 三个角色

- Event：事件，它可以是任意类型，EventBus会根据事件类型进行全局的通知。
- Subscriber：事件订阅者，在EventBus 3.0之前我们必须定义以onEvent开头的那几个方法，分别是onEvent、onEventMainThread、onEventBackgroundThread和onEventAsync，而在3.0之后事件处理的方法名可以随意取，不过需要加上注解@subscribe，并且指定线程模型，默认是POSTING
- Publisher：事件的发布者，可以在任意线程里发布事件。一般情况下，使用EventBus.getDefault()就可以得到一个EventBus对象，然后再调用post(Object)方法即可。

下面是 EventBus官方给出的原理示意图。

![eventbus](/images/github/eventbus/eventbus.png)

### 2.2 四种线程模型

EventBus有四种线程模型，分别是：

- POSTING：默认，表示事件处理函数的线程跟发布事件的线程在同一个线程。
- MAIN：表示事件处理函数的线程在主线程(UI)线程，因此在这里不能进行耗时操作。
- BACKGROUND：表示事件处理函数的线程在后台线程，因此不能进行UI操作。如果发布事件的线程是主线程(UI线程)，那么事件处理函数将会开启一个后台线程，如果果发布事件的线程是在后台线程，那么事件处理函数就使用该线程。
- ASYNC：表示无论事件发布的线程是哪一个，事件处理函数始终会新建一个子线程运行，同样不能进行UI操作。

## 3 EventBus的使用

EventBus的使用步骤分为定义事件、订阅事件、发送事件、处理事件、取消订阅五步。

### 3.1 引入依赖

```groovy
implementation 'org.greenrobot:eventbus:3.1.1'
```

### 3.2 定义事件

```java
public class MyEvent {

    private String message;

    public String getMessage() {
        return message;
    }

    public MyEvent(String message) {
        this.message = message;
    }
}
```

### 3.3 注册订阅事件

```java
@Override
 public void onStart() {
     super.onStart();
     EventBus.getDefault().register(this);
 }
```

### 3.4 取消订阅事件

```java
@Override
 public void onStop() {
     super.onStop();
     EventBus.getDefault().unregister(this);
 }
```

### 3.5 发布事件

```java
EventBus.getDefault().post(new MyEvent("hello"));
```

### 3.6 消费事件

```java
@Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessage(MyEvent event) {
}
```

### 3.7 EventBus高级用法

#### 3.7.1 黏性事件

使用上述的步骤发布事件,如果一个类还没创建,那么该类是不能接受发布的事件的,需要先注册,监听事件的发布才行.这个时候采用黏性事件就能解决.所谓的黏性事件,就是指发送了该事件之后再注册EventBus,订阅者依然能够接收到的事件.使用黏性事件的时候有两个地方需要做些修改.一个是订阅事件的地方,这里我们在先打开的Activity中注册监听黏性事件：

```java
@Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
    public void onMessage(MyEvent event) {
}
```

Subscribe注解多了sticky = true.另一个修改的地方是发布事件的 API :

```java
EventBus.getDefault().postSticky(new MyEvent("hello"));
```

将 post()改为postSticky(),其他不变.

#### 3.7.2 优先级

在`Subscribe`注解中总共有3个参数，上面我们用到了其中的两个，这里我们使用以下第三个参数即`priority`.它用来指定订阅方法的优先级,是一个整数类型的值,默认是0,值越大表示优先级越大.在某个事件被发布出来的时候,优先级较高的订阅方法会首先接受到事件.需要注意以下两点:

1. 只有当相同的订阅方法使用相同的`ThreadMode`参数的时候,它们的优先级才生效,会与`priority`指定的值一致.
2. 只有当某个订阅方法的`ThreadMode`参数为`POSTING`的时候，它才能停止该事件的继续分发.

## 4 EventBus 源码解析

从以下几个方面着手分析EventBus的源码：

- Subscribe注解
- 注册事件订阅方法
- 取消注册
- 发送事件
- 事件处理
- 粘性事件
- Subscriber Index

### 4.1 Subscribe注解

EventBus从3.0开始使用Subscribe注解配置事件订阅方法，不再使用方法名，例如：

```java
@Subscribe(threadMode = ThreadMode.MAIN)
    public void onMessage(MyEvent event) {
      // do something
}
```

其中，事件类型可以是 Java 中已有的类型或者自定义的类型。下面是Subscribe注解的具体实现：

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    // 指定事件订阅方法的线程模式，即在那个线程执行事件订阅方法处理事件，默认为POSTING
    ThreadMode threadMode() default ThreadMode.POSTING;
    // 是否支持粘性事件，默认为false
    boolean sticky() default false;
    // 指定事件订阅方法的优先级，默认为0，如果多个事件订阅方法可以接收相同事件的，则优先级高的先接收到事件
    int priority() default 0;
}
```

可以发现，在使用Subscribe注解时可以根据需求指定threadMode、sticky、priority三个属性。其中，threadMode属性见2.2 小节.

### 4.2 注册事件

使用EventBus时，需要在在需要接收事件的地方注册事件，注册事件的方式如下：

```java
EventBus.getDefault().register(this);
```

点击打开getDefault()会发现，getDefault()是一个单例方法，保证当前只有一个EventBus实例。

```java
public static EventBus getDefault() {
  if (defaultInstance == null) {
    synchronized (EventBus.class) {
      if (defaultInstance == null) {
        defaultInstance = new EventBus();
      }
    }
  }
  return defaultInstance;
}
```

new EventBus()最终会调用EventBus(EventBusBuilder builder) 的构造方法,EventBusBuilder是采用构建者设计模式创建 EventBus 对象,默认采用默认的EventBusBuilder,开发者也可以自定义EventBusBuilder.
创建 EventBus 对象后,就可以注册事件了.

```java
public void register(Object subscriber) {
  // 得到当前要注册类的Class对象
  Class<?> subscriberClass = subscriber.getClass();
  // 根据Class查找当前类中订阅了事件的方法集合，即使用了Subscribe注解、有public修饰符、一个参数的方法
  // SubscriberMethod类主要封装了符合条件方法的相关信息：Method对象、线程模式、事件类型、优先级、是否是粘性事等
  List<SubscriberMethod> subscriberMethods
    subscriberMethodFinder.findSubscriberMethods(subscriberClass);
  synchronized (this) {
    // 循环遍历订阅了事件的方法集合，以完成注册
    for (SubscriberMethod subscriberMethod : subscriberMethods) {
      subscribe(subscriber, subscriberMethod);
    }
  }
}
```

注册事件分为两个步骤:

1. 查找方法:findSubscriberMethods()
2. 遍历注册事件:subscribe()

#### 4.2.1 findSubscriberMethods

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
  // METHOD_CACHE是一个ConcurrentHashMap，直接保存了subscriberClass和对应SubscriberMethod的集合，    以提高注册效率，赋值重复查找。
  List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
  if (subscriberMethods != null) {
    return subscriberMethods;
  }
  // 由于使用了默认的EventBusBuilder，则ignoreGeneratedIndex属性默认为false，即是否忽略注解生成器
  //可以看到，EventBus3.0以上虽然是以注解器，但是如果采用默认的构建者创建EventBus对象，依然是采用反射机制
  if (ignoreGeneratedIndex) {
    subscriberMethods = findUsingReflection(subscriberClass);
  } else {
    subscriberMethods = findUsingInfo(subscriberClass);
  }
  // 如果对应类中没有符合条件的方法，则抛出异常
  if (subscriberMethods.isEmpty()) {
    throw new EventBusException("Subscriber " + subscriberClass
                                + " and its super classes have no public methods with the @Subscribe annotation");
  } else {
    // 保存查找到的订阅事件的方法
    METHOD_CACHE.put(subscriberClass, subscriberMethods);
    return subscriberMethods;
  }
}
```

总的来说,findSubscriberMethods()方法先从缓存中查找,如果找到则直接返回,否则去做下一步的查找过程,然后缓存查找到的集合,根据上边的注释可知findUsingInfo()方法会被调用.

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {       
  FindState findState = prepareFindState();
  findState.initForSubscriber(subscriberClass);
  // 初始状态下findState.clazz就是subscriberClass
  while (findState.clazz != null) {
    findState.subscriberInfo = getSubscriberInfo(findState);
    if (findState.subscriberInfo != null) {
      SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
      for (SubscriberMethod subscriberMethod : array) {
        if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
          findState.subscriberMethods.add(subscriberMethod);
        }
      }
    } else {
      // 通过反射查找订阅事件的方法
      findUsingReflectionInSingleClass(findState);
    }
    // 修改findState.clazz为subscriberClass的父类Class，即需要遍历父类
    findState.moveToSuperclass();
  }
  // 查找到的方法保存在了FindState实例的subscriberMethods集合中。
  // 使用subscriberMethods构建一个新的List<SubscriberMethod>,释放掉findState
  return getMethodsAndRelease(findState);
}
```

findUsingInfo()方法会在当前要注册的类以及其父类中查找订阅事件的方法，具体的查找过程在findUsingReflectionInSingleClass()方法里，它主要通过反射来查找订阅事件的方法。

```java
private void findUsingReflectionInSingleClass(FindState findState) {
  ...
  // 循环遍历当前类的方法，筛选出符合条件的
  for (Method method : methods) {
    // 获得方法的修饰符
    int modifiers = method.getModifiers();
    // 如果是public类型，但非abstract、static等
    if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
      // 获得当前方法所有参数的类型
      Class<?>[] parameterTypes = method.getParameterTypes();
      // 如果当前方法只有一个参数
      if (parameterTypes.length == 1) {
        Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
        // 如果当前方法使用了Subscribe注解
        if (subscribeAnnotation != null) {
          // 得到该参数的类型
          Class<?> eventType = parameterTypes[0];
          // checkAdd()方法用来判断FindState的anyMethodByEventType map是否已经添加过以当前
          //eventType为key的键值对，没添加过则返回true
          if (findState.checkAdd(method, eventType)) {
            // 得到Subscribe注解的threadMode属性值，即线程模式
            ThreadMode threadMode = subscribeAnnotation.threadMode();
            // 创建一个SubscriberMethod对象，并添加到subscriberMethods集合
            findState.subscriberMethods.add(new SubscriberMethod(method,
eventType,threadMode,subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
          }
        }
      } 
      ...
  }
}
```

#### 4.2.2 subscribe

到此register()方法中findSubscriberMethods()流程就分析完了，我们已经找到了当前注册类及其父类中订阅事件的方法的集合。接下来分析下具体的注册流程，即register()中的subscribe()方法。首先，我们看一下subscribe()方法的源码：

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
  // 得到当前订阅了事件的方法的参数类型
  Class<?> eventType = subscriberMethod.eventType;
  // Subscription类保存了要注册的类对象以及当前的subscriberMethod
  Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
  // subscriptionsByEventType是一个HashMap，保存了以eventType为key,Subscription对象集合为value的键值对
  // 先查找subscriptionsByEventType是否存在以当前eventType为key的值
  CopyOnWriteArrayList<Subscription> subscriptions =
    subscriptionsByEventType.get(eventType);
  // 如果不存在，则创建一个subscriptions，并保存到subscriptionsByEventType
  if (subscriptions == null) {
    subscriptions = new CopyOnWriteArrayList<>();
    subscriptionsByEventType.put(eventType, subscriptions);
  } else {
    if (subscriptions.contains(newSubscription)) {
      throw new EventBusException("Subscriber " + subscriber.getClass() + " already
                                  registered to event " + eventType);
    }
  }
  // 添加上边创建的newSubscription对象到subscriptions中
  int size = subscriptions.size();
  for (int i = 0; i <= size; i++) {
    if (i == size || subscriberMethod.priority >
        subscriptions.get(i).subscriberMethod.priority) {
      subscriptions.add(i, newSubscription);
      break;
    }
  }
  // typesBySubscribere也是一个HashMap，保存了以当前要注册类的对象为key，注册类中订阅事件的方法
  //的参数类型的集合为value的键值对
  // 查找是否存在对应的参数类型集合
  List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
  // 不存在则创建一个subscribedEvents，并保存到typesBySubscriber
  if (subscribedEvents == null) {
    subscribedEvents = new ArrayList<>();
    typesBySubscriber.put(subscriber, subscribedEvents);
  }
  // 保存当前订阅了事件的方法的参数类型
  subscribedEvents.add(eventType);
  // 粘性事件相关的流程
  if (subscriberMethod.sticky) {
    ...
  }
}
```

可以发现，subscribe()方法主要是得到了subscriptionsByEventType、typesBySubscriber两个 HashMap。其中，发送事件的时候要用到subscriptionsByEventType，完成事件的处理。当取消 EventBus 注册的时候要用到typesBySubscriber、subscriptionsByEventType，完成相关资源的释放。

### 4.3 取消注册

接下来，我们看一下EventBus 取消事件注册的流程。

```java
EventBus.getDefault().unregister(this);
```


其中，unregister()的源码如下：

```java
public synchronized void unregister(Object subscriber) {
  // 得到当前注册类对象 对应的 订阅事件方法的参数类型 的集合
  List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
  if (subscribedTypes != null) {
    // 遍历参数类型集合，释放之前缓存的当前类中的Subscription
    for (Class<?> eventType : subscribedTypes) {
      unsubscribeByEventType(subscriber, eventType);
    }
    // 删除以subscriber为key的键值对
    typesBySubscriber.remove(subscriber);
  } else {
    logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
  }
}
```


unregister()方法调用了unsubscribeByEventType()方法：

```java
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
  // 得到当前参数类型对应的Subscription集合
  List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
  if (subscriptions != null) {
    int size = subscriptions.size();
    // 遍历Subscription集合
    for (int i = 0; i < size; i++) {
      Subscription subscription = subscriptions.get(i);
      // 如果当前subscription对象对应的注册类对象 和 要取消注册的注册类对象相同，则删除当前subscription对象
      if (subscription.subscriber == subscriber) {
        subscription.active = false;
        subscriptions.remove(i);
        i--;
        size--;
      }
    }
  }
}
```


所以，在unregister()方法中，最主要的就是释放typesBySubscriber、subscriptionsByEventType中缓存的资源。

### 4.4 发送事件

在EventBus中，我们发送一个事件使用的是如下的方式：

```java
EventBus.getDefault().post("Hello EventBus!")
```

#### 4.4.1 post

可以看到，发送事件就是通过post()方法完成的，post()方法的源码如下：

```java
public void post(Object event) {
    // currentPostingThreadState是一个PostingThreadState类型的ThreadLocal
    // PostingThreadState类保存了事件队列和线程模式等信息
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    // 将要发送的事件添加到事件队列
    eventQueue.add(event);
    // isPosting默认为false
    if (!postingState.isPosting) {
        // 是否为主线程
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");           
        }
      try {
        // 遍历事件队列
        while (!eventQueue.isEmpty()) {
          // 发送单个事件
          // eventQueue.remove(0)，从事件队列移除事件
          postSingleEvent(eventQueue.remove(0), postingState);
        }
      } finally {
        postingState.isPosting = false;
        postingState.isMainThread = false;
      }
    }
}
```

可以发现，post()方法先将发送的事件保存到List的事件队列，然后通过循环出队列，将事件交给postSingleEvent()方法处理。

#### 4.4.2 postSingleEvent

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
  Class<?> eventClass = event.getClass();
  boolean subscriptionFound = false;
  // eventInheritance默认为true，表示是否向上查找事件的父类
  if (eventInheritance) {
    // 查找当前事件类型的Class，连同当前事件类型的Class保存到集合
    List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
    int countTypes = eventTypes.size();
    // 遍历Class集合，继续处理事件
    for (int h = 0; h < countTypes; h++) {
      Class<?> clazz = eventTypes.get(h);
      subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);           
    }
  } else {
    subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
  }
  if (!subscriptionFound) {
    if (logNoSubscriberMessages) {
      logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
    }
    if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
        eventClass != SubscriberExceptionEvent.class) {
      post(new NoSubscriberEvent(this, event));
    }
  }
}
```

postSingleEvent()方法中，根据eventInheritance属性，决定是否向上遍历事件的父类型，然后用postSingleEventForEventType()方法进一步处理事件。

#### 4.4.3 postSingleEventForEventType

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
  CopyOnWriteArrayList<Subscription> subscriptions;
  synchronized (this) {
    // 获取事件类型对应的Subscription集合
    subscriptions = subscriptionsByEventType.get(eventClass);
  }
  // 如果已订阅了对应类型的事件
  if (subscriptions != null && !subscriptions.isEmpty()) {
    for (Subscription subscription : subscriptions) {
        // 记录事件
        postingState.event = event;
        // 记录对应的subscription
        postingState.subscription = subscription;
        boolean aborted = false;
        try {
          // 最终的事件处理
          postToSubscription(subscription, event, postingState.isMainThread);
          aborted = postingState.canceled;
        } finally {
          postingState.event = null;
          postingState.subscription = null;
          postingState.canceled = false;
        }
      if (aborted) {
        break;
      }
    }
    return true;
  }
  return false;
}
```

可以发现，postSingleEventForEventType()方法核心就是遍历发送的事件类型对应的Subscription集合，然后调用postToSubscription()方法处理事件。

#### 4.4.4 postToSubscription

postToSubscription()内部会根据订阅事件方法的线程模式，间接或直接的以发送的事件为参数，通过反射执行订阅事件的方法。

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
  // 判断订阅事件方法的线程模式
  switch (subscription.subscriberMethod.threadMode) {
      // 默认的线程模式，在那个线程发送事件就在那个线程处理事件
    case POSTING:
      invokeSubscriber(subscription, event);
      break;
      // 在主线程处理事件
    case MAIN:
      // 如果在主线程发送事件，则直接在主线程通过反射处理事件
      if (isMainThread) {
        invokeSubscriber(subscription, event);
      } else {
        // 如果是在子线程发送事件，则将事件入队列，通过Handler切换到主线程执行处理事件
        // mainThreadPoster 不为空
        mainThreadPoster.enqueue(subscription, event);
      }
      break;
      // 无论在那个线程发送事件，都先将事件入队列，然后通过 Handler 切换到主线程，依次处理事件。
      // mainThreadPoster 不为空
    case MAIN_ORDERED:
      if (mainThreadPoster != null) {
        mainThreadPoster.enqueue(subscription, event);
      } else {
        invokeSubscriber(subscription, event);
      }
      break;
    case BACKGROUND:
      // 如果在主线程发送事件，则先将事件入队列，然后通过线程池依次处理事件
      if (isMainThread) {
        backgroundPoster.enqueue(subscription, event);
      } else {
        // 如果在子线程发送事件，则直接在发送事件的线程通过反射处理事件
        invokeSubscriber(subscription, event);
      }
      break;
      // 无论在那个线程发送事件，都将事件入队列，然后通过线程池处理。
    case ASYNC:
      asyncPoster.enqueue(subscription, event);
      break;
    default:
      throw new IllegalStateException("Unknown thread mode: " +
                                      subscription.subscriberMethod.threadMode);
  }
}
```

可以看到，postToSubscription()方法就是根据订阅事件方法的线程模式、以及发送事件的线程来判断如何处理事件，至于处理方式主要有两种：一种是在相应线程直接通过invokeSubscriber()方法，用反射来执行订阅事件的方法，这样发送出去的事件就被订阅者接收并做相应处理了。
首先，我们来看一下invokeSubscriber()方法

#### 4.4.5 invokeSubscriber

```java
void invokeSubscriber(Subscription subscription, Object event) {
  try {
    subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
  } catch (InvocationTargetException e) {
    handleSubscriberException(subscription, event, e.getCause());
  } catch (IllegalAccessException e) {
    throw new IllegalStateException("Unexpected exception", e);
  }
}
```


如果在子线程发送事件，则直接在发送事件的线程通过反射处理事件。另外一种是先将事件入队列（其实底层是一个List），然后做进一步处理，我们以mainThreadPoster.enqueue(subscription, event)为例简单的分析下，其中mainThreadPoster是HandlerPoster类的一个实例。

```java
public class HandlerPoster extends Handler implements Poster {
  private final PendingPostQueue queue;
  private boolean handlerActive;
  ...
  public void enqueue(Subscription subscription, Object event) {
    // 用subscription和event封装一个PendingPost对象
    PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
    synchronized (this) {
      // 入队列
      queue.enqueue(pendingPost);
      if (!handlerActive) {
        handlerActive = true;
        // 发送开始处理事件的消息，handleMessage()方法将被执行，完成从子线程到主线程的切换
        if (!sendMessage(obtainMessage())) {
          throw new EventBusException("Could not send handler message");
        }
      }
  }
```

```java
@Override
public void handleMessage(Message msg) {
  boolean rescheduled = false;
  try {
    long started = SystemClock.uptimeMillis();
    // 死循环遍历队列
    while (true) {
      // 出队列
      PendingPost pendingPost = queue.poll();
      ...
        // 进一步处理pendingPost
        eventBus.invokeSubscriber(pendingPost);
        ...  
    }
  } finally {
    handlerActive = rescheduled;
  }
}
```
可以发现，HandlerPoster的enqueue()方法主要就是将subscription、event对象封装成一个PendingPost对象，然后保存到队列里，之后通过Handler切换到主线程，在handleMessage()方法将中将PendingPost对象循环出队列，交给invokeSubscriber()方法做进一步处理。

```java
void invokeSubscriber(PendingPost pendingPost) {
  Object event = pendingPost.event;
  Subscription subscription = pendingPost.subscription;
  // 释放pendingPost引用的资源
  PendingPost.releasePendingPost(pendingPost);
  if (subscription.active) {
    // 用反射来执行订阅事件的方法
    invokeSubscriber(subscription, event);
  }
}
```


invokeSubscriber方法比较简单，主要就是从pendingPost中取出之前保存的event、subscription，然后用反射来执行订阅事件的方法，又回到了第一种处理方式。所以mainThreadPoster.enqueue(subscription, event)的核心就是先将将事件入队列，然后通过Handler从子线程切换到主线程中去处理事件。backgroundPoster.enqueue()和asyncPoster.enqueue也类似，内部都是先将事件入队列，然后再出队列，但是会通过线程池去进一步处理事件。

### 4.5 粘性事件

一般情况，我们使用 EventBus 都是准备好订阅事件的方法，然后注册事件，最后在发送事件，即要先有事件的接收者。但粘性事件却恰恰相反，我们可以先发送事件，后续再准备订阅事件的方法、注册事件。
首先，我们看一下EventBus的粘性事件是如何使用的：

```java
EventBus.getDefault().postSticky("Hello EventBus!");
```

黏性事件使用了postSticky()方法，postSticky()方法的源码如下：


```java
public void postSticky(Object event) {
  synchronized (stickyEvents) {
    stickyEvents.put(event.getClass(), event);
  }
  post(event);
}
```

postSticky()方法主要做了两件事：先将事件类型和对应事件保存到stickyEvents中，方便后续使用；然后执行post(event)继续发送事件，这个post()方法就是之前发送的post()方法。所以，如果在发送粘性事件前，已经有了对应类型事件的订阅者，及时它是非粘性的，依然可以接收到发送出的粘性事件。
发送完粘性事件后，再准备订阅粘性事件的方法，并完成注册。核心的注册事件流程还是我们之前的register()方法中的subscribe()方法，前边分析subscribe()方法时，有一段没有分析的代码，就是用来处理粘性事件的。

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
  ...
  // 如果当前订阅事件的方法的Subscribe注解的sticky属性为true，即该方法可接受粘性事件
    if (subscriberMethod.sticky) {
      // 默认为true，表示是否向上查找事件的父类
      if (eventInheritance) {
        // stickyEvents就是发送粘性事件时，保存了事件类型和对应事件
        Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
        for (Map.Entry<Class<?>, Object> entry : entries) {
          Class<?> candidateEventType = entry.getKey();
          // 如果candidateEventType是eventType的子类或
          if (eventType.isAssignableFrom(candidateEventType)) {
            // 获得对应的事件
            Object stickyEvent = entry.getValue();
            // 处理粘性事件
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
          }
        }
      } else {
        Object stickyEvent = stickyEvents.get(eventType);
        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
      }
    }
}
```


可以看到，处理粘性事件就是在 EventBus 注册时，遍历stickyEvents，如果当前要注册的事件订阅方法是粘性的，并且该方法接收的事件类型和stickyEvents中某个事件类型相同或者是其父类，则取出stickyEvents中对应事件类型的具体事件，做进一步处理。
subscribe()方法最核心的就是checkPostStickyEventToSubscription()方法。

```java
private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
  if (stickyEvent != null) {
    postToSubscription(newSubscription, stickyEvent, isMainThread());
  }
}
```

### 4.6 Subscriber索引

回顾上面对 EventBus 注册事件流程的分析，EventBus主要是在项目运行时通过反射来查找订事件的方法信息，如果项目中有大量的订阅事件的方法，必然会对项目运行时的性能产生影响。其实除了在项目运行时通过反射查找订阅事件的方法信息，EventBus 还提供了在项目编译时通过注解处理器查找订阅事件方法信息的方式，生成一个辅助的索引类来保存这些信息，这个索引类就是Subscriber Index，和 ButterKnife 的原理是类似的。
所以，我们在添加EventBus依赖的时候通常是下面这样的：

```groovy
dependencies {
    compile 'org.greenrobot:eventbus:3.1.1'
    // 引入注解处理器
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
```


然后在项目的 Application 中添加如下配置，可以生成一个默认的 EventBus 单例。

```java
EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
```


其中，MyEventBusIndex()方法的源码如下：

```java
public class MyEventBusIndex implements SubscriberInfoIndex {
  private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;
  static {
    SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();
    putIndex(new SimpleSubscriberInfo(MainActivity.class, true, new SubscriberMethodInfo[] {
      new SubscriberMethodInfo("changeText", String.class),
    }));
}
  private static void putIndex(SubscriberInfo info) {
    SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
  }
  @Override
  public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
    SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
    if (info != null) {
      return info;
    } else {
      return null;
    }
  }
}
```

其中SUBSCRIBER_INDEX是一个HashMap，保存了当前注册类的 Class 类型和其中事件订阅方法的信息。接下来，我们再来分析下使用 Subscriber 索引时 EventBus 的注册流程。首先，创建一个EventBusBuilder，然后通过addIndex()方法添加索引类的实例。

```java
public EventBusBuilder addIndex(SubscriberInfoIndex index) {
  if (subscriberInfoIndexes == null) {
    subscriberInfoIndexes = new ArrayList<>();
  }
  subscriberInfoIndexes.add(index);
  return this;
}
```


即把生成的索引类的实例保存在subscriberInfoIndexes集合中，然后用installDefaultEventBus()创建默认的 EventBus实例。

```java
public EventBus installDefaultEventBus() {
  synchronized (EventBus.class) {
    if (EventBus.defaultInstance != null) {
      throw new EventBusException("Default instance already exists." +
                                  " It may be only set once before it's used the first time to ensure consistent behavior.");
    }
    EventBus.defaultInstance = build();
    return EventBus.defaultInstance;
  }
}
```

即用当前EventBusBuilder对象创建一个 EventBus 实例，这样我们通过EventBusBuilder配置的 Subscriber Index 也就传递到了EventBus实例中，然后赋值给EventBus的 defaultInstance成员变量。
所以在 Application 中生成了 EventBus 的默认单例，这样就保证了在项目其它地方执行EventBus.getDefault()就能得到唯一的 EventBus 实例！
由于我们现在使用了 Subscriber Index 所以不会通过findUsingReflectionInSingleClass()来反射解析订阅事件的方法。我们重点来看getSubscriberInfo()方法。

```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
  // 该条件不成立
  if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
    SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
    if (findState.clazz == superclassInfo.getSubscriberClass()) {
      return superclassInfo;
    }
  }
  // 该条件成立
  if (subscriberInfoIndexes != null) {
    // 遍历索引类实例集合
    for (SubscriberInfoIndex index : subscriberInfoIndexes) {
      // 根据注册类的 Class 类查找SubscriberInfo
      SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
      if (info != null) {
        return info;
      }
    }
  }
  return null;
}
```


subscriberInfoIndexes就是在前边addIndex()方法中创建的，保存了项目中的索引类实例，即MyEventBusIndex的实例，继续看索引类的getSubscriberInfo()方法，来到了MyEventBusIndex类中。

```java
@Override
public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
  SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
  if (info != null) {
    return info;
  } else {
    return null;
  }
}
```

即根据注册类的 Class 类型从 SUBSCRIBER_INDEX 查找对应的SubscriberInfo，如果我们在注册类中定义了订阅事件的方法，则 info不为空，进而上边findUsingInfo()方法中findState.subscriberInfo != null成立，到这里主要的内容就分析完了，其它的和之前的注册流程一样。

所以 Subscriber Index 的核心就是项目编译时使用注解处理器生成保存事件订阅方法信息的索引类，然后项目运行时将索引类实例设置到 EventBus 中，这样当注册 EventBus 时，从索引类取出当前注册类对应的事件订阅方法信息，以完成最终的注册，避免了运行时反射处理的过程，所以在性能上会有质的提高。项目中可以根据实际的需求决定是否使用 Subscriber Index。

### 4.7 完整流程

最后我们再来看一下EventBus的流程图。
![eventbus-post](/images/github/eventbus/eventbus-post.png)

![eventbus-register](/images/github/eventbus/eventbus-register.png)

![eventbus-unregister](/images/github/eventbus/eventbus-unregister.png)