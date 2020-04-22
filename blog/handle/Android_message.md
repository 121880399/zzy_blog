Android 消息机制分析

#前言
本文说的Android消息机制主要是指Handler的运行机制。Handler机制主要可以让我们在子线程中处理耗时操作，然后在主线程中更新UI。
本文涉及到的类有
**Handler
Message
Looper
MessageQueue
ActivityThread**

#Handler的运行原理

![android消息机制.png](http://upload-images.jianshu.io/upload_images/1870458-7ddff0ee8b9d19d0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对于Handler的运行原理相信大家都很熟悉了，网络上面也有很多这样的文章。大致就是在主线程中创建Handler，然后通过Handler给MessageQueue发送Message,该MessageQueue是在程序启动的时候ActivityThread在Main方法中调用Looper.prepareMainLooper()创建的，一个线程只有一个Looper，一个MessageQueue,Looper通过ThreadLocal存储在当前的线程中。Looper会使用loop方法不断的从MessageQueue中取Message，通过Message的taget来把消息分发到相应的Handler中进行处理。

#ActivityThread
android程序启动时会首先执行ActivityThread的main方法，我们先来看看代码:
```
public static final void main(String[] args) {
        SamplingProfilerIntegration.start();

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();
        if (sMainThreadHandler == null) {
            sMainThreadHandler = new Handler();
        }

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        Looper.loop();

        if (Process.supportsProcesses()) {
            throw new RuntimeException("Main thread loop unexpectedly exited");
        }

        thread.detach();
        String name = (thread.mInitialApplication != null)
            ? thread.mInitialApplication.getPackageName()
            : "<unknown>";
        Slog.i(TAG, "Main thread of " + name + " is now exiting");
    }
```
可以看到在程序一启动就调用了Looper.prepareMainLooper方法和Looper.loop方法。那么我们就先来看看Looper这个类。

#Looper类

![Looper.png](http://upload-images.jianshu.io/upload_images/1870458-27bb9deb067ee2bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
以上是Looper类主要的方法和变量，我们可以看到它持有了一个自己的引用，一个MessageQueue和一个ThreadLocal对象(当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑采用ThreadLocal)。我们先看prepareMainLooper方法：
```
 public static final void prepareMainLooper() {
        prepare();
        setMainLooper(myLooper());
        if (Process.supportsProcesses()) {
            myLooper().mQueue.mQuitAllowed = false;
        }
    }
```
逻辑很简单，其实就是一些初始化操作，我们继续跟进去。
```
public static final void prepare() {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper());
    }
```
prepare方法首先通过sThreadLocal.get()方法得到当前线程的Looper对象，看是否为空，如果不为空说明已经创建了Looper对象，就抛出异常，从异常打印的语句我们也可以指定，每个线程只能有一个Looper。如果当前线程还没有Looper，那么我们就new一个Looper并且通过sThreadLocal.set方法将其保存起来。
```
 private Looper() {
        mQueue = new MessageQueue();
        mRun = true;
        mThread = Thread.currentThread();
    }
```
Looper类的构造方法，可以看到标明是私有的，也就是说外部不能直接通过new Looper()的方式创建Looper对象。在构造方法中创建了一个MessageQueue，这下我们知道主线程中的Looper和MessageQueue在程序一启动的时候就创建好了，不需要我们手动去创建。
```
private synchronized static void setMainLooper(Looper looper) {
        mMainLooper = looper;
    }

public synchronized static final Looper getMainLooper() {
        return mMainLooper;
    }

public static final Looper myLooper() {
        return (Looper)sThreadLocal.get();
    }
```
这三个方法都很简单，一起看看。我们开始说了Looper类持有了本身的引用mMainLooper,setMainLooper方法就是为这个变量赋值，这是一个私有方法，说明在外部并不能调用该方法对mMainLooper进行赋值。在prepareMainLooper方法中调用它时传入的参数是myLooper方法的返回值，myLooper方法的返回值正好就是当前线程的Looper对象，也就是说在prepare方法中为当前线程创建了一个Looper对象并保存，接下里又通过mylooper方法把该对象取出来赋值给了mMainLooper。
```
public static final void loop() {
    	//得到当前线程的Looper对象
        Looper me = myLooper();
		//得到Looper对象对应的MessageQueue
        MessageQueue queue = me.mQueue;
        
        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
        
        while (true) {
			//从MessageQueue中得到一条消息，如果没有消息则阻塞
            Message msg = queue.next(); // might block
            //if (!me.mRun) {
            //    break;
            //}
            if (msg != null) {
                if (msg.target == null) 
					{
                    // No target is a magic identifier for the quit message.
                    return;
                }
                if (me.mLogging!= null) 
					me.mLogging.println(
                        ">>>>> Dispatching to " + msg.target + " "
                        + msg.callback + ": " + msg.what
                        );
				//回调Message中的Hanlder对象的dispatcheMessage方法
                msg.target.dispatchMessage(msg);
                if (me.mLogging!= null) me.mLogging.println(
                        "<<<<< Finished to    " + msg.target + " "
                        + msg.callback);
                
                // Make sure that during the course of dispatching the
                // identity of the thread wasn't corrupted.
                final long newIdent = Binder.clearCallingIdentity();
                if (ident != newIdent) {
                    Log.wtf("Looper", "Thread identity changed from 0x"
                            + Long.toHexString(ident) + " to 0x"
                            + Long.toHexString(newIdent) + " while dispatching to "
                            + msg.target.getClass().getName() + " "
                            + msg.callback + " what=" + msg.what);
                }
                
                msg.recycle();
            }
        }
    }
```
接着来看loop方法，loop方法内部首先会得到当前线程的Looper对象，然后与该Looper对象对应的MessageQueue，也就是在Looper构造方法中创建的那个，然后通过while（true）不断的从消息队列中获取消息。经过一系列的判断，最终回调Message所对应的targe的dispatchMessage方法把消息分发出去。

#Message类

![Message.png](http://upload-images.jianshu.io/upload_images/1870458-6bfd573368a58c6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Message类其实类似于JavaBean,封装对象提供get,set方法。
```
public final class Message implements Parcelable {
    /**
     * User-defined message code so that the recipient can identify 
     * what this message is about. Each {@link Handler} has its own name-space
     * for message codes, so you do not need to worry about yours conflicting
     * with other handlers.
     */
    public int what;

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg1; 

    /**
     * arg1 and arg2 are lower-cost alternatives to using
     * {@link #setData(Bundle) setData()} if you only need to store a
     * few integer values.
     */
    public int arg2;

    /**
     * An arbitrary object to send to the recipient.  When using
     * {@link Messenger} to send the message across processes this can only
     * be non-null if it contains a Parcelable of a framework class (not one
     * implemented by the application).   For other data transfer use
     * {@link #setData}.
     * 
     * <p>Note that Parcelable objects here are not supported prior to
     * the {@link android.os.Build.VERSION_CODES#FROYO} release.
     */
    public Object obj;

    /**
     * Optional Messenger where replies to this message can be sent.  The
     * semantics of exactly how this is used are up to the sender and
     * receiver.
     */
    public Messenger replyTo;
    
    /*package*/ long when;
    
    /*package*/ Bundle data;
    
    /*package*/ Handler target;     
    
    /*package*/ Runnable callback;   
    
    // sometimes we store linked lists of these things
    /*package*/ Message next;

    private static Object mPoolSync = new Object();
    private static Message mPool;
    private static int mPoolSize = 0;

    private static final int MAX_POOL_SIZE = 10;
    
    /**
     * Return a new Message instance from the global pool. Allows us to
     * avoid allocating new objects in many cases.
     */
    public static Message obtain() {
        synchronized (mPoolSync) {
            if (mPool != null) {
                Message m = mPool;
                mPool = m.next;
                m.next = null;
                mPoolSize--;
                return m;
            }
        }
        return new Message();
    }

    /**
     * Same as {@link #obtain()}, but copies the values of an existing
     * message (including its target) into the new one.
     * @param orig Original message to copy.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Message orig) {
        Message m = obtain();
        m.what = orig.what;
        m.arg1 = orig.arg1;
        m.arg2 = orig.arg2;
        m.obj = orig.obj;
        m.replyTo = orig.replyTo;
        if (orig.data != null) {
            m.data = new Bundle(orig.data);
        }
        m.target = orig.target;
        m.callback = orig.callback;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the value for the <em>target</em> member on the Message returned.
     * @param h  Handler to assign to the returned Message object's <em>target</em> member.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h) {
        Message m = obtain();
        m.target = h;

        return m;
    }

    /**
     * Same as {@link #obtain(Handler)}, but assigns a callback Runnable on
     * the Message that is returned.
     * @param h  Handler to assign to the returned Message object's <em>target</em> member.
     * @param callback Runnable that will execute when the message is handled.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h, Runnable callback) {
        Message m = obtain();
        m.target = h;
        m.callback = callback;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values for both <em>target</em> and
     * <em>what</em> members on the Message.
     * @param h  Value to assign to the <em>target</em> member.
     * @param what  Value to assign to the <em>what</em> member.
     * @return A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what) {
        Message m = obtain();
        m.target = h;
        m.what = what;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>, and <em>obj</em>
     * members.
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param obj  The <em>object</em> method to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.obj = obj;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>, 
     * <em>arg1</em>, and <em>arg2</em> members.
     * 
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param arg1  The <em>arg1</em> value to set.
     * @param arg2  The <em>arg2</em> value to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, int arg1, int arg2) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;

        return m;
    }

    /**
     * Same as {@link #obtain()}, but sets the values of the <em>target</em>, <em>what</em>, 
     * <em>arg1</em>, <em>arg2</em>, and <em>obj</em> members.
     * 
     * @param h  The <em>target</em> value to set.
     * @param what  The <em>what</em> value to set.
     * @param arg1  The <em>arg1</em> value to set.
     * @param arg2  The <em>arg2</em> value to set.
     * @param obj  The <em>obj</em> value to set.
     * @return  A Message object from the global pool.
     */
    public static Message obtain(Handler h, int what, 
            int arg1, int arg2, Object obj) {
        Message m = obtain();
        m.target = h;
        m.what = what;
        m.arg1 = arg1;
        m.arg2 = arg2;
        m.obj = obj;

        return m;
    }

    /**
     * Return a Message instance to the global pool.  You MUST NOT touch
     * the Message after calling this function -- it has effectively been
     * freed.
     */
    public void recycle() {
        synchronized (mPoolSync) {
            if (mPoolSize < MAX_POOL_SIZE) {
                clearForRecycle();
                
                next = mPool;
                mPool = this;
                mPoolSize++;
            }
        }
    }

    /**
     * Make this message like o.  Performs a shallow copy of the data field.
     * Does not copy the linked list fields, nor the timestamp or
     * target/callback of the original message.
     */
    public void copyFrom(Message o) {
        this.what = o.what;
        this.arg1 = o.arg1;
        this.arg2 = o.arg2;
        this.obj = o.obj;
        this.replyTo = o.replyTo;

        if (o.data != null) {
            this.data = (Bundle) o.data.clone();
        } else {
            this.data = null;
        }
    }

    /**
     * Return the targeted delivery time of this message, in milliseconds.
     */
    public long getWhen() {
        return when;
    }
    
    public void setTarget(Handler target) {
        this.target = target;
    }

    /**
     * Retrieve the a {@link android.os.Handler Handler} implementation that
     * will receive this message. The object must implement
     * {@link android.os.Handler#handleMessage(android.os.Message)
     * Handler.handleMessage()}. Each Handler has its own name-space for
     * message codes, so you do not need to
     * worry about yours conflicting with other handlers.
     */
    public Handler getTarget() {
        return target;
    }

    /**
     * Retrieve callback object that will execute when this message is handled.
     * This object must implement Runnable. This is called by
     * the <em>target</em> {@link Handler} that is receiving this Message to
     * dispatch it.  If
     * not set, the message will be dispatched to the receiving Handler's
     * {@link Handler#handleMessage(Message Handler.handleMessage())}.
     */
    public Runnable getCallback() {
        return callback;
    }
    
    /** 
     * Obtains a Bundle of arbitrary data associated with this
     * event, lazily creating it if necessary. Set this value by calling
     * {@link #setData(Bundle)}.  Note that when transferring data across
     * processes via {@link Messenger}, you will need to set your ClassLoader
     * on the Bundle via {@link Bundle#setClassLoader(ClassLoader)
     * Bundle.setClassLoader()} so that it can instantiate your objects when
     * you retrieve them.
     * @see #peekData()
     * @see #setData(Bundle)
     */
    public Bundle getData() {
        if (data == null) {
            data = new Bundle();
        }
        
        return data;
    }

    /** 
     * Like getData(), but does not lazily create the Bundle.  A null
     * is returned if the Bundle does not already exist.  See
     * {@link #getData} for further information on this.
     * @see #getData()
     * @see #setData(Bundle)
     */
    public Bundle peekData() {
        return data;
    }

    /**
     * Sets a Bundle of arbitrary data values. Use arg1 and arg1 members 
     * as a lower cost way to send a few simple integer values, if you can.
     * @see #getData() 
     * @see #peekData()
     */
    public void setData(Bundle data) {
        this.data = data;
    }

    /**
     * Sends this Message to the Handler specified by {@link #getTarget}.
     * Throws a null pointer exception if this field has not been set.
     */
    public void sendToTarget() {
        target.sendMessage(this);
    }

    /*package*/ void clearForRecycle() {
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        when = 0;
        target = null;
        callback = null;
        data = null;
    }

    /** Constructor (but the preferred way to get a Message is to call {@link #obtain() Message.obtain()}).
    */
    public Message() {
    }

    public String toString() {
        return toString(SystemClock.uptimeMillis());
    }

    String toString(long now) {
        StringBuilder   b = new StringBuilder();
        
        b.append("{ what=");
        b.append(what);

        b.append(" when=");
        TimeUtils.formatDuration(when-now, b);

        if (arg1 != 0) {
            b.append(" arg1=");
            b.append(arg1);
        }

        if (arg2 != 0) {
            b.append(" arg2=");
            b.append(arg2);
        }

        if (obj != null) {
            b.append(" obj=");
            b.append(obj);
        }

        b.append(" }");
        
        return b.toString();
    }

    public static final Parcelable.Creator<Message> CREATOR
            = new Parcelable.Creator<Message>() {
        public Message createFromParcel(Parcel source) {
            Message msg = Message.obtain();
            msg.readFromParcel(source);
            return msg;
        }
        
        public Message[] newArray(int size) {
            return new Message[size];
        }
    };
        
    public int describeContents() {
        return 0;
    }

    public void writeToParcel(Parcel dest, int flags) {
        if (callback != null) {
            throw new RuntimeException(
                "Can't marshal callbacks across processes.");
        }
        dest.writeInt(what);
        dest.writeInt(arg1);
        dest.writeInt(arg2);
        if (obj != null) {
            try {
                Parcelable p = (Parcelable)obj;
                dest.writeInt(1);
                dest.writeParcelable(p, flags);
            } catch (ClassCastException e) {
                throw new RuntimeException(
                    "Can't marshal non-Parcelable objects across processes.");
            }
        } else {
            dest.writeInt(0);
        }
        dest.writeLong(when);
        dest.writeBundle(data);
        Messenger.writeMessengerOrNullToParcel(replyTo, dest);
    }

    private final void readFromParcel(Parcel source) {
        what = source.readInt();
        arg1 = source.readInt();
        arg2 = source.readInt();
        if (source.readInt() != 0) {
            obj = source.readParcelable(getClass().getClassLoader());
        }
        when = source.readLong();
        data = source.readBundle();
        replyTo = Messenger.readMessengerOrNullFromParcel(source);
    }
}
```
Message类实在很简单，实现了Parcelable接口，Parcelable接口的介绍与serializable的区别请自行百度。在Message类中有一个消息池，池中的消息数最大为10，用mPoolSize变量来记录当前池中的消息数量，通过obtain和recycle方法来达到重复利用Message。在Message中要需要注意一下target和callback这两个成员变量，前者是一个Handler，用来指明该Message是哪个Handler发出的，这样在从MessageQueue取出Message以后知道该回调哪个Handler的dispatchMessage方法。后者是一个Runnable对象，如果这个Runnable对象不为空的话，会优先执行这个对象的run方法。

#MessageQueue

![MessageQueue.png](http://upload-images.jianshu.io/upload_images/1870458-ea4864afdddfea6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

MessageQueue很多都是native方法，用的比较多的就是上图列出的这两个。一个把消息入队，一个获取下一个Message。
```
final boolean enqueueMessage(Message msg, long when) {
        if (msg.when != 0) {
            throw new AndroidRuntimeException(msg
                    + " This message is already in use.");
        }
        if (msg.target == null && !mQuitAllowed) {
            throw new RuntimeException("Main thread not allowed to quit");
        }
        final boolean needWake;
        synchronized (this) {
            if (mQuiting) {
                RuntimeException e = new RuntimeException(
                    msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            } else if (msg.target == null) {
                mQuiting = true;
            }

            msg.when = when;
            //Log.d("MessageQueue", "Enqueing: " + msg);
            Message p = mMessages;
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked; // new head, might need to wake up
            } else {
                Message prev = null;
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
                msg.next = prev.next;
                prev.next = msg;
                needWake = false; // still waiting on head, no need to wake up
            }
        }
        if (needWake) {
            nativeWake(mPtr);
        }
        return true;
    }

 final Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;

        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            nativePollOnce(mPtr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                final Message msg = mMessages;
                if (msg != null) {
                    final long when = msg.when;
                    if (now >= when) {
                        mBlocked = false;
                        mMessages = msg.next;
                        msg.next = null;
                        if (Config.LOGV) Log.v("MessageQueue", "Returning message: " + msg);
                        return msg;
                    } else {
                        nextPollTimeoutMillis = (int) Math.min(when - now, Integer.MAX_VALUE);
                    }
                } else {
                    nextPollTimeoutMillis = -1;
                }

                // If first time, then get the number of idlers to run.
                if (pendingIdleHandlerCount < 0) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount == 0) {
                    // No idle handlers to run.  Loop and wait some more.
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // Run the idle handlers.
            // We only ever reach this code block during the first iteration.
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // Reset the idle handler count to 0 so we do not run them again.
            pendingIdleHandlerCount = 0;

            // While calling an idle handler, a new message could have been delivered
            // so go back and look again for a pending message without waiting.
            nextPollTimeoutMillis = 0;
        }
    }
```
代码我就不具体分析了，虽然MessageQueue是消息队列，但是这里是采用单链表来实现的。关于单链表的知识[请看这](http://www.cnblogs.com/smyhvae/p/4761593.html)。

#Handler类

![Handler.png](http://upload-images.jianshu.io/upload_images/1870458-3b29fd86e2f61d74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Handler类中定义了一个Callback接口:
```
  public interface Callback {
        public boolean handleMessage(Message msg);
    }
```
从Handler的构造方法我们知道，可以实现该接口并传入。我们接下来看看Handler的构造方法:
```
//第一种 无参构造方法
public Handler() {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = null;
    }

//第二种 带有Callback参数的构造方法
 public Handler(Callback callback) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
    }
 //第三种 带有Looper的构造方法
   public Handler(Looper looper) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = null;
    }

  //第四种 带有Looper和Callback的构造方法
public Handler(Looper looper, Callback callback) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
    }
```
以上是Handler的四种构造方法，别看代码挺多，无非就是对mLooper,mQueue,mCallback这三个成员变量赋值。1,2两种构造方法都是使用当前线程的Looper。3,4两种构造方法可以指定使用其他的Looper对象，也就是说子线程的Looper对象，发送Message也会发送到子线程的MessageQueue中。需要注意的是，如果在子线程中使用1,2两种构造方法来创建Handler，必须在创建Handler之前调用Looper.prepare()方法来为当前线程创建一个Looper对象，否则在1，2两种构造方法内部使用Looper.myLooper获取当前线程的Looper对象时会返回null,这样就会抛出RuntimeException。
```
public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

  private final Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

 public boolean sendMessageAtTime(Message msg, long uptimeMillis)
    {
        boolean sent = false;
        MessageQueue queue = mQueue;
        if (queue != null) {
            msg.target = this;
            sent = queue.enqueueMessage(msg, uptimeMillis);
        }
        else {
            RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
        }
        return sent;
    }
```
不论使用post方法还是sendMessage方法，最后都是调用sendMessageAtTime方法。只不过调用post方法时会把传入的Runnable对象包装成一个Message，还记我们Message类中有一个Runnable成员变量吗？就是把通过post方法传入的Runnable对象传给它进行包装然后返回一个Message。在最终的sendMessageAtTime方法中，我们先得到当前的MessageQueue，然后把Message中的target变量赋值为当前handler，还记得Message中的target吧？它是一个Handler对象，这样在Looper的loop方法中获取消息以后，就可以通过Message的target属性得到Handler然后回调它的dispatchMessage方法。所以target是有指向作用的，它让Message知道自己是哪个Handler发送过来的，最后要由哪个Handler来处理。接着就调用MessageQueue的enqueueMessage方法把Message入队。
```
public void dispatchMessage(Message msg) {
    //如果message中有runnable对象，那么就回调runnable的run方法
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
        //如果通过handler带callBack参数的构造方法创建并实现了Callback接口
        //则调用Callback接口中的handleMessage方法
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
			//回调创建Handler时复写的handleMessage方法
            handleMessage(msg);
        }
    }

 private final void handleCallback(Message message) {
        message.callback.run();
    }

public void handleMessage(Message msg) {
    }
```
以上就是在loop方法中回调的dispatchMessage方法，从代码中我们看出，这里能够处理三种情况：
1.Message中指定了Runnable对象的。（也就是使用post方法提交的）
2.创建Handler时实现了内部Callback接口的。（使用3,4种构造方法创建Handler的）
3.创建Handler时重写了handleMessage方法的。
这里要注意一个细节，如果在创建Handler是重写了handleMessage方法，并且使用post方法提交消息的，那么在执行完Message.callback.run()方法以后，还会回调重写的handleMessage方法。当然如果在创建Handler时实现了内部Callback接口的，也重写了handleMessage方法，但是在调用Callback接口中的handleMessage方法返回了false，那么也会回调重写的handleMessage方法。我们可以看见默认的handleMessage方法是空实现。

到此为止Handler的运行机制基本分析完毕，由于笔者水平有限，难免有所疏漏，望请包涵。
本文为原创文章，转载请注明出处:http://www.jianshu.com/p/12db39650356