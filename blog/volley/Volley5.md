Volley都干了些什么？（四）

#目录
 (一) [CacheDispatcher的解析](http://www.jianshu.com/p/85bcd878a676)
 (二) [NetworkDispatcher的解析](http://www.jianshu.com/p/0a0dc214c2c0)
 (三) [RequestQueue中add方法的分析](http://www.jianshu.com/p/4dbc76e3f548)
 (四) [ImageRequest的分析](http://www.jianshu.com/p/d6609049325d)
 (五) [ImageLoader的分析](http://www.jianshu.com/p/cb6ba58f9f3e)
#前言
前几篇文章分析了通过StringRequest来发起一个请求，Volley都干了些什么。今天我们来分析一下Volley中的ImageRequest。
#ImageRequest的分析
ImageRequest的使用与StringRequest的使用非常的相似。
首先我们需要建立一个RequestQueue对象：
```
RequestQueue mQueue=Volley.newRequestQueue(context);
```
其次需要建立一个ImageRequest对象：
```
ImageRequest imageRequest=new ImageRequest(url,new Response.Listener<Bitmap>()
{
       @override
      public void onResponse(Bitmap response)
      {
        //这里进行成功返回Bitmap后的操作
      }
},0,0,Config.ARGB_8888,new Response.ErrorListener()
{
      @override
      public void onErrorResponse(VolleyError error)
      {
          //如果返回出错，在这里进行处理
      }
}
)；
```
这段代码相信大家并不陌生，url指明了请求的地址，onResponse,onErrorResponse分别是成功返回和返回出错的回调，第3个，第4个参数标明用给定的宽高去解码所得到的图片，如果设置的宽高小于图片原始的宽高，就对图片进行压缩，如果设置为0,0那么就不对图片做任何压缩处理，关于这一点在接下来的文章中会有详细的说明。第5个参数是图片的编码格式。常用的主要由ARGB_8888，RGB_565,ARGB_8888使用4个字节，其中ARGB各占8位。RGB_565使用2个字节，R占5位，G占6位，B占5位一共16位。使用ARGB_8888来对图片编码肯定会比使用RGB_565质量更好，但是使用的空间也会更大，具体使用哪种编码格式，大家自己权衡，Picasso默认使用ARGB_8888，Glide默认使用RGB_565。

请求队列和请求都创建好以后，我们要做的就是把请求放入请求队列中：
```
mQueue.add(imageRequest);
```
以上就是ImageRequest的使用，是不是很简单？接下来我们就要分析一下，看似简单的背后，Volley为我们做了些什么？

在前几篇文章中我们已经对Volley的执行流程分析了一遍，上面代码的执行流程也是一样的，唯一不同的是我们这次使用的是ImageRequest而不是StringRequest。通过前几篇文章我们指定，不论是缓存调度器还是网络调度器，在其run方法中都会执行request.parseNetworkResponse方法。这个方法会在不同的请求中重写，那我们就来看看ImageRequest中的parseNetworkResponse方法：
```
 @Override
    protected Response<Bitmap> parseNetworkResponse(NetworkResponse response)
    {
        // Serialize all decode on a global lock to reduce concurrent heap usage.
        synchronized (sDecodeLock)
        {
            try
            {
                return doParse(response);
            } catch (OutOfMemoryError e)
            {
                VolleyLog.e("Caught OOM for %d byte image, url=%s", response.data.length, getUrl());
                return Response.error(new ParseError(e));
            }
        }
    }
```
这个方法很简单，直接调用了doParse方法：
```
 private Response<Bitmap> doParse(NetworkResponse response)
    {
        byte[] data = response.data;
        BitmapFactory.Options decodeOptions = new BitmapFactory.Options();
        Bitmap bitmap = null;
        //如果在构造ImageRequest时，传入的宽高为0
        if (mMaxWidth == 0 && mMaxHeight == 0)
        {
            decodeOptions.inPreferredConfig = mDecodeConfig;
            bitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
        } else
        {
            // If we have to resize this image, first get the natural bounds.
            decodeOptions.inJustDecodeBounds = true;
            BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);
            //得到原始图片的长度和宽度
            int actualWidth = decodeOptions.outWidth;
            int actualHeight = decodeOptions.outHeight;

            // Then compute the dimensions we would ideally like to decode to.
            int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight, actualWidth, actualHeight, mScaleType);
            int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth, actualHeight, actualWidth, mScaleType);

            // Decode to the nearest power of two scaling factor.
            decodeOptions.inJustDecodeBounds = false;
            // TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
            // decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
            decodeOptions.inSampleSize = findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
            Bitmap tempBitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);

            // If necessary, scale down to the maximal acceptable size.
            if (tempBitmap != null && (tempBitmap.getWidth() > desiredWidth || tempBitmap.getHeight() > desiredHeight))
            {
                bitmap = Bitmap.createScaledBitmap(tempBitmap, desiredWidth, desiredHeight, true);
                tempBitmap.recycle();
            } else
            {
                bitmap = tempBitmap;
            }
        }

        if (bitmap == null)
        {
            return Response.error(new ParseError(response));
        } else
        {
            return Response.success(bitmap, HttpHeaderParser.parseCacheHeaders(response));
        }
    }
```
首先response.data得到返回的数据，然后创建一个BitmapFactory.Option对象，通过这个对象可以对图片进行编码。mMaxWidth与mMaxHeight就是我们在创建ImageRequest的时候传入的第3,4个参数，如果当时传入的都是0，那么把mDecodeConfig的值赋给decodeOptions.inPreferredConfig,inPreferredConfig表示对图片进行编码时所采用的颜色模式，我们当时使用的是ARGB_8888。然后通过BitmapFactory.decodeByteArray方法得到一个bitmap对象。最后如果这个bitmap对象不为空，就调用Response.success方法包装成一个Response对象返回。

如果我们在创建ImageRequest对象时传入了一个宽，高值，那么我们就要对网络所返回的原始图片进行编码了，因为有可能原始图片宽、高比我们所指定的宽、高要大。

对图片重新编码的第一步就是要把decodeOption.inJustDecodeBounds的值设置为true,这个时候使用BitmapFactory.decodeByteArray只会解析图片的原始宽、高信息，并不会真正的加载图片。然后通过decodeOptions.outWidth、decodeOptions.outHeight得到原始图片的宽高。得到原始图片的实际宽高以后，我们就要进入一个非常重要的方法getResizedDimension（），解析ImageRequest源码，不讲这个方法的都是耍流氓（玩笑）。。。
```
 private static int getResizedDimension(int maxPrimary, int maxSecondary, int actualPrimary, int actualSecondary, ScaleType scaleType)
    {

        // If no dominant value at all, just return the actual.
        //如果没有指定大小，则使用原图的大小
        if ((maxPrimary == 0) && (maxSecondary == 0))
        {
            return actualPrimary;
        }

        // If ScaleType.FIT_XY fill the whole rectangle, ignore ratio.
        if (scaleType == ScaleType.FIT_XY)
        {
            if (maxPrimary == 0)
            {
                return actualPrimary;
            }
            return maxPrimary;
        }

        // If primary is unspecified, scale primary to match secondary's scaling ratio.
        if (maxPrimary == 0)
        {
            double ratio = (double) maxSecondary / (double) actualSecondary;
            return (int) (actualPrimary * ratio);
        }

        if (maxSecondary == 0)
        {
            return maxPrimary;
        }

        //得到原图的宽高比
        double ratio = (double) actualSecondary / (double) actualPrimary;
        int resized = maxPrimary;

        // If ScaleType.CENTER_CROP fill the whole rectangle, preserve aspect ratio.
        if (scaleType == ScaleType.CENTER_CROP)
        {
            if ((resized * ratio) < maxSecondary)
            {
                resized = (int) (maxSecondary / ratio);
            }
            return resized;
        } 
        if ((resized * ratio) > maxSecondary)
        {
            resized = (int) (maxSecondary / ratio);
        }
        return resized;
    }
```
这个方法不长，却包含了很多种情况。
1.在构建ImageRequest时没有指定宽高的情况
2.ScaleType为FIT_XY的情况
3.在构建ImageRequest时指定了宽高，但是其中一个为0的情况
4.ScaleType为CENTER_CROP的情况
5.其他情况
可以看见就这么一个小小的方法就包含了5种情况，我打算采用举例子的方式来逐一分析这5种情况。首先我们来看是如何调用该方法的？
```
//mMaxWidth指定宽度 ，mMaxHeight指定的高度，actualWidth原图的宽度，actualHeight原图的高度，mScaleType比例类型
int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight, actualWidth, actualHeight, mScaleType);
int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth, actualHeight, actualWidth, mScaleType);
```
我们从传入的参数位置可以发现，我们要得到desiredWidth那么我们就把指定的宽度和原图的宽度放在前面，我们要得到desiredHeight就把指定的高度和原图的高度放在前面。
```
getResizedDimension(int maxPrimary, int maxSecondary, int actualPrimary, int actualSecondary, ScaleType scaleType)
```
与getResizedDimension对比来看，要得到desiredWidth时，
maxPrimary=mMaxWidth //指定宽度
maxSecondary=mMaxHeight //指定高度
actualPrimary=actualWidth //原图宽度
actualSecondary=actualHeight //原图高度
scaleType=mScaleType //比例类型
要得到desiredWidth时只要把指定宽高和原图宽高调换一下位置。

知道参数传入的顺序很重要，这会直接影响到返回的结果。
我们假设原图的实际大小为宽300，高600。
现在我们来看第一种情况，在构建ImageRequest时没有指定宽高，也就是maxPrimary=mMaxWidth=0,maxSecondary=mMaxHeight=0的情况。
```
       if ((maxPrimary == 0) && (maxSecondary == 0))
        {
            return actualPrimary;
        }
```
由于maxPrimary，maxSecondary 都为0，所以就直接返回actualPrimary的值，所以这时的desiredWidth=300;同理可得desiredHeight=600;由此我们可以指定如果我们在构建ImageRequest时没有指定宽高，那么就以原图的实际宽高来显示。

接下来看第二种情况---ScaleType为FIT_XY:
我们假设指定的宽为200，高为400,scaleType为FIT_XY。
```
        if (scaleType == ScaleType.FIT_XY)
        {
            if (maxPrimary == 0)
            {
                return actualPrimary;
            }
            return maxPrimary;
        }
```
由于我们指定的宽高都不为0，所以最后desiredWidth=200,desiredHeight=400。

我们假设指定宽为0，高为400，scaleType为FIT_XY，这时desiredWidth=300,desiredHeight=400。

从这里我们可以总结出ScaleType为FIT_XY的三个特性：
1.如果指定的宽高都不为0，那么返回指定宽高
2.如果指定宽高中有其中一个为0，那么用原图的宽高来代替。
3.模式为FIT_XY且宽高其一未指定时不保存原图的宽高比

第三种情况，在构建ImageRequest时指定了宽高，但是其中一个为0的情况：
假设指定的宽为0，高为200，scaleType不为FIT_XY（注意这里0代表未指定）
```
      if (maxPrimary == 0)
        {
            double ratio = (double) maxSecondary / (double) actualSecondary;
            return (int) (actualPrimary * ratio);
        }

        if (maxSecondary == 0)
        {
            return maxPrimary;
        }
```
先来计算desiredWidth的值，这时候maxPrimary==0，那么ratio=maxSecondary 指定高度/actualSecondary原图高度,然后再通过原图的宽度去乘以ratio得到宽度。这里的ratio其实不叫宽高比，而是指定高度跟原图高度的比例,ratio=200/600。最后desiredWidth=100;

接着计算desiredHeight的值，这时候maxPrimary=200,maxSecondary=0;那么就直接返回maxPrimary，也就是desiredHeight=maxPrimary=200;

第四种情况，ScaleType为CENTER_CROP：
假设指定宽度为100，高为500，scaleType为CENTER_CROP,那么执行的就是以下逻辑
```
      //原图的宽高比
        double ratio = (double) actualSecondary / (double) actualPrimary;
        int resized = maxPrimary;

        // If ScaleType.CENTER_CROP fill the whole rectangle, preserve aspect ratio.
        if (scaleType == ScaleType.CENTER_CROP)
        {
            if ((resized * ratio) < maxSecondary)
            {
                resized = (int) (maxSecondary / ratio);
            }
            return resized;
        }
```
还是先来计算desiredWidth的值，首先得到原图的宽高比，ratio=600/300（注意这里是用原图的高度除以原图的宽度）,resized=100;resized*ratio的值为200，小于maxSecondary=500;那么resized=500/2 最后desiredWidth=250。

接着算desiredHeight的值，这时候
maxPrimary=500,
maxSecondary=100,
actualPrimary=600,
actualSecondary=300,
ratio=300/600,
resized=500,
resized*ratio不小于maxSecondary，所以直接返回resized,desiredHeight=500。

如果第四种情况只讲到这里的话，就没有任何意义，因为计算谁都会。我们这里要探讨的是代码为什么要这么写？我们这里假设maxPrimary是指定的宽，maxSecondary是指定的高，actualPrimary是原图的宽，actualSecondary是原图的高。那么actualSecondary/actualPrimary得到了原图的宽高比ratio。这里都容易理解，接着判断了scaleType是否为CENTER_CROP，为什么要判断呢？肯定因为CENTER_CROP比较特殊，那么特殊的地方在哪？看谷歌官方对CENTER_CROP的解释：

**Scale the image uniformly (maintain the image's aspect ratio) so that both dimensions (width and height) of the image will be equal to or larger than the corresponding dimension of the view (minus padding). **
这里有个关键词equal to or larger than，等于或者大于。有了这个条件我们就比较容易理解为什么要判断
(resized * ratio) < maxSecondary，我们看上面计算desiredWidth的例子，原图宽高比算出来是2，resized=100，resized*ratio的意思是用指定的宽乘以原图的宽高比得到一个指定的高度，这样指定图片的宽高比例会跟原图的宽高比例一致。由于resized*ratio的值为200小于指定的高度500，那么我们就需要用指定的高度来除以宽高比，逆向得到宽度（宽*比例=高度，高度/比例=宽度），也就是说如果我们按照100这个宽度来得到高度的话，得到的结果为200，小于我们需要的500，这就不满足谷歌官方的条件了，谷歌说明使用CENTER_CROP模式，图片在缩放的时候必须要大于等于指定的宽高，所以为了满足谷歌的条件，我们只能反推，用500/2得到250这个宽度而不能用100这个宽度，这样最后的结果---宽度250也大于指定的100，高度为指定的500。（说明：如果把指定的宽高100,500看作是ImageView的宽高的话，如果使用100的宽度，按照宽高比来得到高为200就不满足CENTER_CROP的要求，因为高度没有大于或者等于指定高度，所以宽度不能用100这个值）
这一段有点难理解，大家可以多思考思考，消化一下。

第五种情况，除了以上说的情况以外全都属于这一类
假设指定宽度为200，高度为300，scaleType为CENTER_INSIDE
```
//得到原图宽高比
double ratio = (double) actualSecondary / (double) actualPrimary;
int resized = maxPrimary;
if ((resized * ratio) > maxSecondary)
{
      resized = (int) (maxSecondary / ratio);
}
      return resized;
```
以上就是从getResizedDimension方法中抽取出来的逻辑，我们先来计算desiredWidth,ratio=600/300,
resized=200,resized*ratio大于maxSecondary,所以resized= (int) (maxSecondary / ratio);最后desiredWidth=150;同理我们可以算出desiredHeight=300。
原图的高度除以宽度为2，我们得到的desiredHeight除以desiredWidth也等于2，所以宽高比跟原图是一样的。
关键点来了，这里为什么要判断(resized * ratio) > maxSecondary？通过情况4我们应该可以推测出跟scaleType这个参数有关系，以下是谷歌官方说明：
**Scale the image uniformly (maintain the image's aspect ratio) so that both dimensions (width and height) of the image will be equal to or less than the corresponding dimension of the view (minus padding). **
我们可以看到这种模式的要求是小于或者等于，跟CENTER_CROP的大于或者等于不一样，所以前面要单独来判断是否是CENTER_CROP模式。具体的我就不再分析了，无非是指定的宽度乘以原图得到的宽高比以后发现得到的高度大于指定的高度了，所以不能使用这个宽度，只能使用给定的高度来处于宽高比得到一个宽度。这里值得注意的是，如果在构建ImageRequest时没有指定ScaleType那么默认为CENTER_INSIDE模式。

到这里为止我们就解析完了getResizedDimension方法，也成功得到了desiredWidth和desiredHeight。从代码中我们可以看出，如果在构建ImageRequest时没有指定宽高，那么就使用原始图片实际宽高，如果指定了宽高，最后得到的结果也不一定是指定的，Volley会根据原图的宽高比还有使用的比例模式来生成一个最优宽高。

我们接下来回到doParse方法
```
            ....
            int desiredWidth = getResizedDimension(mMaxWidth, mMaxHeight, actualWidth, actualHeight, mScaleType);
            int desiredHeight = getResizedDimension(mMaxHeight, mMaxWidth, actualHeight, actualWidth, mScaleType);

            // Decode to the nearest power of two scaling factor.
            decodeOptions.inJustDecodeBounds = false;
            // TODO(ficus): Do we need this or is it okay since API 8 doesn't support it?
            // decodeOptions.inPreferQualityOverSpeed = PREFER_QUALITY_OVER_SPEED;
            decodeOptions.inSampleSize = findBestSampleSize(actualWidth, actualHeight, desiredWidth, desiredHeight);
            Bitmap tempBitmap = BitmapFactory.decodeByteArray(data, 0, data.length, decodeOptions);

            // If necessary, scale down to the maximal acceptable size.
            if (tempBitmap != null && (tempBitmap.getWidth() > desiredWidth || tempBitmap.getHeight() > desiredHeight))
            {
                bitmap = Bitmap.createScaledBitmap(tempBitmap, desiredWidth, desiredHeight, true);
                tempBitmap.recycle();
            } else
            {
                bitmap = tempBitmap;
            }
        }

        if (bitmap == null)
        {
            return Response.error(new ParseError(response));
        } else
        {
            return Response.success(bitmap, HttpHeaderParser.parseCacheHeaders(response));
        }
}
```
我们得到desiredWidth和desiredHeight后，把decodeOptions.inJustDecodeBounds的值设为false，这样下次加载才会真正的加载图片。接下来findBestSampleSize方法可以得到decodeOptions.inSampleSize的值，通过BitmapFactory.Options来缩放图片，主要是通过inSampleSize参数来确定缩放程度，inSampleSize也就是采样率，当值小于等于1时，采样后的图片跟原图一样，也就是没有缩放效果，当inSampleSize的值大于1，比如为2时，那么采样后的宽、高均为原图的1/2，而图片大小为原图的1/4大小。谷歌官方建议inSampleSize的值最好为2的指数，比如2的一次方，2的二次方，2的三次方等。。。当inSampleSize的值为2的三次方也就是8时，压缩后的图片大小为原图的1/64。也就是说压缩比例为1/inSampleSize的二次方。我们看看findBestSampleSize方法的实现：
```
static int findBestSampleSize(int actualWidth, int actualHeight, int desiredWidth, int desiredHeight)
    {
        double wr = (double) actualWidth / desiredWidth;
        double hr = (double) actualHeight / desiredHeight;
        double ratio = Math.min(wr, hr);
        float n = 1.0f;
        while ((n * 2) <= ratio)
        {
            n *= 2;
        }

        return (int) n;
    }
```
这个方法很短，逻辑也很简单，用原图的宽度除以我们计算后的指定宽度，用原图高度除以我们计算后的指定高度，按正常情况下这两个比值是一样的，也就是wr==hr的，只有一种情况，就是ScaleType为FIT_XY且宽或者高之一未指定时，wr!=hr。为什么呢？你想，我们计算后的宽度和高度都是按照原图的宽高比得到的,所以原图的宽高比跟计算后的宽高比也应该是一样的。
desiredWidth/desiredHeight=actualWidth/actualHeight 
这个数学公式大家都能看懂，变化以后
actualWidth / desiredWidth=actualHeight / desiredHeight
那为什么下面还会有Math.min(wr, hr)这段代码呢？就是因为在ScaleType为FIT_XY且宽或者高之一未指定时，计算后的宽与高不是按原图的宽高比得到的。这时候wr就不等于hr了，于是就要判断wr和hr谁更小。为什么判断谁小而不是谁大呢？我的想法是避免过度的缩放。
可以看见我们n的值是每次乘以2增长的，这也是为了遵循谷歌的建议，inSampleSize的值最好为2的指数。

得到inSampleSize的值后，我们再用decodeByteArray方法去压缩图片得到一个Bitmap对象，如果压缩后的bitmap不为空且宽度，高度都还大于我们计算后的指定宽高，那么我们再用createScaledBitmap方法去裁剪这个Bitmap对象，得到一个新的bitmap。注意createScaledBitmap这个方法，如果传入的宽高跟要裁剪Bitmap的宽高一致，那么就直接返回原bitmap，而不会创建新的bitmap对象。

如果压缩后的bitmap宽高小于或者等于计算后的指定宽高，那么就直接执行bitmap=tempBitmap；在bitmap不为null时，将其包装成一个Response对象返回。

这里思考一个问题，为什么不直接在一开始用BitmapFactory.decodeByteArray直接生成一个原图的bitmap，然后计算出desiredWidth，desiredHeight值后，直接用Bitmap.createScaledBitmap(bitmap, desiredWidth, desiredHeight, true);来得到最后的bitmap对象。而是要去计算采样率，然后通过采样率先去压缩一次图片？这个问题希望有大神帮忙解答一下。

到目前为止doParse方法我们也解析完了，这也代表着parseNetworkResponse方法的结束，由前几篇文章我们知道，最后会在ExetutorDelivery中的内部类ResponseDeliveryRunnable的run方法中调用imageRequest的deliveryResponse方法，imageRequest的deliveryResponse方法实现很简单：
```
@Overrideprotected void deliverResponse(Bitmap response){    mListener.onResponse(response);}
```
方法中只是简单的回调我们在创建ImageRequest时重写的onResponse方法。

到目前为止ImageRequest类就算是分析完了，如果你觉得本篇文章帮助到了你，希望大爷能够打赏。

本文为原创文章，转载请注明出处：http://www.jianshu.com/p/d6609049325d