Handler的运行机制

#前言
由于最近有想换工作的想法，所以就打算整理一下工作内容，也复习一下android的基础知识，毕竟面试的时候总会问。不论现在实际项目中是否还使用Handler，面试都会问Handler的运行机制，索性又重新看了一遍源码。为了防止以后又得重新捋一遍，干脆就直接写篇文章记录。

#1.为什么要用Handler？
我们在开发过程中使用Handler的场景一般是将一个任务切换到某个指定的线程中去执行。为什么要在线程间来回切换呢？这是因为在android开发过程中，比较耗时的任务我们要放到子线程中执行，而更新UI又必须在主线程，所以我们要使用Handler。
那为什么不能子线程更新UI呢?这主要是因为android的UI是非线程安全的，在多线程中并发访问可能会导致UI处于不可预期的状态。

#2.ThreadLocal
ThreadLocal是一个线程内部的数据存储类，当某些数据是以线程为作用域并且不同线程具有不同的数据副本的时候，就可以考虑用ThreadLocal。为什么要提到这个类呢？因为要了解Handler的运行机制，就牵扯到Handler,MessageQueue,Looper。而Looper的作用域就是线程，并且不同的线程具有不同的Looper,这时候通过ThreadLocal就可以轻松实现Looper的存取。

#3.MessageQueue
MessageQueue叫消息队列，但是它内部实现并不是队列，而是通过一个单链表的数据结构来维护消息列表，因为但链表在插入和删除上比较有优势。

消息队列主要包括两个操作：插入，读取。我们先来看插入操作。

```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
由于队列是先进先出，所以插入队列的时候其实就是把Message添加到链表的末尾。
```
 Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
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
                    Log.wtf(TAG, "IdleHandler threw exception", t);
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
这里的读取方法是一个无限循环方法，如果消息队列中没有消息，那么就会一直阻塞在这，当有消息到来，就会返回这条消息，并且将消息从链表中移除。

#4.Looper
Looper在android消息机制当中扮演着消息循环的角色，也就是它会不停的从MessageQueue中取消息。我们先来看Looper的构造方法：
```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
我们可以看见Looper的构造方法是私有的，并且在构造方法中创建了MessageQueue，还把mThread的值指定为当前的线程。既然不能通过构造方法创建Looper，那我们如何创建Looper呢？
```
public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
我们可以看到Looper提供了两个prepare方法供我们创建Looper对象。其中在第二个prepare方法中我们可以看见，首先使用了ThreadLocal.get()方法获取当前线程的Looper对象，如果存在的话就抛出一个异常，不存在就创建一个新的Looper对象，并且使用ThreadLocal.set()方法保存起来。从这段代码我们可以知道，一个线程只能存在一个Looper对象。

除此之外，Looper还提供了prepareMainLooper方法，该方法主要是给主线程创建Looper使用的，内部的实现也是通过prepare方法。

```
 public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
由于主线程比较特殊，Looper还提供了一个得到主线程Looper对象的方法getMainLooper。
```
 public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
```
使用getMainLooper方法可以在任何地方得到主线程的Looper对象。

创建完Looper之后，就需要运行它，Looper中最重要的一个方法就是loop,只有调用了loop,消息循环系统才会真正的起作用。
```
 public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
loop方法是一个死循环，会不断的从MessageQueue中读取Message，读取到以后交给Handler的dispatchMessage方法处理。如果没有Message则一直阻塞。那怎么样才能够退出循环呢？Looper提供了quit和quitSafely方法。
```
 public void quit() {
        mQueue.quit(false);
    }

 public void quitSafely() {
        mQueue.quit(true);
    }
```
quit方法会直接退出Looper,而quitSafely只是设定一个退出标记，知道队列中已有消息处理完毕以后才安全地退出。从代码中我们可以看见，其实这两个方法都是调用了MessageQueue中的quit方法。还记得我们MessageQueue中的next方法么？该方法也是一个阻塞方法，正是由于该方法的阻塞导致了loop方法的阻塞，如果MessageQueue的next方法返回null,那么loop方法就可以结束循环了。

建议如果在子线程中手动的创建了Looper对象，那么等所有事情完成以后应该调用quit方法来终止消息循环。

#5.Handler
Handler的主要工作包括发送消息和接收处理消息。Handler发送消息时在MessageQueue中插入一条消息，MessageQueue的next方法就会返回这条消息给Looper，在Looper的loop方法中得到该消息以后就会调用Handler的dispatchMessage方法，这时候就进入了消息处理阶段。
```
 public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
从代码中我们可以看见，首先要判断传入的Message中是否指定了callback变量，该变量是一个Runnale对象。如果指定了就调用handleCallback方法。
```
 private static void handleCallback(Message message) {
        message.callback.run();
    }
```
handleCallback方法也很简单，只是调用Runnable对象的run方法。

如果在定义Message对象时并没有指定callback变量。那么就判断在创建Handler的时候是否传入了Callback对象赋值给mCallback变量。
```
public interface Callback {
        public boolean handleMessage(Message msg);
    }
```
Callback是一个接口，里面只有一个handleMessage方法，我们在创建Handler时，如果要传入Callback接口，那么就得实现它的handleMessage方法，这样在dispatchMessage方法中就会调用该方法了。 

如果在创建Handler时也没有传入Callback对象，那么我们就调用Handler的handleMessage方法。
```
  public void handleMessage(Message msg) {
    }
```
可以看见Handler中的handleMessage方法是空实现。所以我们在创建Handler的时候必须复写该方法，在该方法中处理分发过来的消息。但是要注意，如果是采用匿名内部类的方式复写该方法，有可能会导致内存泄漏问题。这样就需要我们采用静态类加弱引用的方式来避免。最好在Activity的onDestroy方法中调用Handler的：
removeCallback
removeMessages
removeCallbackAndMessages
其中的一个方法来切断它们的联系。
最后我们来看Handler中比较重要的一个构造方法：
```
public Handler(Callback callback, boolean async) {
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
        mAsynchronous = async;
    }
```
从这里我们可以知道，如果Looper为null的话，就会抛出运行时异常。这说明如果要使用Handler必须要创建Looper对象。也就是执行Looper对象的prepare方法，在主线程中之所以可以不用创建，是因为在ActivityThread中系统已经创建过Looper对象。 

#6.关系图

![194707_TItG_1391648.png](http://upload-images.jianshu.io/upload_images/1870458-01e030d8d720222f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最后盗用一张别人的图来说明Handler,MessageQueue,Looper之间的关系。

如果你觉得本篇文章帮助到了你，希望大爷能够给瓶买水钱。
本文为原创文章，转载请注明出处！