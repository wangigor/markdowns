# EventBus源码解析

[EventBusExplained](https://github.com/google/guava/wiki/EventBusExplained)

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



