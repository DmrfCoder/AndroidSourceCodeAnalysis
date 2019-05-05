## 概述

`OkHttp`是一个适用于`Android`和`Java`应用程序的`HTTP` + `HTTP/2`框架。

## 使用示例

```java
				//创建OkHttpClient.Builder
		OkHttpClient.Builder builder = new OkHttpClient.Builder();

				//创建OkHttpClient
        OkHttpClient okHttpClient = builder.build();

				//创建Request
        Request request = new Request.Builder().build();

				//创建Call
        Call call = okHttpClient.newCall(request);

				//发起同步请求
        try {
            Response response = call.execute();
        } catch (IOException e) {
            e.printStackTrace();
        }

				//发起异步请求
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
```

一步一步来看，首先第一步是获取一个`OkHttpClient.Builder`对象，然后第二步通过这个`builder`对象`build()`出来了一个`OkHttp`对象，不用说，这是简单的建造者（`Builder`）设计模式，看一眼`OkHttpClient.Builder`的构造方法：

```java
 public Builder() {
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      if (proxySelector == null) {
        proxySelector = new NullProxySelector();
      }
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      callTimeout = 0;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
    }
```

这里初始化了一堆东西，我们重点注意一下第2行的dispatcher对象，对应的Dispathcher类时OkHttp中的核心类，我们后面会重点详细解析，大家先有个印象即可。

第三步是通过`Request request = new Request.Builder().build()`初始化了一个`Request`对象，也是建造者模式，这个`Request`对象主要是描述要发起请求的详细信息。

第四步通过` Call call = okHttpClient.newCall(request)`创建了一个`Call`的对象，来看看这个`newCall()`方法：

```java
 @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }

```

再继续深入到`RealCall.newRealCall()`中：

```java
  static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.transmitter = new Transmitter(client, call);
    return call;
  }
```

到这里我们就明白了，我们获取到的`Call`对象实际上是一个`RealCall`对象，看看`Call`和`RealCall`的声明：

`Call`：

```java
public interface Call extends Cloneable
```

`RealCall`：

```java
final class RealCall implements Call 
```

明白了，`Call`是一个接口，而`RealCall`实现了这个接口，所以返回`new RealCall()`给`call`对象当然是没问题的。

然后，如果我们想发起同步网络请求，则执行：

```java
 Response response = call.execute();
```

如果想发起异步网络请求，则执行：

```java
 call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
```

我们先来分析同步的情况。

## RealCall.excute()

我们上面提到，这个`call`实际上是一个`RealCall`对象，那么我们看看这个`RealCall.excute()`方法的源码：

```java
   @Override public Response execute() throws IOException {
        synchronized (this) {
          if (executed) throw new IllegalStateException("Already Executed");
          executed = true;
        }
        transmitter.timeoutEnter();
        transmitter.callStart();
        try {
          client.dispatcher().executed(this);
          return getResponseWithInterceptorChain();
        } finally {
          client.dispatcher().finished(this);
        }
  }
```

可以看到，这里首先使用了一个`synchronized`锁判断了`executed`标志位的值，如果`executed`为`true`，则抛出异常，异常信息为`"Already Executed"`，否则将`executed`置为`true`，继续执行下面的逻辑。所以，这个`executed`就是用来标记当前`RealCall`的`excute（）`方法是否已经被执行过，后面到异步请求`enqueue（）`的代码中我们会发现同样使用了这个`executed`标志位做了相同逻辑的判断，所以我们可以得出一个`Call`对象只能被执行一次（不管是同步请求还是异步请求）的结论。

==那么可能有同学会有疑问了，为什么OkHttp中的一个Call对象只能发起一次请求？这个和OkHttp中的连接池有关系，我们会在后面讲ConnectInterceptor拦截器的时候详细分析。==

如果`excuted`判断没有问题之后，就会执行：

```java
transmitter.timeoutEnter();
transmitter.callStart();
try {
    client.dispatcher().executed(this);
    return getResponseWithInterceptorChain();
} finally {
  	client.dispatcher().finished(this);
}
```

我们抓住重点，直接从第4行开始看，这里执行了 `client.dispatcher().executed(this)`，注意这个`client`是我们刚才传进来的`OkHttpClient`对象，`dispather`对象是我们刚才在上面提到的`OkHttpClient.Builder`中初始化的，我们来看看这个`Dispatcher.excuted()`方法：

```java
 /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

可以看到这里主要是将当前这次请求的`call`对象加入到了`runningSyncCalls`中，我们看看这份`runningSyncCalls`的声明：

```java
 /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

这个`runningSyncCalls`是一个队列，从其源代码的注释我们可以得知这个`runningSyncCalls`的作用是存储当前`OkHttpClient`正在执行的同步请求。

好，下一步我们来分析：

```java
@Override public Response execute() throws IOException {
        synchronized (this) {
          if (executed) throw new IllegalStateException("Already Executed");
          executed = true;
        }
        transmitter.timeoutEnter();
        transmitter.callStart();
        try {
          client.dispatcher().executed(this);
          return getResponseWithInterceptorChain();
        } finally {
          client.dispatcher().finished(this);
        }
  }
```

中第9行之后的代码，可以看到，第10行直接返回了一个`getResponseWithInterceptorChain()`，而`public Response execute()`方法返回的是一个`Response`对象，所以说这个`getResponseWithInterceptorChain()`方法返回的也是一个`Response`对象，即这个`getResponseWithInterceptorChain()`方法中执行了真正的同步请求的逻辑并返回了`Response`对象，其具体实现细节我们后面详细分析。

注意，`Response execute()`方法的第11行到第13行，这是`try...finally`语句块中的`finally`体，也就是说无论`try`中语句的执行结构如何，都会执行这个`finally`块中代码，其中只有一行代码：

```java
client.dispatcher().finished(this);
```

我们来看看`Dispatcher.finished()`方法的实现：

```java
 /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call);
  }

  private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }

    boolean isRunning = promoteAndExecute();

    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
  }

```

`client.dispatcher().finished(this)`调用了`dispatcher().finished(this)`方法并将自身(`call`)传递了进去，在`finished(RealCall call)`方法中又调用了`finished(Deque<T> calls, T call)`方法，传入了`runningSyncCalls`和我们当前的`call`对象，还记得这个`runningSyncCalls`在哪里出现过吗？对的，它在`dispater.excuted()`方法中出现过，当时的操作是将当前`call`对象加入到这个`runningSyncCalls`队列中，那么现在请求结束了，`finished()`方法中应该做什么？当然是将当前`call`对象从`runningSyncCalls`队列中移除，在代码中就是：

```java
synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }
```

可以看到为了考虑线程安全，这里使用了`synchronized`锁保证了线程同步，然后将当前`call`从`runningSyncCalls`队列中移除。

到这里我们就分析完了同步请求的大致流程，现在我们来看一下`OkHttp`中发起请求的核心方法`getResponseWithInterceptorChain()`，可以看到在同步请求中仅仅调用了这一个方法就得到了返回的`Response`，我们来看一下它的源码：

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,originalRequest, this, client.connectTimeoutMillis(),client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
      Response response = chain.proceed(originalRequest);
      if (transmitter.isCanceled()) {
        closeQuietly(response);
        throw new IOException("Canceled");
      }
      return response;
    } catch (IOException e) {
      calledNoMoreExchanges = true;
      throw transmitter.noMoreExchanges(e);
    } finally {
      if (!calledNoMoreExchanges) {
        transmitter.noMoreExchanges(null);
      }
    }
  }
```

第3行到第12行是`new`了一个`List`并依次将`用户自定义的应用拦截器集合`（第4行）、`OkHttp内置的拦截器集合`（第5-8行）、`用户自定义的网络拦截器集合`（第9-11行）添加了进去，构成了一个大的拦截器集合。

然后执行：

```java
 Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,originalRequest, this, client.connectTimeoutMillis(),client.readTimeoutMillis(), client.writeTimeoutMillis());
```

再看看RealInterceptorChain类的构造方法：

```java
public RealInterceptorChain(List<Interceptor> interceptors, Transmitter transmitter,
      @Nullable Exchange exchange, int index, Request request, Call call,
      int connectTimeout, int readTimeout, int writeTimeout) {
    this.interceptors = interceptors;
    this.transmitter = transmitter;
    this.exchange = exchange;
    this.index = index;
    this.request = request;
    this.call = call;
    this.connectTimeout = connectTimeout;
    this.readTimeout = readTimeout;
    this.writeTimeout = writeTimeout;
  }
```

主要是初始化了一个`RealInterceptorChain`类的对象，注意一下传入的第1个参数（`interceptors`）和第4个参数(`index`)，分别传入的是我们刚才构成的拦截器集合以及0，一会我们会用到。

初始化好`RealInterceptorChain`的对象后继续往下执行，关注一下第18行，可以看到，真正返回的`response`就是从这里的 `chain.proceed(originalRequest)`方法返回的，当前这个`chain`是`RealInterceptorChain`类的对象，所以我们来看看`RealInterceptorChain.proceed()`方法中做了什么：

```java
 @Override public Response proceed(Request request) throws IOException {
    return proceed(request, transmitter, exchange);
  }
```

可以看到，虽然我们调用的是`chain.proceed(originalRequest)`，但是实际上它内部执行的是 `proceed(request, transmitter, exchange)`，我们来看看这个方法的源码：

```java
public Response proceed(Request request, Transmitter transmitter, @Nullable Exchange exchange) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.exchange != null && !this.exchange.connection().supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.exchange != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (exchange != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    if (response.body() == null) {
      throw new IllegalStateException(
          "interceptor " + interceptor + " returned a response with no body");
    }

    return response;
  }
```

第2行首先检查了一下`index`是否超出了`interceptors`的`size`，还记得`index`和`interceptors`是什么吗？对的，就是我们在`getResponseWithInterceptorChain()`源码的第14行传入的0和我们初始化的拦截器集合，为什么要检测`index`和`interceptors`的`size`之间的关系呢？猜想是想通过`index`访问·中的元素，我们继续往下看，注意第19行到第21行，我们把这几行代码拿下来：

```java
 RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```

这里又初始化了一个`RealInterceptorChain`对象，那这里初始化的这个`RealInterceptorChain`对象和当前`RealInterceptorChain`有什么区别呢？我们再看看当前`RealInterceptorChain`对象初始化的代码：

```java
Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,originalRequest, this, client.connectTimeoutMillis(),client.readTimeoutMillis(), client.writeTimeoutMillis());
```

可以发现，只有一个参数不一样，就是第4个参数，对应`RealInterceptorChain`构造方法中的`index`参数，现在这个`RealInterceptorChain next`对象的构造方法中的`index`值传入的是当前`RealInterceptorChain`对象的`index+1`。然后下一行果然通过`index`拿到了`interceptors`中的元素`interceptor`，这也是`proceed()`方法的开头为什么先检测`index`和`interceptors.size()`大小关系的原因，就是为了防止这发生越界异常。拿到`interceptor`对象之后，下一行执行了`interceptor.intercept(next)`并返回了`response`，而最后也是将这个`response`最终作为当前`proceed()`方法的返回值，这时候我们就有必要深入一下`interceptor.intercept(next)`的源码了，我们尝试跟踪源码，会发现这个`Interceptor`其实是一个接口：
![在这里插入图片描述](https://ws4.sinaimg.cn/large/006tNc79gy1g2nzbq4i8kj312w0u0n3p.jpg)


我们看看`Interceptor`都有哪些实现类：
 ![image-20190503114437541](https://ws3.sinaimg.cn/large/006tNc79gy1g2nzcxk6ytj31gw0a80xc.jpg)
我们看到了5个拦截器类，由于当前`interceptor`是通过`interceptors.get(index)`拿到的，而`index`当前传入的值为0，所以第一次执行的应该是第一个加入拦截器集合的那个拦截器类的`intercept()`方法，这里我们先不考虑用户自添加的拦截器，那么第一个拦截器就应该是`RetryAndFollowUpInterceptor`拦截器，我们来看看它的`intercept()`方法：

```java
 @Override public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Transmitter transmitter = realChain.transmitter();

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      transmitter.prepareToConnect(request);

      if (transmitter.isCanceled()) {
        throw new IOException("Canceled");
      }

      Response response;
      boolean success = false;
      try {
        response = realChain.proceed(request, transmitter, null);
        success = true;
      } catch (RouteException e) {
        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), transmitter, false, request)) {
          throw e.getFirstConnectException();
        }
        continue;
      } catch (IOException e) {
        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, transmitter, requestSendStarted, request)) throw e;
        continue;
      } finally {
        // The network call threw an exception. Release any resources.
        if (!success) {
          transmitter.exchangeDoneDueToException();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }

      Exchange exchange = Internal.instance.exchange(response);
      Route route = exchange != null ? exchange.connection().route() : null;
      Request followUp = followUpRequest(response, route);

      if (followUp == null) {
        if (exchange != null && exchange.isDuplex()) {
          transmitter.timeoutEarlyExit();
        }
        return response;
      }

      RequestBody followUpBody = followUp.body();
      if (followUpBody != null && followUpBody.isOneShot()) {
        return response;
      }

      closeQuietly(response.body());
      if (transmitter.hasExchange()) {
        exchange.detachWithViolence();
      }

      if (++followUpCount > MAX_FOLLOW_UPS) {
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      request = followUp;
      priorResponse = response;
    }
  }
```

代码很长，请大家暂时忽略其他代码，只关注第3行和第18行，第3行将当前传入的`Chain chain`对象类型转化为了`RealInterceptorChain realChain`对象，第18行执行了`realChain.proceed(request, transmitter, null)`并返回了`response`，注意，这个`realChain`是我们调用当前`intercept()`方法时传入的`chain`参数，而这个`chain`参数传入的是：

```java
 RealInterceptorChain next = new RealInterceptorChain(interceptors, transmitter, exchange,index + 1, request, call, connectTimeout, readTimeout, writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```

即由`interceptors`和`index+1`构建的新的`RealInterceptorChain`，所以整个逻辑就是：![image-20190505084646823](https://ws4.sinaimg.cn/large/006tNc79gy1g2q5gk655ij31440u0wst.jpg)

可以发现，整个拦截器集合构成了一个链式结构，当前拦截器执行完对应的拦截方法后激活下一个拦截器开始工作，直到最后一个拦截器，这也告诉我们如果要添加自定义的拦截器，则必须在重写`intercept（Chain chain）`方法时返回`chain.proceed()`，否则责任链就会断链，自然不会成功地发起网络请求。

注意由于`CallServerInterceptor`这个拦截器是最后一个拦截器，所以它的`intercept`方法中没有像之前的拦截器那样执行`next.proceed()`，而是直接使用传入的`Chain chain`参数直接发起了网络请求并返回`Response`。

至此，我们已经分析完了`OkHttp`同步请求的完整流程，总结一下：

- `RealCall.excute()`发起网络同步请求
- 利用`excuted`标志位判断是否当前`call`对象已经执行过，若执行过抛出异常
- 调用`client.dispatcher.excuted()`，将当前`call`对象加入`runningSyncCalls`这个队列
- 调用`getResponseWithInterceptorChain()`方法，内部利用责任链模式依次执行拦截器链中的拦截器，最终发起网络请求并返回`Response`
- 调用`client.dispatcher.finished()`，将当前`call`对象从`runningSyncCalls`队列中移除
- 返回`Response`

## Realcall.enqueue()

先看一下发起`OkHttp`异步网络请求的典型代码：

```java
 call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
```

可以看到，传入了一个`CallBack`的回调对象，该`CallBack`类中有`onFailure（）`和`onResponse（）`两个方法，分别代表网络请求发起成功和失败的情况，这里抛出一个问题供大家思考：`onFailure（）`和`onResponse()`这两个方法是处于主线程的还是子线程的？这个问题我会在后面的分析中解答。

那么我们看一下`RealCall.enqueue(Callback callback)`的源码：

```java
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.callStart();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

可以看到，和同步的`excute()`一样，开头先会检测`excuted`标志位判断当前`call`对象是否已经被执行过，如果已经被执行过，抛出异常。

如果当前`call`对象没有被执行过，则执行第7行，调用`Dispatcher`的`enqueue()`方法，传入了一个`AsyncCall`参数，我们先看看`Dispatcher.enqueue()`方法的源码：

```java
void enqueue(AsyncCall call) {
    synchronized (this) {
        readyAsyncCalls.add(call);

        // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
        // the same host.
        if (!call.get().forWebSocket) {
          AsyncCall existingCall = findExistingCallWithHost(call.host());
          if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
        }
    }
    promoteAndExecute();
  }
```

首先加锁将传入的`AsyncCall call`加入`readyAsyncCalls`这个队列，然后执行了第7到第10行，首先判断`call.get().forWebSocket`的值，其声明及初始化如下：

```java
 final boolean forWebSocket;
 private RealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
		 ......
     this.forWebSocket = forWebSocket;
 }
```

而`RealCall`的构造方法的调用代码如下：

```java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```

即`forWebSocket`的值默认为false，所以，会执行`Dispatcher.enqueue()`方法中 的第8-9行：

```java
AsyncCall existingCall = findExistingCallWithHost(call.host());
if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
```

看看`findExistingCallWithHost()`方法 的源码：

```java
@Nullable private AsyncCall findExistingCallWithHost(String host) {
    for (AsyncCall existingCall : runningAsyncCalls) {
      if (existingCall.host().equals(host)) return existingCall;
    }
    for (AsyncCall existingCall : readyAsyncCalls) {
      if (existingCall.host().equals(host)) return existingCall;
    }
    return null;
  }
```

可以看到这个方法就是遍历了一下当前正在执行的和准备执行的异步网络请求的call队列，看看有没有某一个`call`的`host`和当前`call`的`host`相同，如果有就返回。

然后：

```java
if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
```

判断了一下`findExistingCallWithHost（）`返回的是否为`null`，如果不为`null`，调用`call.reuseCallsPerHostFrom(existingCall)`：

```java
 void reuseCallsPerHostFrom(AsyncCall other) {
      this.callsPerHost = other.callsPerHost;
    }
```

就是对`call`对象的`callsPerHost`进行了更新，注意这里是直接使用了`=`对`this.callsPerHost`进行了赋值，而且在`java`中参数默认传递的是引用，所以当前`callsPerHost`和`findExistingCallWithHost()`中返回的那个`AsyncCall`对象的`callsPerHost`是同一个引用，那么再延伸一下，所有`host`相同的`AsyncCall`对象中的`callsPerHost`都是同一个引用，即如果改变其中一个`AsyncCall`对象的`callsPerHost`值，其他的所有`AsyncCall`对象的 `callsPerHost`的值也会随之改变，下面我们会看到作者巧妙地利用了这一点更新了所有`host`相同的 `AsyncCall`对象的callsPerHost值，实在是非常优秀。 这个`callPerHost`的声明如下：

```java
 private volatile AtomicInteger callsPerHost = new AtomicInteger(0);
```

即初始为0。

然后执行 `promoteAndExecute()`方法，我们看看其源码：

```java
private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
```

这个方法中遍历了`readyAsyncCalls`这个队列，对于`readyAsyncCalls`中的每一个被遍历的当前`AsyncCall`对象，会首先判断`runningAsyncCalls`这个队列的长度是否大于了`maxRequests`，这个`maxRequests`默认是64，意思就是说要求当前正在执行的网络请求不能超过64个(为了节约资源考虑)，如果`runningAsyncCalls`的元素数量不超过`maxRequests`，则判断`asyncCall.callsPerHost().get()`是否大于`maxRequestsPerHost`，`maxRequestsPerHost`的值默认为5，上面我们说到`callsPerHost`的初始为0，那么`asyncCall.callsPerHost().get()`初始应该是小于`maxRequestsPerHost`的，这里这个判断的意思就是当前正在执行的所有的请求中，与`asyncCall`对应的主机(`host`)相同请求的数量不能超过`maxRequestsPerHost`也就是5个，如果满足条件即同一个`host`的请求不超过5个，则往下执行13-16行，首先将当前`AsyncCall`对象从`readyAsyncCalls`中移除，然后执行`asyncCall.callsPerHost().incrementAndGet()`，就是将`callsPerHost`的值增1，上面我提到了，所有`host`相同的`AsyncCall`对象中的`callsPerHost`都是同一个引用，所以这里对当前这个`callsPerHost`的值增1实际上是更新了`readyAsyncCalls`中的所有`AsyncCall`对象中的`callsPerHost`的值，这样`callsPerHost`这个属性值就能够表示请求`host`与当前`host`相同的请求数量。

然后下面15行是将当前`asyncCall`对象加入到`executableCalls`中，下面会执行所有`executableCalls`中的请求，16行就是将当前这个`asyncCall`对象加入到`runningAsyncCalls`中表示其现在已经是正在执行了，注意这时`executableCalls`和`runningAsyncCalls`两个集合的不同。

然后下面第21行到24行，主要是对`executableCalls`进行了遍历，对于`executableCalls`中的每一个`AsyncCall`对象，执行`asyncCall.executeOn()`方法，传入了一个`executorService()`，我们首先看`executorService()`的源码：

```java
private @Nullable ExecutorService executorService;
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

可以看到这里是利用单例模式返回了一个线程池的对象，线程池的核心线程数为0，最大线程数为`Integer.MAX_VALUE`即不加限制，这里将线程池的最大线程数置位`Integer.MAX_VALUE`的原因是我们在`Dispatcher`中默认使用了`maxRequests`控制了同时并发的最大请求数量，所以这里就不用在线程池中加以控制了，然后设置了最大存活时间为60s，也就是说如果当前线程的任务执行完成了，60s内本线程不会被销毁，如果此时有其他网络请求的任务，就不用新建线程，可以直接复用之前的线程，如果60s后还没有被复用，则本线程会被销毁。

然后我们看`asyncCall.executeOn()`的源码：

```java
void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        transmitter.noMoreExchanges(ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }

```

重点是第5行，主要是调用线程池的`execute()`方法执行当前`asyncCall`，所以`AsyncCall`类应该实现了`Runnable`接口，我们看看`AsyncCall`类的声明：

```java
final class AsyncCall extends NamedRunnable
```

再看看`NamedRunnable`接口的源码：

```java
public abstract class NamedRunnable implements Runnable {
  protected final String name;

  public NamedRunnable(String format, Object... args) {
    this.name = Util.format(format, args);
  }

  @Override public final void run() {
    String oldName = Thread.currentThread().getName();
    Thread.currentThread().setName(name);
    try {
      execute();
    } finally {
      Thread.currentThread().setName(oldName);
    }
  }

  protected abstract void execute();
}

```

可以看到，`NamedRunnable`实现了`Runnable`接口，其`run（）`方法中主要是调用了抽象方法`execute()`，而`execute()`方法会被`AsyncCall`类实现，所以，`AsyncCall`类的`run()`方法实际上执行的是其`execute()`中的内容：

```java
@Override protected void execute() {
      boolean signalledCallback = false;
      transmitter.timeoutEnter();
      try {
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        responseCallback.onResponse(RealCall.this, response);
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
```

可以看到，`execute()`方法中发起了真正的网络请求，核心方法是`getResponseWithInterceptorChain()`，这个方法我们在解析`RealCall.excute()`方法时已经解释过，其作用是发起网络请求，返回`Response`，注意这里使用了`try...catch`语句块对异常进行了捕捉，如果发生异常，则调用`responseCallback.onFailure(RealCall.this, e）`，而这个`responseCallback`就是我们发起异步请求时传入的那个`CallBack`对象，所以就是在这里回调的`onFailure()`方法。如果请求成功，则调用`responseCallback.onResponse(RealCall.this, response)`即我们的`onResponse()`回调方法。那么由于当前`execute()`方法是在`Runnable`接口的`run()`方法中被调用的，而`asyncCall`又被传入了`executorService.execute()`中，所以当前`execute()`方法会在线程池中被执行，即`onFailure()`和`onResponse()`这两个回调方法会在子线程被调用，这也说明了我们不能再`RealCall.enqueue()`方法的回调中直接更新UI，因为其回调方法都是在子线程被调用的。

最后关注一下第16行的`finally`语句块中的内容，主要是执行了`client.dispatcher().finished(this)`，`this`指的是当前asyncCall对象，看看这个`Dispatcher.finished()`的源码：

```java
/** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    call.callsPerHost().decrementAndGet();
    finished(runningAsyncCalls, call);
  }
```

可以看到，首先执行 `call.callsPerHost().decrementAndGet()`将`asyncCall`对象的`callsPerHost`的值减1，因为当前`asyncCall`请求结束了，那么就应该将与本`asyncCall`具有相同`host`的请求数量减1，然后调用了：

```java
private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }

    boolean isRunning = promoteAndExecute();

    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
  }
```

主要做的工作就是将当前`call`对象从`runningAsyncCalls`中移除。

至此，我们分析完了`OkHttp`异步网络请求的整体流程，可以发现，在异步网络请求中`Dispatcher`类扮演了相当重要的角色。

总结一下`OkHttp`异步请求的步骤：

- 调用`call.enqueue(CallBack callback)`，传入`callback`回调
- 将当前`call`对象转化为`asyncCall`对象，调用`client.dispater.enqueue()`，调用`OkHttpClient`的`dispatcher`对象的`enqueue()`方法
- 将当前`asyncCall`对象加入`readyAsyncCalls`队列
- 遍历`readyAsyncCalls`，将符合条件的`asyncCall`对象移除并加入`executableCalls`和`runningAsyncCalls`集合
- 遍历`executableCalls`集合，执行每一个`asyncCall`对象的`executeOn()`方法，传入线程池
- 在线程池中发起当前`asyncCall`的网络请求
- 回调成功或失败对应的回调方法
- 将当前`asyncCall`对象从`runningAsyncCalls`中移除

到这里就分析完了OkHttp中同步请求和异步请求的执行流程，之后会推出OkHttp中内置的5大拦截器的源码分析，深入分析每一个拦截器的实现原理与作用，欢迎大家关注。