Volley都干了些什么？（五）

#目录
 (一) [CacheDispatcher的解析](http://www.jianshu.com/p/85bcd878a676)
 (二) [NetworkDispatcher的解析](http://www.jianshu.com/p/0a0dc214c2c0)
 (三) [RequestQueue中add方法的分析](http://www.jianshu.com/p/4dbc76e3f548)
 (四) [ImageRequest的分析](http://www.jianshu.com/p/d6609049325d)
 (五) [ImageLoader的分析](http://www.jianshu.com/p/cb6ba58f9f3e)
#前言
在上篇文章我们分析了ImageRequest的源码，如果你对ImageRequest还不了解，请先看上篇文章的讲解。因为接下来的文章或多或少都会与ImageReqeust有所关联。

#ImageLoader的使用
在解析ImageLoader源代码之前，我们还是来回顾一下ImageLoader的一般用法。
1.创建一个RequestQueue对象
```
RequestQueue mQueue=Volley.newRequestQueue(mContext);
```
2.创建一个ImageLoader对象
```
ImageLoader imageLoader=new ImagerLoader(mQueue,new ImageCache()
{
       @Override  
    public void putBitmap(String url, Bitmap bitmap)
    {  
    }  
  
    @Override  
    public Bitmap getBitmap(String url) 
    {  
        return null;  
    }  
});
```
这里的putBitmap和getBitmap我们先空着，这里怎么写比较好以后再说。

3.得到一个ImageListener对象
```
ImageListener mListener=ImageLoader.getImageListener(imageView,
                                        R.drawable.default_image,R.drawable.failed_image);
```
这里传入一个ImageView对象，一张默认显示的图片id，一张失败显示的图片id。

4.调用ImageLoader的get方法获取网络图片
```
imageLoader.get("https://ss0.bdstatic.com/5aV1bjqh_Q23odCf/static/superman/img/logo/bd_logo1_31bdc765.png",mListener);

imageLoader.get("https://ss0.bdstatic.com/5aV1bjqh_Q23odCf/static/superman/img/logo/bd_logo1_31bdc765.png",mListener,200,400);

imageLoader.get("https://ss0.bdstatic.com/5aV1bjqh_Q23odCf/static/superman/img/logo/bd_logo1_31bdc765.png",mListener,200,400,ImageView.ScaleType.CENTER_CROP);
```
这里可以看到ImageLoader的get方法有三种重载方法，如果看过上篇文章的话，其实不用看代码就能猜出这些参数是干什么用的。

#ImageLoader的解析
上面我们回顾了一下ImageLoader的基本用法，现在我们就按照以上步骤一步步去跟踪一下ImageLoader的源码吧。由于RequestQueue的创建在前几篇文章中已经分析过了，所以我们直接从第二步创建ImageLoader对象开始分析。
```
public ImageLoader(RequestQueue queu e, ImageCache imageCache)
    {
        mRequestQueue = queue;
        mCache = imageCache;
    }
```
ImageLoader对象的创建就只有2行代码，简单的给成员变量赋值。这里简要的提一下ImageCache这个接口，ImageCache是ImageLoader类中定义的一个内部接口。只有putBitmap和getBitmap两种方法，putBitmap方法的作用是把bitmap对象存入缓存中，getBimap方法的作用是从缓存中得到一个bitmap对象。在上面的代码中这两个方法我们都是空实现，所以我们现在只要知道这两个方法要实现的功能就行。

接下来我们看getImageListener方法如何得到一个ImageListener对象：
```
public static ImageListener getImageListener(final ImageView view, final int defaultImageResId, final int errorImageResId)
    {
        return new ImageListener()
        {
            /**
             * 如果返回出错，则显示错误图片
             * */
            @Override
            public void onErrorResponse(VolleyError error)
            {
                if (errorImageResId != 0)
                {
                    view.setImageResource(errorImageResId);
                }
            }

            /**
             * 如果响应图片为空的时候就显示默认图片，响应图片不为空，则显示响应图片。
             * */
            @Override
            public void onResponse(ImageContainer response, boolean isImmediate)
            {
                if (response.getBitmap() != null)
                {
                    view.setImageBitmap(response.getBitmap());
                } else if (defaultImageResId != 0)
                {
                    view.setImageResource(defaultImageResId);
                }
            }
        };
    }
```
这个方法的逻辑很简单，直接new了一个ImageListener对象，并且实现了其中的onErrorResponse和onResponse方法。这两个方法会在适当的时候被回调，具体是什么时候我们接下来会讲到，从这两个方法的方法名和方法中的代码我们可以知道，如果还没收到返回的Bitmap对象则先显示默认的图片，收到返回的Bitmap对象后才显示收到的图片，当返回出错的时候会把错误的图片显示在指定的ImageView上。

最后看ImageLoader的get方法：
```
 public ImageContainer get(String requestUrl, ImageListener imageListener, int maxWidth, int maxHeight, ScaleType scaleType)
    {

        // only fulfill requests that were initiated from the main thread.
        //如果该方法不在主线程中执行则抛出异常
        throwIfNotOnMainThread();

        //使用requestUrl,maxWidth,maxHeight,scaleType构造一个cacheKey
        final String cacheKey = getCacheKey(requestUrl, maxWidth, maxHeight, scaleType);

        // Try to look up the request in the cache of remote images.
        //通过cachekey从缓存中得到Bitmap对象
        Bitmap cachedBitmap = mCache.getBitmap(cacheKey);
        if (cachedBitmap != null)
        {
            // Return the cached bitmap.
            ImageContainer container = new ImageContainer(cachedBitmap, requestUrl, null, null);
            imageListener.onResponse(container, true);
            return container;
        }

        // The bitmap did not exist in the cache, fetch it!
        ImageContainer imageContainer = new ImageContainer(null, requestUrl, cacheKey, imageListener);

        // Update the caller to let them know that they should use the default bitmap.
        //如果缓存中没有图片，就先显示默认图片
        imageListener.onResponse(imageContainer, true);

        // Check to see if a request is already in-flight.
        BatchedImageRequest request = mInFlightRequests.get(cacheKey);
        if (request != null)
        {
            // If it is, add this request to the list of listeners.
            //把imageContainer加入到BatchedImageRequest的list中
            request.addContainer(imageContainer);
            return imageContainer;
        }

        // The request is not already in flight. Send the new request to the network and
        // track it.
        Request<Bitmap> newRequest = makeImageRequest(requestUrl, maxWidth, maxHeight, scaleType, cacheKey);

        mRequestQueue.add(newRequest);
        mInFlightRequests.put(cacheKey, new BatchedImageRequest(newRequest, imageContainer));
        return imageContainer;
    }
```
前面说了，get有三个重载方法，不过最后都是调用到这个上面。首先判断get方法是否在主线程中执行。
```
private void throwIfNotOnMainThread()
    {
        if (Looper.myLooper() != Looper.getMainLooper())
        {
            throw new IllegalStateException("ImageLoader must be invoked from the main thread.");
        }
    }
```
判断的规则很简单，就是判断当前线程的Looper是否等于主线程的Looper。
接着使用getCacheKey方法构造一个cacheKey：
```
 private static String getCacheKey(String url, int maxWidth, int maxHeight, ScaleType scaleType)
    {
        return new StringBuilder(url.length() + 12).append("#W").append(maxWidth).append("#H").append(maxHeight).append("#S").append(scaleType.ordinal()).append(url).toString();
    }
```
这段代码用url，指定的宽度，高度，ScaleType来构造一个String对象。
比如我们的url为：http://www.baidu.com/1.jpg
maxWdith=200;
maxHeight=400;
ScaleType=ScaleType.CENTER_CROP
构造的结果为：
"38#W200#H400#S6http://www.baidu.com/1.jpg"
scaleType.ordinal返回的是当前模式在枚举中的位置，CENTER_CROP为6。

接下来使用mCache.getBitmap(cacheKey)从缓存中得到一个Bitmap对象，getBitmap这个方法我前面简单提到过，我们当时是空实现。如果得到的bitmap对象不为空，那么我们就构造一个ImageContainer对象，这个ImageContainer对象是什么鬼？简单来说，它是对图片请求相关数据的一个包装。我们来看看它定义的成员变量和构造方法：
```
public class ImageContainer
    {
        /**
         * The most relevant bitmap for the container. If the image was in cache, the
         * Holder to use for the final bitmap (the one that pairs to the requested URL).
         */
        private Bitmap mBitmap;

        private final ImageListener mListener;

        /** The cache key that was associated with the request */
        private final String mCacheKey;

        /** The request URL that was specified */
        private final String mRequestUrl;

        /**
         * Constructs a BitmapContainer object.
         * @param bitmap The final bitmap (if it exists).
         * @param requestUrl The requested URL for this container.
         * @param cacheKey The cache key that identifies the requested URL for this container.
         */
        public ImageContainer(Bitmap bitmap, String requestUrl, String cacheKey, ImageListener listener)
        { 
            mBitmap = bitmap;
            mRequestUrl = requestUrl;
            mCacheKey = cacheKey;
            mListener = listener;
        }
        ...
}
```
在构造方法中对成员变量进行了赋值，包括bitmap对象，要回调的接口ImageListener,缓存key,请求url。

如果我在从缓存中得到的bitmap对象不为空的话，那么直需要把从缓存得到的bitmap和请求url传入ImageContainer中进行包装，然后回调ImageListener接口的onResponse方法，该方法会把bitmap对象显示在ImageView上面，这在前面已经提到过了，之后返回ImageContainer对象。

如果缓存中没有bitmap对象，也就是cacheDBitmap对象为null,那么我们创建ImageContainer对象时，就不用传入bitmap作为参数，但是请求的url,缓存key,监听接口都要传入。就算这时候没有bitmap对象，我们也要回调ImageListener中的onResponse方法，作用当然是先显示默认的图片。

接下来这句代码可能会让你有点懵
```
BatchedImageRequest request = mInFlightRequests.get(cacheKey);
```
因为mInFlightRequests和BatchedImageRequest是什么，我们并没有介绍过，现在一起来看看。
```
private final HashMap<String, BatchedImageRequest> mInFlightRequests = new HashMap<String, BatchedImageRequest>();
```
以上是mInFlightRequests的定义，我们可以看到它是一个HashMap对象，key是String类型，value是BatchedImageRequest类型。接着我们看一下BatchedImageRequest的源码:
```
private class BatchedImageRequest
    {
        /** The request being tracked */
        private final Request<?> mRequest;

        /** The result of the request being tracked by this item */
        private Bitmap mResponseBitmap;

        /** Error if one occurred for this response */
        private VolleyError mError;

        /** List of all of the active ImageContainers that are interested in the request */
        private final LinkedList<ImageContainer> mContainers = new LinkedList<ImageContainer>();

        /**
         * Constructs a new BatchedImageRequest object
         * @param request The request being tracked
         * @param container The ImageContainer of the person who initiated the request.
         */
        public BatchedImageRequest(Request<?> request, ImageContainer container)
        {
            mRequest = request;
            mContainers.add(container);
        }

        /**
         * Set the error for this response
         */
        public void setError(VolleyError error)
        {
            mError = error;
        }

        /**
         * Get the error for this response
         */
        public VolleyError getError()
        {
            return mError;
        }

        /**
         * Adds another ImageContainer to the list of those interested in the results of
         * the request.
         */
        public void addContainer(ImageContainer container)
        {
            mContainers.add(container);
        }

        /**
         * Detatches the bitmap container from the request and cancels the request if no one is
         * left listening.
         * @param container The container to remove from the list
         * @return True if the request was canceled, false otherwise.
         */
        public boolean removeContainerAndCancelIfNecessary(ImageContainer container)
        {
            mContainers.remove(container);
            if (mContainers.size() == 0)
            {
                mRequest.cancel();
                return true;
            }
            return false;
        }
    }
```
BatchedImageRequest是在ImageLoader中定义的内部类，它是一个包装类。从成员变量中我们可以看到它包装了一个Request对象，还有请求返回的bitmap，处理错误的VelleyError，存放ImageContainer的LinkedList集合。内部的方法都很简单，无非都是set,get类似的方法，这里就不细讲了。

回到ImageLoader的get方法中，如果从mInFlightRequest.get方法中得到的BatchedImageRequest对象不为空，说明已经有相同的请求了，那么我们就把包装了该次请求的ImageContainer加入到BatchedImageRequest当中的LinkedList集合中去并返回该ImageContainer。
如果从mInFlightRequest.get方法中得到的BatchedImageRequest为空，那么就创建一个新的ImageRequest请求，执行makeImageRequest方法:
```
protected Request<Bitmap> makeImageRequest(String requestUrl, int maxWidth, int maxHeight, ScaleType scaleType, final String cacheKey)
    {
        return new ImageRequest(requestUrl, new Listener<Bitmap>()
        {
            @Override
            public void onResponse(Bitmap response)
            {
                onGetImageSuccess(cacheKey, response);
            }
        }, maxWidth, maxHeight, scaleType, Config.RGB_565, new ErrorListener()
        {
            @Override
            public void onErrorResponse(VolleyError error)
            {
                onGetImageError(cacheKey, error);
            }
        });
    }
```
makeImageRequest方法很简单，只是直接new了一个ImageRequest对象并返回，ImageRequest对象已经在上篇文章中分析过了。我们主要来看当成功返回以后调用的onGetImageSuccess方法：
```
    protected void onGetImageSuccess(String cacheKey, Bitmap response)
    {
        // cache the image that was fetched.
        mCache.putBitmap(cacheKey, response);

        // remove the request from the list of in-flight requests.
        BatchedImageRequest request = mInFlightRequests.remove(cacheKey);

        if (request != null)
        {
            // Update the response bitmap.
            request.mResponseBitmap = response;

            // Send the batched response
            batchResponse(cacheKey, request);
        }
    }
```
成功返回图片后，做了这么几件事。首先使用ImageCache的putBitmap方法把返回的结果缓存起来。其次从mInFlightRequests中移除并得到与cachekey有关的BatchedImageRequest对象，如果得到的对象不为空，那么把我们把成功返回的bitmap对象放入BatchedImageRequest。接着调用batcheResponse方法：
```
 private void batchResponse(String cacheKey, BatchedImageRequest request)
    {
        mBatchedResponses.put(cacheKey, request);
        // If we don't already have a batch delivery runnable in flight, make a new one.
        // Note that this will be used to deliver responses to all callers in mBatchedResponses.
        if (mRunnable == null)
        {
            mRunnable = new Runnable()
            {
                @Override
                public void run()
                {
                    for (BatchedImageRequest bir : mBatchedResponses.values())
                    {
                        for (ImageContainer container : bir.mContainers)
                        {
                            // If one of the callers in the batched request canceled the request
                            // after the response was received but before it was delivered,
                            // skip them.
                            if (container.mListener == null)
                            {
                                continue;
                            }
                            if (bir.getError() == null)
                            {
                                container.mBitmap = bir.mResponseBitmap;
                                container.mListener.onResponse(container, false);
                            } else
                            {
                                container.mListener.onErrorResponse(bir.getError());
                            }
                        }
                    }
                    mBatchedResponses.clear();
                    mRunnable = null;
                }

            };
            // Post the runnable.
            mHandler.postDelayed(mRunnable, mBatchResponseDelayMs);
        }
    }
```
在看方法之前我们先思考一下，在mInFlightRequests中根据cachekey的不同存放了不同的BatchedImageRequest对象，而BatchedImageRequest对象中的LinkedList集合中又存放了相同cachekey的请求。当其中一个请求被处理成功以后，那么其余相同的请求该怎么办？是全部放入网络队列中去等待处理？这显然是没有必要的，既然请求的图片都是相同的，那么我们直接把成功返回的图片转给其他一样的请求不就行了吗？带着这样的想法我们一起去看batchResponse这个方法。

mBatchedResonses是从哪冒出来的？我们看看它的定义：
```
private final HashMap<String, BatchedImageRequest> mBatchedResponses = new HashMap<String, BatchedImageRequest>();
```
从定义我们发现它跟mInFilghtRequests一样，都是用来保存BatchedImageRequest。方法一开始就使用put方法把从onGetImageSuccess方法中传过来的cachekey和BatchedImageRequest存入mBatchedResponses中。接着判断mRunnable是否为空，如果为空就创建一个Runnable对象，在run方法遍历mBatchedResponses，然后再遍历每个BatchedImageRequest中的LinkedList集合，得到ImageContainer对象，如果ImageContainer中有传入回调接口并且VolleyError的值为空，那么就回调接口中的onResponse方法，并把成功返回的图片传入。这跟我们前面说的思想是一样的。出错的话就回调onErrorResponse方法，把VolleyError传入。遍历完成以后清空mBatchedResponse，把mRunnable也置为空。最后使用Handler的postDelayed方法发送到主线程的MessageQueue中。为什么知道发送到主线程呢？因为Handler的定义：
```
private final Handler mHandler = new Handler(Looper.getMainLooper());
```
到这ImageRequest中回调的onResponse方法要执行的代码解析完了，onErrorResponse的过程也大体一致，这里就不跟进了，大家可以自己看看，很简单。

现在回到ImageLoader的get方法继续看，执行完makeImageRequest方法以后会返回一个Request对象。
```
        Request<Bitmap> newRequest = makeImageRequest(requestUrl, maxWidth, maxHeight, scaleType, cacheKey);

        mRequestQueue.add(newRequest);
        mInFlightRequests.put(cacheKey, new BatchedImageRequest(newRequest, imageContainer));
        return imageContainer;
```
把这个请求加入到RequestQueue中，再创建一个BatchedImageRequest对象加入到mInFlightRequests中,最后返回ImageContainer。

到此我们已经把整个ImageLoader的使用流程解析了一遍。最主要的都集中在了get方法中。我们把get方法的逻辑再理一遍，首先我们要确保get方法是在主线程中执行，然后通过参数创建了一个cacheKey，有了cacheKey我们就尝试从缓存中去获取数据，如果有数据那么我们就直接回调ImageListener中的onResponse方法显示图片。如果没有数据，我们有也还是要回调ImageListner中的onResponse方法先显示默认的图片，完了我们去mInFlightRequests这个HasheMap中通过cacheKey获取BatchedImageRequest，如果获取到，说明已有相同的cacheKey存在，那么只需要把创建好的ImageContainer对象放入BatchedImageRequest的LinkedList集合中，等上一个相同cacheKey的请求执行完毕返回图片以后，会遍历LinkedList集合，把返回的图片传给ImageContainer，然后通过ImageContainer中的回调接口把图片显示。如果通过cacheKey没有在mInFlightRequests中得到BatchedImageRequest对象，说明是第一次发起该请求，那么我们就创建一个ImageRequest对象，加入到RequestQueue中等待执行，并且创建一个BatchedImageRequest对象存入mInFlightRequests中。这样下次有相同cacheKey时获取到的BatchedImageRequest就不为空了。

##ImageCache的实现
上面的代码中，我们只提到ImageCache是用来实现缓存操作的，但是我们并没有去实现它，这里我们采用LruCache和DiskLruCache来实现二级缓存。
```
public class ImageCacheUtil implements ImageLoader.ImageCache
{
    private static final String TAG=ImageCacheUtil.class.getSimpleName();
    private static final int DISK_CACHE_INDEX=0;
    private static final long DISK_CACHE_SIZE=1024*1024*50;//设备缓存大小为50M
    private boolean mIsDiskLruCacheCreated=false;

    private LruCache<String,Bitmap> mMemoryCache;
    private DiskLruCache mDiskLruCache;
    private Context mContext;


    public ImageCacheUtil(Context context)
    {
        mContext=context.getApplicationContext();
        //对LRUCache的初始化
        int maxMemory=(int) (Runtime.getRuntime().maxMemory()/1024);
        int cacheSize=maxMemory/8;
        mMemoryCache=new LruCache<String,Bitmap>(cacheSize)
        {
            //完成bitmap大小的计算
            @Override
            protected int sizeOf(String key, Bitmap value)
            {
                return value.getRowBytes()*value.getHeight()/1024;
            }
        };
        //对DiskLRUCache的初始化
        File diskCacheDir=getDiskCacheDir(mContext, "bitmap");
        if(!diskCacheDir.exists())
        {
            diskCacheDir.mkdirs();
        }
        if(getUsableSpace(diskCacheDir)>DISK_CACHE_SIZE)
        {
            try{
                mDiskLruCache=DiskLruCache.open(diskCacheDir,1,1,DISK_CACHE_SIZE);
                mIsDiskLruCacheCreated=true;
            }
            catch (IOException e)
            {
                e.printStackTrace();
            }
        }


    }

    @Override
    public void putBitmap(String url, Bitmap bitmap)
    {
        //加入内存缓存
        addBitmapToMemoryCache(url,bitmap);
        try
        {
            if (loadBitmapFromDiskCache(url) == null)
            {
                //加入设备缓存
                addBitmapToDiskCache(url,bitmap);
            }
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }



    @Override
    public Bitmap getBitmap(String url)
    {
        //先从内存缓存中获取，如果不存在再从设备缓存中获取
        Bitmap bitmap=loadBitmapFromMemCache(url);
        if(bitmap!=null)
        {
            Log.i(TAG, "从内存缓存获取");
            return bitmap;
        }
        try
        {
            bitmap=loadBitmapFromDiskCache(url);
            return bitmap;
        }catch (IOException e)
        {
            e.printStackTrace();
        }
        return null;
    }

    /**
     *  添加Bitmap对象到设备缓存
     * */
    private void addBitmapToDiskCache(String url,Bitmap bitmap) throws IOException
    {
        DiskLruCache.Editor editor=mDiskLruCache.edit(url);
        if(editor!=null)
        {
            OutputStream outputStream=editor.newOutputStream(DISK_CACHE_INDEX);
            if(bitmap.compress(Bitmap.CompressFormat.JPEG,100,outputStream))
            {
                editor.commit();
            }
            else
            {
                editor.abort();
            }
        }
        mDiskLruCache.flush();
    }


    /**
     * 从设备缓存中获取Bitmap对象
     * */
    private Bitmap loadBitmapFromDiskCache(String url) throws IOException
    {
        Bitmap bitmap=null;
        DiskLruCache.Snapshot snapshot=mDiskLruCache.get(url);
        if(snapshot!=null)
        {
            bitmap= BitmapFactory.decodeStream(snapshot.getInputStream(DISK_CACHE_INDEX));
            if(bitmap!=null)
            {
                //把bitmap添加进内存缓存
                addBitmapToMemoryCache(url, bitmap);
                Log.i(TAG,"从存储设备缓存获取");
            }
        }
        return bitmap;
    }

    /**
     * 添加Bitmap到内存缓存
     * */
    private void addBitmapToMemoryCache(String url, Bitmap bitmap)
    {
        if(loadBitmapFromMemCache(url)==null)
        {
            mMemoryCache.put(url,bitmap);
        }
    }

    /**
     * 从内存缓存中得到Bitmap
     * */
    private Bitmap loadBitmapFromMemCache(String url)
    {
        return mMemoryCache.get(url);
    }



    /**
     * 得到可用空间大小
     * */
    private long getUsableSpace(File path)
    {
        if(Build.VERSION.SDK_INT>= Build.VERSION_CODES.GINGERBREAD)
        {
            return path.getUsableSpace();
        }
        final StatFs stats=new StatFs(path.getPath());
        return stats.getBlockSizeLong()*stats.getAvailableBlocksLong();
    }

    /**
     * 得到缓存文件夹
     * */
    public File getDiskCacheDir(Context context,String uniqueName)
    {
        boolean externalStorageAvailable= Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED);
        final String cachePath;
        //判断设备存储器是否可用，如果可用使用私有存储，如果不可用使用内部存储
        if(externalStorageAvailable)
        {
            cachePath=context.getExternalCacheDir().getPath();
        }
        else
        {
            cachePath=context.getCacheDir().getPath();
        }
        return new File(cachePath+File.separator+uniqueName);
    }
}
```
我们自己写了一个工具类来继承ImageCache，LruCache使用的是V4包下的，DiskLruCache需要大家自己去下载。上面的代码很简单，都是围绕着getBitmap和putBitmap来实现，我就不分析了。这里提供的只是一个示例代码，大家可以根据自己的需求来更改实现。好了，这次的源码分析就到这啦。

本文为原创文章，转载请注明出处:http://www.jianshu.com/p/cb6ba58f9f3e