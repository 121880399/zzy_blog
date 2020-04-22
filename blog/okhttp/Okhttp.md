Okhttp 源码之旅

#前言
目前android最流行的网络请求库应该非Okhttp莫属了，我们都知道Okhttp使用起来很方便，而且很高效，那它是怎么做到的呢？它跟其他的网络请求库有什么区别呢？从这篇文章起，我们就一起来研究一下Okhttp的源码。

#使用方法
跟Volley不同，Okhttp有同步请求和异步请求两种。但是在使用上面区别并不是很大，我们先来看同步请求：
```
OkHttpClient client = new OkHttpClient.Builder().build();
Request request = new Request.Builder()
        .url("http://www.baidu.com")
        .build();
Call call=client.newCall(request);
Response response = call.execute();
```
异步请求：
```  
OkHttpClient okHttpClient = new OkHttpClient();
Request request = new Request.Builder()
                    .url("http://www.baidu.com")
                    .build();
Call call = okHttpClient.newCall(request);
call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                // TODO: 17-1-4  请求失败
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                // TODO: 17-1-4 请求成功
                //获得返回体
                ResponseBody body = response.body();
            }
        });
```
从上面代码我们可以看见，不论是同步请求还是异步请求，首先都是构造一个OkHttpClient对象，然后创建一个Request请求，通过OkHttpClient的newCall方法得到一个Call对象，同步请求调用Call对象中的execute方法，异步请求调用enqueue方法。所以在使用上来说，同步请求和异步请求的代码差别不大。在这里我们按照异步请求的方式去分析代码。

#OkHttpClient
OkHttpClient这个类，我每次看都觉得头大，因为里面有一堆的变量。其实这个类没有太复杂的逻辑，只是使用了Builder模式来初始化一些参数。我们在使用的时候第一步就是创建OkHttpClient这个对象，我们看关键代码：
```
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
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
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
```
以上是在Builder类中的代码。

#Request
既然是网络通信，那么肯定就有请求，对于Http协议来说，请求包含了：请求行，请求头，请求体。

![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1870458-821e06b68bc0dea5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从网上盗了一张Http请求的抓包图。上面对Http请求标注的很清楚。既然Http请求需要这些参数，那么我们Request类肯定要提供这些参数。
```
  final HttpUrl url;
  final String method;
  final Headers headers;
  final @Nullable RequestBody body;
  final Object tag;

  private volatile CacheControl cacheControl; // Lazily initialized.

```
以上是Request类的成员变量。
Request类也使用了Builder模式，我们在上面的使用中调用了url方法，给请求指定了请求地址，值得注意的是，如果没有指定Method，那么默认为Get方式请求。

#Call
接下来就是使用OkHttpClient中的newCall方法得到一个Call对象。
```
 /**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```
以上代码中实际是调用了RealCall中的newRealCall方法，我们继续跟踪进去。
```
 static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    call.eventListener = client.eventListenerFactory().create(call);
    return call;
  }
```
在代码中首先创建了一个RealCall对象。然后通过OkHttpClient得到了一个EventListener对象，最后返回这个RealCall。我们来看看这个EventListener是什么？
```
public abstract class EventListener {
  public static final EventListener NONE = new EventListener() {
  };

  static EventListener.Factory factory(final EventListener listener) {
    return new EventListener.Factory() {
      public EventListener create(Call call) {
        return listener;
      }
    };
  }
public void callStart(Call call) {
  }
 public void dnsStart(Call call, String domainName) {
  }
  public void dnsEnd(Call call, String domainName, @Nullable List<InetAddress> inetAddressList) {
  }
 public void connectStart(Call call, InetSocketAddress inetSocketAddress, Proxy proxy) {
  }
public void secureConnectStart(Call call) {
  }
 public void secureConnectEnd(Call call, @Nullable Handshake handshake) {
  }
public void connectEnd(Call call, InetSocketAddress inetSocketAddress,
      @Nullable Proxy proxy, @Nullable Protocol protocol) {
  }
 public void connectFailed(Call call, InetSocketAddress inetSocketAddress,
      @Nullable Proxy proxy, @Nullable Protocol protocol, @Nullable IOException ioe) {
  }
 public void connectionAcquired(Call call, Connection connection) {
  }
 public void connectionReleased(Call call, Connection connection) {
  }
public void requestHeadersStart(Call call) {
  }
public void requestHeadersEnd(Call call, Request request) {
  }
 public void requestBodyStart(Call call) {
  }
public void requestBodyEnd(Call call, long byteCount) {
  }
 public void responseHeadersStart(Call call) {
  }
 public void responseHeadersEnd(Call call, Response response) {
  }
public void responseBodyStart(Call call) {
  }
public void responseBodyEnd(Call call, long byteCount) {
  }
public void callEnd(Call call) {
  }
public void callFailed(Call call, IOException ioe) {
  }
public interface Factory {
    /**
     * Creates an instance of the {@link EventListener} for a particular {@link Call}. The returned
     * {@link EventListener} instance will be used during the lifecycle of the {@code call}.
     *
     * <p>This method is invoked after the {@code call} is created. See
     * {@link OkHttpClient#newCall(Request)}.
     *
     * <p><strong>It is an error for implementations to issue any mutating operations on the
     * {@code call} instance from this method.</strong>
     */
    EventListener create(Call call);
  }
}
```
可以看到EventListener是一个抽象类，其中定义了很多开始结束的空方法。所以我们可以知道，这个类的作用是在请求执行到每个步骤时提供的接口回调。

得到RealCall以后，我们又调用了enqueue方法。
```
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
以上代码中，首先判断这个Call是否执行过？如果执行了就抛出IllegalStateException异常，executed是一个boolean类型的变量，默认为false，如果没有执行过，那么把executed标记为true。
接着调用EventListener中的callStart方法，通知Call开始。这么多说一句这个EventListener要怎么用，我们可以直接继承EventListener这个抽象类，重写其中的方法，然后在OkHttpClient创建的时候使用Builder模式调用eventListener方法，把自己创建好的EventListener子类传入，这样在执行请求的关键步骤的时候，就会回调我们重写的方法。
接着使用OkHttpClient中的dispatcher（）方法得到一个Dispatcher对象，调用Dispatcher对象的enqueue方法，并创建一个AsyncCall传入。
我们先来看看AsyncCall这个类:
```
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }

    String host() {
      return originalRequest.url().host();
    }

    Request request() {
      return originalRequest;
    }

    RealCall get() {
      return RealCall.this;
    }

    /**
     * 线程会执行该方法
     * */
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        //从这里开始执行拦截链
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```
AsyncCall是RealCall中的一个内部类，继承与NameRunnale类
```
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
```
而NamedRunnable类实现了Runnbale接口，并在run方法中对当前线程进行名字的修改，再调用execute方法。AsyncCall继承了NameRunnable类并且实现了execute方法，关于这个方法的代码，我们待会再详细看。AsyncCall这个类中有一个Callback类型的成员变量，这个CallBack是我们在一开始使用call.enqueue方法时传入的。
了解了AsyncCall这个类，我们接着看Dispatcher中的enqueue方法。
```
  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```
Dispatcher这个类在我们创建OkHttpClient的时候就在Builder构造方法中创建好了，还记得那一大堆变量赋值么？在Dispatcher这个类中初始化了一些成员变量。
```
 private int maxRequests = 64;
  private int maxRequestsPerHost = 5;

  /** Executes calls. Created lazily. */
  private @Nullable ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```
maxRequest定义了最大请求数量64
maxRequestsPerHost定义了每个host最大请求数量5
executorService是一个线程池
readyAsyncCalls用于存放那些准备被执行的AsyncCall
runningAsyncCalls用于存放那些已经在运行当中的AsyncCall
runningSyncCalls用于存放那些正在运行的RealCall

知道这些以后我们再来看Dispatcher中的enqueue方法，首先需要判断当前正在运行的AsyncCall是否小于最大请求数量？并且当前的AsyncCall在已经运行的AsyncCall队列中的数量是否小于host的最大请求数量5？如果都不超过限制，那么把这个AsyncCall加入到正在运行的AsyncCall队列中，并且调用executorService方法。
```
 public synchronized ExecutorService executorService() {
    if (executorService == null) {
      //核心线程数为0，非核心线程数为int的最大值，非核心线程数的闲置时间为60秒，如果超过这个时间就被回收。
      //SynchronousQueue保证任务马上被执行
      //也就是说该线程池不存在核心线程，只要有任务进来就开启一个非核心线程立即执行任务，处理完任务线程闲置60秒，还没有
      //任务进来就会被销毁。
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
executorService方法很简单，如果executorService为null，那么就创建一个线程池并返回。返回后调用executorService的execute方法，把AsyncCall传入进去，也就是从线程池中调用一根线程去执行AsyncCall。那么接下来就会执行AysncCall中的execute方法。
如果正在运行的AsyncCall的数量已经超过最大请求数量，或者当前AsyncCall所请求的host数量已经超过5个，那么就把当前AsyncCall加入到准备执行的AsyncCall队列。
我们先假设为true的情况，也就是能够马上执行。来看AsyncCall的execute方法。
```
@Override protected void execute() {
      boolean signalledCallback = false;
      try {
        //从这里开始执行拦截链
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```
这个方法的逻辑很简单，首先通过getResponseWithInterceptorChain方法会得到返回的Response，怎么得到的，我们一会再看。接着判断重定向拦截器是否取消，如果取消那么就回调失败的接口。如果没有取消就调用成功的接口。如果有异常就判断是否要给用户返回信号，如果不返回就直接打印LOG日志，如果返回的话就回调失败的接口。最后调用Dispatcher的finished方法来把当前的AsyncCall从正在运行的队列中移除，并且将准备运行队列中的AsyncCall加入一定数量到正在运行队列并运行。
接下来我们就看最重要的getResponseWithInterceptorChain方法，前方核能，请抓好安全带。
```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    //得到OkHttpClient.Builder中添加的自定义interceptor
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
            interceptors,//拦截器列表
            null,//StreamAllocation对象
            null,//HttpCodec对象
            null,//RealConnection对象
            0,//index标识当前取第几个拦截器
        originalRequest,//Request对象
            this,//Call对象
            eventListener,//EventListener对象，监听当前的通信进度
            client.connectTimeoutMillis(),//连接超时时间
        client.readTimeoutMillis(),//读超时
            client.writeTimeoutMillis()//写超时
    );

    return chain.proceed(originalRequest);
  }
```
这个方法可以说是OkHttp的精髓所在，使用了拦截链的设计模式来发送请求得到响应。如果对拦截链设计模式不清楚的童鞋可以自己去百度找一下资料，这里就不详细说了。
在以上代码中首先创建了一个ArrayList的链表，里面用于存放所有的拦截器，接着就把自定义拦截器，重定向拦截器，BridgeInterceptor拦截器，CacheInterceptor拦截器，ConnectInterceptor拦截器，CallServerInterceptor拦截器加进链表。
然后创建了一个拦截链RealInterceptorChain，执行了RealInterceptorChain的proceed方法。
```
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.创建下一个拦截链
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
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
以上代码中大部分都是一些错误判断，我们来看主要代码
```
    // Call the next interceptor in the chain.创建下一个拦截链
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```
在proceed方法里又重新创建了一个RealInterceptorChain，这次创建的RealInterceptorChain和在getResponseWithInterceptorChain方法中创建的RealInterceptorChain不同的地方在于，index的值加了1。接着又从拦截器链表中取到第0个拦截器并执行它的intercept方法。如果没有添加自定义拦截器的话，目前获取到的拦截器应该是重定向拦截器，我们看看它的intercept方法。
```
@Override public Response intercept(Chain chain) throws IOException {
    //这里就是我们自己创建的Request对象，包括URL之类的
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    //创建一个StreamAllocation对象
    streamAllocation = new StreamAllocation(
            client.connectionPool()//连接池
            , createAddress(request.url())//Address
            , call//call
            , eventListener//事件监听者
            , callStackTrace);//Object

    //用于记录重定向数
    int followUpCount = 0;
    Response priorResponse = null;
    //无限循环，除非return
    while (true) {
      //如果取消
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {

        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;

      } catch (RouteException e) {

        // The attempt to connect via a route failed. The request will not have been sent.
        if (!recover(e.getLastConnectException(), false, request)) {
          throw e.getLastConnectException();
        }

        releaseConnection = false;
        continue;

      } catch (IOException e) {

        // An attempt to communicate with a server failed. The request may have been sent.
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;

      } finally {
        //只有在proceed的时候发生了为止的异常，releaseConnection才会为true
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
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

      //进行重定向
      Request followUp = followUpRequest(response);

      //为空说明不需要重定向，则释放流并且返回response
      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
        return response;
      }

      closeQuietly(response.body());

      //对重定向次数加1，如果大于最大重定向次数则关闭流
      if (++followUpCount > MAX_FOLLOW_UPS) {
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      //如果请求体是不可重复请求体，则关闭流
      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      //如果上一次请求跟重定向后的请求不能复用同一个连接，则释放掉流并且重新创建一个新流
      if (!sameConnection(response, followUp.url())) {

        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);

      } else if (streamAllocation.codec() != null) {

        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");

      }

      request = followUp;
      priorResponse = response;
    }
  }
```
大部分的代码我都有注释，现在捡重点看看。
首先在这个方法里创建了一个StreamAllocation对象。
```
   //创建一个StreamAllocation对象
    streamAllocation = new StreamAllocation(
            client.connectionPool()//连接池
            , createAddress(request.url())//Address
            , call//call
            , eventListener//事件监听者
            , callStackTrace);//Object
```
接着会调用RealInterceptorChain类的proceed方法，把request和streamAllocation传入。我们接着跟进proceed方法。
```
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.创建下一个拦截链
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
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
是不是感觉很眼熟？因为开始我们已经讲过这个方法了，在这个方法中又会创建一个新的RealInterceptorChain对象，但是这次的RealInterceptorChain对象的streamAllocation值不为null了，我们第一次进入的时候该对象的值为null。接着又取了下一个拦截器BridgeInterceptor，执行它的intercept方法。
```
@Override public Response intercept(Chain chain) throws IOException {
    //得到一个用户请求
    Request userRequest = chain.request();

    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();

    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        //将RequestBody中设置的contentType设置到请求头中
        requestBuilder.header("Content-Type", contentType.toString());
      }

      //返回内容的长度
      long contentLength = body.contentLength();
      //如果内容长度不为-1，则添加Content-Length头，并删除Transfer-Encoding头
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        //如果内容长度为-1，则添加Transfer-Encoding头，删除Content-Length头
        //表示输出的内容长度不能确定
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    //从用户请求中得到Host请求头，如果为null则设置Host请求头
    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    ///从用户请求中得到Connection请求头，如果为Null则设置Connection请求头
    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    //是否转换为Gzip传输
    boolean transparentGzip = false;
    //如果用户没有设置Accept-Encoding，也没有设置Range头，则采用Giz压缩
    // Accept-Encoding:浏览器申明自己接收的编码方法，通常指定压缩方法，是否支持压缩，支持什么压缩
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    //根据用户请求中的Url返回一个cookie列表
    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    //如果cookie不为空，则把列表中的cookie转话为字符串，存入请求头
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    //如果User-Agent为null,则设置为"okhttp/${project.version}"
    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }
    //以上代码是把用户的请求转换为网络请求，是请求之前的准备

    //关键
    Response networkResponse = chain.proceed(requestBuilder.build());

    //以下代码是把网络返回的响应进行处理，返回
    //解析响应头，对cookie进行处理
    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    //将用户的请求跟响应关联在一起
    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
        //giz的处理
        GzipSource responseBody = new GzipSource(networkResponse.body().source());

        //从响应头中删除Content-Encoding和Content-Length两个字段
        Headers strippedHeaders = networkResponse.headers().newBuilder()
            .removeAll("Content-Encoding")
            .removeAll("Content-Length")
            .build();
        //删除后的头重新存入Response中
        responseBuilder.headers(strippedHeaders);

        String contentType = networkResponse.header("Content-Type");

        //将响应体重新包装然后存入response中
        responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
    }
    //将包装后的response返回
    return responseBuilder.build();
  }

```
BridgeInterceptor拦截器是桥接应用层的代码和网络层代码，在这个方法里主要把用户请求转换为真正的Http请求，设置请求头之类的。代码我基本都有加注释，由于是拦截器模式，所以我们可以预计肯定会执行RealInterceptorChain的proceed方法，并且再次创建一个新的RealInterceptorChain，然后得到下一个拦截器CacheInterceptor，执行它的intercept方法。
```
@Override public Response intercept(Chain chain) throws IOException {
    //如果内部缓存不为null,则试着把用户请求传入，看是否存在缓存
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    //得到当前的时间
    long now = System.currentTimeMillis();


    //通过工厂得到缓存策略
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();


    //针对这两个对象，进行判断，做出响应操作
    //newnetworkRequest==null,cacheResponse==null :only-if-cached 不进行网络请求，如果缓存不存在或者过期返回504
    //newnetworkRequest==null,cacheResponse!=null :不进行网络请求，缓存可用，直接返回缓存
    //newnetworkRequest!=null,cacheResponse==null :需要进行网络请求，缓存不存在或者过期，直接访问网络
    //newnetworkRequest!=null,cacheResponse!=null :Header中含有ETag/Last-Modified标签，
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      //跟踪一个Response看是否满足缓存策略
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    //如果使用网络被禁止，也没有缓存，则返回一个Response，代码为504网关超时
    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // If we don't need the network, we're done.
    //如果不需要使用网络，就是用CacheResponse构建一个Response并且把CacheRespone
    //去除body后关联到里面
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    //声明一网络响应
    Response networkResponse = null;

    try {
      //能执行到这一步说明networkRequest不为null,也就是说需要网络请求
      //以上代码是执行网络请求之前的操作,判断是否有缓存，是否需要网络请求
      networkResponse = chain.proceed(networkRequest);

    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }
    //以下代码是请求返回后需要执行的操作

    // If we have a cache response too, then we're doing a conditional get.
    //如果缓存中存在Response，并且网络请求返回的是304，则构建一个response返回
    if (cacheResponse != null) {
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    //如果缓存中不存在Response，通过网络返回的Response构建一个response返回
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      //如果响应有响应体，并且可以缓存，则缓存响应
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```
这个方法主要是判断是否存在缓存，是否使用缓存，如果不使用那么继续请求网络。我们先看不需要缓存的情况，在不需要缓存的情况下，就会继续执行RealInterceptorChain的proceed方法，并且再次创建一个新的ConnectInterceptor拦截器，执行intercept方法。
```

  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
这个方法的代码很短，但是很重要，在代码中通过streamAllocation的newStream方法得到了一个HttpCodec对象，我们跟进这个方法。
```
public HttpCodec newStream(OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```
可以看见在这个方法中先是得到了连接超时时间，读超时时间，写超时时间。然后通过findHealthyConnection方法把这些参数传入，构建出了一个RealConnection对象，我们继续跟进去。
```
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate;
    }
  }
```
这里面得到的RealConnection对象也是通过findConnection方法得到的，我们继续跟进。
```
 /**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        // We had an already-allocated connection and it's good.
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      if (result == null) {
        // Attempt to get a connection from the pool.
        Internal.instance.get(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.get(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(
        connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;

      // Pool the connection.
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```
在这个方法中我们会得到一个RealConnection对象，在Okhttp3中有一套连接池机制，首先会在连接池中寻找可用连接，找不到时才会创建新的连接，创建新的连接以后会调用RealConnection的connect方法。
```
public void connect(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled, Call call, EventListener eventListener) {
    if (protocol != null) throw new IllegalStateException("already connected");

    RouteException routeException = null;
    List<ConnectionSpec> connectionSpecs = route.address().connectionSpecs();
    ConnectionSpecSelector connectionSpecSelector = new ConnectionSpecSelector(connectionSpecs);

    if (route.address().sslSocketFactory() == null) {
      if (!connectionSpecs.contains(ConnectionSpec.CLEARTEXT)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication not enabled for client"));
      }
      String host = route.address().url().host();
      if (!Platform.get().isCleartextTrafficPermitted(host)) {
        throw new RouteException(new UnknownServiceException(
            "CLEARTEXT communication to " + host + " not permitted by network security policy"));
      }
    }

    while (true) {
      try {
        if (route.requiresTunnel()) {
          connectTunnel(connectTimeout, readTimeout, writeTimeout, call, eventListener);
          if (rawSocket == null) {
            // We were unable to connect the tunnel but properly closed down our resources.
            break;
          }
        } else {
          connectSocket(connectTimeout, readTimeout, call, eventListener);
        }
        establishProtocol(connectionSpecSelector, call, eventListener);
        eventListener.connectEnd(call, route.socketAddress(), route.proxy(), protocol);
        break;
      } catch (IOException e) {
        closeQuietly(socket);
        closeQuietly(rawSocket);
        socket = null;
        rawSocket = null;
        source = null;
        sink = null;
        handshake = null;
        protocol = null;
        http2Connection = null;

        eventListener.connectFailed(call, route.socketAddress(), route.proxy(), null, e);

        if (routeException == null) {
          routeException = new RouteException(e);
        } else {
          routeException.addConnectException(e);
        }

        if (!connectionRetryEnabled || !connectionSpecSelector.connectionFailed(e)) {
          throw routeException;
        }
      }
    }

    if (route.requiresTunnel() && rawSocket == null) {
      ProtocolException exception = new ProtocolException("Too many tunnel connections attempted: "
          + MAX_TUNNEL_ATTEMPTS);
      throw new RouteException(exception);
    }

    if (http2Connection != null) {
      synchronized (connectionPool) {
        allocationLimit = http2Connection.maxConcurrentStreams();
      }
    }
  }
```
在这个方法里面会判断是否建立隧道连接，如果是隧道连接就执行connectTunnel方法，否则执行connectSocket方法。但是在connectTunnel方法中也调用了connectSocket方法，所以我们直接来看这个方法。
```
/** Does all the work necessary to build a full HTTP or HTTPS connection on a raw socket. */
  private void connectSocket(int connectTimeout, int readTimeout, Call call,
      EventListener eventListener) throws IOException {
    Proxy proxy = route.proxy();
    Address address = route.address();

    rawSocket = proxy.type() == Proxy.Type.DIRECT || proxy.type() == Proxy.Type.HTTP
        ? address.socketFactory().createSocket()
        : new Socket(proxy);

    eventListener.connectStart(call, route.socketAddress(), proxy);
    rawSocket.setSoTimeout(readTimeout);
    try {
      //建立socket连接
      Platform.get().connectSocket(rawSocket, route.socketAddress(), connectTimeout);
    } catch (ConnectException e) {
      ConnectException ce = new ConnectException("Failed to connect to " + route.socketAddress());
      ce.initCause(e);
      throw ce;
    }

    // The following try/catch block is a pseudo hacky way to get around a crash on Android 7.0
    // More details:
    // https://github.com/square/okhttp/issues/3245
    // https://android-review.googlesource.com/#/c/271775/
    try {
      source = Okio.buffer(Okio.source(rawSocket));
      sink = Okio.buffer(Okio.sink(rawSocket));
    } catch (NullPointerException npe) {
      if (NPE_THROW_WITH_NULL.equals(npe.getMessage())) {
        throw new IOException(npe);
      }
    }
  }
```
在这个方法里面首先创建了一个Socket对象，创建的时候要判断是采用直接连接，还是使用了代理？直接连接就使用SokcetFactorty来创建Socket对象，代理连接就手动new一个Socket对象。接着就得到当前客户端使用的平台信息，通过该信息建立Socket连接。
```
public void connectSocket(Socket socket, InetSocketAddress address,
      int connectTimeout) throws IOException {
    socket.connect(address, connectTimeout);
  }
```
连接建立成功以后，就需要给对方发送数据和接收对方数据，在Okhttp3中使用了Okio这个包。关于Okio如何使用大家自己去百度，这里不做介绍。在该方法中就初始化了用于写数据和读数据的Source和Sink对象。
```
   source = Okio.buffer(Okio.source(rawSocket));
      sink = Okio.buffer(Okio.sink(rawSocket));
```
所以这个方法很重要，真正的建立Socket连接和读写数据对象的创建都在这里。

在得到RealConnection对象以后，我们就要调用它的newCodec方法来得到一个HttpCodec对象。
```
  public HttpCodec newCodec(OkHttpClient client, Interceptor.Chain chain,
      StreamAllocation streamAllocation) throws SocketException {
    if (http2Connection != null) {
      return new Http2Codec(client, chain, streamAllocation, http2Connection);
    } else {
      socket.setSoTimeout(chain.readTimeoutMillis());
      source.timeout().timeout(chain.readTimeoutMillis(), MILLISECONDS);
      sink.timeout().timeout(chain.writeTimeoutMillis(), MILLISECONDS);
      return new Http1Codec(client, streamAllocation, source, sink);
    }
  }
```
这里如果Http2Connection不为空那么返回Http2Codec，如果为空就返回Http1Codec。
跟踪到这里，我们就可以返回到ConnectInterceptor的intercept方法中了。
```
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
在ConnectInterceptor的intercept方法中我们又得到了HttpCodec，RealConnection对象，这时候再调用realChain.proceed方法，把新得到的这两个对象传下去。在processd方法中又一次创建新的RealInterceptorChain对象，并且得到下一个拦截器，执行它的intercept方法。

CallServerInterceptor:
```
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    realChain.eventListener()
        .responseHeadersEnd(realChain.call(), response);

    int code = response.code();
    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```
在这个方法里面首先利用HttpCodec写请求头，我们这么假定使用的是Http1Codec。我们来看看它的writeRequestHeaders方法。
```
@Override public void writeRequestHeaders(Request request) throws IOException {
    String requestLine = RequestLine.get(
        request, streamAllocation.connection().route().proxy().type());
    writeRequest(request.headers(), requestLine);
  }
```
首先使用RequestLine中的get方法将Request对象中封装的method，url,还有http协议版本转换成一个String对象。
```
public static String get(Request request, Proxy.Type proxyType) {
    StringBuilder result = new StringBuilder();
    result.append(request.method());
    result.append(' ');

    if (includeAuthorityInRequestLine(request, proxyType)) {
      result.append(request.url());
    } else {
      result.append(requestPath(request.url()));
    }

    result.append(" HTTP/1.1");
    return result.toString();
  }
```
接着开始写请求行和请求头。
```
 /** Returns bytes of a request header for sending on an HTTP transport. */
  public void writeRequest(Headers headers, String requestLine) throws IOException {
    if (state != STATE_IDLE) throw new IllegalStateException("state: " + state);
    sink.writeUtf8(requestLine).writeUtf8("\r\n");
    for (int i = 0, size = headers.size(); i < size; i++) {
      sink.writeUtf8(headers.name(i))
          .writeUtf8(": ")
          .writeUtf8(headers.value(i))
          .writeUtf8("\r\n");
    }
    sink.writeUtf8("\r\n");
    state = STATE_OPEN_REQUEST_BODY;
  }
```
我们可以看见这里使用的Sink对象，就是前面connectSocket方法中创建的。

接着我们判断请求头中Expect是否为100-continue，如果是的话，那么需要分两步走：
1.发送一个请求，询问Server是否愿意接受数据。
2.接受到Server返回的100-continue回应后，才把数据POST到Server。
接着使用request.body().writeTo(bufferedRequestBody)方法，把body写入服务器。之后接着读响应头，构建Response对象并且返回。

返回以后会回到ConnectInterceptor中接着执行剩下的代码，执行完以后又接着执行CacheInterceptor中未执行的代码，一直递归上去。

![InterceptorChain.png](http://upload-images.jianshu.io/upload_images/1870458-9133ba58c57ee5f5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里画了一副简单的流程图帮助大家理解。一直递归到RealCall的execute方法中，并且回调onResponse方法，这样一次完整的Http请求就完成了。

#后记
本文只是根据OkHttp3的简单使用流程进行了跟踪，Okhttp3的源码博大精深，很多值得我们学习的地方，后续还会有相关的文章对OkHttp3进行不同角度的分析。

如果你觉得本篇文章帮助到了你，希望大爷能够给瓶买水钱。
本文为原创文章，转载请注明出处！