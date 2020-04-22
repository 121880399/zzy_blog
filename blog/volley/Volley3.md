Volley 都干了些什么？（二）

#目录
 (一) [CacheDispatcher的解析](http://www.jianshu.com/p/85bcd878a676)
 (二) [NetworkDispatcher的解析](http://www.jianshu.com/p/0a0dc214c2c0)
 (三) [RequestQueue中add方法的分析](http://www.jianshu.com/p/4dbc76e3f548)
 (四) [ImageRequest的分析](http://www.jianshu.com/p/d6609049325d)
 (五) [ImageLoader的分析](http://www.jianshu.com/p/cb6ba58f9f3e)
#前言
上篇文章我们主要分析了Volley的缓存调度器，如果你还没有了解请移步[这里](http://www.jianshu.com/p/85bcd878a676)。今天我们接着上文继续解析Volley源码。
#网络调度器
上篇文章由于篇幅问题，我们只是对RequestQueue.start方法中的缓存调度器初始化做了分析。接下来我们来分析网络调度器的初始化。
```
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
mDispatchers是一个NetworkDispatcher数组，大小为4。这个4是怎么来的呢？前文介绍了RequestQueue的构造函数中使用了DEFAULT_NETWORK_THREAD_POOL_SIZE来构造RequestQueue,这个DEFAULT_NETWORK_THREAD_POOL_SIZE的值就是4。也就是说我们这里开启4个线程来处理网络请求。可以看见Volley中并没有使用ThreadPoolExecutor这样的线程池。

看了上篇文章的朋友肯定对初始化NetworkDispatcher的参数有所了解。mNetworkQueue是个PriorityBlockingQueue的优先级阻塞队列。mNetwork是BasicNetwork,它实现了Network接口，用来执行网络请求。mCache是DiskBasedCache用来执行缓存操作的。mDelivery是ExecutorDelivery用来传递响应的结果。

由于NetworkDispatcher继承了Thread类，所以执行networkDispatcher.start（）后，我们得去看看NetworkDispatcher类中的run方法。
```
@Override
    public void run()
    {
        //设置线程优先级
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        Request<?> request;
        while (true)
        {
            //得到从开机到现在的时间，单位是毫秒
            long startTimeMs = SystemClock.elapsedRealtime();
            // release previous request object to avoid leaking request object when mQueue is drained.
            request = null;
            try
            {
                // Take a request from the queue.
                request = mQueue.take();//从网络队列中获取一个请求
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
                request.addMarker("network-queue-take");

                // If the request was cancelled already, do not perform the
                // network request.
                if (request.isCanceled())//如果请求被取消，则不执行网络请求
                {
                    request.finish("network-discard-cancelled");
                    continue;
                }

                addTrafficStatsTag(request);

                // Perform the network request.执行网络请求
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                if (networkResponse.notModified && request.hasHadResponseDelivered())
                {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                if (request.shouldCache() && response.cacheEntry != null)
                {
                    mCache.put(request.getCacheKey(), response.cacheEntry);//如果请求可以缓存，则把结果放入到缓存当中
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);//返回请求结果
            } catch (VolleyError volleyError)
            {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e)
            {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
        }
    }
```
首先设置线程的优先级，注意这里的Process不是Java.lang中的，而是android.os包下面的。THREAD_PRIORITY_BACKGROUND)的值为10。

接着不断的从网络队列中去获取请求，如果队列为空就阻塞。不为空就接着判断请求是否被取消了？如果取消就结束本次循环，重新从网络队列中获取请求。

addTrafficStatsTag方法中会去判断当前SDK的版本，如果API>=14,那么就可以使用TrafficStats.setThreadStatsTag（）来统计线程内部发生的数据传输情况。这个类的用法在这里不细讲，大家可以去百度。setTreadStatsTag接受一个int型参数，这个参数是request中url的host部分的hash码。
```
 @TargetApi(Build.VERSION_CODES.ICE_CREAM_SANDWICH)
    private void addTrafficStatsTag(Request<?> request)
    {
        // Tag the request (if API >= 14)
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH)
        {
            //标记线程内部发生的数据传输情况
            TrafficStats.setThreadStatsTag(request.getTrafficStatsTag());
        }
    }
```

接下来我们看执行网络请求的方法BasicNetwork.performRequest();
```
 @Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError
    {
        //从开机到现在的毫秒数
        long requestStart = SystemClock.elapsedRealtime();
        while (true)
        {
            HttpResponse httpResponse = null;
            //响应内容
            byte[] responseContents = null;
            //响应头
            Map<String, String> responseHeaders = Collections.emptyMap();
            try
            {
                // Gather headers.
                Map<String, String> headers = new HashMap<String, String>();
                //如果请求中有缓存数据，那么添加额外的请求头
                addCacheHeaders(headers, request.getCacheEntry());
                //使用HttpClient或者HttpUrlConnnection来执行请求并返回HttpResonse对象
                httpResponse = mHttpStack.performRequest(request, headers);
                //得到响应的状态
                StatusLine statusLine = httpResponse.getStatusLine();
                //得到状态码
                int statusCode = statusLine.getStatusCode();
                //把HttpResonse中的响应头全部放入一个Map对象中
                responseHeaders = convertHeaders(httpResponse.getAllHeaders());
                // Handle cache validation. 处理缓存验证
                //HttpStatus.SC_NOT_MODIFIED=304 Not Modified
                if (statusCode == HttpStatus.SC_NOT_MODIFIED)
                {

                    Entry entry = request.getCacheEntry();
                    if (entry == null)
                    {
                        return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, null, responseHeaders, true, SystemClock.elapsedRealtime() - requestStart);
                    }

                    // A HTTP 304 response does not have all header fields. We
                    // have to use the header fields from the cache entry plus
                    // the new ones from the response.
                    // http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.3.5
                    entry.responseHeaders.putAll(responseHeaders);
                    return new NetworkResponse(HttpStatus.SC_NOT_MODIFIED, entry.data, entry.responseHeaders, true, SystemClock.elapsedRealtime() - requestStart);
                }

                // Handle moved resources 处理资源移动的情况
                //SC_MOVED_PERMANENTLY=301 Moved Permanently SC_MOVED_TEMPORARILY=302 Moved Temporarily
                if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || statusCode == HttpStatus.SC_MOVED_TEMPORARILY)
                {
                    //Location表示客户应当到哪里去提取文档
                    String newUrl = responseHeaders.get("Location");
                    request.setRedirectUrl(newUrl);
                }

                // Some responses such as 204s do not have content.  We must check.
                //判断是否返回了响应实体,状态码为204是没有内容的
                if (httpResponse.getEntity() != null)
                {
                    //把一个HttpEntity对象转化为一个byte数组
                    responseContents = entityToBytes(httpResponse.getEntity());
                } else
                {
                    // Add 0 byte response as a way of honestly representing a
                    // no-content request.
                    responseContents = new byte[0];
                }

                // if the request is slow, log it.
                //得到request的一个生命周期
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                //如果是debug模式，或者requestLifetime大于3000ms，就用Log打印请求
                logSlowRequests(requestLifetime, request, responseContents, statusLine);
                //如果状态码小于200，大于299返回IO异常
                if (statusCode < 200 || statusCode > 299)
                {
                    throw new IOException();
                }
                return new NetworkResponse(statusCode, responseContents, responseHeaders , false, SystemClock.elapsedRealtime() - requestStart);
            } catch (SocketTimeoutException e)
            {
                attemptRetryOnException("socket", request, new TimeoutError());
            } catch (ConnectTimeoutException e)
            {
                attemptRetryOnException("connection", request, new TimeoutError());
            } catch (MalformedURLException e)
            {
                throw new RuntimeException("Bad URL " + request.getUrl(), e);
            } catch (IOException e)
            {
                int statusCode = 0;
                NetworkResponse networkResponse = null;
                if (httpResponse != null)
                {
                    statusCode = httpResponse.getStatusLine().getStatusCode();
                } else
                {
                    throw new NoConnectionError(e);
                }
                if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || statusCode == HttpStatus.SC_MOVED_TEMPORARILY)
                {
                    VolleyLog.e("Request at %s has been redirected to %s", request.getOriginUrl(), request.getUrl());
                } else
                {
                    VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                }
                if (responseContents != null)
                {
                    networkResponse = new NetworkResponse(statusCode, responseContents, responseHeaders, false, SystemClock.elapsedRealtime() - requestStart);
                    if (statusCode == HttpStatus.SC_UNAUTHORIZED || statusCode == HttpStatus.SC_FORBIDDEN)
                    {
                        attemptRetryOnException("auth", request, new AuthFailureError(networkResponse));
                    } else if (statusCode == HttpStatus.SC_MOVED_PERMANENTLY || statusCode == HttpStatus.SC_MOVED_TEMPORARILY)
                    {
                        attemptRetryOnException("redirect", request, new RedirectError(networkResponse));
                    } else
                    {
                        // TODO: Only throw ServerError for 5xx status codes.
                        throw new ServerError(networkResponse);
                    }
                } else
                {
                    throw new NetworkError(e);
                }
            }
        }
    }
```
以上代码做了比较详细的注释，我们挑重点代码看。
首先看addCacheHeaders(headers, request.getCacheEntry());
```
private void addCacheHeaders(Map<String, String> headers, Cache.Entry entry)
    {
        // If there's no cache entry, we're done.
        if (entry == null)
        {
            return;
        }

        if (entry.etag != null)
        {
            headers.put("If-None-Match", entry.etag);
        }

        if (entry.lastModified > 0)
        {
            Date refTime = new Date(entry.lastModified);
            headers.put("If-Modified-Since", DateUtils.formatDate(refTime));
        }
    }
```
首先我们来思考一下，为什么会有这个方法的存在？还记得我们上篇文章介绍的CacheDispatcher类吗？其中run方法里面有3种情况会把请求加入到网络队列中。
1.当从本地获取缓存为空的时候
2.请求硬过期的时候
3.请求软过期的时候
在这里我们先解释一下网络请求的一般思想，如果客户端发起一个请求，这个请求被缓存过，并且缓存未过期就直接使用缓存数据。如果这个请求被缓存过，但是缓存过期了怎么办？缓存过期并不意味着这个资源真的就过期了，在服务器上这个资源可能一直没有发生变化，所以我们需要发送一个验证请求来判断资源是否发生变化，如果没有发生变化就返回304代码，这时候服务器返回一个新的响应头这里面包括新的过期时间。如果资源发生变化服务器就直接返回200代码，这时候服务器返回的就是发生变化后的全新资源。

那么问题来了，如何判断缓存中的资源跟服务器上的资源一致？
首先服务器在第一次返回资源的时候，响应头中会有Expires头部，Expires的值是一个绝对的时间值，当前客户端的时间超过这个值资源就过期了。或者响应头中Cache-Control头部的值是max-age，max-age的值是毫秒数，是一个相对的时间，指的是资源在客户端过了多少毫米以后就过期。如果max-age和Expires同时存在，则max-age覆盖Expires.

这里值得一提的是Expires的使用存在着缺陷，因为Expires返回的是服务器的时间，如果客户端的时间和服务器的时间相差较大的话，那么就会有误差，所以在Http 1.1版本开始，使用Cache-control：max-age来替代。

上面已经说了资源在什么情况下会过期，过期了要发送验证请求来确认服务器的资源是否修改过，如果修改过就返回200状态码，如果没有修改过就返回304状态码。那么我们如何判断服务器资源是否修改过呢？
服务器在第一次返回资源的时候，在头部还添加了Last-Modified和Etag头部。
Last-Modified标记了资源文件在服务器的最后修改时间，类似于Last-Modified:Tue, 24 Feb 2016 08:01:04 GMT，当客户端由于缓存过期发起请求时，请求头要使用If-Modified-Since头部，它的值就是第一次服务器返回的Last-Modified。服务器收到这个时间后，跟当前的资源文件最后修改时间进行对比，如果服务器中资源文件的最后修改时间新与Last-Modified的值，那么说明资源文件进行修改过。
Etag头部是资源实体标记，格式类似于Etag:“5d83a2aeedda8d6a:3119″，它是资源的唯一标识。在服务器第一次返回数据的时候，响应头中会包含这个头部。当客户端由于缓存过期发起请求时会使用If-None-Match头部，它的值就是Etag返回的值。服务器收到这个值后，跟当前资源的Etag值进行对比，如果是相同的，说明是同一个资源，就会返回304状态码。否则返回200状态码。
If-Modified-Since和Etag一个保证资源是否修改，一个保证是否是同一个资源。
需要注意的是在分布式系统里，需要保证多台服务器之间资源的Last-Modified都一致。Etag头部尽量不要使用，因为每台机器生产的Etag都不一样。

有了以上的知识，我们就很容易看懂addCacheHeaders方法都干了些什么，我就不做过多的解释了。

接下来我们继续看BasicNetwork.performRequest()方法，添加完请求头以后就该真正执行请求了，这里的mHttpStack会根据当前的SDK版本来选择，有可能是HurlStack或者HttpClientStack,上篇文章有说过，这两个类的performRequest的实现我们就不跟进去了，大家有兴趣自己去翻一下源码。这里我们只用知道这个方法会返回一个HttpResponse对象。

HttpResponse中可以得到状态码，如果状态码为HttpStatus.SC_NOT_MODIFIED，也就是304状态码，说明服务器的资源没有变化，既然没有变化，那么我们就可以直接使用缓存中的数据来构建一个NetWorkResponse对象，getCacheEntry会得到一个Cache.Entry对象，这个对象保存了缓存中的内容，如果entry为空，那么我们构建的NetworkResponse对象就不传入数据返回。如果entry不为空，那么就把entry对象中的响应头改为新的响应头，然后构造NetworkResponse对象返回。

如果状态码为HttpStatus.SC_MOVED_PERMANENTLY ：301、HttpStatus.SC_MOVED_TEMPORARILY：302，说明服务器资源位置发生了改变，那么我们就要设置request中的redirectUrl。

如果返回的状态码是204说明返回的消息当中没有实体，那么我们就使用一个零byte的数组来代表响应内容。最后如果状态码不是小于200或者大于299则使用得到的响应体，响应头，状态码，请求所花的时间来构建一个NetworkResponse对象返回。

看完了BasicNetwork中的performRequest方法，我现在返回NetworkDispatcher类中去继续看run方法。
```
              // Perform the network request.执行网络请求
                NetworkResponse networkResponse = mNetwork.performRequest(request);
                request.addMarker("network-http-complete");

                // If the server returned 304 AND we delivered a response already,
                // we're done -- don't deliver a second identical response.
                //hasHadResponseDelivered中mResponseDelivered默认值为false
                if (networkResponse.notModified && request.hasHadResponseDelivered())
                {
                    request.finish("not-modified");
                    continue;
                }

                // Parse the response here on the worker thread.
                Response<?> response = request.parseNetworkResponse(networkResponse);
                request.addMarker("network-parse-complete");

                // Write to cache if applicable.
                // TODO: Only update cache metadata instead of entire record for 304s.
                //shouldCache中的mShouldCache值默认为true
                if (request.shouldCache() && response.cacheEntry != null)
                {
                    mCache.put(request.getCacheKey(), response.cacheEntry);//如果请求可以缓存，则把结果放入到缓存当中
                    request.addMarker("network-cache-written");
                }

                // Post the response back.
                request.markDelivered();
                mDelivery.postResponse(request, response);//返回请求结果
            } catch (VolleyError volleyError)
            {
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                parseAndDeliverNetworkError(request, volleyError);
            } catch (Exception e)
            {
                VolleyLog.e(e, "Unhandled exception %s", e.toString());
                VolleyError volleyError = new VolleyError(e);
                volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
                mDelivery.postError(request, volleyError);
            }
```
在通过mNetwork.performRequest方法得到一个NetworkResponse对象以后，接着判断了networkResponse.notModified是否为true，如果当时返回的响应头是304就为true。另外一个request.hasHadResponseDelivered方法返回的是一个mResponseDelivered变量的值，这个值默认为false,只有手动调用了request.markDelivered()才为true。如果判断中的两个值都为true，那么就结束本次循环，也就是不需要回调接口把结果返回了。

接下来的代码在上篇文章都有涉及个别的也有详细的注释，这里就不做解析了。到此为止NetworkDispatcher中的run方法也解析完毕。接下来的源码解析请看下篇文章。

![NetworkDispatcher.png](http://upload-images.jianshu.io/upload_images/1870458-8e4b7d61de6e28be.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

本文为原创文章，转载请注明出处：http://www.jianshu.com/p/0a0dc214c2c0