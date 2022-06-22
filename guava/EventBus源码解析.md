# EventBus源码解析

[EventBusExplained](https://github.com/google/guava/wiki/EventBusExplained)

## 成员变量

```java
  //such as name
  private final String identifier;
  //方法执行器 默认同步执行
  private final Executor executor;
  //异常处理器
  private final SubscriberExceptionHandler exceptionHandler;
  //消费者注册中心
  private final SubscriberRegistry subscribers = new SubscriberRegistry(this);
  //派发器「详见Dispatcher」
  private final Dispatcher dispatcher;
```

## 构造器

```java
//默认
public EventBus() {
  this("default");
}
//指定名称
public EventBus(String identifier) {
  this(
      identifier,
      MoreExecutors.directExecutor(),
      Dispatcher.perThreadDispatchQueue(),
      LoggingHandler.INSTANCE);
}
//指定异常处理
public EventBus(SubscriberExceptionHandler exceptionHandler) {
  this(
      "default",
      MoreExecutors.directExecutor(),
      Dispatcher.perThreadDispatchQueue(),
      exceptionHandler);
}
//全参数构造器
EventBus(
    String identifier,
    Executor executor,
    Dispatcher dispatcher,
    SubscriberExceptionHandler exceptionHandler) {
  this.identifier = checkNotNull(identifier);
  this.executor = checkNotNull(executor);
  this.dispatcher = checkNotNull(dispatcher);
  this.exceptionHandler = checkNotNull(exceptionHandler);
}
```

- 名称默认为`default`
- 方法执行默认使用`MoreExecutors.directExecutor()`
- 派发默认使用`Dispatcher.perThreadDispatchQueue()`
- 异常处理默认使用单例`LoggingHandler`

## Executor

> guava提供了Executor的多种实现方式，详见「guava的线程池工厂」

事件处理默认采用同步执行。

```java
@GwtCompatible
@ElementTypesAreNonnullByDefault
enum DirectExecutor implements Executor {
  INSTANCE;

  @Override
  public void execute(Runnable command) {
    command.run();
  }

  @Override
  public String toString() {
    return "MoreExecutors.directExecutor()";
  }
}
```

## SubscriberRegistry

消费者的注册中心。按接收消息的`Class`分组。

先来看看测试用例

- 按照类型获取消费者

  ```java
  public void testRegister() {
    assertEquals(0, registry.getSubscribersForTesting(String.class).size());
  
    registry.register(new StringSubscriber());
    assertEquals(1, registry.getSubscribersForTesting(String.class).size());
  
    registry.register(new StringSubscriber());
    assertEquals(2, registry.getSubscribersForTesting(String.class).size());
  
    registry.register(new ObjectSubscriber());
    assertEquals(2, registry.getSubscribersForTesting(String.class).size());
    assertEquals(1, registry.getSubscribersForTesting(Object.class).size());
  }
  ```

- 按照实例获取消费者

  ```java
  public void testGetSubscribers() {
    assertEquals(0, Iterators.size(registry.getSubscribers("")));
  
    registry.register(new StringSubscriber());
    assertEquals(1, Iterators.size(registry.getSubscribers("")));
  
    registry.register(new StringSubscriber());
    assertEquals(2, Iterators.size(registry.getSubscribers("")));
  
    registry.register(new ObjectSubscriber());
    //注意这里。
    //可以获取实例类型和实例超类类型
    assertEquals(3, Iterators.size(registry.getSubscribers("")));
    assertEquals(1, Iterators.size(registry.getSubscribers(new Object())));
    assertEquals(1, Iterators.size(registry.getSubscribers(1)));
  
    registry.register(new IntegerSubscriber());
    assertEquals(3, Iterators.size(registry.getSubscribers("")));
    assertEquals(1, Iterators.size(registry.getSubscribers(new Object())));
    assertEquals(2, Iterators.size(registry.getSubscribers(1)));
  }
  ```

注册中心只有两个成员变量

```java
  //分组
  private final ConcurrentMap<Class<?>, CopyOnWriteArraySet<Subscriber>> subscribers =
      Maps.newConcurrentMap();

  //对应EventBus的弱引用「保证bus被回收之后，registry不会继续占用内存，可以被gc」
  @Weak private final EventBus bus;
```

`Subscriber`是按照被`@Subscribe`标注的个数展开的。

```java
  //bus的引用
  @Weak private EventBus bus;
  //测试用例可见的消费者实例对象
  @VisibleForTesting final Object target;
  //消费方法
  private final Method method;
  //消费方法执行器  
  private final Executor executor;
```

下面是消费者注册方法

```java
void register(Object listener) {
  //解析消息类型和消费方法的一对多关系
  //这个Subscriber是消费方法
  Multimap<Class<?>, Subscriber> listenerMethods = findAllSubscribers(listener);

  
  for (Entry<Class<?>, Collection<Subscriber>> entry : listenerMethods.asMap().entrySet()) {
    Class<?> eventType = entry.getKey();
    Collection<Subscriber> eventMethodsInListener = entry.getValue();

    CopyOnWriteArraySet<Subscriber> eventSubscribers = subscribers.get(eventType);

    if (eventSubscribers == null) {
      CopyOnWriteArraySet<Subscriber> newSet = new CopyOnWriteArraySet<>();
      eventSubscribers =
          MoreObjects.firstNonNull(subscribers.putIfAbsent(eventType, newSet), newSet);
    }

    eventSubscribers.addAll(eventMethodsInListener);
  }
}
```

`findAllSubscribers`是按照消费消息类型分组的解析方法，举个例子

```java
  public static class MultiSubscriber {
    @Subscribe
    public void handle(String s) {}
    @Subscribe
    public void handle1(String s) {}
    @Subscribe
    public void handle2(Integer s) {}
  }
  //会被解析为
  //String
  //      --MultiSubscriber.handle
  //      --MultiSubscriber.handle1
  //Integer
  //      --MultiSubscriber.handle2
```

`MultiSubscriber`这里可能有继承关系，每一层按照class类型做了缓存。

```java
  private static final LoadingCache<Class<?>, ImmutableList<Method>> subscriberMethodsCache =
      CacheBuilder.newBuilder()
          .weakKeys()
          .build(
              new CacheLoader<Class<?>, ImmutableList<Method>>() {
                @Override
                public ImmutableList<Method> load(Class<?> concreteClass) throws Exception {
                  return getAnnotatedMethodsNotCached(concreteClass);
                }
              });

```

## Dispatcher

```java
abstract class Dispatcher {

  static Dispatcher perThreadDispatchQueue() {
    return new PerThreadQueuedDispatcher();
  }

  static Dispatcher legacyAsync() {
    return new LegacyAsyncDispatcher();
  }

  static Dispatcher immediate() {
    return ImmediateDispatcher.INSTANCE;
  }

  abstract void dispatch(Object event, Iterator<Subscriber> subscribers);
}
```

是一个接口类，也是一个工厂类。

支持三种事件派发规则

- `PerThreadQueuedDispatcher`广度优先的广播派发
- `LegacyAsyncDispatcher`有序异步派发
- `ImmediateDispatcher`立即执行的广播派发

### PerThreadQueuedDispatcher

确保所有事件按照发布顺序发出。

由单线程顺序调用消费者的消息处理方法。

```java
  private static final class PerThreadQueuedDispatcher extends Dispatcher {

  //线程本地队列
  //单次事件派发结束后清除「重新生成」
  private final ThreadLocal<Queue<Event>> queue =
      new ThreadLocal<Queue<Event>>() {
        @Override
        protected Queue<Event> initialValue() {
          //非线程安全的双端队列
          return Queues.newArrayDeque();
        }
      };

  //线程分配状态，防止重入事件派发
  private final ThreadLocal<Boolean> dispatching =
      new ThreadLocal<Boolean>() {
        @Override
        protected Boolean initialValue() {
          return false;
        }
      };

  @Override
  void dispatch(Object event, Iterator<Subscriber> subscribers) {
    //参数校验略……
    //创建线程本地队列
    Queue<Event> queueForThread = queue.get();
    //加入队列
    queueForThread.offer(new Event(event, subscribers));

    //防止充入事件派发
    if (!dispatching.get()) {
      dispatching.set(true);
      try {
        Event nextEvent;
        //广度派发
        while ((nextEvent = queueForThread.poll()) != null) {
          while (nextEvent.subscribers.hasNext()) {
            nextEvent.subscribers.next().dispatchEvent(nextEvent.event);
          }
        }
      } finally {
        //在下次事件派发时重建
        dispatching.remove();
        queue.remove();
      }
    }
  }
	
  //事件对象和消费「观察」者包装对象
  //多消费者
  private static final class Event {
    private final Object event;
    private final Iterator<Subscriber> subscribers;

    private Event(Object event, Iterator<Subscriber> subscribers) {
      this.event = event;
      this.subscribers = subscribers;
    }
  }
}
```

### LegacyAsyncDispatcher

注意。这里的加入队列和消费不是两个嵌套的循环。

两个循环先后处理，可能是为了如果有其他线程执行够快，可以抢先执行这个事件的消费。

> emmmmmmmmmm，这是我的一个猜想，因为还没有看到其他部分。
>
> 如果事件A、B同时发布，每个事件有两个消费者，都进入全局队列之后，A事件的第一个消费者处理事件较长，则可以由事件B的dispatch线程执行三个消费者处理方法。

```java
  private static final class LegacyAsyncDispatcher extends Dispatcher {

    //全局支持并发的无界队列
    private final ConcurrentLinkedQueue<EventWithSubscriber> queue =
        Queues.newConcurrentLinkedQueue();

    @Override
    void dispatch(Object event, Iterator<Subscriber> subscribers) {
      //加入队列
      while (subscribers.hasNext()) {
        queue.add(new EventWithSubscriber(event, subscribers.next()));
      }
			//消费
      EventWithSubscriber e;
      while ((e = queue.poll()) != null) {
        e.subscriber.dispatchEvent(e.event);
      }
    }

    //事件对象和消费「观察」者包装对象
    //单个消费者
    private static final class EventWithSubscriber {
      private final Object event;
      private final Subscriber subscriber;

      private EventWithSubscriber(Object event, Subscriber subscriber) {
        this.event = event;
        this.subscriber = subscriber;
      }
    }
  }
```

### ImmediateDispatcher

同步广度派发

```java
  private static final class ImmediateDispatcher extends Dispatcher {
    //单例
    private static final ImmediateDispatcher INSTANCE = new ImmediateDispatcher();

    @Override
    void dispatch(Object event, Iterator<Subscriber> subscribers) {
      //派发
      while (subscribers.hasNext()) {
        subscribers.next().dispatchEvent(event);
      }
    }
  }
```

## 默认异常处理-日志输出

```java
  /** Simple logging handler for subscriber exceptions. */
  static final class LoggingHandler implements SubscriberExceptionHandler {
    static final LoggingHandler INSTANCE = new LoggingHandler();

    @Override
    public void handleException(Throwable exception, SubscriberExceptionContext context) {
      Logger logger = logger(context);
      if (logger.isLoggable(Level.SEVERE)) {
        logger.log(Level.SEVERE, message(context), exception);
      }
    }

    private static Logger logger(SubscriberExceptionContext context) {
      return Logger.getLogger(EventBus.class.getName() + "." + context.getEventBus().identifier());
    }

    private static String message(SubscriberExceptionContext context) {
      Method method = context.getSubscriberMethod();
      return "Exception thrown by subscriber method "
          + method.getName()
          + '('
          + method.getParameterTypes()[0].getName()
          + ')'
          + " on subscriber "
          + context.getSubscriber()
          + " when dispatching event: "
          + context.getEvent();
    }
  }
```

## 消息发送

```java
public void post(Object event) {
  //按照类型匹配所有消费方法「包括超类」
  Iterator<Subscriber> eventSubscribers = subscribers.getSubscribers(event);
  if (eventSubscribers.hasNext()) {
    //同步派发
    dispatcher.dispatch(event, eventSubscribers);
  } else if (!(event instanceof DeadEvent)) {
    //如果没有消费者且不是DeadEvent类型，包装成DeadEvent类型发送。
    post(new DeadEvent(this, event));
  }
}
```

## 异步

前面消息处理是同步的，guava提供异步派发的处理方案`AsyncEventBus`+`LegacyAsyncDispatcher`

```java
public class AsyncEventBus extends EventBus {

  public AsyncEventBus(String identifier, Executor executor) {
    super(identifier, executor, Dispatcher.legacyAsync(), LoggingHandler.INSTANCE);
  }
  public AsyncEventBus(Executor executor, SubscriberExceptionHandler subscriberExceptionHandler) {
    super("default", executor, Dispatcher.legacyAsync(), subscriberExceptionHandler);
  }
  public AsyncEventBus(Executor executor) {
    super("default", executor, Dispatcher.legacyAsync(), LoggingHandler.INSTANCE);
  }
```

手动传入Executor即可。

测试用例如下

```java
  protected void setUp() throws Exception {
    super.setUp();
    executor = new FakeExecutor();
    bus = new AsyncEventBus(executor);
  }

  public void testBasicDistribution() {
    StringCatcher catcher = new StringCatcher();
    bus.register(catcher);

    // We post the event, but our Executor will not deliver it until instructed.
    bus.post(EVENT);

    List<String> events = catcher.getEvents();
    assertTrue("No events should be delivered synchronously.", events.isEmpty());

    // Now we find the task in our Executor and explicitly activate it.
    List<Runnable> tasks = executor.getTasks();
    assertEquals("One event dispatch task should be queued.", 1, tasks.size());

    tasks.get(0).run();

    assertEquals("One event should be delivered.", 1, events.size());
    assertEquals("Correct string should be delivered.", EVENT, events.get(0));
  }
```

