# EventBus源码解析

`EventBus`是一个基于`订阅者-发布者`模式框架，该模式定义了一种一对多的依赖关系，让多个订阅者对象同时监听一个对象，通过这种方式对订阅者和主题发布者进行充分解耦，主要用于`Android`组件间相互通信、线程间互相通信及其他线程与`UI`线程之间互相通信等。代替了传统的`Handler`、`BroadCastReceiver`、`Interface`回调等通信方式，相比之下`EventBus`的优点是代码简洁，使用简单，并将事件发布和订阅充分解耦。

## <span id="using-demo">基本用法</span>

1：添加依赖

```
implementation 'org.greenrobot:eventbus:3.1.1'
```

2：自定义一个数据类

```java

/**
 * @author dmrfcoder
 * @date 2019/4/22
 */

public class MessageEvent {
    public String getEventMessage() {
        return eventMessage;
    }

    private final String eventMessage;

    public MessageEvent(String eventMessage) {
        this.eventMessage = eventMessage;
    }
}

```

3：注册、反注册

```java
 		@Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        EventBus.getDefault().register(this);
    }

    @Override
    public void onStop() {
        super.onStop();
        EventBus.getDefault().unregister(this);
    }
```

4：添加处理事件的方法

```java
 @Subscribe
    public void receiveMessage(MessageEvent messageEvent) {
        receivedMessage.setText(messageEvent.getEventMessage());
    }


```

5：发送事件

```java
 private void sendMessage() {
        EventBus.getDefault().post(new MessageEvent("这是来自FragmentA的消息！"));
    }

```

整个流程如下图所示：

![image-20190423141222855](https://ws2.sinaimg.cn/large/006tNc79gy1g2cjgbga6fj31400u00xt.jpg)

那么接下来我们根据上图展示的事件总线来深入源码理解一下其实现方法。

## EventBus.getDefault()

不管是注册(`register`)、反注册(`unregister`)还是发送消息(`post`)，我们一般都会直接使用`EventBus.getDefault()`方法来获取一个`EventBus`对象，那么这个方法里面做了什么？源码如下：

```java
 		static volatile EventBus defaultInstance;

		/** Convenience singleton for apps using a process-wide EventBus instance. */
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

可以看到，这是一个很经典的单例模式，首先将`defaultInstance`声明为`static`的，并且使用`volatile`关键字修饰`defaultInstance`，就是为了保证线程安全，然后在`getDefault()`中使用了`synchronized`锁实现单例，最后返回一个`EventBus`对象。

值得一提的是`getDefault()`方法中执行了`new EventBus()`去实例化`EventBus`对象，我们看看其构造方法的源码：

```java

 	private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
 
/**
     * Creates a new EventBus instance; each instance is a separate scope in which events are delivered. To use a
     * central bus, consider {@link #getDefault()}.
     */
    public EventBus() {
        this(DEFAULT_BUILDER);
    }

    EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadSupport = builder.getMainThreadSupport();
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```

可以看到，默认的构造方法是`public`的，这说明`EventBus`并不是一个真正的单例类，它允许我们实例化多个`EventBus`的对象，而通过`getDefault()`获得的是`EventBus`类为我们维护的一个`EventBus`对象，这样做的好处是既可以让用户通过获取单例对象简便地实现消息通信，又可以支持用户根据自己的需求定制自己的`EventBus`，一般来说我们使用`EventBus`默认提供的`getDefault()`即可。

另外需要注意的一点是这里使用了经典的建造者（`Builder`）模式，我们来看一下这个`EventBusBuilder`：

![image-20190423142512823](https://ws2.sinaimg.cn/large/006tNc79gy1g2cjsxkdmtj30p216mjxv.jpg)

可以看到，`EventBusBuilder`中主要是对`EventBus`的一些基础的配置信息，其中的关键信息我们会在下面展开讲解。

总结一下，`EventBus`的`getDefault()`方法主要涉及的就是**单例模式**以及**建造者模式**这两种较为常见的设计模式。

## register()

使用`EventBus.getDefault()`获取到`EventBus`对象之后，我们就可以向`EventBus`注册**订阅者**了，常见的注册方法如下：

```java
 EventBus.getDefault().register(this);
```

我们来看一下register()方法的实现：

```java
 /**
     * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
     * are no longer interested in receiving events.
     * <p/>
     * Subscribers have event handling methods that must be annotated by {@link Subscribe}.
     * The {@link Subscribe} annotation also allows configuration like {@link
     * ThreadMode} and priority.
     */
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

可以看到，`register()`方法要求我们传入一个`Object subscriber`即**订阅者**对象，我们一般会将`Activity`或者`Fragment`的`this`传入表示将本类注册为**订阅者**，在`register()`方法中首先会调用`subscriber(订阅者对象）`的`getClass()`方法获取订阅者的`Class对象`，然后会执行11行的`subscriberMethodFinder.findSubscriberMethods(subscriberClass)`，从方法名我们可以看出这个方法的作用是在寻找**订阅者**类中的所有订阅方法，返回一个`SubscriberMethod`的`List`对象。下一节我们来深入探寻一下

`findSubscriberMethods(Class<?> subscriberClass)`方法到的源码。

### SubscriberMethodFinder.findSubscriberMethods(Class<?> subscriberClass)

源码如下：

```java
 List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

这里出现了一个`SubscriberMethod`类，我们先来看看这个类的源码：

#### SubscriberMethod

该类的类图如下：

![image-20190423143609472](https://ws2.sinaimg.cn/large/006tNc79gy1g2ck4bkv3kj30p20e0myl.jpg)

可以看到，这个`SubscriberMethod`类主要描述的是**订阅方法**，即**订阅者**中用来订阅`Event`的方法，就是诸如[基本用法示例](#using-demo)中的`public void receiveMessage(MessageEvent messageEvent) `这类被`@Subscribe`注解修饰的方法。该类的主要成员变量有`方法信息(method)`、`事件类型(eventType)`、`优先级(priority)`、`是否粘滞(stick)`等。

明确了`SubscriberMethod`类的内容之后我们回过头来来看看`findSubscriberMethods(Class<?> subscriberClass)`方法的逻辑：

```java
class SubscriberMethodFinder {
      private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
  
  		....

      List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
              List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
              if (subscriberMethods != null) {
                  return subscriberMethods;
              }

              if (ignoreGeneratedIndex) {
                  subscriberMethods = findUsingReflection(subscriberClass);
              } else {
                  subscriberMethods = findUsingInfo(subscriberClass);
              }
              if (subscriberMethods.isEmpty()) {
                  throw new EventBusException("Subscriber " + subscriberClass
                          + " and its super classes have no public methods with the @Subscribe annotation");
              } else {
                  METHOD_CACHE.put(subscriberClass, subscriberMethods);
                  return subscriberMethods;
              }
          }
  
  		....
  
}
```

可以看到，`SubscriberMethodFinder`类中首先有一个类型为`Map<Class<?>, List<SubscriberMethod>>` 的对象，其`key`是`类对象（Class）`，`value`是是`SubscriberMethod`对应的`List`集合。在`findSubscriberMethods(Class<?> subscriberClass)`方法中会首先从`METHOD_CACHE` 这个`Map`中根据**订阅类**的**类对象**查询看是否之前已经存在该**订阅者**的**订阅方法集合**了，如果已经存在（应该是之前缓存的），直接从`METHOD_CACHE` 这个`Map`中拿到对应的`List`并返回，如果之前没有缓存过即`METHOD_CACHE` 这个`Map`中没有以**当前**`subscriberClass`为`key`的键值对，则需要从`subscriberClass`类中去找订阅方法，关键代码即上面的12到16行代码所示：

```java
   if (ignoreGeneratedIndex) {
     subscriberMethods = findUsingReflection(subscriberClass);
   } else {
     subscriberMethods = findUsingInfo(subscriberClass);
   }
```

可以看到，这里根据一个`ignoreGeneratedIndex`标志位值的不同执行了不同的逻辑，`ignoreGeneratedIndex`是`subscriberMethodFinder`对象的成员变量，我们寻根溯源找一下它的值：

`EventBus`的构造方法：

```java
 EventBus(EventBusBuilder builder) {
        ...
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        ...
    }

```

`SubscriberMethodFinder`的构造方法：

```java
SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                           boolean ignoreGeneratedIndex) {
        this.subscriberInfoIndexes = subscriberInfoIndexes;
        this.strictMethodVerification = strictMethodVerification;
        this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    }
```

可以看到`subscriberMethodFinder`中的`ignoreGeneratedIndex`应该和`builder.ignoreGeneratedIndex`中的值是一样的，那么查看一下`EventBusBuilder`中`ignoreGeneratedIndex`的声明：

```java
public class EventBusBuilder {
	...
    boolean ignoreGeneratedIndex;
  ...
  }
    
```

可以看到，`EventBusBuilder`中只是简单对其做了声明，并未对其赋值，所以`ignoreGeneratedIndex`的值应该是默认的`false`，所以：

```java
 if (ignoreGeneratedIndex) {
     subscriberMethods = findUsingReflection(subscriberClass);
   } else {
     subscriberMethods = findUsingInfo(subscriberClass);
   }
```

默认会执行：

```
 subscriberMethods = findUsingInfo(subscriberClass);
```

再往下走，我们看看findUsingInfo(subscriberClass)方法：

#### SubscriberMethodFinder.findUsingInfo(Class<?> subscriberClass) 

源码如下：

```java
 private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

```

可以看到，首先调用`prepareFindState()`获取了一个`FindState`对象，看一下`prepareFindState()`:

##### SubscriberMethodFinder.prepareFindState()

```java
 class SubscriberMethodFinder{
   ...
     	private static final int POOL_SIZE = 4;
      private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

      private FindState prepareFindState() {
              synchronized (FIND_STATE_POOL) {
                  for (int i = 0; i < POOL_SIZE; i++) {
                      FindState state = FIND_STATE_POOL[i];
                      if (state != null) {
                          FIND_STATE_POOL[i] = null;
                          return state;
                      }
                  }
              }
              return new FindState();
          }
   
 }
```

`prepareFindState()`方法中首先从一个已有的`FindState[] FIND_STATE_POOL`**对象池**中找，看对象池中有没有哪一个位置是**非空(null)**的，如果有，就将该位置的值置为空（null），然后将该位置原来的对象返回，如果最后发现对象池中没有一个位置的对象值为null即对象池中不存在可用的对象，再`new`一个新的`FindState`对象并返回，这是一种经典的使用对象池达到对象复用以减少内存开销的方法，但是这和我们之前见过的对象池复用机制好像不太一样，为什么返回了原有的对象之后要将对象池原来的位置的对象值置为空？一会我们来解释，这是作者的一个巧妙设计。

我们看一下这个`FindState`类到底是个什么东西：

##### FindState

`FindState`类的类图如下：

![image-20190423145543264](https://ws4.sinaimg.cn/large/006tNc79gy1g2ckoodzdzj30n60hetaz.jpg)

首先，`FindState`是`SubscriberMethodFinder`的一个内部类，上图贴出了其成员变量及主要的方法。

那么我们回过头来继续分析这段代码：

```java
 private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }


```

获取了`findState`对象之后，执行了 `findState.initForSubscriber(subscriberClass)`，我们看看这个方法是在干什么：

##### FindStat.initForSubscriber(subscriberClass)

```java
FindState{
      Class<?> subscriberClass;
      Class<?> clazz;
      boolean skipSuperClasses;
      SubscriberInfo subscriberInfo;

      void initForSubscriber(Class<?> subscriberClass) {
          this.subscriberClass = clazz = subscriberClass;
          skipSuperClasses = false;
          subscriberInfo = null;
      }
}
```

可以看到这里主要是根据当前**订阅者对象**对刚才获取到的`findState`做了一些配置。

接下来，`findUsingInfo(Class<?> subscriberClass)` 方法会执行：

```java
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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
   }
```

这是一个循环，循环终止的条件是`findState.clazz != null`，通过`FindState.initForSubscriber(Class<?> subscriberClass)`方法我们可以看到`findState.clazz`对象当前应该是等于我们的**订阅者对象**`subscriberClass`，所以第一次会默认进入循环，然后循环中：

首先调用`getSubscriberInfo(findState)`：

##### SubscriberMethodFinder.getSubscriberInfo(FindState findState)

其源码如下

```java
private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }

```

首先会判断`findState.subscriberInfo` 是否为`null`，刚才从 `findState.initForSubscriber(subscriberClass)`方法中我们看到：

```java
  subscriberInfo = null;
```

所以第一个`if`条件为`false`，跳过，然后判断`subscriberInfoIndexes`是否为`null`，这个`subscriberInfoIndexes`声明如下：

```java
 private List<SubscriberInfoIndex> subscriberInfoIndexes;

//SubscriberMethodFinder的构造方法
SubscriberMethodFinder(List<SubscriberInfoIndex> subscriberInfoIndexes, boolean strictMethodVerification,
                           boolean ignoreGeneratedIndex) {
        this.subscriberInfoIndexes = subscriberInfoIndexes;
        this.strictMethodVerification = strictMethodVerification;
        this.ignoreGeneratedIndex = ignoreGeneratedIndex;
    }
```

那么我们现在就需要知道构造`SubscriberMethodFinder`的时候传入的`List<SubscriberInfoIndex> subscriberInfoIndexes`是什么。我们来看看这个`subscriberMethodFinder`是如何构造的：

```java
public class EventBusBuilder {
	...
   List<SubscriberInfoIndex> subscriberInfoIndexes;
   boolean strictMethodVerification;
   boolean ignoreGeneratedIndex;
   ...
}

EventBus(EventBusBuilder builder){
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
}
```

到现在就清楚了，这个`subscriberInfoIndexes`当前为`null`，所以 `SubscriberMethodFinder.getSubscriberInfo(FindState findState)`方法最终也会返回`null`，所以我们继续看这个`FindStat.initForSubscriber(subscriberClass)`中的这个循环：

```java
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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
   }
```

根据以上分析`，findState.subscriberInfo` 最终为`null`，所以再往下走会执行 `findUsingReflectionInSingleClass(findState)`，我们看看其源码：

##### SubscriberMethodFinder.findUsingReflectionInSingleClass(findState)

```java
 private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

我们还是一句一句分析，首先调用` methods = findState.clazz.getDeclaredMethods()`，`findState.clazz`是什么？我们来看看源码：

```java
FindState{
      Class<?> subscriberClass;
      Class<?> clazz;
      boolean skipSuperClasses;
      SubscriberInfo subscriberInfo;

      void initForSubscriber(Class<?> subscriberClass) {
          this.subscriberClass = clazz = subscriberClass;
          skipSuperClasses = false;
          subscriberInfo = null;
      }
}
```

是的，`findState.clazz`就是刚才我们的**订阅者对象**，`findUsingReflectionInSingleClass（）`的第5行调用了它的`getDeclaredMethods()`方法，这个方法是`java`反射机制中支持的方法，主要作用是获取**当前类的所有方法**，**不包括父类和接口的**，至于这个方法怎么实现的就不展开赘述了，感兴趣的读者可以自行研究`java`的**反射机制**。

获取到订阅者类的所有方法之后，下面使用了一个`for..each`循环对订阅者对象对应的**订阅者**类中的所有方法进行了遍历：

```java
for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
```

首先执行`int modifiers = method.getModifiers()`，这个`getModifiers()`方法是`Method`类提供的，作用是返回该方法的`Java`语言修饰符。

接下来执行了:

```java
 if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0)
```

对方法的**修饰符**进行了判断，我们拆开看：

```
(modifiers & Modifier.PUBLIC) != 0 
```

这句意思是判断`modifiers`是否为`Modifier.PUBLIC`，如果`modifiers`是`Modifier.PUBLIC`的，则该条件为真。

```java
(modifiers & MODIFIERS_IGNORE) == 0
```

这里的`MODIFIERS_IGNORE`定义如下：

```java
private static final int BRIDGE = 0x40;
private static final int SYNTHETIC = 0x1000;
private static final int MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
```

所以易知这一句是为了判断该方法是否是**抽象方法**、**静态方法**等，如果不是，则这个条件为真，否则为假。

到这里我们就明白了，` if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0)`这句是在根据方法的修饰符筛选方法，筛选出所有由`public`修饰的且**非抽象**、**非静态**的方法，这也说明了为什么我们的订阅方法必须声明为`public`的。筛选出满足`java`修饰符要求的方法后会执行：

```java
Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
```

这里首先通过`Class<?>[] parameterTypes = method.getParameterTypes();`获取到当前方法的**参数类型**，然后判断该方法的参数的数量是否为1，如果不是1就抛出异常，这就是为什么订阅方法必须**只能指定一个参数**的原因。确定了当前方法的参数长度为1之后，执行了`method.getAnnotation(Subscribe.class)`，这个`method.getAnnotation()`方法的原型如下：

```java
public <T extends Annotation> T getAnnotation(Class<T> annotationClass)
```

作用是判断当前方法是否存在`annotationClass`对应的注解，方法如果存在这样的注解，则返回指定类型的元素的注解内容，否则返回`null`。这里的`Subscribe`自然对应的就是我们配置订阅方法时使用的`Subscribe`注解了，其源码如下：

```java

@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING;

    /**
     * If true, delivers the most recent sticky event (posted with
     * {@link EventBus#postSticky(Object)}) to this subscriber (if event available).
     */
    boolean sticky() default false;

    /** Subscriber priority to influence the order of event delivery.
     * Within the same delivery thread ({@link ThreadMode}), higher priority subscribers will receive events before
     * others with a lower priority. The default priority is 0. Note: the priority does *NOT* affect the order of
     * delivery among subscribers with different {@link ThreadMode}s! */
    int priority() default 0;
}


```

然后执行：

```java
  if (subscribeAnnotation != null) {
    Class<?> eventType = parameterTypes[0];
    if (findState.checkAdd(method, eventType)) {
      ThreadMode threadMode = subscribeAnnotation.threadMode();
      findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                                           subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
    }
 }
```

首先判断当前方法含有`Subscribe`注解，然后使用`Class<?> eventType = parameterTypes[0]`获取到当前方法的第一个(唯一一个)参数的类型，然后执行了`findState.checkAdd(method, eventType)`，这个`findState.checkAdd()`方法是干什么的呢？

###### FindState.checkAdd(Method method, Class<?> eventType)

看源码：

```java
  boolean checkAdd(Method method, Class<?> eventType) {
    // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
    // Usually a subscriber doesn't have methods listening to the same event type.
    Object existing = anyMethodByEventType.put(eventType, method);
    if (existing == null) {
      return true;
    } else {
      if (existing instanceof Method) {
        if (!checkAddWithMethodSignature((Method) existing, eventType)) {
          // Paranoia check
          throw new IllegalStateException();
        }
        // Put any non-Method object to "consume" the existing Method
        anyMethodByEventType.put(eventType, this);
      }
      return checkAddWithMethodSignature(method, eventType);
    }
  }

```

首先执行`Object existing = anyMethodByEventType.put(eventType, method)`，这个`anyMethodByEventType`的声明如下：

```java
 static class FindState {
 				 ...
       final Map<Class, Object> anyMethodByEventType = new HashMap<>();
  			...
        
  }
```

可以看到，这个`anyMethodByEventType`是一个`Map`，存储着`Class-Object`的映射关系，这里调用`Object existing = anyMethodByEventType.put(eventType, method)`，试图将`<eventType(我们订阅方法中唯一一个参数的类型即事件类型),method(我们的订阅方法)>` 添加到这个`Map`中，那么这个`put`的返回值应该是什么呢？根据`Map.put`的源码注解我们可以了解到，当`anyMethodByEventType`中之前存储有`eventType`为`key`的键值对时这里返回键`eventType`之前对应的值，如果之前`anyMethodByEventType`中之前不存在以`eventType`为`key`的键值对，则这里返回`null`。

执行完  `Object existing = anyMethodByEventType.put(eventType, method)`之后会执行：

```java
  if (existing == null) {
    return true;
  } else {
    if (existing instanceof Method) {
      if (!checkAddWithMethodSignature((Method) existing, eventType)) {
        // Paranoia check
        throw new IllegalStateException();
      }
      // Put any non-Method object to "consume" the existing Method
      anyMethodByEventType.put(eventType, this);
    }
    return checkAddWithMethodSignature(method, eventType);
  }
```

这里首先判断`existing`是否为`null`，本质就是判断之前 `anyMethodByEventType`中是否存在以`eventType`为键的键值对，如果不存在，这里`existing`为`null`，则直接返回`true`，由于初始时`anyMethodByEventType`为空，所以`existing`为`null`，所以这里会直接返回`true`。

如果之前存在以`eventType`为键的键值对，则这里的`existing`为之前以`eventType`为键的值，使用`existing instanceof Method`判断了一下之前键`eventType`对应的值是否是一个`Method`对象，如果是，检查了`checkAddWithMethodSignature((Method) existing, eventType)`的返回结果，那么这个

`checkAddWithMethodSignature()`是干什么的呢？

- **FindState.checkAddWithMethodSignature(Method method, Class<?> eventType)**

看源码：

```java
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
            methodKeyBuilder.setLength(0);
            methodKeyBuilder.append(method.getName());
            methodKeyBuilder.append('>').append(eventType.getName());

            String methodKey = methodKeyBuilder.toString();
            Class<?> methodClass = method.getDeclaringClass();
            Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
            if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) 			{
                // Only add if not already found in a sub class
                return true;
            } else {
                // Revert the put, old class is further down the class hierarchy
                subscriberClassByMethodKey.put(methodKey, methodClassOld);
                return false;
            }
        }
```

这个方法中使用了一个`methodKeyBuilder`对象，我们来看看这个`methodKeyBuilder`对象：

```java
 final StringBuilder methodKeyBuilder = new StringBuilder(128);
```

就是一个普通的`StringBuilder`对象，这里首先将其当前长度设为0，然后将当前方法的`method.getName()`追加进去，然后追加了`>`以及`eventType.getName()`，所以最终这个

`methodKeyBuilder`的值为：`method.getName()>eventType.getName()`。

然后将其转化为了`String`存储在`methodKey`中，然后执行 `Class<?> methodClass = method.getDeclaringClass()`，这个`getDeclaringClass()`方法的原型如下：

```java
public Class<T> getDeclaringClass()
```

这个方法的作用是返回声明此`Method`类的`Class`对象。

所以，`method.getDeclaringClass()`的返回值就是我们的**订阅类**的`Class`对象，然后执行`Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass)`，再看看这个`subscriberClassByMethodKey`：

```java
static class FindState {
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        }
```

也是一个`Map`，这里执行`Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass)`之后，如果之前`subscriberClassByMethodKey`中存在以`methodKey`为键的键值对，则`methodClassOld`为`methodKey`键之前对应的值(类)，否则`methodClassOld`的值为`null`。

然后执行`if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass))`，`if`里面两个条件，第一个`methodClassOld == null`就不多说了，已经解释过它何时为真。说一下`methodClassOld.isAssignableFrom(methodClass)`，<span id="isAssignableFrom">`isAssignableFrom（）`</span>方法是用来判断两个类的之间的关联关系，也可以说是一个类是否可以被强制转换为另外一个实例对象，具体解释：

有两个`Class`类型的对象，一个是调用`isAssignableFrom`方法的类对象（`对象A`），另一个是作为方法中参数的这个类对象（称之为`对象B`），这两个对象如果满足以下条件则返回`true`，否则返回`false`：

- A对象所对应类信息是B对象所对应的类信息的父类或者是父接口，简单理解即A是B的父类或接口

- A对象所对应类信息与B对象所对应的类信息相同，简单理解即A和B为同一个类或同一个接口

所以，当`methodClassOld`和`methodClass`对应的是同一个类或者`methodClassOld`是`methodClass`的父类或接口时，`methodClassOld.isAssignableFrom(methodClass)`为`true`，否则为`false`。

```java
 if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) 			{
                // Only add if not already found in a sub class
                return true;
            } else {
                // Revert the put, old class is further down the class hierarchy
                subscriberClassByMethodKey.put(methodKey, methodClassOld);
                return false;
            }
        }
```

综上，当`subscriberClassByMethodKey`中不存在以`methodKey`为键的键值对或`subscriberClassByMethodKey`中存在以`methodKey`为键的键值对，但是之前`methodKey`键对应的值（类）是当前类的父类，则直接返回`true`，相当于将当前类对象（子类）加入`subscriberClassByMethodKey`集合中，否则执行 `subscriberClassByMethodKey.put(methodKey, methodClassOld)`即将当前`methodKey`和其之前对应的值`put`进`subscriberClassByMethodKey`，也就是不将当前的类对象加入`subscriberClassByMethodKey`集合中。

那么对应到：

```java
if (!checkAddWithMethodSignature((Method) existing, eventType)) {
        // Paranoia check
        throw new IllegalStateException();
      }
```

中，如果`subscriberClassByMethodKey`中不存在以由 `existing`, `eventType`转化的`methodKey`为键的键值对或`subscriberClassByMethodKey`中存在以`methodKey`为键的键值对，但是之前`methodKey`键对应值（类）的是当前类的父类，则这里`if`中的条件为假，否则说明`subscriberClassByMethodKey`中存在由 `existing`, `eventType`转化的`methodKey`为键的键值对，而且其之前对应的订阅类不是当前订阅类的父类，则抛出异常。

如果`if`中的条件为真，则执行：

```java
anyMethodByEventType.put(eventType, this);
```

将`<eventType,findstate>`放入`anyMethodByEventType`中。然后返回`checkAddWithMethodSignature(method, eventType)`。`checkAddWithMethodSignature()`的源码在上面已经分析过。

好现在继续回到：

```java
      if (findState.checkAdd(method, eventType)) {
        ThreadMode threadMode = subscribeAnnotation.threadMode();
        findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                                             subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
      }
```

我们上面提到这里第一次`findState.checkAdd(method, eventType)`会直接返回`true`，即进入到`if`语句块中，`if`语句块中首先执行 `ThreadMode threadMode = subscribeAnnotation.threadMode()`获取`Subscribe`注解中的`ThreadMode`属性值，通过查看`Subscribe`的源码发现`ThreadMode`默认是`ThreadMode.POSTING`的：

```java
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING;
    }
```

然后执行：

```java
findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
```

`findState.subscriberMethods`的声明如下：

```java
  static class FindState {
  		final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
  }
```

其是一个`List`，这里给`subscriberMethods`这个`List`中`add`了一个新的对象：

```java
new SubscriberMethod(method, eventType, threadMode,subscribeAnnotation.priority(), subscribeAnnotation.sticky())
```

`SubscriberMethod`构造方法的原型如下：

```java
 public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }
```

对应到实例化的代码，这里使用`当前订阅方法`、`当前订阅方法的eventTye`、`当前订阅方法对应Subscribe注解的优先级`、`当前订阅方法对应Subscrible注解的sticky属性`实例化了一个`SubscriberMethod`，然后将其添加到`findState`的`subscriberMethods`中去。

然后继续看`findUsingInfo()`方法：

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

刚才我们分析完了 `findUsingReflectionInSingleClass(findState)`，其作用是将当前订阅方法添加到`findState`对应的`List`中去，然后执行 `findState.moveToSuperclass()`：

##### FindState.moveToSuperclass()

我们看这个方法的源码：

```java
void moveToSuperclass() {
            if (skipSuperClasses) {
                clazz = null;
            } else {
                clazz = clazz.getSuperclass();
                String clazzName = clazz.getName();
                /** Skip system classes, this just degrades performance. */
                if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
                    clazz = null;
                }
            }
        }
```

首先判断了`skipSuperClasses`的值，从FindState的构造方法中：

```java
void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
```

我们可以看到该值默认为`false`，那么就会执行：

```java
clazz = clazz.getSuperclass();
String clazzName = clazz.getName();
/** Skip system classes, this just degrades performance. */
if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
  clazz = null;
}
```

首先明确`clazz`原本的值是什么：

```java
 void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }

```

可以看到`clazz`的值为`initForSubscriber()`中传入的参数，回想一下我们是在哪里调用`initForSubscriber()`的：

```java

Class<?> subscriberClass = subscriber.getClass();
List<SubscriberMethod> subscriberMethods =subscriberMethodFinder.findSubscriberMethods(subscriberClass);
subscriberMethods = findUsingInfo(subscriberClass);
findState.initForSubscriber(subscriberClass);
```

所以，这个`clazz`就是`subscriberClass`，也就是我们的**订阅类**。

继续看这段代码：

```java
    clazz = clazz.getSuperclass();
    String clazzName = clazz.getName();
    /** Skip system classes, this just degrades performance. */
    if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android.")) {
      clazz = null;
    }
```

首先使用`clazz = clazz.getSuperclass()`获取到当前订阅者类的父类，然后使用`String clazzName = clazz.getName()`获取当前订阅者类的父类的类名，然后判断：

```java
 if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") || clazzName.startsWith("android."))
```

就是判断了一下当前订阅者类的父类是否是以`java`、`javax`、`android`开头的，也就是判断当前类的父类是否是`android`本身的类，如果是，则将`clazz`置为`null`。

所以这个`void moveToSuperclass() `方法的作用就是将`findState`中的`clazz`置为当前订阅者类的父类，当然，如果当前订阅者类没有父类或者当前订阅者类的父类是系统的类，则将`clazz`置为空（`null`）。

再继续看`findUsingInfo（）`：

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

我们分析完了 `findState.moveToSuperclass()`，注意，执行完了 `findState.moveToSuperclass()`之后相当于第一次`while`循环执行完了，现在会去进行下一个`while`循环的判断，那么`while`进入循环的条件是什么？是`findState.clazz != null`，所以，如果之前订阅者类没有父类（或者有父类但是父类是`android`自身的类），则这个循环就会跳出，否则如果之前订阅者类还有父类，就会进入下一个循环，其实是一个递归的过程，从初始订阅者类开始，一级一级向上遍历父类，直到最顶级。

然后，`while`循环完了之后，会返回`getMethodsAndRelease(findState)`。

##### SubscriberMethodFinder.getMethodsAndRelease(FindState findState);

我们看看这个方法的源码：

```java
 private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```

这个方法首先拿到了`findState.subscriberMethods`，也就是**订阅者**中的所有订阅方法的`List`，注意这里是`new`了一个`List`并将`findState.subscriberMethods`的值拷贝了过去，而不是简单地使用`=`，这样做的目的是为了下一步对`findState.subscriberMethods`做修改时不会改变这里的`subscriberMethods`，也就是`findState.recycle()`：

```java
void recycle() {
  subscriberMethods.clear();
  anyMethodByEventType.clear();
  subscriberClassByMethodKey.clear();
  methodKeyBuilder.setLength(0);
  subscriberClass = null;
  clazz = null;
  skipSuperClasses = false;
  subscriberInfo = null;
}
```

可以看到，这个`recycle()`方法其实是将当前`findState`恢复成初始状态，这里使用了 `subscriberMethods.clear()`清空了`subscriberMethods`，如果之前简单使用`=`，则这里执行`clear`之后`subscriberMethods`也会被`clear`掉，这涉及到`java`的引用机制，这里不展开赘述。

执行`findState.recycle()`将`findState`恢复初态之后，执行了：

```java
synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
```

这个逻辑是否有点眼熟？对的，它和前面介绍的：

```java
class SubscriberMethodFinder{
   ...
     	private static final int POOL_SIZE = 4;
      private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

      private FindState prepareFindState() {
              synchronized (FIND_STATE_POOL) {
                  for (int i = 0; i < POOL_SIZE; i++) {
                      FindState state = FIND_STATE_POOL[i];
                      if (state != null) {
                          FIND_STATE_POOL[i] = null;
                          return state;
                      }
                  }
              }
              return new FindState();
          }
   
 }
```

中的：

```java
 synchronized (FIND_STATE_POOL) {
                  for (int i = 0; i < POOL_SIZE; i++) {
                      FindState state = FIND_STATE_POOL[i];
                      if (state != null) {
                          FIND_STATE_POOL[i] = null;
                          return state;
                      }
                  }
              }
```

逻辑是类似的，只不过`prepareFindState()`中是从对象池中拿出了一个`findState`并将其对应的位置置为`null`，相当于**借出**，`getMethodsAndRelease()`是查看对象池中的空缺位置然后将现在不用的`findState`放进去，相当于**归还**。借出时将其坑位置为`null`，归还时再将坑位补上，完美地利用了内存，一点也没浪费，不得不佩服作者的神勇。

归还完`findState`之后 `private List<SubscriberMethod> getMethodsAndRelease(FindState findState)`会执行最后一行：

```
 return subscriberMethods;
```

即返回当前订阅者类的所有订阅方法。

至此，我们分析完了：

```java
   private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

```

方法，记得这个方法返回的是当前订阅者类及其父类的所有订阅方法。

我们继续回退一级：

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

这分析完了`subscriberMethods = findUsingInfo(subscriberClass)`这一行，然后会判断`subscriberMethods`即当前**订阅者类**及其父类的**订阅方法**集合是否为空，如果为空，抛出异常，因为订阅者类中都没有订阅方法，其存在没有任何意义。如果订阅方法集合不为空，则将`<当前订阅者类，当前订阅者类及其父类的所有订阅方法的集合>` `put`到`METHOD_CACHE`这个`Map`缓存中，最后返回`subscriberMethods`。

这时候再回退一级：

```java
public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

`     List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);`获取了当前订阅者类及其父类的所有订阅方法，然后下面使用：

```java
synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
```

即对每一个订阅方法执行了：

```java
 subscribe(subscriber, subscriberMethod);
```

### EventBus.subscribe(Object subscriber, SubscriberMethod subscriberMethod)

我们看看这个方法的源码：

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
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

这个方法的源码有点长，我们慢慢分析。

首先，`Class<?> eventType = subscriberMethod.eventType`获取了当前订阅方法对应的事件类型，然后执行`Subscription newSubscription = new Subscription(subscriber, subscriberMethod)`实例化了一个`Subscription`类的对象.

#### Subscription

我们看看这个类的类图：

![image-20190423234532550](https://ws2.sinaimg.cn/large/006tNc79gy1g2czzz7qn0j30gm08wjs6.jpg)

其中关注一下`active`这个属性：

```java
 /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;

```

看注释的意思是如果订阅者执行了**反注册**，则这个值会置为`false`。

构造方法：

```java
 Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        active = true;
    }

```

所以，这里是用订阅者的类对象和一个订阅方法实例化了一个`Subscription`类。

然后执行`CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType)` ,  `CopyOnWriteArrayList`是`Java`并发包中提供的一个并发容器，它是个线程安全且读操作无锁的`ArrayList`，具体实现原理不在这里展开赘述。那么这个<span id="subscriptionsByEventType">`subscriptionsByEventType`是什么？

```java
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

 EventBus(EventBusBuilder builder) {
   ...
        subscriptionsByEventType = new HashMap<>();
        ...
  }
```

可以看到，这个`subscriptionsByEventType`实际上是一个`Map`，初始里面什么内容都没有，所以第一次执行`CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);`一定会得的`subscriptions`为`null`，所以下面：

```java
  if (subscriptions == null) {
    subscriptions = new CopyOnWriteArrayList<>();
    subscriptionsByEventType.put(eventType, subscriptions);
  } else {
    if (subscriptions.contains(newSubscription)) {
      throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                                  + eventType);
    }
  }
```

会进入到`subscriptions == null`对应的代码块，即：

```java
 subscriptions = new CopyOnWriteArrayList<>();
 subscriptionsByEventType.put(eventType, subscriptions);
```

就是先`new`了一个`CopyOnWriteArrayList`，然后将`<eventType, subscriptions>`   `put`到`subscriptionsByEventType`中去，所以`subscriptionsByEventType`中存储的就是`<事件类型,事件类型对应的订阅方法/订阅类的集合>`这样的键值对。如果之后执行`CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType)`取到的值不为`null`即说明之前已经有`<eventType, subscriptions>`被`put`到`subscriptionsByEventType`里面过，就会执行：

```java
if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
```

主要是判断之前`eventType`对应的订阅方法中是否已经存在当前的这个订阅方法，如果存在，就说明这个方法已经被注册过了，一般是由于订阅者类多次`register()`导致的，就抛出异常。

继续往下走，执行：

```java
 int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
```

这里遍历了`subscriptions`，即遍历了与当前订阅方法对应的事件类型相同的所有订阅方法，

然后将当前订阅方法按照优先级（`priority`）的顺序插入到该事件类型对应的`List`集合中去。

然后执行：

```java
List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
if (subscribedEvents == null) {
  subscribedEvents = new ArrayList<>();
  typesBySubscriber.put(subscriber, subscribedEvents);
}
subscribedEvents.add(eventType);
```

这里首先执行`List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber)`，看看`typesBySubscriber`是什么：

```java
private final Map<Object, List<Class<?>>> typesBySubscriber;
EventBus(EventBusBuilder builder) {
	...
  typesBySubscriber = new HashMap<>();
  ...
}
```

可以看到这个`typesBySubscriber`也是一个`Map`，那么也可得出初始时`subscribedEvents`为`null`的结论，那么就会执行：

```java
 subscribedEvents = new ArrayList<>();
 typesBySubscriber.put(subscriber, subscribedEvents);
```

这里就是将`<subscriber, subscribedEvents>`添加到了`typesBySubscriber`这个`map`中，所以可以知道`typesBySubscriber`这个`map`存储的就是订阅者类和订阅者类对应的所有订阅方法的事件类型集合的映射对。

然后执行`subscribedEvents.add(eventType)`将当前订阅方法对应的事件类型添加进去。

接下来执行：

```java
if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
```

判断了一下当前订阅方法是否是`sticky(粘滞)`的，如果是，执行：

```java
if (eventInheritance) {
  // Existing sticky events of all subclasses of eventType have to be considered.
  // Note: Iterating over all events may be inefficient with lots of sticky events,
  // thus data structure should be changed to allow a more efficient lookup
  // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
  Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
  for (Map.Entry<Class<?>, Object> entry : entries) {
    Class<?> candidateEventType = entry.getKey();
    if (eventType.isAssignableFrom(candidateEventType)) {
      Object stickyEvent = entry.getValue();
      checkPostStickyEventToSubscription(newSubscription, stickyEvent);
    }
  }
} else {
  Object stickyEvent = stickyEvents.get(eventType);
  checkPostStickyEventToSubscription(newSubscription, stickyEvent);
 }
```

<span id="eventInheritance">首先判断`eventInheritance`的真值，这个对象是什么？它代表什么意思？</span>

```java
EventBus:
private final boolean eventInheritance;

EventBus(EventBusBuilder builder) {
		eventInheritance = builder.eventInheritance;
}
EventBusBuilder:
boolean eventInheritance = true;

```

所以这个`eventInheritance`的值默认是`true`的，那么就会执行：

```java
 // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
for (Map.Entry<Class<?>, Object> entry : entries) {
  Class<?> candidateEventType = entry.getKey();
  if (eventType.isAssignableFrom(candidateEventType)) {
    Object stickyEvent = entry.getValue();
    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
  }
}
```

这个<span id="stickyEvents">`stickyEvents`</span>的声明如下：

```java
private final Map<Class<?>, Object> stickyEvents;
EventBus(EventBusBuilder builder) {
				...
        stickyEvents = new ConcurrentHashMap<>();
        ...
}
```

所以它就是一个线程安全的`HashMap`，`stickyEvents.entrySet()`返回的是`stickyEvents`中所有的`entry`对，比如：

```java
hash_map = new ConcurrentHashMap<Integer, String>(); 
  
        // Mapping string values to int keys 
        hash_map.put(10, "Geeks"); 
        hash_map.put(15, "4"); 
        hash_map.put(20, "Geeks"); 
        hash_map.put(25, "Welcomes"); 
        hash_map.put(30, "You"); 
        System.out.println("The set is: " + hash_map.entrySet()); 
```

输出为：

```java
The set is: [20=Geeks, 25=Welcomes, 10=Geeks, 30=You, 15=4]
```

初始时`stickyEvents = new ConcurrentHashMap<>()`执行完后`stickyEvents`一定为空，不会执行`for`语句里面的内容。但是我们还是简要分析一下for语句里面的代码逻辑：

```java
for (Map.Entry<Class<?>, Object> entry : entries) {
    Class<?> candidateEventType = entry.getKey();
    if (eventType.isAssignableFrom(candidateEventType)) {
      Object stickyEvent = entry.getValue();
      checkPostStickyEventToSubscription(newSubscription, stickyEvent);
    }
}
```

这里遍历了`stickyEvents`中的所有`entry`，对每一个`entry`，首先通过    `Class<?> candidateEventType = entry.getKey()`获取到该`entry`对的键(`key`)，然后判断`eventType.isAssignableFrom(candidateEventType)`的真值，还记得这个`isAssignableFrom()`的功能吗？忘记的话看[这里](#isAssignableFrom)，这个方法的作用就是判断`eventType`这个类对象对应的类是否是`candidateEventType`即当前`entry`对的键对应的类对象的类的父类，如果可以，执行：

```java
 Object stickyEvent = entry.getValue();
 checkPostStickyEventToSubscription(newSubscription, stickyEvent);
```

重点是这个<span id="checkPostStickyEventToSubscription">`checkPostStickyEventToSubscription（）`</span>方法：

```java
 private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }

```

这里对`stickyEvent`做了一下判空，然后调用 `postToSubscription(newSubscription, stickyEvent, isMainThread())`，该方法我会在讲解`post()`的时候详细分析

### register()总结

至此，我们暂时分析完了`register()`的过程，总结一下其主要做了哪些工作：

- 通过反射获取订阅者中定义的处理事件的方法集合(订阅方法集合)
- 遍历订阅方法集合，调用`subscribe`方法，在这个`subscribe`方法内：
  - 初始化并完善`subscriptionsByEventType`这个`map`对象
  - 初始化并完善`typesBySubscriber`这个`map`对象
  - 触发`stick`事件的处理

看一下register()方法中逻辑的思维导图：

![register分支](https://ws4.sinaimg.cn/large/006tNc79gy1g2esrbr1j7j31vr0u04qp.jpg)

如果上图看不清，可[下载原图查看](https://github.com/DmrfCoder/AndroidSourceCodeAnalysis/blob/master/EventBus/EventBus-Xmind/register%E5%88%86%E6%94%AF.png)。

## post(Event event)

然后我们分析一下post()方法的执行逻辑：

```java
 EventBus.getDefault().post(new MessageEvent("这是来自FragmentA的消息！"));
```

post()方法的源码如下：

```java
 /** Posts the given event to the event bus. */
    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

首先执行了`PostingThreadState postingState = currentPostingThreadState.get()`，看一下这个`currentPostingThreadState`是什么：

```java
 private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
        @Override
        protected PostingThreadState initialValue() {
            return new PostingThreadState();
        }
    };
```

这是一个`ThreadLocal<PostingThreadState>`类型的对象，主要是为每个线程存储一个唯一的`PostingThreadState`对象，看看<span id="PostingThreadState">`PostingThreadState`</span>是什么：

```java
    /** For ThreadLocal, much faster to set (and get multiple values). */
    final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
```

里面有一个`List`以及一些其他的成员变量。

那么这个`PostingThreadState postingState = currentPostingThreadState.get()`语句做的就是获取到当前线程对应的`PostingThreadState`对象。

然后执行 `List<Object> eventQueue = postingState.eventQueue`拿到了`postingState`对象中的那个`List`，然后执行`eventQueue.add(event)`将当前事件添加进了`eventQueue`这个`List`中。

然后执行：

```java
if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
```

首先判断`postingState.isPosting`的真值，从[PostingThreadState](#PostingThreadState)的源码我们可以看到这个`isPosting`默认是`false`的，所以会进入if语句块内执行相应逻辑。

首先：

```java
postingState.isMainThread = isMainThread();
postingState.isPosting = true;
```

给`postingState.isMainThread`和`postingState.isPosting`赋值。

然后判断`postingState.canceled`，从`PostingThreadState`源码知其默认为`false`，不会触发异常。然后遍历`eventQueue`这个`List`，执行了：

```java
 postSingleEvent(eventQueue.remove(0), postingState);
```

### EventBus.postSingleEvent(Object event, PostingThreadState postingState)

我们看看这个`postSingleEvent()`的源码：

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
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

初始执行：

```java
 Class<?> eventClass = event.getClass();
 boolean subscriptionFound = false;
```

第一句将事件`event`的类对象赋给`eventClass`，第二句将`false`赋给`subscriptionFound`。

然后判断`eventInheritance`的真值，这个`eventInheritance`我们在[上面](#eventInheritance)分析过，其值默认为`true`，所以执行：

```java
List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
int countTypes = eventTypes.size();
for (int h = 0; h < countTypes; h++) {
  Class<?> clazz = eventTypes.get(h);
  subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
}
```

首先执行了`List<Class<?>> eventTypes = lookupAllEventTypes(eventClass)`，看一下`lookupAllEventTypes()`方法.

##### EventBus.lookupAllEventTypes(Class<?> eventClass)

源码：

```java
 /** Looks up all Class objects including super classes and interfaces. Should also work for interfaces. */
    private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
                    eventTypes.add(clazz);
                    addInterfaces(eventTypes, clazz.getInterfaces());
                    clazz = clazz.getSuperclass();
                }
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
    }
```

这里涉及到一个`eventTypesCache`，我们看看它的声明：

```java
private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();
```

它是一个`HashMap`，这里使用了`synchronized`锁保证其线程安全，然后执行 `List<Class<?>> eventTypes = eventTypesCache.get(eventClass)`，易知`eventTypes`应该为`null`，则会执行：

```java
eventTypes = new ArrayList<>();
Class<?> clazz = eventClass;
while (clazz != null) {
  eventTypes.add(clazz);
  addInterfaces(eventTypes, clazz.getInterfaces());
  clazz = clazz.getSuperclass();
}
eventTypesCache.put(eventClass, eventTypes);
```

这里涉及到一个 `addInterfaces(eventTypes, clazz.getInterfaces())`:

###### EventBus.addInterfaces(List<Class<?\>\> eventTypes, Class<?\>[] interfaces)

源码：

```java
 /** Recurses through super interfaces. */
    static void addInterfaces(List<Class<?>> eventTypes, Class<?>[] interfaces) {
        for (Class<?> interfaceClass : interfaces) {
            if (!eventTypes.contains(interfaceClass)) {
                eventTypes.add(interfaceClass);
                addInterfaces(eventTypes, interfaceClass.getInterfaces());
            }
        }
    }
```

还是先明确调用这个方法时传入了什么参数：

- `eventTypes`

  > 一个`List`，存储着事件类的类对象(`eventClass`)

- `interfaces`

  > 传入了eventClass.getInterfaces()`，这个`getInterfaces()`是一个反射方法，返回的是`clazz`类实现的所有的接口的集合(数组)。

那么`addInterfaces()`方法执行的逻辑就很明确了：遍历当前事件类所实现的所有接口，判断这些接口是否在`eventTypes`这个`List`中，如果不在，则将该接口加入进去并递归调用，从而实现将事件者类实现的所有的祖先接口都加入到`eventTypes`这个`List`中去。

最后，执行`return eventTypes`将这个包含了订阅者类实现的所有接口的`List`集合`eventTypes`返回。

好，继续回到`postSingleEvent()`方法：

```java
 private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
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

现在我们分析完了`List<Class<?>> eventTypes = lookupAllEventTypes(eventClass)`这一行，得知`eventTypes`代表的是当前事件类实现的所有祖先接口的集合。然后执行了：

```java
int countTypes = eventTypes.size();
for (int h = 0; h < countTypes; h++) {
  Class<?> clazz = eventTypes.get(h);
  subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
}
```

就是遍历了这个接口集合，然后对每一个接口执行：

```java
 subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
```

即：

```java
subscriptionFound =subscriptionFound | postSingleEventForEventType(event, postingState, clazz);
```

注意`subscriptionFound`是一个`Boolean`类型的对象，所以这句的意思是只要`subscriptionFound`或者 `postSingleEventForEventType(event, postingState, clazz)`为true，则将`true`赋值给`subscriptionFound`。

我们需要分析一下`postSingleEventForEventType(event, postingState, clazz)`方法，看看它什么时候返回true：

##### EventBus.postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass)

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
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

还是先明确传入的参数：

- `event`

  > 事件对象

- `postingState`

  > 当前线程特定的的PostingThreadState对象

- `eventClass`

  > 当前订阅者类的某一个祖先接口的类对象

`postSingleEventForEventType()`方法中首先声明了一个`CopyOnWriteArrayList<Subscription> subscriptions`，然后在`synchronized`锁中执行了：

```java
subscriptions = subscriptionsByEventType.get(eventClass);
```

这个`subscriptionsByEventType`在[上面](#subscriptionsByEventType)已经讨论过了，它存储的是`事件类型`和其对应的`订阅方法&订阅类`的键值对，所以`subscriptions`就是当前事件类型对应的`订阅方法&订阅者`的集合。

然后执行：

```java
if (subscriptions != null && !subscriptions.isEmpty()) {
  for (Subscription subscription : subscriptions) {
    postingState.event = event;
    postingState.subscription = subscription;
    boolean aborted = false;
    try {
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
```

如果`subscriptions`为`null`或者其中没有内容，则返回`false`。

如果`subscriptions`不为空，则对其进行遍历，对每一个`subscriptions`中的`Subscription`对象，执行：

```java
		postingState.event = event;
    postingState.subscription = subscription;
    boolean aborted = false;
    try {
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
```

前三行主要是赋值操作，然后下面执行了：

```java
 postToSubscription(subscription, event, postingState.isMainThread);
```

这个方法是否有点眼熟？是的，在上面[`checkPostStickyEventToSubscription`](#checkPostStickyEventToSubscription)中曾经调用过这个方法，我说在讲解`post()`的时候详细解析这个方法，现在是兑现承诺的时候了。

###### EventBus.postToSubscription(Subscription subscription, Object event, boolean isMainThread) 

源码：

```java
 private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }
```

首先明确传入的实参是什么：

- `subscription` 

  > `Subscription`对象，主要成员变量是一个`订阅方法`和一个`订阅者`

- `event`

  > 事件类的类对象

- `isMainThread`

  > 表示当前订阅者是否在`MainThread`中

然后就是一个`switch`语句：

```java
switch (subscription.subscriberMethod.threadMode) {
						case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
}
```

判断了当前订阅方法的`threadMode`的值，主要是按照订阅方法的线程值分为了四个主要部分处理，`POSTING`代表着当前线程，在哪个线程发送事件，就在哪个线程处理事件；`MAIN`代表只在主线程处理事件；`BACKGROUND`代表只在非主线程处理事件；`ASYNC`也是代表在非主线程处理事件。

`BACKGROUND`和`ASYNC`都是在非主线程处理事件，那么二者有什么区别呢？
从代码中可以直观的看到：`BACKGROUND`的逻辑是如果在主线程，那么会开启一条新的线程处理事件，如果不在主线程，那么直接处理事件；而`ASYNC`的逻辑则是不管你处于什么线程，我都新开一条线程处理事件。

我来详细解释一下这四个不同的线程类型：

- `POSTING`

  ```java
  case POSTING:
      invokeSubscriber(subscription, event);
      break;
  ```

  这里调用了<span id="invokeSubscriber">`invokeSubscriber(subscription, event)`</span>。

  ###### EventBus.invokeSubscriber(Subscription subscription, Object event) 

  源码如下：

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

  还是要明确传入的参数是什么：

  - `subscription` 一个`Subscription`对象，描述了当前事件类对象对应的订阅方法和订阅者类
  - `event` 当前事件对象

   `subscription.subscriberMethod.method.invoke()`方法的意思就是调用`subscription`中订阅方法的`invoke()`方法，传入了订阅者类和事件对象两个参数，实际上就是调用了订阅方法，并将当前事件作为参数传递进去，这样订阅方法就实现了事件的接收。

- `MAIN`

  ```java
  case MAIN:
  if (isMainThread) {
  	invokeSubscriber(subscription, event);
  } else {
  	mainThreadPoster.enqueue(subscription, event);
  }
  break;
  
  ```

  `MAIN`中首先判断当前线程是否是主线程，如果是，则直接执行`invokeSubscriber`，通过反射直接执行事件处理方法。反之则通过`mainThreadPoster.enqueue(subscription, event)`来执行事件的处理，那么它是如何切换线程的呢？我们来看一下这个`mainThreadPoster`的声明：

  ```java
  EventBus{
  	...
    private final Poster mainThreadPoster;
    ...
    EventBus(EventBusBuilder builder) {
    	...
       mainThreadSupport = builder.getMainThreadSupport();
       mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
       ...
    }
  }
  
  ```

  `mainThreadPoster`的实例化和`mainThreadSupport`有关，我们来寻根溯源：

  ```java
  EventBusBuilder{
    ...
    MainThreadSupport getMainThreadSupport() {
            if (mainThreadSupport != null) {
                return mainThreadSupport;
            } else if (Logger.AndroidLogger.isAndroidLogAvailable()) {
                Object looperOrNull = getAndroidMainLooperOrNull();
                return looperOrNull == null ? null :
                        new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
            } else {
                return null;
            }
        }
    ...
  }
  ```

  这个

  ```java
   Object looperOrNull = getAndroidMainLooperOrNull();
  ```

  从名字看就是得到`Android`当前的主线程的`Looper`，看源码：

  ```java
  EventBusBuilder{
    ...
  Object getAndroidMainLooperOrNull() {
          try {
              return Looper.getMainLooper();
          } catch (RuntimeException e) {
              // Not really a functional Android (e.g. "Stub!" maven dependencies)
              return null;
          }
      }
   ... 
  }
  ```

  `getMainThreadSupport（）`最终返回的有效对象是：

  ```
  new MainThreadSupport.AndroidHandlerMainThreadSupport((Looper) looperOrNull);
  ```

  `AndroidHandlerMainThreadSupport`的源码如下：

  ```java
  public interface MainThreadSupport {
  
      boolean isMainThread();
  
      Poster createPoster(EventBus eventBus);
  
      class AndroidHandlerMainThreadSupport implements MainThreadSupport {
  
          private final Looper looper;
  
          public AndroidHandlerMainThreadSupport(Looper looper) {
              this.looper = looper;
          }
  
          @Override
          public boolean isMainThread() {
              return looper == Looper.myLooper();
          }
  
          @Override
          public Poster createPoster(EventBus eventBus) {
              return new HandlerPoster(eventBus, looper, 10);
          }
      }
  
  	}
  ```

  所以

  ```java
   mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
  ```

  中有效的值为`AndroidHandlerMainThreadSupport`中`public Poster createPoster(EventBus eventBus)`的返回值：`new HandlerPoster(eventBus, looper, 10)`看看这个`HandlerPoster`是什么东西.

  ###### HandlerPoster
  
  源码：
  
  ```java
  public class HandlerPoster extends Handler implements Poster {
  
      private final PendingPostQueue queue;
      private final int maxMillisInsideHandleMessage;
      private final EventBus eventBus;
      private boolean handlerActive;
  
      protected HandlerPoster(EventBus eventBus, Looper looper, int maxMillisInsideHandleMessage) {
          super(looper);
          this.eventBus = eventBus;
          this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
          queue = new PendingPostQueue();
    }
    ..
}
  ```
  
  可以看到这个`HandlerPoster`也就是`mainThreadPoster`实际上是一个主线程的`Handler`，因为它对应的`looper`是从主线程拿的。至此再看这段代码：
  
  ```java
  case MAIN:
      if (isMainThread) {
        invokeSubscriber(subscription, event);
      } else {
        mainThreadPoster.enqueue(subscription, event);
      }
		break;
  

  ```
  
  如果当前线程不是主线程，则使用主线程中的一个`Handler`对象实现线程切换，那么我们深入看一下`mainThreadPoster.enqueue(subscription, event)`的源码：
  
  ```java
  public class HandlerPoster extends Handler implements Poster {
  				...
          public void enqueue(Subscription subscription, Object event) {
                  PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
                  synchronized (this) {
                      queue.enqueue(pendingPost);
                      if (!handlerActive) {
                          handlerActive = true;
                          if (!sendMessage(obtainMessage())) {
                              throw new EventBusException("Could not send handler message");
                          }
                      }
                  }
            }
              ...
   }
  ```

  这里首先执行`PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event)`获取了一个`PendingPost`对象，我们看看这个`PendingPost`类：

  ![image-20190424135944937](https://ws4.sinaimg.cn/large/006tNc79gy1g2dooqy0dgj30m60acjsg.jpg)

  这个`PendingPost`类中有一个`PendingPost next`属性，应该类似于链表中的"`下一个`"指针。
  
  深入一下`obtainPendingPost()`方法的源码：
  
  ```java
  static PendingPost obtainPendingPost(Subscription subscription, Object event) {
          synchronized (pendingPostPool) {
              int size = pendingPostPool.size();
              if (size > 0) {
                  PendingPost pendingPost = pendingPostPool.remove(size - 1);
                  pendingPost.event = event;
                  pendingPost.subscription = subscription;
                  pendingPost.next = null;
                  return pendingPost;
              }
          }
        return new PendingPost(event, subscription);
      }

  ```
  
  首先使用`synchronized`对`pendingPostPool`对象进行了加锁，看一下这个`pendingPostPool`对象的声明：

  ```java
 private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();
  ```

  它是一个`List`对象，然后如果该`List`的`size`不为0，则返回该`List`的最后一个元素，否则就`new`一个`PendingPost`对象并返回。
  
  那么继续回到这段代码：
  
  ```java
  public class HandlerPoster extends Handler implements Poster {
  				...
          public void enqueue(Subscription subscription, Object event) {
                  PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
                  synchronized (this) {
                      queue.enqueue(pendingPost);
                      if (!handlerActive) {
                          handlerActive = true;
                          if (!sendMessage(obtainMessage())) {
                              throw new EventBusException("Could not send handler message");
                          }
                      }
                  }
            }
              ...
   }
  ```
  
  获取到`PendingPost`之后使用`synchronized`锁对当前对象进行了加锁，然后将该`pendingPost`对象加入队列，这里再看一下和这个`queue`的声明：

  ```java
 private final PendingPostQueue queue;
  ```
  
  再深入看看这个`PendingPostQueue`：
  
  ```java
  final class PendingPostQueue {
      private PendingPost head;
      private PendingPost tail;
  
      synchronized void enqueue(PendingPost pendingPost) {
          if (pendingPost == null) {
              throw new NullPointerException("null cannot be enqueued");
          }
          if (tail != null) {
              tail.next = pendingPost;
              tail = pendingPost;
          } else if (head == null) {
              head = tail = pendingPost;
          } else {
              throw new IllegalStateException("Head present, but no tail");
          }
          notifyAll();
      }
  
      synchronized PendingPost poll() {
          PendingPost pendingPost = head;
          if (head != null) {
              head = head.next;
              if (head == null) {
                  tail = null;
              }
          }
          return pendingPost;
      }
  
      synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
          if (head == null) {
              wait(maxMillisToWait);
          }
          return poll();
      }

  }

  ```

  好了，明白了，这个`PendingPostQueue`实际上就是作者自己写的一个双端队列，实现了入队(`enqueue`)和出队(`poll`)操作。
  
  所以继续看这段代码：
  
  ```java
  public class HandlerPoster extends Handler implements Poster {
  				...
          public void enqueue(Subscription subscription, Object event) {
                  PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
                  synchronized (this) {
                      queue.enqueue(pendingPost);
                      if (!handlerActive) {
                          handlerActive = true;
                          if (!sendMessage(obtainMessage())) {
                              throw new EventBusException("Could not send handler message");
                          }
                      }
                  }
            }
              ...
   }
  ```
  
  `queue.enqueue(pendingPost)`就是将由当前订阅者和当前事件获取到的`PendingPost`对象加入到`queue`这个双向队列中。然后判断了一下`handlerActive`的真值，`handlerActive`的声明如下：

  ```java
 private boolean handlerActive;
  ```
  
  所以其默认值为`false`，至于其作用我会在下面说到，这里进入`if`代码块内，执行：
  
  ```java
  handlerActive = true;
if (!sendMessage(obtainMessage())) {
    throw new EventBusException("Could not send handler message");
}
  ```
  
  首先将`handlerActive`置为了`true`，然后执行`sendMessage(obtainMessage())`方法，并判断其返回结果，执行完`sendMessage()`方法之后会在本`Handler`类的`handleMessage(Message msg)`方法中收到消息，`handleMessage(Message msg)`源码如下：
  
  ```java
  @Override
      public void handleMessage(Message msg) {
          boolean rescheduled = false;
          try {
              long started = SystemClock.uptimeMillis();
              while (true) {
                  PendingPost pendingPost = queue.poll();
                  if (pendingPost == null) {
                      synchronized (this) {
                          // Check again, this time in synchronized
                          pendingPost = queue.poll();
                          if (pendingPost == null) {
                              handlerActive = false;
                              return;
                          }
                      }
                  }
                  eventBus.invokeSubscriber(pendingPost);
                  long timeInMethod = SystemClock.uptimeMillis() - started;
                  if (timeInMethod >= maxMillisInsideHandleMessage) {
                      if (!sendMessage(obtainMessage())) {
                          throw new EventBusException("Could not send handler message");
                      }
                      rescheduled = true;
                      return;
                  }
              }
          } finally {
            handlerActive = rescheduled;
          }
    }
  ```

  首先声明了 `boolean rescheduled = false`，然后获取了当前时间： `long started = SystemClock.uptimeMillis()`。
  
  然后是一个循环：
  
  ```java
  while (true) {
                  PendingPost pendingPost = queue.poll();
                  if (pendingPost == null) {
                      synchronized (this) {
                          // Check again, this time in synchronized
                          pendingPost = queue.poll();
                          if (pendingPost == null) {
                              handlerActive = false;
                              return;
                          }
                      }
                  }
                  eventBus.invokeSubscriber(pendingPost);
                  long timeInMethod = SystemClock.uptimeMillis() - started;
                  if (timeInMethod >= maxMillisInsideHandleMessage) {
                      if (!sendMessage(obtainMessage())) {
                          throw new EventBusException("Could not send handler message");
                      }
                      rescheduled = true;
                    return;
                  }
            }
  ```
  
  循环中，首先从队列`queue`中出队一个元素，然后判断其是否为`null`，如果不为`null`，则使用`synchronized`锁锁定当前对象，再次执行一次出队操作。我来解释一下这里为什么判断了两次`queue.poll()`:
  
  解释一下：
  
  - 第一次判断`queue.poll()`是否为`null`
      第一次判断是在`Synchronized`同步代码块外进行判断，是为了在`queue.poll()`不为`null`的情况下，避免进入同步代码块，**提升效率**。
  
  - 第二次判断`queue.poll()`是否为`null`
      第二次判断是为了避免以下情况的发生。 
      (1)假设：线程A已经经过第一次判断，判断`pendingPost == null`，准备进入同步代码块. 
      (2)此时线程B获得时间片，去执行`enqueue（）`操作成功入队。
      (3)此时，线程A再次获得时间片，由于刚刚经过第一次判断`pendingPost == null`(不会重复判断)，进入同步代码块再次判`null`，然后拿到不为`null`的`pendingPost`。如果不加第二次判断的话，线程A直到下一次被调用`handleMessage（）`才有可能会处理刚才线程B加入的事件，这样不利于事件的快速传递，所以第二次判断是很有必要的。

  ```java
synchronized (this) {
    // Check again, this time in synchronized
    pendingPost = queue.poll();
    if (pendingPost == null) {
      handlerActive = false;
    return;
    }
}
  ```

  这里加锁之后如果出队结果还是`null`，那么就说明队列中真的没有元素了，将`handlerActive`置为`false`然后`return`。到这里`handlerActive`的作用就显而易见了，其就是标记队列是否为空的，初始时队列为空，`handlerActive`为`false`，顺利入队后队列一定不为空了，将`handlerActive`置为`true`，现在出队直到队列为空时再次将`handlerActive`置为`false`，这样仅仅通过`handlerActive`就可判断队列是否为空，而不用访问队列再判空。

  那么如果队列出队的结果不为`null`，就会执行：

  ```java
  eventBus.invokeSubscriber(pendingPost);
  long timeInMethod = SystemClock.uptimeMillis() - started;
  if (timeInMethod >= maxMillisInsideHandleMessage) {
    if (!sendMessage(obtainMessage())) {
      throw new EventBusException("Could not send handler message");
    }
  rescheduled = true;
    return;
}
  ```

  首先执行`eventBus.invokeSubscriber(pendingPost)`， [invokeSubscriber](#invokeSubscriber)方法我在上面已经解析过，作用就是执行订阅方法，将事件传递过去，这样订阅方法就能收到订阅内容了。

  `eventBus.invokeSubscriber(pendingPost)`之后下一行计算了本方法迄今为止的耗时，如果已经超过了`maxMillisInsideHandleMessage`，那么调用`sendMessage(obtainMessage())`代表再次尝试，如果该方法返回`false`，则抛出异常，否则将`rescheduled` 置位`true`，并`return`。

  那么迄今为止本方法的耗时没有超过`maxMillisInsideHandleMessage`，则执行：

  ```java
  @Override
      public void handleMessage(Message msg) {
          boolean rescheduled = false;
          try {
              long started = SystemClock.uptimeMillis();
              while (true) {
                  PendingPost pendingPost = queue.poll();
                  if (pendingPost == null) {
                      synchronized (this) {
                          // Check again, this time in synchronized
                          pendingPost = queue.poll();
                          if (pendingPost == null) {
                              handlerActive = false;
                              return;
                          }
                      }
                  }
                  eventBus.invokeSubscriber(pendingPost);
                  long timeInMethod = SystemClock.uptimeMillis() - started;
                  if (timeInMethod >= maxMillisInsideHandleMessage) {
                      if (!sendMessage(obtainMessage())) {
                          throw new EventBusException("Could not send handler message");
                      }
                      rescheduled = true;
                      return;
                  }
              }
          } finally {
            handlerActive = rescheduled;
          }
      }
  ```

  最后的`finally`语句中还将 `rescheduled`赋值给了`handlerActive` 。

至于`postToSubscription(Subscription subscription, Object event, boolean isMainThread)`中其他几个`Switch`的情况和上述类似这里不做过多解释。

### post(Event event总结)

好，现在我们总结一下`post()`方法的逻辑：

- 将事件加入当前线程的事件队列
- 遍历这个事件队列
  - 找到当前事件类型所有的父类事件类型，加入事件类型集合并遍历
    - 通过`subscriptionsByEventType`获取`subscriptions`集合，遍历这个`subscriptions`集合
      - 在`POSTING`、`MAIN`、`MAIN_ORDERED`、`BACKGROUND`、`ASYNC`五种线程模式下通过反射执行事件处理方法

对应逻辑的思维导图如下：

![post分支](https://ws1.sinaimg.cn/large/006tNc79gy1g2ezrt1emnj32lb0u07q2.jpg)

如果上图看不清，可[下载原图查看](https://github.com/DmrfCoder/AndroidSourceCodeAnalysis/blob/master/EventBus/EventBus-Xmind/post%E5%88%86%E6%94%AF.png)。

## postSticky(Event event)

这里再看一下`postSticky`方法：

```java
public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }

```

首先用了一个锁锁住了`stickyEvents`，这个[`stickyEvents`](#stickyEvents)我在上面讲过：

```java
  private final Map<Class<?>, Object> stickyEvents;
 EventBus(EventBusBuilder builder) {
 ...
	 stickyEvents = new ConcurrentHashMap<>();
	 ...
 }
```

可能有的同学会有疑问，`ConcurrentHashMap`它是一个线程安全的类啊，这里为什么还要对它加锁呢？是的，`ConcurrentHashMap`是一个线程安全的类，但是他的线程安全指的是其内部的方法是原子的、线程安全的，但是它不能保证其内部方法的复合操作也是线程安全的。也就是说它能保证`put()`操作是线程安全的，但是不能保证在进入`postSticky()`之后执行`put()`操作之前没有其他线程对`stickyEvents`执行其他操作，所以这里对`stickyEvents`加了一个`synchronized`锁。

然后将`<事件对应的类型-事件>`加入了该`Map`中，然后执行`Post()`方法，之后的逻辑和上面解析的`post()`的执行一样。

## unregister(Object object)

反注册的源码如下：

```java
 /** Unregisters the given subscriber from all event classes. */
    public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }

```

首先我们上面通过分析得知`typesBySubscriber`这个`map`存储的是订阅者类和订阅者类对应的所有订阅方法的事件类型集合的映射对，这里调用`typesBySubscriber.get(subscriber)`应该是获取了当前订阅者类的事件类型的类对象集合。

然后进行了判空，如果不为空，则执行：

```java
for (Class<?> eventType : subscribedTypes) {
  unsubscribeByEventType(subscriber, eventType);
}
typesBySubscriber.remove(subscriber);
```

对所有事件类型的类对象进行遍历，依次执行 `unsubscribeByEventType(subscriber, eventType)`，最后将本订阅类从订阅类的`List`中移除。

### EventBus.unsubscribeByEventType(Object subscriber, Class<?> eventType) 

我们深入一下 `unsubscribeByEventType(subscriber, eventType)`的源码：

```java
 /** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
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

首先从`subscriptionsByEventType`中拿出了所有的以`eventType`为事件类型类对象的订阅方法，然后对其进行遍历，如果当前方法的订阅类和当前要反注册的类相同，则将该订阅方法移除。

所以总结一下，`unregister（）`方法的核心内容就是删除全局变量`subscriptionsByEventType`中所有包含当前订阅者的`subscription`，这也就很好的解释了为什么在解绑后我们不会再收到`EventBus`发送的事件了。因为发送事件的核心是根据事件类型从`subscriptionsByEventType`中获取`subscriptions`这个集合，遍历集合通过`subscription`可以拿到`subscriber`和`subscriberMethod`,之后再通过`subscription.subscriberMethod.method.invoke(subscription.subscriber, event)`，采用反射来进行事件的处理。`unregister`方法删除了全局变量`subscriptionsByEventType`中所有包含当前订阅者的`subscription`，在遍历`subscriptions`的时候是不会获取包含当前订阅者的`subscription`，所以自然不会再收到事件。

###  观察者模式与EventBus

其实EventBus的核心要点就是**观察者模式**。

经典的观察者模式中，**被观察者**只有一个，而观察者的个数不确定。被观察者内部维持着一个观察者的集合，当**被观察者**要发送事件时会遍历内部的观察者集合，拿到观察者并调用观察者的相关方法。

具体到EventBus，EventBus就是一个**被观察者**，它的内部存放着一个**subscriptionsByEventType**（订阅方法）集合，这个集合包含了我们所有的观察者，也就是调用了register的所有Activity或者Fragment中的订阅方法。当使用post发送事件时，会遍历**subscriptionsByEventType**，拿到观察者，通过反射调用观察者中的事件处理方法。

## 总结

这里给出EventBus的整个思维导图：

![EventBus](https://ws3.sinaimg.cn/large/006tNc79gy1g2ezwvvinzj31g90u0b2a.jpg)

如果上图看不清，可下载[原图](https://github.com/DmrfCoder/AndroidSourceCodeAnalysis/blob/master/EventBus/EventBus-Xmind/EventBus.png)或[源文件](https://github.com/DmrfCoder/AndroidSourceCodeAnalysis/blob/master/EventBus/EventBus-Xmind/EventBus.xmind)查看。

