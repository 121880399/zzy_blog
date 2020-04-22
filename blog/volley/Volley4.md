Volley都干了些什么？（三）

#目录
 (一) [CacheDispatcher的解析](http://www.jianshu.com/p/85bcd878a676)
 (二) [NetworkDispatcher的解析](http://www.jianshu.com/p/0a0dc214c2c0)
 (三) [RequestQueue中add方法的分析](http://www.jianshu.com/p/4dbc76e3f548)
 (四) [ImageRequest的分析](http://www.jianshu.com/p/d6609049325d)
 (五) [ImageLoader的分析](http://www.jianshu.com/p/cb6ba58f9f3e)
#前言
在上两篇文章中，我们已经对Volley.newRequestQueue操作的内部实现做了分析。在这篇文章中，我们要对StringRequest的创建和RequestQueue.add(request)操作进行分析。

#把请求加入队列
由于StringRequest的创建太简单，没有什么可以说的，这里就略过了。直接来看RequestQueue.add操作。
以下是RequestQueue类中的add方法:
```
public <T> Request<T> add(Request<T> request)
    {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests)
        {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");

        // If the request is uncacheable, skip the cache queue and go straight to the network.
        //如果该Request不能被缓存，则加入到网络队列当中
        if (!request.shouldCache())
        {
            mNetworkQueue.add(request);
            return request;
        }
        //以下情况是针对可以缓存的请求
        // Insert request into stage if there's already a request with the same cache key in flight.
        synchronized (mWaitingRequests)
        {
            String cacheKey = request.getCacheKey();
            if (mWaitingRequests.containsKey(cacheKey))//如果等待请求队列中有相同的cacheKey则取出
        {
            // There is already a request in flight. Queue up.
            //通过cachekey从等待队列中得到队列
            Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
            if (stagedRequests == null)
            {
                stagedRequests = new LinkedList<Request<?>>();
            }
            stagedRequests.add(request);
            mWaitingRequests.put(cacheKey, stagedRequests);//更新等待队列
            if (VolleyLog.DEBUG)
            {
                VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
            }
        } else
        {
            //如果等待队列中没有相同的请求，则通过cacheKey在等待队列中创建一个为空的，并把这个请求加入到缓存队列中
            // Insert 'null' queue for this cacheKey, indicating there is now a request in
            // flight.
            mWaitingRequests.put(cacheKey, null);
            mCacheQueue.add(request);
        }
            return request;
        }
    }
```
我们依然是挑重点看，在分析这个方法中的代码前，我们先来解释一下两个变量。
```
private final Map<String, Queue<Request<?>>> mWaitingRequests = new HashMap<String, Queue<Request<?>>>();
private final Set<Request<?>> mCurrentRequests = new HashSet<Request<?>>();
```
从变量的命名来看，一个是等待请求，一个是当前请求。这两个集合跟我们前两篇文章中所提到的缓存队列和网络队列有什么关系呢？

我们接着看代码，进入add方法体后，我们会把传入的request请求存入mCurrentRequest这个set集合当中，也就是说mCurrentRequest这个集合存放了所有的请求。
接着request.shouldCache判断该请求是否可以缓存，默认情况返回true,如果不能缓存就把请求加入到网络队列当中并返回请求。

接下来的情形都是针对可以缓存的请求进行的判断。使用request.getCacheKey以后我们会判断mWaitingRequests这个集合中是否包含该缓存key，我们先看不包含的情况，如果不包含那么我们就往mWaitingRequests集合中加入一个以cachekey为key,以null为value的空值，然后把请求放入缓存队列。

这时候如果再把一个有着相同cacheKey的请求加入，那么mWaitingRequests集合中就包含了该CacheKey，那么就从mWaitingRequests中取出对应的Value来，这里的Value是一个Queue对象，接着会把请求放入这个Queue中，然后更新mWaitingRequests。

也就是说如果一个请求可以被缓存，那么我们就要在mWaitingRequests这个集合中去添加一个以当前请求为key的Queue队列，以后相同的请求都会加入到这个Queue当中。

那么现在就很清楚mCurrentRequests和mWaitingRequests这两个集合的作用是什么以及它们与缓存队列，网络队列的关系。

好了，RequestQueue中的add方法也解析完了，接下来我们要看看当一个请求完成的时候发生了什么？我们大致思考一下，请求完成的方式大致有两种，一种是以缓存命中的方式结束，一种是以网络请求完成来结束。但是不论请求以什么样的形式结束，都会调用request.finish()方法:
```
void finish(final String tag)
    {
        if (mRequestQueue != null)
        {
            mRequestQueue.finish(this);
            onFinish();
        }
        if (MarkerLog.ENABLED)
        {
            final long threadId = Thread.currentThread().getId();
            if (Looper.myLooper() != Looper.getMainLooper())
            {
                // If we finish marking off of the main thread, we need to
                // actually do it on the main thread to ensure correct ordering.
                Handler mainThread = new Handler(Looper.getMainLooper());
                mainThread.post(new Runnable()
                {
                    @Override
                    public void run()
                    {
                        mEventLog.add(tag, threadId);
                        mEventLog.finish(this.toString());
                    }
                });
                return;
            }

            mEventLog.add(tag, threadId);
            mEventLog.finish(this.toString());
        }
    }
```
finish方法很简单，如果该请求的RequestQueue不为空，那么我们就调用该RequestQueue的finish方法，并把当前请求传入，然后再调用onFinish方法。onFinish方法很简单，只是把回调的错误接口重置为null。
而后面if的条件判断是针对Debug模式的，我们可以不看。我们主要来看RequestQueue中的finish方法：
```
<T> void finish(Request<T> request)
    {
        // Remove from the set of requests currently being processed.
        //从当前请求队列中删除该请求
        synchronized (mCurrentRequests)
        {
            mCurrentRequests.remove(request);
        }
        synchronized (mFinishedListeners)
        {
            for (RequestFinishedListener<T> listener : mFinishedListeners)
            {
                listener.onRequestFinished(request);
            }
        }

        if (request.shouldCache())//如果该请求可以被缓存
        {
            synchronized (mWaitingRequests)
            {
                String cacheKey = request.getCacheKey();//得到缓存key
                Queue<Request<?>> waitingRequests = mWaitingRequests.remove(cacheKey);//从等待队列中得到并移除该缓存key中的值
                if (waitingRequests != null)
                {
                    if (VolleyLog.DEBUG)
                    {
                        VolleyLog.v("Releasing %d waiting requests for cacheKey=%s.", waitingRequests.size(), cacheKey);
                    }
                    // Process all queued up requests. They won't be considered as in flight, but
                    // that's not a problem as the cache has been primed by 'request'.
                    mCacheQueue.addAll(waitingRequests);//将该cachekey对应的所有请求都加入到缓存队列中去
                }
            }
        }
    }
```
前文我们说了mCurrentRequests集合中包含了所有的请求，当请求完成时我们肯定得从这个集合中把请求移除掉。

mFinishedListener是一个List对象，当中存放了所有的RequestFinishedListener接口，这个接口是在RequestQueue中定义的，其中只有一个方法onReqeustFinished，用户可以实现RequestFinishedListener接口并使用addRequestFinishedListener方法把自己实现的接口传入mFinishedListener中，在执行finish方法的时候，就会回调自己实现的方法。

接着判断如果该结束的请求是可以缓存的，那么我们就要通过cacheKey去mWatingRequests中去移除并得到一个Queue，这个Queue中存放了相同的请求，如果存在这些请求，那么我们就把这些请求全部加入到缓存队列中去。换言之，如果我们执行完了一个可以缓存的请求，这个请求不论是通过缓存完成的，还是通过网络完成的，它必定要把完成的结果缓存在本地。那么Queue中这些相同的请求只要从缓存中取就可以了，所以会把这些相同的请求放入缓存队列。

总结一下RequestQueue类中finish方法的逻辑，当一个请求完成，首先要从mCurrentRequests这个集合中删除，如果这个请求是一个可以缓存的请求，那么再去mWaitingRequests中去删除相同的请求队列，并把队列里面的请求全部加入到缓存队列当中。

到目前为止，Volley一般使用流程的代码执行我们已经分析完了，但是Volley博大精深，还有很多实现细节我们没有分析到，由于时间仓促，文章中也难免出现错误，如果有发现错误的读者请与我联系。

本文为原创文章，转载请注明出处：http://www.jianshu.com/p/4dbc76e3f548