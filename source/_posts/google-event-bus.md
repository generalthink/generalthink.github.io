---
title: 如果你知道观察者模式,那你也应该了解下EventBus
date: 2020-09-10 15:19:50
tags: java
---

观察者模式又叫发布-订阅模式，它定义了一种一对多的依赖关系，多个观察者对象可同时监听某一主题对象，当该主题对象状态发生变化时，相应的所有观察者对象都可收到通知。
比如求职者，他们订阅了一些工作发布网站，当有合适的工作机会时，他们会收到提醒。


又或者是当用户注册网站成功的时候，发送一封邮件或者发送一条短信。我们都可以使用观察者模式来解决类似的问题

<!--more-->

关于观察者模式的基本模型代码如下：


```java
public interface Subject {

  void registerObserver(Observer observer);

  void unregisterObserver(Observer observer);

  void notifyObservers(Message message);
}

interface Observer {
    void update(Message message);
}
@Data
class Message {
  String id;
  String name;
}

// 具体的主题
class UserRegisterSubject implements Subject {

    List<Observer> observerList = new ArrayList<Observer>();

    public void registerObserver(Observer observer) {
      observerList.add(observer);
    }

    public void unregisterObserver(Observer observer) {
      observerList.remove(observer);
    }

    public void notifyObservers(Message message) {

      for (Observer observer : observerList) {
        observer.update(message);
      }
    }
}

class RegNotificationObserver implements Observer {

  public void update(Message message) {
    System.out.println("注册成功，已经发送邮件给" + message.getName());
  }
}

class RegOtherObserver implements Observer {

  public void update(Message message) {
    System.out.println("注册成功，发送优惠券给" + message.getName());
  }
}

class Main {

  public static void main(String[] args) {
      
    // 实际使用的时候配合Spring使用
    Subject subject = new UserRegisterSubject();
    subject.registerObserver(new RegNotificationObserver());
    subject.registerObserver(new RegOtherObserver());

    boolean registSuccess = true;
    if(registSuccess) {
      Message msg = new Message();
      msg.setId("123456");
      msg.setName("think123");
      subject.notifyObservers(msg);
    }
  }
}

```

输出结果如下:
注册成功，已经发送邮件给think123
注册成功，发送优惠券给think123


从上面的代码可以看出，观察者模式中我们首先需要注册观察者,然后当某个事件发生的时候通知观察者。

而在google guava中对于观察者模式的框架实现叫做EventBus,实现方式更为优雅,我们来看看如何使用EventBus,然后再深入分析下它的源码。

```java
public class EventBusDemo {

  public static void main(String[] args) {
      EventBus eventBus = new EventBus("think123");

    Offer offer = new Offer();
    offer.setCompany("蚂蚁金服");
    offer.setMoney(20000);

    // 注册观察者
    eventBus.register(new EmailNotificationObserver());

    eventBus.register(new MessageNotificationObserver());
    
    // 触发MessageNotification
    eventBus.post(offer.getCompany());

    // 触发EmailNotification
    eventBus.post(offer);
  }

}

@Data
class Offer {
    private String company;
    private Integer money;
}

// 发送邮件
class EmailNotificationObserver {
    @Subscribe
    public void mailNotification(Offer offer) {
      System.out.println("恭喜你被 " + offer.getCompany() + " 录取,每月工资为" + offer.getMoney() + "元");
    }

}

// 发送消息
class MessageNotificationObserver {
    @Subscribe
    public void messageNotification(String company) {
      System.out.println("恭喜你被" + company + "录取了");
    }
}

```

可以看出来,EventBus的使用更加简单,我们只需要编写自己的observer就可以了，然后在需要处理通知的方法上加上`@Subscribe`注解就行了。然后当post传入参数的时候，就会找到哪些观察者可以处理这样的参数，就调用观察者的这个方法。

可以理解为观察者订阅了某个事件,当事件发生的时候，观察者会执行指定的动作。
比如EmailNotificationObserver订阅了Offer事件(事件就可以认为是参数)，所以在收到通知后会发送邮件(这里使用打印来代替)


让我们看看EventBus的核心代码：


```java
public class EventBus {

  // 标识EventBus,可以理解为name
  private final String identifier;

  // 具体的线程池,实际上directExecutor,它实际上是单线程
  private final Executor executor;

  // 异常处理器，负责处理异常
  private final SubscriberExceptionHandler exceptionHandler;

  // 订阅中心,存储有哪些订阅者。 这里将eventBus传递给了订阅中心
  private final SubscriberRegistry subscribers = new SubscriberRegistry(this);

  // 事件转发器,负责转发event给订阅者
  private final Dispatcher dispatcher;

  // 构造方法
  public EventBus(String identifier) {
    this(
        identifier,
        MoreExecutors.directExecutor(),
        Dispatcher.perThreadDispatchQueue(),
        LoggingHandler.INSTANCE);
  }

  // 注册订阅者
  public void register(Object object) {
    subscribers.register(object);
  }

  // 移除订阅者
  public void unregister(Object object) {
    subscribers.unregister(object);
  }

  // 投送event给所有注册的订阅者
  public void post(Object event) {
    Iterator<Subscriber> eventSubscribers = subscribers.getSubscribers(event);
    if (eventSubscribers.hasNext()) {
      dispatcher.dispatch(event, eventSubscribers);
    } else if (!(event instanceof DeadEvent)) {

      // 没有找到订阅者,则封装成DeadEvent(默认是丢弃掉了)
      post(new DeadEvent(this, event));
    }
  }

```

EventBus中主要的方法就是注册/移除订阅者，然后分发事件。保留了主体流程的同时也让不同的类承担自己的职责，真的很赞。

在注册订阅者中，会调用`findAllSubscribers`方法从缓存中加载已有的订阅者，并且为了保证线程安全，会使用`CopyOnWriteArraySet`来保存对应的订阅者。

订阅者为什么会存在多个(用了set保存)呢？这是因为我们eventBus.post方法的参数是Object类型,而在订阅者中可能会存在多个方法可以处理这个类型的参数(有多个订阅者都订阅了该事件)，所以会是多个。


然后会根据订阅者的Class加载所有标明了`@Subscribe`注解的方法，并将其放到缓存中

```java

void register(Object listener) {

  // 从缓存中获取所有的订阅者
  Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

  for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
    Class<?> eventType = entry.getKey();
    Collection<Subscriber> eventMethodsInListener = entry.getValue();

    // 根据参数类型获取到所有的订阅者
    CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);

    // 使用CopyOnWriteArraySet，保证线程安全
    if (eventSubscribers == null) {
      CopyOnWriteArraySet<Subscriber> newSet = new CopyOnWriteArraySet<>();
      eventSubscribers =
          MoreObjects.firstNonNull(subscribers.putIfAbsent(eventType, newSet), newSet);
    }

    eventSubscribers.addAll(eventMethodsInListener);
  }
}


 private Multimap<Class<?>, Subscriber> findAllSubscribers(Object listener) {
    Multimap<Class<?>, Subscriber> methodsInListener = HashMultimap.create();
    Class<?> clazz = listener.getClass();
    for (Method method : getAnnotatedMethods(clazz)) {
      Class<?>[] parameterTypes = method.getParameterTypes();
      Class<?> eventType = parameterTypes[0];
      // Subscriber中保存了要执行的对象以及方法
      // eventType就是参数类型，这里就形成了参数类型---》订阅者的映射
      // 而订阅者中保存了具体需要执行的类以及方法
      methodsInListener.put(eventType, Subscriber.create(bus, listener, method));
    }
    return methodsInListener;
  }

// 当缓存中没有的时候，会调用这个方法。所以最开始注册订阅者的时候都会调用这个方法
private static ImmutableList<Method> getAnnotatedMethodsNotCached(Class<?> clazz) {
    Set<? extends Class<?>> supertypes = TypeToken.of(clazz).getTypes().rawTypes();
    Map<MethodIdentifier, Method> identifiers = Maps.newHashMap();
    for (Class<?> supertype : supertypes) {
      for (Method method : supertype.getDeclaredMethods()) {
        // 只处理被Subscribe注解标明的方法并且method不能是合成的(isSynthetic)
        if (method.isAnnotationPresent(Subscribe.class) && !method.isSynthetic()) {
          Class<?>[] parameterTypes = method.getParameterTypes();
          // 参数个数只能为1
          checkArgument(
              parameterTypes.length == 1,
              "Method %s has @Subscribe annotation but has %s parameters."
                  + "Subscriber methods must have exactly 1 parameter.",
              method,
              parameterTypes.length);

          MethodIdentifier ident = new MethodIdentifier(method);
          if (!identifiers.containsKey(ident)) {
            identifiers.put(ident, method);
          }
        }
      }
    }
    return ImmutableList.copyOf(identifiers.values());
  }

```

可以看到,EventBus的订阅者之所以不用实现特定的接口实际上是利用了反射将订阅者和要执行的方法对应起来了的。

经过register方法之后，我们就知道每个订阅者分别订阅了哪些事件(能处理什么参数),并且形成了这样的对应关系：

```
事件类型(参数) ---> 订阅者(target object, method)

Offer  --> EmailNotificationObserver::mailNotification

String --> MessageNotificationObserver::messageNotification
```

EventBus中，我们会通过post方法分发事件。在post方法中，首先会根据参数找到我们之前处理好的对应关系,然后通过反射调用对应的方法

```java
public void post(Object event) {
    Iterator<Subscriber> eventSubscribers = subscribers.getSubscribers(event);
    if (eventSubscribers.hasNext()) {
      // 转发事件
      dispatcher.dispatch(event, eventSubscribers);
    } else if (!(event instanceof DeadEvent)) {
      // 对于找不到订阅者的包装成DeadEvent处理,实际上就是丢弃掉
      post(new DeadEvent(this, event));
    }
  }

// PerThreadQueuedDispatcher实现
@Override
void dispatch(Object event, Iterator<Subscriber> subscribers) {
  checkNotNull(event);
  checkNotNull(subscribers);

  // 每个线程都对应一个队列,如果多线程插入则先来的先处理
  Queue<Event> queueForThread = queue.get();
  queueForThread.offer(new Event(event, subscribers));

  if (!dispatching.get()) {
    dispatching.set(true);
    try {
      Event nextEvent;
      // 找到对应的订阅者进行处理
      while ((nextEvent = queueForThread.poll()) != null) {
        while (nextEvent.subscribers.hasNext()) {
          nextEvent.subscribers.next().dispatchEvent(nextEvent.event);
        }
      }
    } finally {
      dispatching.remove();
      queue.remove();
    }
  }
}

```

Dispatcher是一个抽象类，这个类的作用是负责转发event给订阅者，提供不同的event顺序。这里这样的实现主要是考虑到了多线程。

我们的默认实现使用的是PerThreadQueuedDispatcher，看名字的意思就是每个线程一个队列，实行先来先处理的原则。

最终调用Subscriber的invokeSubscriberMethod()方法

```java
final void dispatchEvent(final Object event) {
  executor.execute(
    new Runnable() {
      @Override
      public void run() {
        try {
          invokeSubscriberMethod(event);
        } catch (InvocationTargetException e) {
          bus.handleSubscriberException(e.getCause(), context(event));
        }
      }
    });
}
void invokeSubscriberMethod(Object event) throws InvocationTargetException {
   
   // 省略异常捕获代码
  // 反射调用方法执行 
  method.invoke(target, checkNotNull(event));  
}

```
最终这样就调用了我们使用`@Subscribe`注解标明的方法了。

而这里的executor实际上是创建EventBus的executor,它的execute方法实现如下:

```java
@GwtCompatible
enum DirectExecutor implements Executor {
  INSTANCE;

  @Override
  public void execute(Runnable command) {
    command.run();
  }
}

```

所以说EventBus实际上是同步阻塞执行，那么为什么还要写成线程池的方式呢？虽然EventBus默认是同步执行的，但是它还有一个异步执行的子类AsyncEventBus,异步的EventBus需要指定线程池，所以这里是为了兼容才这么写的。










