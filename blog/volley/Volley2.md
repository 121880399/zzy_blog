Volley 都干了些什么？

#目录
 (一) [CacheDispatcher的解析](http://www.jianshu.com/p/85bcd878a676)
 (二) [NetworkDispatcher的解析](http://www.jianshu.com/p/0a0dc214c2c0)
 (三) [RequestQueue中add方法的分析](http://www.jianshu.com/p/4dbc76e3f548)
 (四) [ImageRequest的分析](http://www.jianshu.com/p/d6609049325d)
 (五) [ImageLoader的分析](http://www.jianshu.com/p/cb6ba58f9f3e)
#前言
 本文着重于Volley源代码的解析，关于如何使用请参考其他文章。
 Volley的整体架构请参考[此文](http://a.codekk.com/detail/Android/grumoon/Volley%20源码解析)。
#从一个例子开始
  我们平时使用Volley最简单的方式如下：
  1.首先得到一个RequestQueue
   ``` 
  RequestQueue mQueue=Volley.newRequestQueue(context);
   ```
  2.创建一个StringRequest
```
StringRequest strRequest = new StringRequest("http://www.baidu.com",  
                        new Response.Listener<String>() {  
                            @Override  
                            public void onResponse(String response) {  
                                Log.d("TAG", response);  
                            }  
                        }, new Response.ErrorListener() {  
                            @Override  
                            public void onErrorResponse(VolleyError error) {  
                                Log.e("TAG", error.getMessage(), error);  
                            }  
                        }); 
```
  3.把StringRequest加入到RequestQueue当中
```mQueue.add(strRequest); ```
  没错！就是这么简单，仅仅3步就能发起一个请求。接下来我们一起来看看，这3步背后，Volley都干了些什么？
#源码分析
  那我们就按照上面的代码一步一步跟踪源码，首先Volley.newRequestQueue(context);    
  ```
  public static RequestQueue newRequestQueue(Context context)
  {    
    return  newRequestQueue(context, null);
  }
  ```
从以上代码我们可以看到，我们调用newRequestQueue(context)后，在其内部会调用其他的重载方法。
```
 public static RequestQueue newRequestQueue(Context context, HttpStack stack)
    {
        return newRequestQueue(context, stack, -1);
    }

 public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes)
    {
        File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);

        String userAgent = "volley/0";
        try
        {
            String packageName = context.getPackageName();
            PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
            userAgent = packageName + "/" + info.versionCode;
        } catch (NameNotFoundException e)
        {
        }

        //如果HttpStack为空则创建
        if (stack == null)
        {
            //判断当前SDK的版本，如果大于9则使用HttpUrlConnection,小于9就使用HttpClient
            if (Build.VERSION.SDK_INT >= 9)
            {
                stack = new HurlStack();
            } else
            {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }
        //使用HttpStack构建一个代表网络的具体实现类BasicNetwork
        Network network = new BasicNetwork(stack);

        RequestQueue queue;
        //使用上面构建的network构建一个请求队列，如果maxDiskCacheByteds小于等于-1，说明是默认的硬盘缓存大小
        if (maxDiskCacheBytes <= -1)
        {
            // No maximum size specified
            queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        } else
        {
            // Disk cache size specified
            queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }

        queue.start();//启动队列

        return queue;
    }

```
我们主要看
public static RequestQueue newRequestQueue(Context context, HttpStack stack, int maxDiskCacheBytes)
由于我们没有传入HttpStack，所以会在其内部帮我们创建一个HttpStack对象，在SDK版本大于等于9的时候会为我们创建一个基于HttpUrlConnection的HurlStack，在SDK版本小于9时会为我们创建一个基于HttpClient的HttpClientStack。
```
//如果HttpStack为空则创建
        if (stack == null)
        {
            //判断当前SDK的版本，如果大于9则使用HttpUrlConnection,小于9就使用HttpClient
            if (Build.VERSION.SDK_INT >= 9)
            {
                stack = new HurlStack();
            } else
            {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
            }
        }
```
接下来会使用刚刚创建好的stack来构建一个Network。
```
 Network network = new BasicNetwork(stack);
```
Network是一个接口，其中只有一个方法：
```
public interface Network {
    /**
     * Performs the specified request.
     * 用于执行特定请求。
     * @param request Request to process
     * @return A {@link NetworkResponse} with data and caching metadata; will never be null
     * @throws VolleyError on errors
     */
    public NetworkResponse performRequest(Request<?> request) throws VolleyError;
}
```
到目前为止我们只需要知道使用Network接口中的performRquest方法可以执行一个请求就行。
接着我们就该创建RquestQueue了，由于我们也没有指定maxDiskCacheBytes的大小，所以默认为-1。
执行 queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
```
       RequestQueue queue;
        //使用上面构建的network构建一个请求队列，如果maxDiskCacheByteds小于等于-1，说明是默认的硬盘缓存大小
        if (maxDiskCacheBytes <= -1)
        {
            // No maximum size specified
            queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
        } else
        {
            // Disk cache size specified
            queue = new RequestQueue(new DiskBasedCache(cacheDir, maxDiskCacheBytes), network);
        }
```
我们看看RequestQueue构造方法中都做了些什么？
```
public RequestQueue(Cache cache, Network network)
    {
        this(cache, network, DEFAULT_NETWORK_THREAD_POOL_SIZE);
    }
```
在这个方法内部调用了3个参数的构造方法，其中DEFAULT_NETWORK_THREAD_POOL_SIZE值为4；
```
public RequestQueue(Cache cache, Network network, int threadPoolSize)
    {
        this(cache, network, threadPoolSize, new ExecutorDelivery(new Handler(Looper.getMainLooper())));
    }
```
在这个方法内部又调用了4个参数的构造方法，并传入了一个ExecutorDelivery对象，ExecutorDelivery实现了ResponseDelivery。ResponseDelivery是什么鬼？我们就暂时把它理解为用来传递请求结果的接口。
```
 public RequestQueue(Cache cache, Network network, int threadPoolSize, ResponseDelivery delivery)
    {
        mCache = cache;
        mNetwork = network;
        mDispatchers = new NetworkDispatcher[threadPoolSize];
        mDelivery = delivery;
    }
```
这个方法内部终于没有再调用其他构造方法了，而是对RequestQueue中的一些成员变量进行了初始化。
```
  //cacheDir的地址为/data/data/项目包名/cache
 Cache mCache=new DiskBasedCache(cacheDir);
 //stack根据SDK的版本分别采用不同的实现方式
 NetWork mNetwork=new BasicNetwork(stack);
//threadPoolSize的值为4
 NetworkDispatcher[] mDispatchers=new NetworkDispatcher[threadPoolSize];
 ResponseDelivery mDispatchers =new ExecutorDelivery(new Handler(Looper.getMainLooper()));
```
为了方便理解，我把构造方法里的成员变量初始化写全了。以上并不是源代码，而是我为了大家方便理解而补全的代码。
NetworkDispatcher是网络调度器，它继承了Thread类，没错，也就是初始化了4条用于网络请求的线程。
DiskBasedCache是实际调用缓存的类。
构造完RequestQueue后，我们接着要执行开启请求队列的操作了。
```
queue.start();
```
以下是RquestQueue类中的start方法
```
/**
     * Starts the dispatchers in this queue.
     * 开始队列
     */
    public void start()
    {
        stop();  // Make sure any currently running dispatchers are stopped.开始前先停止
        // Create the cache dispatcher and start it.创建一个缓存调度线程并开启他，CacheDispatcher是一个线程
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        //mDispatchers.length默认为4
        for (int i = 0; i < mDispatchers.length; i++)
        {
            //创建网络调度线程并开启
            NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork, mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```
首先我们可以看到，方法在一开始就调用了stop方法。这是确保在开始之前所有的线程都处于停止状态。
```
 /**
     * Stops the cache and network dispatchers.
     * 停止缓存调度线程和网络调度线程
     */
    public void stop()
    {
        //如果缓存调度器不为空
        if (mCacheDispatcher != null)
        {
            mCacheDispatcher.quit();//退出缓存调度器
        }
        //把所有的网络调度器都退出，默认为4个网络调度器
        for (int i = 0; i < mDispatchers.length; i++)
        {
            if (mDispatchers[i] != null)
            {
                mDispatchers[i].quit();
            }
        }
    }
```
stop方法很简单，就是停止所有的缓存线程和网络线程。在这里我们不深入的去追究是如何停止的。
接着我们初始化了缓存调度器，我们来看看它的构造方法：
```
    public CacheDispatcher(BlockingQueue<Request<?>> cacheQueue, BlockingQueue<Request<?>> networkQueue, Cache cache, ResponseDelivery delivery)
    {
        mCacheQueue = cacheQueue;
        mNetworkQueue = networkQueue;
        mCache = cache;
        mDelivery = delivery;
    }
```
这里讲RequestQueue中的mCacheQueue, mNetworkQueue, mCache, mDelivery传入来初始化CacheDispatcher中的成员变量。我们来解释一下这4个参数。
PriorityBlockingQueue<Request<?>> mCacheQueue = new PriorityBlockingQueue<Request<?>>();
PriorityBlockingQueue<Request<?>> mNetworkQueue = new PriorityBlockingQueue<Request<?>>();
mCacheQueue和mNetworkQueue是两个优先级阻塞队列。用来分别存放缓存请求和网络请求的。
mCache刚刚在传入RuestQueue的构造方法时已经解释过了。
mDelivery是用于传递请求结果的类。
构建完CacheDispatcher对象以后，我们就要使用mCacheDispatcher.start();启动该线程了。
```
 @Override
    public void run()
    {
        if (DEBUG)
            VolleyLog.v("start new dispatcher");
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // Make a blocking call to initialize the cache.
        mCache.initialize();

        Request<?> request;
        while (true)
        {
            // release previous request object to avoid leaking request object when mQueue is drained.
            request = null;
            try
            {
                // Take a request from the queue.
                request = mCacheQueue.take();//从缓存队列中得到一个请求
            } catch (InterruptedException e)
            {
                // We may have been interrupted because it was time to quit.
                if (mQuit)
                {
                    return;
                }
                continue;
            }
            try
            {
                request.addMarker("cache-queue-take");

                // If the request has been canceled, don't bother dispatching it.
                if (request.isCanceled())
                {
                    request.finish("cache-discard-canceled");
                    continue;
                }

                // Attempt to retrieve this item from cache.
                Cache.Entry entry = mCache.get(request.getCacheKey());
                //如果缓存没有命中，就把请求加入到网络队列中去调度处理
                if (entry == null)
                {
                    request.addMarker("cache-miss");
                    // Cache miss; send off to the network dispatcher.
                    mNetworkQueue.put(request);
                    continue;
                }

                // If it is completely expired, just send it to the network.
                //如果请求已经过期的，就要把请求加入到网络队列中去调度处理
                if (entry.isExpired())
                {
                    request.addMarker("cache-hit-expired");
                    request.setCacheEntry(entry);
                    mNetworkQueue.put(request);
                    continue;
                }

                // We have a cache hit; parse its data for delivery back to the request.
                request.addMarker("cache-hit");
                //将缓存结果解析为Response
                Response<?> response = request.parseNetworkResponse(new NetworkResponse(entry.data, entry.responseHeaders));
                request.addMarker("cache-hit-parsed");
                //数据是否要更新，如果不更新则传递
                if (!entry.refreshNeeded())
                {
                    // Completely unexpired cache hit. Just deliver the response.
                    mDelivery.postResponse(request, response);
                } else
                {
                    // Soft-expired cache hit. We can deliver the cached response,
                    // but we need to also send the request to the network for
                    // refreshing.
                    request.addMarker("cache-hit-refresh-needed");
                    request.setCacheEntry(entry);

                    // Mark the response as intermediate.
                    response.intermediate = true;

                    // Post the intermediate response back to the user and have
                    // the delivery then forward the request along to the network.
                    final Request<?> finalRequest = request;
                    mDelivery.postResponse(request, response, new Runnable()
                    {
                        @Override
                        public void run()
                        {
                            try
                            {
                                mNetworkQueue.put(finalRequest);
                            } catch (InterruptedException e)
                            {
                                // Not much we can do about this.
                            }
                        }
                    });
                }
            } catch (Exception e)
            {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
            }
        }
    }
```
以上是CacheDispatcher中的run方法，代码有点长，我们一点一点来看。

首先我们设置了线程的优先级为THREAD_PRIORITY_BACKGROUND，接着调用了mcache.initialize（）方法，该方法会扫描缓存目录得到所有缓存数据的摘要信息放入内存。注意这里只是缓存数据的摘要，不包含数据。

接着就会使用take方法不断的从缓存队列中去得到一个请求。take方法会取走BlockingQueue里排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到BlockingQueue有新的对象被加入为止 。

得到请求以后就要判断该请求是否被取消，如果被取消掉了就不继续进行下面的操作。如果没有取消掉，就通过request.getCacheKey()得到对应的缓存key。缓存key的格式为---请求方法：url，比如：Get:http://www.baidu.com

有了请求的缓存key我们就可以通过Cache这个接口去我们缓存文件夹下面去获取缓存，并把缓存数据包装成一个entry对象。如果得到的entry为空，就把这个请求加入到网络队列中去重新请求，如果得到的entry过期了，就把缓存放入请求当中再加入到网络队列当中。请注意这里的过期判断方法entry.isExpired()，我们看看它的源码:
```
 public boolean isExpired() {
            return this.ttl < System.currentTimeMillis();
        }
```
可以看到判断缓存是否过期的条件是ttl小于当前时间的毫秒数。如果小于则为过期，如果大于等于缓存就没有过期。那ttl是什么鬼？在这里先告诉大家ttl是硬过期时间，具体什么是硬过期，下文会说明。

一切验证完毕以后，我们就要把entry对象中的数据包装成一个NetworkResponse对象，再调用Request类中的parseNetworkResponse方法去解析这个NetworkResponse对象。在这个例子中我们使用的是StringRequest，我们看一下StringRequest是如何实现parseNetworkResponse方法的。
```
 @Override
    protected Response<String> parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {
            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
        } catch (UnsupportedEncodingException e) {
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }
```
从以上代码我们可以看到，parseNetworkResponse方法无非就是把NetworkResonse中的数据转换为了String对象，然后包装成一个Response对象返回。从这里我们可以想象一下如果使用的是JsonObjectRequest来构建请求，这里就会把数据转换为一个JsonObject对象如何包装成Response返回。
以后我们自定义其他类型的Request，也要重写这个方法，把数据转换为我们需要的类型。

我们可以看到在最后一行代码里面有个parseCacheHeaders方法，这个方法很重要，我们跟进去看看。
```
public static Cache.Entry parseCacheHeaders(NetworkResponse response)
    {
        long now = System.currentTimeMillis();

        Map<String, String> headers = response.headers;

        long serverDate = 0;
        long lastModified = 0;
        long serverExpires = 0;
        long softExpire = 0;
        long finalExpire = 0;
        long maxAge = 0;
        long staleWhileRevalidate = 0;
        boolean hasCacheControl = false;
        boolean mustRevalidate = false;

        String serverEtag = null;
        String headerValue;
        //服务器时间
        headerValue = headers.get("Date");
        if (headerValue != null)
        {
            serverDate = parseDateAsEpoch(headerValue);
        }

        //判断响应头中是否有cache-control首部
        headerValue = headers.get("Cache-Control");
        if (headerValue != null)
        {
            hasCacheControl = true;
            String[] tokens = headerValue.split(",");
            for (int i = 0; i < tokens.length; i++)
            {
                String token = tokens[i].trim();
                //cache-强制每次请求直接发送给源服务器，而不经过本地缓存版本的校验。
                // 这对于需要确认认证应用很有用（可以和public结合使用），
                // 或者严格要求使用最新数据 的应用（不惜牺牲使用缓存的所有好处）；
                //no-store-强制缓存在任何情况下都不要保留任何副本
                if (token.equals("no-cache") || token.equals("no-store"))
                {
                    return null;
                }
                //缓存的内容将在 xxx 秒后失效, 这个选项只在HTTP 1.1可用
                else if (token.startsWith("max-age="))
                {
                    try
                    {
                        maxAge = Long.parseLong(token.substring(8));
                    } catch (Exception e)
                    {
                    }
                }
                //为过了缓存时间后还可以继续使用缓存的时间，即真正的缓存时间是“max-age=” + “stale-while-revalidate=”的总时间
                else if (token.startsWith("stale-while-revalidate="))
                {
                    try
                    {
                        staleWhileRevalidate = Long.parseLong(token.substring(23));
                    } catch (Exception e)
                    {
                    }
                }
                //有“must-revalidate”或者“proxy-revalidate”字段则过了缓存时间缓存就立即请求服务器
                else if (token.equals("must-revalidate") || token.equals("proxy-revalidate"))
                {
                    mustRevalidate = true;
                }
            }
        }
        //响应过期的日期和时间，是一个绝对时间 。Expires 的一个缺点就是，返回的到期时间是服务器端的时间，
        // 这样存在一个问题，如果客户端的时间与服务器的时间相差很大，那么误差就很大，
        // 所以在HTTP 1.1版开始，使用Cache-Control: max-age=秒替代。
        headerValue = headers.get("Expires");
        if (headerValue != null)
        {
            serverExpires = parseDateAsEpoch(headerValue);
        }
        //请求资源的最后修改时间
        headerValue = headers.get("Last-Modified");
        if (headerValue != null)
        {
            lastModified = parseDateAsEpoch(headerValue);
        }
        //被请求变量的实体标记,在分布式系统中避免使用，每台机器生成的Etag都不一样
        serverEtag = headers.get("ETag");

        // Cache-Control takes precedence over an Expires header, even if both exist and Expires
        // is more restrictive.
        if (hasCacheControl)//是否有Cache-Control头，且非no-cache和no-store方式
        {
            //软过期时间为当前的时间+cachecontrol中maxage的时间*1000
            softExpire = now + maxAge * 1000;
            //硬过期的时间先要判断是否有“must-revalidate”或者“proxy-revalidate”字段，如果有的话
            //硬过期的时间跟软过期一致，否则为软过期时间+stale-while-revalidate字段的时间*1000
            finalExpire = mustRevalidate ? softExpire : softExpire + staleWhileRevalidate * 1000;
        }
        //如果响应头中没有cache-control头，且响应头中返回的服务器时间大于0并且缓存过期时间大于返回的服务器时间
        else if (serverDate > 0 && serverExpires >= serverDate)
        {
            // Default semantic for Expire header in HTTP specification is softExpire.
            //软过期等于当前时间加上（过期时间-返回的服务器时间）
            softExpire = now + (serverExpires - serverDate);
            //硬过期时间等于软过期时间
            finalExpire = softExpire;
        }

        Cache.Entry entry = new Cache.Entry();
        entry.data = response.data;
        entry.etag = serverEtag;
        entry.softTtl = softExpire;
        entry.ttl = finalExpire;
        entry.serverDate = serverDate;
        entry.lastModified = lastModified;
        entry.responseHeaders = headers;

        return entry;
    }
```
以上代码我做了很详细的注解，这个方法主要功能是把NetworkResponse中头信息提取出来，存入Cache.Entry中。从这段代码中我们就知道了前文提到的ttl这个硬过期时间怎么来的了，其实硬过期时间是我自己编的，只是为了对应softTtl软过期时间。严格来叫应该是最终过期时间。

现在我们回到CacheDispatcher类的run方法中继续往下看。得到一个response对象以后，我们接着判断缓存中的数据是否需要更新，以下是Cache.Entry类中的refreshNeeded方法：
```
 public boolean refreshNeeded() {
            return this.softTtl < System.currentTimeMillis();
        }
```
我们可以看到这里判断是否需要更新是通过softTtl这个值来判断的，也就是我们的软过期时间。

如果不需要更新，我们就直接使用ResponseDelivery中的postResonse方法去传递返回的响应结果response。ResponseDelivery是一个接口，我们看他的实现类ExecutorDelivery。
```
/**
 * Delivers responses and errors.
 * 请求结果传输接口具体实现类。
 * 在 Handler 对应线程中传输缓存调度线程或者网络调度线程中产生的请求结果或请求错误，
 * 会在请求成功的情况下调用 Request.deliverResponse(…) 函数，
 * 失败时调用 Request.deliverError(…) 函数。
 */
public class ExecutorDelivery implements ResponseDelivery {
    /** Used for posting responses, typically to the main thread. */
    private final Executor mResponsePoster;

    /**
     * Creates a new response delivery interface.
     * @param handler {@link Handler} to post responses on
     */
    public ExecutorDelivery(final Handler handler) {
        // Make an Executor that just wraps the handler.
        mResponsePoster = new Executor() {
            @Override
            public void execute(Runnable command) {
                handler.post(command);
            }
        };
    }

    /**
     * Creates a new response delivery interface, mockable version
     * for testing.
     * @param executor For running delivery tasks
     */
    public ExecutorDelivery(Executor executor) {
        mResponsePoster = executor;
    }

    @Override
    public void postResponse(Request<?> request, Response<?> response) {
        postResponse(request, response, null);
    }

    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

    @Override
    public void postError(Request<?> request, VolleyError error) {
        request.addMarker("post-error");
        Response<?> response = Response.error(error);
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, null));
    }

    /**
     * A Runnable used for delivering network responses to a listener on the
     * main thread.
     */
    @SuppressWarnings("rawtypes")
    private class ResponseDeliveryRunnable implements Runnable {
        private final Request mRequest;
        private final Response mResponse;
        private final Runnable mRunnable;

        public ResponseDeliveryRunnable(Request request, Response response, Runnable runnable) {
            mRequest = request;
            mResponse = response;
            mRunnable = runnable;
        }

        @SuppressWarnings("unchecked")
        @Override
        public void run() {
            // If this request has canceled, finish it and don't deliver.
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // Deliver a normal response or error, depending.
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }

            // If this is an intermediate response, add a marker, otherwise we're done
            // and the request can be finished.
            if (mResponse.intermediate) {
                mRequest.addMarker("intermediate-response");
            } else {
                mRequest.finish("done");
            }

            // If we have been provided a post-delivery runnable, run it.
            if (mRunnable != null) {
                mRunnable.run();
            }
       }
    }
}
```
还记得我们在构造RequestQueue的时候创建了一个ExecutorDelivery吗？
new ExecutorDelivery(new Handler(Looper.getMainLooper())；
我们当时调用的是参数为Handler的这个构造方法，Handler中传入的又是主线程的Looper，这就意味着Handler send或者Post的对象都是放到主线程的MessageQueue中，然后在主线程中执行。

我们可以看到ExecutorDelivery的构造方法中初始化了一个Executor对象，重写了execute方法，在方法中使用handler.post发送一个runnable到MessageQueue中。

而在CacheDispatche类的run方法中，如果缓存不需要更新，就调用了postResponse(Request<?> request, Response<?> response) 这个方法。这个方法内部又调用了重载方法public void postResponse(Request<?> request, Response<?> response, Runnable runnable) 。这个方法很简单，我就不详细解释了，我们直接来到ExecutorDelivery的内部类ResponseDeliveryRunnbale的run方法当中。

首先还是要判断请求是否被取消。如果这时候被取消，就直接结束传递。我们回忆一下，到目前为止，请求是否被取消的判断用了几次？2次对吧？第一次是从缓存队列中得到请求以后，如果用户取消了，那么就不继续执行了。而这次是用户已经得到了缓存，在传递回响应结果之前判断，如果这时候用户取消请求，那么结果就不继续传递了。所以我们以后自己写网络框架的时候也可以参考Volley的请求取消思路。

接着判断响应结果是否成功，这里isSuccess方法实现很简单，只是通过判断Response中第一个VolleyError是否为空。如果不为空就说明成功了，就调用Request中的deliverResponse方法，把Response中的result传入进去。因为我们使用StringRequest为例，那么我们进看看那StringRequest中的deliverResponse方法是如何实现的。
```
  protected void deliverResponse(String response) {
        if (mListener != null) {
            mListener.onResponse(response);
        }
    }
```
deliverResponse方法中首先判断mListener是否为空，这个mListener就是我们在构建StringRequest对象时传入的，还记得吗？我们还复写了onResponse方法。所以deliverResponse的目的就是要回调我们复写的onResponse方法，把数据传递过去。

如果Response.isSuccess返回为false，说明VolleyError不为空，那么我们就调用Requet.deliverError这个方法。
```
public void deliverError(VolleyError error)
{    if (mErrorListener != null)   
     {      
        mErrorListener.onErrorResponse(error); 
   }
}
```
deliverError方法跟deliverResponse方法类似，都是回调我们在创建StringRequest时创建的接口。
接着会判断Response.intermediate这个变量是否为true,如果不为true那么就执行Request.finish方法，这个请求也就结束了。那intermediate是什么东西呢？我们暂时不知道，我们选择返回CacheDispatcher中的run方法接着看。

如果缓存需要更新的话，我们把缓存中的数据放入request，然后设置response.intermediate为true.这时候我们就可以猜测intermediate是一个标识位，如果缓存需要更新那么这个标识位就为true。

接着创建了一个新的Request对象，然后用ExecutorDelivery.postResponse方法传递数据。这里为什么要新建一个Request对象而不直接把当前的request对象传递过去呢？因为在ResponseDeliveryRunnbale的run方法中执行了request.finish方法，这个方法先暂时理解为把该请求从各种队列中删除。

由于我们这次调用的postResponse方法传递了一个Runnable对象，那么ResponseDeliveryRunnbale的run方法中就要去执行Runnable对象的run方法。也就是把新创建的finalRquest加入到网络队列中去，从这里我们发现，如果缓存软过期，我们可以返回当前缓存中的数据，但是我们还需要把这次请求加入到网络队列中去获取新的数据。

到目前为止，我们已经分析完了CacheDispatche类中的run方法了。由于文章过长，所以RequstQueue类中的start方法后半部分，也就是网络调度器NetworkDispatcher的创建我们下篇文章再继续分析。

![CacheDispatcher.png](http://upload-images.jianshu.io/upload_images/1870458-df56b30e93ecc28c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

本文为原创文章，转载请注明出处：http://www.jianshu.com/p/85bcd878a676