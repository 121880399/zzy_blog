AsyncTask各版本源码分析

#前言
在android开发中相信大家或多或少都有使用过AsyncTask来执行异步任务然后更新Ui,在AsyncTask刚出来的时候一度受到了大家的追捧，因为可以告别使用Thread+Handler的线程模式。但是在不同SDK版本上AsyncTask有着比较大的差异，今天我们就来分析一下AsyncTask的源码以及各版本之间的差异，让你彻底理解AsyncTask。

#AsyncTask的基本用法
我们先一起回顾一下AsyncTask的基本用法，AsyncTask是一个抽象类，我们必须创建一个子类去继承它。
```
public class DownloadTask extends AsyncTask<String,Integer,Boolean>
{
    private static final String TAG=DownloadTask.class.getSimpleName();
    private ProgressDialog mDialog;
    private ImageView mImageView;
    private Context mContext;

    public DownloadTask(Context context,ProgressDialog dialog,ImageView imageView)
    {
        mContext=context;
        mDialog=dialog;
        mImageView=imageView;
    }
    /**
     * 在doInBackground前执行，该方法在主线程中执行
     * */
    @Override
    protected void onPreExecute()
    {
        super.onPreExecute();
        mDialog.show();
    }

    /**
     * 该方法会在工作线程中执行，在此方法中要添加程序是否被终止的判断逻辑
     * */
    @Override
    protected Boolean doInBackground(String... params)
    {
        String url=params[0];
        boolean flag=true;
        while(flag)
        {
            if(isCancelled())
            {
                return false;
            }
            int progress = downloadFile(url);
            publishProgress(progress);
            if(progress>=100)
            {
                flag=false;
            }
        }
        return true;
    }

    /**
     * 如果任务没有被取消，执行完doInBackground后会执行该方法
     * */
    @Override
    protected void onPostExecute(Boolean aBoolean)
    {
        mDialog.dismiss();
        if(isCancelled())
        {
            Log.i(TAG,"任务被取消！");
        }
        else
        {
            if (aBoolean)
            {
                //这里模拟把下载好的图片显示在ImageView上面
                mImageView.setImageResource(R.mipmap.ic_launcher);
                Toast.makeText(mContext, "下载成功!", Toast.LENGTH_SHORT).show();
            } else
            {
                Toast.makeText(mContext, "下载失败!", Toast.LENGTH_SHORT).show();
            }
        }
    }

    /**
     *如果在doInBackground方法中调用了publishProgress，就会执行该方法，该方法可以对UI进行更新操作
     * */
    @Override
    protected void onProgressUpdate(Integer... values)
    {
        mDialog.setProgress(values[0]);
    }



    /**
     * 在doInBackground中如果调用了cancel方法，则不执行onPostExecute方法而执行onCancelled方法
     * */
    @Override
    protected void onCancelled(Boolean aBoolean)
    {
        mDialog.dismiss();
        Log.i(TAG, "任务被取消！");
    }
}

```
DownloadTask模拟一个下载图片任务，下载之前会显示进度条，进度条显示下载过程，下载完成以后把图片显示在ImageView上面并关闭进度条。需要注意的是AsyncTack的子类最好不要写在Activity的内部，容易造成内存泄漏，如果一定要写，请写出静态内部类并使用弱引用。AsyncTaskd的cancel方法只是简单把标志位改为true，最后使用Thread.interrupt去中断线程执行，但是这并不保证会马上停止执行，所以可以在doInBackground中使用isCancelled来判断任务是否被取消，如果被取消则停止任务。任务被取消以后不会调用onPostExeute方法，而是调用onCancelled方法，要在该方法里面做好任务被取消后的相关逻辑。在onPostExecute中也要判断任务是否被取消，虽然耗时任务已经执行完成，但是在用户依然选择了取消操作，那么接下来的逻辑就没必要执行下去，在本段例子中就是不用在把下载好的图片显示在ImageView上。

接着我们使用execute来执行任务:
```
new DownloadTask(getApplicationContext(),mdialog,mImageView).execute(url);
```

#4.4版的AsyncTask
接下来我们就一起来看看AsyncTask的源码吧，我们对上面的代码进行跟踪。首先AsyncTask可以指定三个泛型参数<Params, Progress, Result>,Params为执行AsyncTask时需要传入的参数类型，Progress用于显示进度的类型，Result指定返回结果的类型。由于DownloadTask继承于AsyncTask,所以new DownloadTask时会先执行AsyncTask的构造方法。
```
public AsyncTask()
    {
        mWorker = new WorkerRunnable<Params, Result>()
        {
            public Result call() throws Exception
            {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };

        mFuture = new FutureTask<Result>(mWorker)
        {
            @Override
            protected void done()
            {
                try
                {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e)
                {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e)
                {
                    throw new RuntimeException("An error occured while executing doInBackground()", e.getCause());
                } catch (CancellationException e)
                {
                    postResultIfNotInvoked(null);
                }
            }
        };
    }
```
别看AsyncTask的构造方法只是简单的创建了WorkerRunnable和FutureTask对象，但是这两个对象很重要。我们先来看看WorkerRunnable是什么？
```
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result>
    {
        Params[] mParams;
    }
```
这就是WorkerRunnable的全部代码了，里面只有一个Params数组。关键是它实现了Callable接口，这个很关键，我们来看看Callable的代码：
```
public interface Callable<V>
 {   
    V call() throws Exception;
}
```
可以看到Callable接口里面只定义了一个call方法，返回一个泛型对象。这么短的几行代码，为什么说它重要呢？因为不论是继承Thread类还是实现Runnable方法，执行完任务以后都无法直接返回结果。而Callable接口就弥补了这个缺陷，当call方法执行完毕以后会返回一个泛型对象。WorkerRunnable实现了Callable接口，也就要实现该方法。该方法中的内容我们暂时先不看。

在AsyncTask的构造方法中也创建了FutureTask对象并重写了done方法，这个方法的内容我们也暂时不看。先了解一下FutureTask这个类。FutureTask实现了RunnableFuture<V>接口，而RunnableFuture又同时继承了Runnable和Future<V>接口。Runnable接口我们很了解，其中只有一个run方法，并且无返回值。Future<V>接口主要是针对Runnable和Callable任务的。
提供了三种功能：
1.判断任务是否完成 
2.能够中断任务
3.能够获取任务执行的结果
FutureTask同时实现了Runnable和Future接口，也就是说FutureTask既能够被线程执行，又能提供线程执行任务后返回的结果。对于Futrue的介绍就先到这，之后我们会接着看。

在执行完AsyncTask和DownloadTask的构造方法后，就该接着执行AsyncTask中的execute方法了。
```
 public final AsyncTask<Params, Progress, Result> execute(Params... params)
    {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```
我们可以看到在方法内部调用了executeOnExecutor方法并传入了sDefaultExecutor对象，我们来看看sDefaultExecutor对象的定义。
```
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
sDefaultExecutor实际上就是SerialExecutor对象，SerialExecutor实现了Executor接口，在SerialExecutor内部定义了一个双端队列ArrayDeque，ArrayDeque的内部是使用数组形式来实现双端队列的，我们知道队列是FIFO的，只能在队头删除元素，队尾添加元素，而双端队列是在队头和队尾都能够删除和添加元素。需要注意的是ArrayDeque没有容量的限制，队列满了以后会自动进行扩充。关于ArrayDeque的相关知识可以自行谷歌。对于sDefaultExecutor我们暂时只需要知道它是一个SerialExecutor对象就行。

我们接着看executeOnExecutor方法:
```
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
方法很简单，首先判断当前的状态，如果是在运行当中或者已经任务结束都会抛出异常。接着就会执行onPreExecute方法，是不是很眼熟？没错就是我们在DownloadTask中重写的方法，我们当时的注释是在doInBackground前执行，现在我们知道这个方法在什么时候被回调了吧？接着把我们传入进来的参数（也就是url）赋值给mWorker.mParams，mWorker我刚刚已经看了其定义，内部有一个Params的数组来存放传入的参数。接着我们就使用exec.execute方法了，这里的exec就是我们刚刚讲的SerialExecutor对象，现在调用它的exeute方法并把我们在AsyncTask构造方法中创建的mFuture对象传入，还记得mFuture对象吧？它实现了Runnable接口。
```

    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
同样的代码，这次我们得一步步来看。在execute方法接收一个Runnable对象，我们传入的mFuture正好实现了该接口，方法内部首先通过双端队列ArrayDeque的offer方法把一个新创建的Runnable对象入队。然后判断mActive是否为空？第一次执行肯定是为空的，所以会调用scheduleNext方法，scheduleNext方法首先从双端队列中得到一个元素，如果不为空，就调用THREAD_POOL_EXECUTOR.execute方法，THREAD_POOL_EXECUTOR是一个线程池，也就是说我们把从双端队列中得到的任务交给线程池去执行（接下来的代码都在工作线程中执行），那么肯定就要执行到Runnable对象的run方法，该方法中的内容就是调用mFuture中的run方法:
```
 public void run() 
 {
        if (state != NEW ||
            !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            // runner must be non-null until state is settled to
            // prevent concurrent calls to run()
            runner = null;
            // state must be re-read after nulling runner to prevent
            // leaked interrupts
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
该run方法为FutureTask类中的run方法，不要搞混淆了。这个方法我们不细讲，我们只看重点，方法中的callable就是我们在AsyncTask构造方法中创建FutureTask时传入的mWorker对象，我们知道mWorker是实现了callable接口的。然后调用了mWorker的call方法：
```
 mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);

                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                return postResult(doInBackground(mParams));
            }
        };
```
还记得这段代码吗？这是AsyncTask构造方法中创建WorkerRunnable对象的代码。我们看它的call方法。首先将mTaskInvoked这个标志位设置为true，然后设置了线程优先级，接着我们看见在这里回调了doInBackground方法，所以doInBackground方法是在子线程中执行的，doInBackground方法中会有一个返回值，这个返回值会传入postResult方法中。
```
   private Result postResult(Result result) {
        @SuppressWarnings("unchecked")
        Message message = sHandler.obtainMessage(MESSAGE_POST_RESULT,
                new AsyncTaskResult<Result>(this, result));
        message.sendToTarget();
        return result;
    }
```
postResult中的逻辑就是获取一个Message，然后发送该Message给Handler处理。我们看sHandler的定义：
```
private static final InternalHandler sHandler = new InternalHandler();
private static class InternalHandler extends Handler
    {
        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg)
        {
            AsyncTaskResult result = (AsyncTaskResult) msg.obj;
            switch (msg.what)
            {
                case MESSAGE_POST_RESULT:
                    // There is only one result
                    result.mTask.finish(result.mData[0]);
                    break;
                case MESSAGE_POST_PROGRESS:
                    result.mTask.onProgressUpdate(result.mData);
                    break;
            }
        }
    }
```
这是sHandler在AsyncTaskResult中的定义，很显然它是在主线程中定义的。在Loop从MessageQueue中获取到Message后会分发到handleMessage这个方法中来处理，我们可以看见在Switch代码块中有两个分支，一个是用来处理结果的，一个是用来更新进度的。我们先看处理结果的分支，处理结果的分支会调用AsyncTask中的finish方法并把任务的结果传入。
```
  private void finish(Result result)
    {
        if (isCancelled())
        {
            onCancelled(result);
        } else
        {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }
```
finish方法中的逻辑很简单，判断用户是否调用了cancel方法，如果调用了那么就回调onCancelled方法，如果没有调用就回调onPostExecute方法，这跟我们在讲AsyncTask基本用法时说的逻辑一样。最后标志任务状态为完成。（注意：以上代码是在主线程中执行）
我们接着看更新进度的分支，我们先想一下更新进度时在什么时候被调用？是不是在doInBackground中调用了publishProgress方法？所以我们先看看publishProgress方法：
```
protected final void publishProgress(Progress... values)
    {
        if (!isCancelled())
        {
            sHandler.obtainMessage(MESSAGE_POST_PROGRESS, new AsyncTaskResult<Progress>(this, values)).sendToTarget();
        }
    }

```
从publishProgress方法我们知道在doInBackground中调用了publishProgress方法传入值后，如果任务没有被取消，那么就会发送一个Message，接着就回到handleMessage方法中的处理进度代码块，回调onProgressUpdate方法，这时候我们就可以对进度条进行更新。
我们可以看见在使用sHandler获取一个Message时都传入了一个AsyncTaskResult对象，我们看看它的定义：
```
private static class AsyncTaskResult<Data> {
        final AsyncTask mTask;
        final Data[] mData;

        AsyncTaskResult(AsyncTask task, Data... data) {
            mTask = task;
            mData = data;
        }
    }
```
AsyncTaskResult只是简单对AsyncTask和返回结果做了封装。

我们回到FutureTask类的run方法中，当调用了callable.call以后是会有一个返回值的，得到该返回值以后，我们使用set(result)方法把返回值保存。这样当使用FutureTask的get方法就可以得到任务的结果了。由此我们可以看出Callable,Runnable,Future之间的关系，以及FutureTask是如果把这三者结合在一起的。我们看看set方法:
```
  protected void set(V v) {
        if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
            outcome = v;
            U.putOrderedInt(this, STATE, NORMAL); // final state
            finishCompletion();
        }
    }
```
set方法中把传入的参数交给outcome保管，当调用get方法时就把这个outcome返回。之后调用了finishCompletion方法：
```
private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (U.compareAndSwapObject(this, WAITERS, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```
该方法其他的我们都不用看，只要看倒数第二句，会回调done方法，这个方法是不是很熟悉？没错，在创建FutureTask的时候我们重写了该方法。我们看看done方法的实现：
```
 mFuture = new FutureTask<Result>(mWorker)
        {
            @Override
            protected void done()
            {
                try
                {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e)
                {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e)
                {
                    throw new RuntimeException("An error occured while executing doInBackground()", e.getCause());
                } catch (CancellationException e)
                {
                    postResultIfNotInvoked(null);
                }
            }
        };

private void postResultIfNotInvoked(Result result)
    {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked)
        {
            postResult(result);
        }
    }
```
done方法中就使用了FutureTask的get方法得到返回结果，并传入postResultIfNotInvoked方法中，postResultIfNotInvoked中我们判断mTaskInvoked是否为false？还记得在创建WorkerRunnable对象时call方法中我们把mTaskInvoked设置为了true，mTaskInvoked是AtomicBoolean类型。如果mTaskInvoked为false说明没有调用，那么我们就调用postResult方法，如果为true则说明已经调用过了，不需要再调用。注意done方法中的代码是在子线程中执行。

到目前为止，AsyncTask的执行流程我们就跟踪解析完了，是不是很晕？我自己都晕了，没关系，画图一向很糟糕的我，决定画一张流程图帮助大家理解。
![AsyncTask流程图.png](http://upload-images.jianshu.io/upload_images/1870458-d446a90b89c3a6cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这个流程图画的不标准，但是我认为这样会比较容易理解。

#各版本AsyncTask之间的差异
各版本AsyncTask之间的差异主要集中在线程池的使用这一块。主要的分界点有2个，分为3个阶段。第一阶段是3.0以前的版本，第二是3.0-4.4版本的阶段，第三个是4.4版本以后的阶段。那我们一个一个阶段来看。

#3.0之前版本中AsyncTask
这里使用2.3版本的AsyncTask源代码:
```
    private static final int CORE_POOL_SIZE = 5;
    private static final int MAXIMUM_POOL_SIZE = 128;
    private static final int KEEP_ALIVE = 1;

    private static final BlockingQueue<Runnable> sWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final ThreadPoolExecutor sExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE,
            MAXIMUM_POOL_SIZE, KEEP_ALIVE, TimeUnit.SECONDS, sWorkQueue, sThreadFactory);
```
在3.0版本以前AsyncTask的线程只有1个线程池，核心线程数为5，最大线程数为128，任务队列容量为10。
也就是说当线程池中的线程数量没到5个，那么有新的任务会直接启动一个核心线程来执行任务，如果线程池中的线程数量达到了5个，然后任务会被插入到任务队列中等待执行，要是任务队列也满了，就会判断线程池中的数量是否已经达到最大线程数128，如果没有达到就会立刻启动一个非核心线程来执行任务。如果线程数量已经达到线程池规定的最大值，那么就会拒绝执行该任务。也就是说该线程池最多能同时接纳138个任务，其中有128个任务可以同时执行。而且该版本只有一个execute(Params... params) 方法，说明不能自定义线程池来执行任务。
#3.0-4.4版本中AsyncTask
这里使用4.2版本的AsyncTask源代码：
```
 private static final int CORE_POOL_SIZE = 5;
    private static final int MAXIMUM_POOL_SIZE = 128;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(10);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
 //以下为新增部分
  public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
  private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
  private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }

    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```
这个版本的线程池与上个版本并没有什么不同，只是新增加了一个SerialExecutor，从代码中我们可以看到这是一个顺序执行任务的Executor，虽然最后任务还是交给了THREAD_POOL_EXECUTOR来执行，但是使用这个Executor可以保证任务时按先进先出的顺序来执行。同时新增加了executeOnExecutor(Executor exec,Params... params) 方法，这个方法被声明是public的并且接受一个Executor参数，说明我们可以自定义线程池或者使用SerialExecutor来执行任务。如果使用execute（）方法的话，默认会使用SerialExecutor来执行任务。
#4.4版本以后AsyncTask
这里使用4.4版本的AsyncTask源代码：
```
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
    private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    private static final int KEEP_ALIVE = 1;

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);

public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
4.4版本以后的线程池数量改为了动态的，以双核心为例，先获取CPU的核心数为2，线程池的核心线程为3，最大线程数为5,而阻塞队列的容量变为了128。为什么会有这样的改动？可能谷歌公司觉得开启的线程数过多会影响效率吧。而阻塞队列从容量为10变为了128是一个很有意思的事情。在4.4以前的版本，如果已经达到了线程池的核心线程数5，切阻塞队列也达到了10，再有任务加入，就会启动新的非核心线程，也就是说只要同时又16个任务进入就会开启非核心线程。而现在需要132（3+128+1）个任务加入才会开启非核心线程。也就是说要开启新的线程的成本更大了。

#AsyncTask的各种坑
1.在Activity中定义AsyncTask导致内存泄漏
由于AsyncTask是Activity的内部类，所以会持有外部的一个引用，如果Activity已经退出，但是AsyncTask还没有执行完毕，那么Activity就无法释放导致内存泄漏。对于这个问题我们可以把AsyncTask定义为静态内部类并且采用弱引用。

2.各版本对AsyncTask的实现不一样
对于这个问题，可以自己扩展一下AsyncTask在其内部也对版本做出判断，对于不同版本做一些不同的处理。

3.不能及时取消任务
以4.4版本双核手机为例，如果用户在A界面发起5个任务，由于使用SerialExecutor来执行任务，那么任务将一个一个顺序执行，由于第一个任务执行时间过长，其他任务只能在队列中等待，导致阻塞，所以可以考虑把执行时间较短的任务优先加入。如果在第一个任务执行过程中，用户跳转到了B界面，而A界面发起的任务已经没有必要执行，所以我们要在Activity的生命周期结束的时候取消掉任务。如果任务没取消掉，B界面又发起新的任务，就会导致B界面的所有请求阻塞。如果有需要我们可以直接使用executeOnExecutor方法，然后直接使用THREAD_POOL_EXECUTOR线程池来执行，这样可以3个线程同时执行，并且在doInBackground方法中判断任务是否被取消，这样可以提高效率。

好啦，以上就是AsyncTask的全部解析，如果发现更多的坑我会继续更新。

