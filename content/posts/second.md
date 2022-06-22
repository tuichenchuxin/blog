---
title: "guava EventBus 使用以及注意点"
date: 2022-04-06T10:39:09+08:00
draft: true
---
## EventBus 的使用及注意点
### 使用
- 创建全局实例
``` java
static final EventBus INSTANCE = new EventBus();
```
- 将含有 `@Subscribe` 的方法注册到全局实例上
``` java
INSTANCE.register(this);
```
- 发送 event
``` java
INSTANCE.post(event);
```
### 流程分析
采用上述方式时有以下几个注意点
- 发送事件并且接受后处理事件是同步执行的，即同一个线程执行
- 如果在接受事件方法中有嵌套发送事件，那么该嵌套发送的事件不会立即执行，会等到第一个事件的接收方法完成后，再执行第二个事件。
## EventBus 调用流程分析
- 创建 EventBus 实例
- 将订阅者注册到 EventBus 实例上

### 创建 EventBus 实例
``` java
public EventBus() {
    this("default");
  }

  /**
   * Creates a new EventBus with the given {@code identifier}.
   *
   * @param identifier a brief name for this bus, for logging purposes. Should be a valid Java
   *     identifier.
   */
  public EventBus(String identifier) {
    this(
        identifier,
        // 同步执行器
        MoreExecutors.directExecutor(),
        Dispatcher.perThreadDispatchQueue(),
        LoggingHandler.INSTANCE);
  }
```

``` java
private static final class PerThreadQueuedDispatcher extends Dispatcher {

    // This dispatcher matches the original dispatch behavior of EventBus.

    /** Per-thread queue of events to dispatch. */
    private final ThreadLocal<Queue<Event>> queue =
        new ThreadLocal<Queue<Event>>() {
          @Override
          protected Queue<Event> initialValue() {
            return Queues.newArrayDeque();
          }
    };
```

### 注册 EventBus 订阅者
```java
  /**
   * Registers all subscriber methods on {@code object} to receive events.
   *
   * @param object object whose subscriber methods should be registered.
   */
  public void register(Object object) {
    subscribers.register(object);
  }

  /** Registers all subscriber methods on the given listener object. */
  void register(Object listener) {
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

  /**
   * Returns all subscribers for the given listener grouped by the type of event they subscribe to.
   */
  private Multimap<Class<?>, Subscriber> findAllSubscribers(Object listener) {
    Multimap<Class<?>, Subscriber> methodsInListener = HashMultimap.create();
    Class<?> clazz = listener.getClass();
    for (Method method : getAnnotatedMethods(clazz)) {
      Class<?>[] parameterTypes = method.getParameterTypes();
      Class<?> eventType = parameterTypes[0];
      methodsInListener.put(eventType, Subscriber.create(bus, listener, method));
    }
    return methodsInListener;
  }

    /** Creates a {@code Subscriber} for {@code method} on {@code listener}. */
  static Subscriber create(EventBus bus, Object listener, Method method) {
    return isDeclaredThreadSafe(method)
        ? new Subscriber(bus, listener, method)
        // 调用方法采用 synchronized
        : new SynchronizedSubscriber(bus, listener, method);
  }
```

### 发送 event
``` java 
 /**
   * Posts an event to all registered subscribers. This method will return successfully after the
   * event has been posted to all subscribers, and regardless of any exceptions thrown by
   * subscribers.
   *
   * <p>If no subscribers have been subscribed for {@code event}'s class, and {@code event} is not
   * already a {@link DeadEvent}, it will be wrapped in a DeadEvent and reposted.
   *
   * @param event event to post.
   */
  public void post(Object event) {
    Iterator<Subscriber> eventSubscribers = subscribers.getSubscribers(event);
    if (eventSubscribers.hasNext()) {
      dispatcher.dispatch(event, eventSubscribers);
    } else if (!(event instanceof DeadEvent)) {
      // the event had no subscribers and was not itself a DeadEvent
      post(new DeadEvent(this, event));
    }
  }
  // 这里需要注意，如果当前线程在调用的 post 方法中，嵌套调用 post，第二个 post方法不会立刻执行，而是会加入 队列中，等到队列中第一个任务处理完，才会继续处理。
  @Override
    void dispatch(Object event, Iterator<Subscriber> subscribers) {
      checkNotNull(event);
      checkNotNull(subscribers);
      Queue<Event> queueForThread = queue.get();
      queueForThread.offer(new Event(event, subscribers));

      if (!dispatching.get()) {
        dispatching.set(true);
        try {
          Event nextEvent;
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
