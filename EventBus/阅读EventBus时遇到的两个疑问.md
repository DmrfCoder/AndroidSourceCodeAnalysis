# 问题一

`EventBus`中的`HandlerPoster`类，源码如下：

```java

public class HandlerPoster extends Handler implements Poster {

    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    private boolean handlerActive;

    @Override
    public void handleMessage(Message msg) {
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
    }
}
```

这里第11行先调用`queue.poll()`，如果获取到的结果为`null`，使用`synchronized`块执行一次`queue.poll()`，如果这次还是返回`null`才确定`queue.poll()`确实为`null`，猜想是为了保证线程安全，`PendingPostQueue queue`对应的`PendingPostQueue`源码如下：

```java

final class PendingPostQueue {
    private PendingPost head;
    private PendingPost tail;

    synchronized void enqueue(PendingPost pendingPost) {
        ...
    }

    synchronized PendingPost poll() {
       ...
    }

    synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
       ...
    }

}
```

可是个PendingPostQueue类对其方法都加了synchronized锁，不就说明这个PendingPostQueue类对应的对象是线程安全的吗？为什么HandlerPoster.handleMessage(Message msg)还需要用synchronized锁二次执行`queue.poll()`方法进行确定？

## 回答

解释一下：

- 第一次判断`queue.poll()`是否为`null`
    第一次判断是在`Synchronized`同步代码块外进行判断，是为了在`queue.poll()`不为`null`的情况下，避免进入同步代码块，**提升效率**。

- 第二次判断`queue.poll()`是否为`null`
    第二次判断是为了避免以下情况的发生。 
    (1)假设：线程A已经经过第一次判断，判断`pendingPost == null`，准备进入同步代码块. 
    (2)此时线程B获得时间片，去执行`enqueue（）`操作成功入队。
    (3)此时，线程A再次获得时间片，由于刚刚经过第一次判断`pendingPost == null`(不会重复判断)，进入同步代码块再次判`null`，然后拿到不为`null`的`pendingPost`。如果不加第二次判断的话，线程A直到下一次被调用`handleMessage（）`才有可能会处理刚才线程B加入的事件，这样不利于事件的快速传递，所以第二次判断是很有必要的。

# 问题二

`EventBus.postStick()`的源码如下：

```java
public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
```

这里对`stickyEvents`加了`synchronized`锁以保证线程安全，可是`stickyEvents`的声明如下：

```java
private final Map<Class<?>, Object> stickyEvents;
 EventBus(EventBusBuilder builder) {
 ...
	 stickyEvents = new ConcurrentHashMap<>();
	 ...
 }
```

它虽然声明为`Map`类，但是实例化的时候用的是`new ConcurrentHashMap<>()`，所以它本身应该就是线程安全的，为什么还需要在`postStick()`中对`stickyEvents`对象进行加锁？

## 回答

ConcurrentHashMap是线程安全的指的是ConcurrentHashMap内部的方法执行时时原子的、线程安全的。但是不能保证使用ConcurrentHashMap中的方法进行复合操作的时候是线程安全的，比如：

```
T1:                                        T2
concurrent_map.insert(key1,val1)           concurrent_map.contains(key1)
```

T1 和T2 的单个操作使线程安全的.

但如果T1 和T2 是

```text
if (! concurrent_map.contains(key1) {
    concurrent_map.insert(key1,val1)
}
```

即使 `contains` 和 `insert` 他们本身是线程安全的，但是有可能在T1执行完了之后要执行T2之前又有另一个删除key1的操作执行了，那么上面的代码块就和预期的执行结果不一样了。

再举个例子：

```
ThreadA：						
map.put(1,1);
map.put(2,2);
```



```
ThreadB:
map.put(a,a);
map.put(a,a);
```

那么当两个线程并发的时候你能知道map最终的值吗？当然不知道，因为ConcurrentHashMap只能保证put()操作是原子的是线程安全的，但是他没法保证你执行的复合操作使线程安全的。