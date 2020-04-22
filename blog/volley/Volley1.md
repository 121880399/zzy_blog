Volley 总纲

#前言
之前写过很多篇关于volley源码的分析文章，这些文章都是深入到volley源码的细节去分析和扩展。最近重新来看这些文章，发现这些文章给人一种只见树木不见森林的感觉，看完以后觉得自己好像明白了什么，却又还是什么都不明白，所以抽时间出来画了两张图，给大家总结一下Volley的整体框架。

#网络请求部分
 之前的文章中有介绍过，volley这个网络请求框架其实可以分为缓存部分和网络请求部分。我们先来看网络请求部分。

![volley网络请求.jpg](http://upload-images.jianshu.io/upload_images/1870458-628dac1db9607f0f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图是我用windows的画图工具画的，不是特别的精细，凑合着看。
但凡是网络请求框架就少不了，请求，请求队列，响应这三部分。Volley也不例外，从使用上我们也能感受到，首先会有一个请求队列，用来存放请求。其次有一个网络请求器，在这里是NetworkDisptcher，不停的从请求队列中获取请求进行处理。
我们以一次请求举例，我们假设这次请求是没有被缓存过的。
1.首先NetworkDispatcher会不停的从RequestQueue中取请求，由于RequesQueue中使用的是BlockingQueue,也就是阻塞队列，当队列里没有请求的时候，是会阻塞掉的。
2.我们发起了一次请求，并且把请求加入到请求队列当中。
3.因为请求队列中存在请求，所以NetworkDispatcher就得到了一个请求，并且根据andriod的版本来判断是否采用HttpClient进行网络请求还是采用HttpUrlConnection。
4.网络请求成功以后会把网络响应包装成NetworkResponse，再根据不同的请求包装成不同的Response，比如是StringRequest就把Response包装成Sring类型的，如果是ImageRequest就包装成Response<Bitmap>。
5.如果需要缓存，那么就把该请求缓存到/data/data/你应用的包名/cache/vollley目录下，并且在DiskBasedCache中维护的一个LinkedHashMap中把缓存头给保存下来。接着通过ResponseDelivery把Response传递给用户。

在这里需要注意的：
1.volley中的所有请求都是异步的。
2.volley中默认会创建4个NetworkDispacher。
3.每一个NetworkDispatcher都是一个Thread,也就是开启了4条线程。
4.volley并没有使用线程池。
5.RequestQueue中的队列使用的是PriorityBlockingQueue，这说明请求是可以有优先级的。
6.由于只有4条网络线程，如果每条线程处理的网络任务时间过长，那么其他任务可能就会出于等待状态，这时的效率会很低。

#缓存部分

![volley缓存.jpg](http://upload-images.jianshu.io/upload_images/1870458-5670c29ea99368e5.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

缓存部分其实跟网络请求部分很类似。
1.首先CacheDispatcher会不从RequesQueue中的CacheQueue中取Request,如果没有将被阻塞。
2.取到Request以后会根据url到DisBasedCache中去找CacheHeader。DisBasedCache中有一个初始大小为16，加载因子为0.75的LinkedHashMap，所有已经缓存的请求都会 把url，硬过期时间，软过期时间响应头等信息包装成CacheHeader存放在这个LinkedHashMap当中。也就是说这个Map中的数据跟缓存是一一对应的。
3.如果不存在缓存，那么就把该请求加入到网络请求队列(NetworkQueue)中，如果存在缓存，那么判断缓存是否硬过期，如果过期也加入到网络请求队列。如果没过期，说明缓存命中了。
4.将请求封装成相应的Response，之后还要判断该请求结果是否软过期了，如果软过期那么在把Response通过ResponseDispatcher传递给用户的同时还要把请求加入到网络请求队列。如果不存在软过期，就直接返回给用户。

这里需要注意的是：
1.CacheDispatcher也是一个Thread。
2.CacheDispatcher跟NetworkDispatcher不一样，CacheDispatcher只有一个，也就是只有1条缓存线程。
3.请求是否过期依赖于Http头。

#总结：
Volley网络框架还是比较轻量级的，代码也不多，整体结构也清晰，适合大家阅读和提高。更多关于Volley的分析，大家可以看我之前写的关于volley的文章。

如果你觉得本篇文章帮助到了你，希望大爷能够给瓶买水钱。
本文为原创文章，转载请注明出处！