# 模型

- Message：消息分为硬件产生的消息（如按钮、触摸）和软件生成的消息；
- MessageQueue：消息队列的主要功能向消息池投递消息 `MessageQueue.enqueueMessage()` 和取走消息池的消息 `MessageQueue.next()`；
- Handler：消息辅助类，主要功能向消息池发送各种消息事件 `Handler.sendMessage()` 和处理相应消息事件 `Handler.handleMessage()`；
- Looper：不断循环执行 `Looper.loop()`，按分发机制将消息分发给目标处理者。
# 类图

![[Handler class graph.png]]
# 实例分析

官方源码中给出了一个典型的用法：

```Java
class LooperThread extends Thread {
    public Handler mHandler;
    
    public void run() {
        Looper.prepare();
        
        mHandler = new Handler {
            public void handleMessage(Message msg) {
                
            }
        }
        
        Looper.loop();
    }   
}
```
# Looper

## Looper.prepare()

首先会执行 `Looper.prepare()`，其源码如下，主要用于将一个新的 `Looper` 对象放入线程存储区。

```Java
private static void prepare(boolean quitAllowed) {
    // 如果 TLS 已经有了一个 Looper 对象，抛出异常
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    // 创建新的 Looper 对象
    sThreadLocal.set(new Looper(quitAllowed));
}
```

这里的 ThreadLocal 是指线程本地存储区（Thread Local Storage，简称为 TLS），每个线程都有自己的私有的本地存储区域，不同线程之间彼此不能访问对方的 TLS 区域。

`Looper` 对象的构造函数如下：

```Java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
} 
```

另外，与 `prepare()` 相近功能的，还有一个 `prepareMainLooper()` 方法，该方法主要在 `ActivityThread` 类中使用。

```Java
public static void prepareMainLooper() {
    prepare(false); // 设置不允许退出的Looper
    synchronized (Looper.class) {
        // 将当前的Looper保存为主Looper，每个线程只允许执行一次。
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}
```

## Looper.loop()

```Java
public static void loop() {
    // 取出当前线程的 Looper
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    if (me.mInLoop) {
        Slog.w(TAG, "Loop again would have the queued messages be executed"
                + " before this one completed.");
    }
    // 打上循环标记
    me.mInLoop = true;

    // 确保在权限检查时基于本地进程，而不是调用进程，并且对此线程进行追踪
    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    // 获取调试人员设置的慢操作的阈值
    final int thresholdOverride = getThresholdOverride();

    me.mSlowDeliveryDetected = false;

    for (;;) {
        if (!loopOnce(me, ident, thresholdOverride)) {
            return;
        }
    }
}

private static boolean loopOnce(final Looper me,
        final long ident, final int thresholdOverride) {
    Message msg = me.mQueue.next(); // 可能会阻塞
    if (msg == null) {
        // 没有消息就直接返回
        return false;
    }
    final Printer logging = me.mLogging;
    if (logging != null) {
        logging.println(">>>>> Dispatching to " + msg.target + " "
                + msg.callback + ": " + msg.what);
    }
    // 保证在一个事务中，观察者不会改变
    final Observer observer = sObserver;
    final long traceTag = me.mTraceTag;
    long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
    long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
    final boolean hasOverride = thresholdOverride >= 0;
    if (hasOverride) {
        slowDispatchThresholdMs = thresholdOverride;
        slowDeliveryThresholdMs = thresholdOverride;
    }
    final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0 || hasOverride)
            && (msg.when > 0);
    final boolean logSlowDispatch = (slowDispatchThresholdMs > 0 || hasOverride);
    final boolean needStartTime = logSlowDelivery || logSlowDispatch;
    final boolean needEndTime = logSlowDispatch;
    if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
        Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
    }
    final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
    final long dispatchEnd;
    Object token = null;
    if (observer != null) {
        token = observer.messageDispatchStarting();
    }
    long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
    // 下方是日志记录
    try {
        msg.target.dispatchMessage(msg); // 分配消息
        if (observer != null) {
            observer.messageDispatched(token, msg);
        }
        dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
    } catch (Exception exception) {
        if (observer != null) {
            observer.dispatchingThrewException(token, msg, exception);
        }
        throw exception;
    } finally {
        ThreadLocalWorkSource.restore(origWorkSource);
        if (traceTag != 0) {
            Trace.traceEnd(traceTag);
        }
    }
    if (logSlowDelivery) {
        boolean slow = false;
        if (!me.mSlowDeliveryDetected || NoImagePreloadHolder.sVerboseLogging) {
            slow = showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart,
                    "delivery", msg);
        }
        if (me.mSlowDeliveryDetected) {
            if (!slow && (dispatchStart - msg.when) <= 10) {
                Slog.w(TAG, "Drained");
                me.mSlowDeliveryDetected = false;
            }
        } else if (slow) {

            me.mSlowDeliveryDetected = true;
        }
    }
    if (logSlowDispatch) {
        showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
    }
    if (logging != null) {
        logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
    }
    
    // 恢复调用者信息
    final long newIdent = Binder.clearCallingIdentity();
    if (ident != newIdent) {
        Log.wtf(TAG, "Thread identity changed from 0x"
                + Long.toHexString(ident) + " to 0x"
                + Long.toHexString(newIdent) + " while dispatching to "
                + msg.target.getClass().getName() + " "
                + msg.callback + " what=" + msg.what);
    }
    msg.recycleUnchecked(); // 将Message放入消息池
    return true;
}
```

`loop()` 进入循环模式，不断重复下面的操作，直到没有消息时退出循环：

- 读取 `MessageQueue` 的下一条 `Message`；
- 把 `Message` 分发给相应的 target；
- 再把分发后的 `Message` 回收到消息池，以便重复利用。

## Looper.quit()

```Java
public void quit() {
    mQueue.quit(false); // 消息移除
}

public void quit() {
    mQueue.quit(true); // 安全的消息移除
}
// ------------------
// MessageQueue.java
void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }
    // 选择两种队列实现
    if (mUseConcurrent) { // 新版实现用于系统进程，具有更高的并发性
        synchronized (mIdleHandlersLock) {
            if (sQuitting.compareAndSet(this, false, true)) {
                if (safe) {
                    removeAllFutureMessages(); // 移除所有未来的消息
                } else {
                    removeAllMessages(); // 移除所有消息
                }
                // 由于 sQuitting 之前是 false，我们可以认为 mPtr 不为空
                nativeWake(mPtr);
            }
        }
    } else { // 默认采用旧版实现
        synchronized (this) {
            if (mQuitting) { // 防止多次队列退出操作
                return;
            }
            mQuitting = true;
            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }
            // 由于 mQuitting 之前是 false，我们可以认为 mPtr 不为空
            nativeWake(mPtr);
        }
    }
}
```

## Looper.myLooper()

用于获取 TLS 存储的 Looper 对象。

```Java
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

## Looper.post()

原来这个方法是用于发送消息，并设置消息的 callback。现在删除了该方法，发送消息和设置的功能已经完全移到了 `Handler` 类中。

# Handler

## Handler generator

```Java
public Handler(@NonNull Looper looper, @Nullable Callback callback) {
    this(looper, callback, false);
}

public Handler(@Nullable Callback callback, boolean async) {
    // 匿名类、内部类或本地类都必须申明为 static，否则会警告可能出现内存泄露
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    // 在获取当前 Looper 对象前需要调用 Looper.prepare()
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread " + Thread.currentThread()
                    + " that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue; // 将 Looper 的消息队列设置为 Handler 的消息队列
    mCallback = callback; // 回调方法
    mAsynchronous = async; // 设置消息是否异步处理
    mIsShared = false;
}

public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
    this(looper, callback, async, /* shared= */ false);
}

public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async,
        boolean shared) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
    mIsShared = shared;
}
```

当不指定 Looper 时，会自动获取当前线程的 Looper。

## 消息分发

```Java
public void dispatchMessage(@NonNull Message msg) {
    if (msg.callback != null) {
        // 当 msg 有自己的回调方法，直接调用 msg 的回调方法
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            // 当 Looper 设置了回调方法，则调用 Looper 设置的回调方法
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        // 调用 Looper 类自己的回调方法，默认为空，需要子类进行覆写
        handleMessage(msg);
    }
} 
```

## 消息发送

调用链如图所示：

![[Send Message.png]]

## Handler.obtainMessage()

```Java
public final Message obtainMessage() {
    return Message.obtain(this);
}
```

## Handler.**removeMessages()**

```Java
public final void removeMessages(int what) {
    mQueue.removeMessages(this, what, null);
}
```

# MessageQueue

MessageQueue 是消息机制的 Java 层和 C++ 层的连接纽带，大部分核心方法都交给 native 层来处理，其中 MessageQueue 类中涉及的 native 方法如下：

```Java
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

## MessageQueue generator

```Java
MessageQueue(boolean quitAllowed) {
    initIsProcessAllowedToUseConcurrent(); // 通过进程号等
    mUseConcurrent = sIsProcessAllowedToUseConcurrent && !isInstrumenting();
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

## MessageQueue.next()

本函数用于提取下一个 Message，通过调用 `nativePollOnce()` 来等待新的消息，当设置 `nextPollTimeoutMillis` 为 -1 时，表明队列为空，需要一直等待下去。

```Java
Message next() {
    if (mUseConcurrent) {
        return nextConcurrent(); // 如果被打上了并发标记，则进入 nextConcurrent()
    }
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }
    int pendingIdleHandlerCount = -1; // 首次循环迭代为 -1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        // 阻塞操作，当等待 nextPollTimeoutMillis 时长，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            // 尝试提取下一个 Message
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // 寻找异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // 下一个 Message 还没有准备好，需要设置定时器
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 得到一个 Message，将其从链表中取出并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                        if (prevMsg.next == null) {
                            mLast = prevMsg;
                        }
                    } else {
                        mMessages = msg.next;
                        if (msg.next == null) {
                            mLast = null;
                        }
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG_L, "Returning message: " + msg);
                    msg.markInUse();
                    if (msg.isAsynchronous()) {
                        mAsyncMessageCount--;
                    }
                    if (TRACE) {
                        Trace.setCounter("MQ.Delivered", mMessagesDelivered.incrementAndGet());
                    }
                    return msg;
                }
            } else {
                // 没有更多的 Message
                nextPollTimeoutMillis = -1;
            }
            // 消息正在退出，返回 null
            if (mQuitting) {
                dispose();
                return null;
            }
            // 当消息队列为空，或是消息队列的第一个消息时
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // 没有 idle handlers 需要运行，需要循环并等待
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        
        // 第一次迭代运行 idle handlers
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // 释放 handler 的引用
            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG_L, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        // 重置 idle hanlers 的数量为 0
        pendingIdleHandlerCount = 0;
        // 当调用一个空闲 handler 时，一个新 message 能够被分发，因此无需等待可以直接查询 pending message.
        nextPollTimeoutMillis = 0;
    }
}
```

## MessageQueue.enqueueMessage()

```Java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (mUseConcurrent) {
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
        return enqueueMessageUnchecked(msg, when);
    }
    synchronized (this) {
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }
        if (mQuitting) { // 正在退出时，回收 msg，加入到消息池
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG_L, e.getMessage(), e);
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // 当 MessageQueue 没有消息，或者 msg 的触发时间是队列中最早的
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; // 阻塞时需要唤醒
            if (p == null) {
                mLast = mMessages;
            }
        } else {
            // 将消息按时间顺序插入到 MessageQueue。一般地，不需要唤醒事件队列，除非
            // 消息队头存在 barrier，并且同时 Message 是队列中最早的异步消息。
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            // 为了可读性，我们将这部分函数按照 tail tracking 是否使能被分成了两块
            if (Flags.messageQueueTailTracking()) {
                if (when >= mLast.when) {
                    needWake = needWake && mAsyncMessageCount == 0;
                    msg.next = null;
                    mLast.next = msg;
                    mLast = msg;
                } else {
                    // 在队列中间进行插入
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
                    if (p == null) {
                        // 在队尾进行插入
                        mLast = msg;
                    }
                    msg.next = p;
                    prev.next = msg;
                }
            } else {
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
                msg.next = p;
                prev.next = msg;
                mLast = null;
            }
        }
        if (msg.isAsynchronous()) {
            mAsyncMessageCount++;
        }
        // 我们可以假设 mPtr 不为空，因为消息没有退出
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

## MessageQueue.removeMessages()

这里之所以采用两个循环，是因为一个循环需要考虑边界条件，第二个循环就不需要了。

```Java
void removeMessages(Handler h, int what, Object object) {
    if (h == null) {
        return;
    }
    if (mUseConcurrent) {
        findOrRemoveMessages(h, what, object, null, 0, mMatchHandlerWhatAndObject, true);
        return;
    }
    synchronized (this) {
        Message p = mMessages;
        // 从头部开始移除符合条件的所有信息
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            if (p.isAsynchronous()) {
                mAsyncMessageCount--;
            }
            p.recycleUnchecked();
            p = n;
        }
        if (p == null) {
            mLast = mMessages;
        }
        // 移除剩余符合要求的信息
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                        && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    if (n.isAsynchronous()) {
                        mAsyncMessageCount--;
                    }
                    n.recycleUnchecked();
                    p.next = nn;
                    if (p.next == null) {
                        mLast = p;
                    }
                    continue;
                }
            }
            p = n;
        }
    }
}
```

## MessageQueue.postSyncBarrier()

有一类特殊的 message 没有 target，即同步 barrier token。这个消息的价值就是用于拦截同步消息，所以并不会唤醒 Looper。

```Java
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    if (mUseConcurrent) {
        final int token = mNextBarrierTokenAtomic.getAndIncrement();
        mNextBarrierToken = token + 1;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.arg1 = token;
        if (!enqueueMessageUnchecked(msg, when)) {
            Log.wtf(TAG_C, "Unexpected error while adding sync barrier!");
            return -1;
        }
        return token;
    }
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;
        if (Flags.messageQueueTailTracking() && mLast != null && mLast.when <= when) {
            mLast.next = msg;
            mLast = msg;
            msg.next = null;
            return token;
        }
        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (p == null) {
            mLast = msg;
        }
        if (prev != null) {
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

和删除普通 Message 一样，`MessageQueue.removeSyncBarrier()` 可以对同步消息进行删除。
# Message

## 消息对象

| 数据类型     | 成员变量     | 解释     |
| -------- | -------- | ------ |
| int      | what     | 消息类别   |
| long     | when     | 消息触发时间 |
| int      | arg1     | 参数1    |
| int      | arg2     | 参数2    |
| Object   | obj      | 消息内容   |
| Handler  | target   | 消息响应方  |
| Runnable | callback | 回调方法   |

创建消息的时候就是填写上述内容的一项或多项。

## 消息池

Message 类会有一个消息池，减少了创建的对象的开销。静态变量 `sPool` 的数据类型为 `Message` 通过 `next` 成员变量，维护一个消息池；静态变量 `MAX_POOL_SIZE` 代表消息池的可用大小；默认大小为 50。

## obtain

把消息池表头的 Message 取走，再把表头指向 next。

```Java
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // 清除使用标签
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}
```

## recycle

把不再使用的消息加入消息池。

```Java
public void recycle() {
    if (isInUse()) {
        if (gCheckRecycle) {
            throw new IllegalStateException("This message cannot be recycled because it "
                    + "is still in use.");
        }
        return;
    }
    recycleUnchecked();
}

@UnsupportedAppUsage
void recycleUnchecked() {
    // Mark the message as in use while it remains in the recycled object pool.
    // Clear out all other details.
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = UID_NONE;
    workSourceUid = UID_NONE;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) {
            next = sPool;
            sPool = this;
            sPoolSize++;
        }
    }
}
```

# Java 层总结

![[Handler Java.png]]
# Handler 通信机制（Native C++ 层）

`MessageQueue` 类里面涉及到多个 native 方法，除了 `MessageQueue` 的 native 方法，Native 层本身也有一套完整的消息机制，用于处理 Native 层的消息，如下图 Native 层的消息机制。

![[Native Handler.png]]
在整个消息机制中，而 `MessageQueue` 是连接 Java 层和 Native 层的纽带，换言之，Java 层可以向 `MessageQueue` 消息队列中添加消息，Native 层也可以向 `MessageQueue` 消息队列中添加消息，接下来来看看 `MessageQueue`。
## nativeInit()

初始化过程的调用链如下：

![[Native Handler Init.png]]
当 `MessageQueue` 的构造函数调用 `MessageQueue.nativeInit()` 后，其通过 JNI 被映射到了 `android_os_Message_nativeInit()` 函数中。

```C++
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env); // 增加引用计数
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

然后我们来看 `NativeMessageQueue` 的构造函数：

```C++
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread(); // 获取 Thread 中的 Looper 对象
    if (mLooper == NULL) {
        mLooper = new Looper(false); // 创建 native 层的 Looper
        Looper::setForThread(mLooper); // 保存 native 层的 Looper 到 TLS 
    }
}
```

其中又调用了 Native C++ 层的 Looper 的构造函数：

```C++
Looper::Looper(bool allowNonCallbacks)
    : mAllowNonCallbacks(allowNonCallbacks),
      mSendingMessage(false),
      mPolling(false),
      mEpollRebuildRequired(false),
      mNextRequestSeq(WAKE_EVENT_FD_SEQ + 1),
      mResponseIndex(0),
      mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd.reset(eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC)); // 重置唤醒事件 fd
    LOG_ALWAYS_FATAL_IF(mWakeEventFd.get() < 0, "Could not make wake event fd: %s", strerror(errno));

    AutoMutex _l(mLock);
    rebuildEpollLocked(); // 重建 Epoll 事件
}
```

查看 `rebuildEpollLocked()` 函数：

将 `Looper` 对象中的 `mWakeEventFd` 添加到 epoll 监控，以及 `mRequests` 也添加到 epoll 的监控范围内。

```C++
void Looper::rebuildEpollLocked() {
    // 将之前的 epoll fd 关闭
    if (mEpollFd >= 0) {
#if DEBUG_CALLBACKS
        ALOGD("%p ~ rebuildEpollLocked - rebuilding epoll set", this);
#endif
        mEpollFd.reset();
    }

    // 新建 epoll fd
    mEpollFd.reset(epoll_create1(EPOLL_CLOEXEC));
    LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance: %s", strerror(errno));

    epoll_event wakeEvent = createEpollEvent(EPOLLIN, WAKE_EVENT_FD_SEQ);
    int result = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, mWakeEventFd.get(), &wakeEvent);
    LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake event fd to epoll instance: %s",
                        strerror(errno));

    for (const auto& [seq, request] : mRequests) {
        epoll_event eventItem = createEpollEvent(request.getEpollEvents(), seq);

        int epollResult = epoll_ctl(mEpollFd.get(), EPOLL_CTL_ADD, request.fd, &eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set: %s",
                  request.fd, strerror(errno));
        }
    }
}
```

## nativeDestory()

调用链为：

![[Native Handler Destory.png]]
`MessageQueue.dispose()` 调用了 `nativeDestory()`。

```Java
private void dispose() {
    if (mPtr != 0) {
        nativeDestroy(mPtr);
        mPtr = 0;
    }
}
```

其通过 JNI 映射到了 `android_os_MessageQueue_nativeDestroy()`。

```C++
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->decStrong(env);
}
```

其调用 `nativeMessageQueue->decStrong(env)`

```C++
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->removeStrongRef(id); // 移除强引用
    const int32_t c = refs->mStrong.fetch_sub(1, std::memory_order_release);
#if PRINT_REFS
    ALOGD("decStrong of %p from %p: cnt=%d\n", this, id, c);
#endif
    LOG_ALWAYS_FATAL_IF(
            BAD_STRONG(c),
            "decStrong() called on %p too many times, possible memory corruption. Consider "
            "compiling with ANDROID_UTILS_REF_BASE_DISABLE_IMPLICIT_CONSTRUCTION for better errors",
            refs);
    if (c == 1) {
        std::atomic_thread_fence(std::memory_order_acquire);
        refs->mBase->onLastStrongRef(id);
        int32_t flags = refs->mFlags.load(std::memory_order_relaxed);
        if ((flags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            delete this; // 在此情况下，析构函数不会删除引用。
        }
    }
    refs->decWeak(id); // 移除弱引用
}
```

## nativePollOnce()

调用链为：

![[Native Handler Poll Once.png]]
在 `MessageQueue.next()` 中，调用了 `nativePollOnce()`，其通过 JNI 映射到了 `android_os_MessageQueue_nativePollOnce()`。

```C++
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    // 将 ptr 转化为 NativeMEssageQueue*
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}
```

其调用了 `nativeMessageQueue->pollOnce(env, obj, timeoutMillis)`。

```C++
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis);
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```

进入 `Looper->pollOnce(timeoutMillis)`。

```C++
inline int pollOnce(int timeoutMillis) {
    return pollOnce(timeoutMillis, nullptr, nullptr, nullptr);
}

/*
timeoutMillis：超时时长
outFd：发生事件的文件描述符
outEvents：当前 outFd 上发生的事件，包含以下 4 类事件
    EVENT_INPUT 可读
    EVENT_OUTPUT 可写
    EVENT_ERROR 错误
    EVENT_HANGUP 中断
outData：上下文数据
*/
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        // 先处理没有 Callback 方法的 Response 事件
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                // ident 大于 0，则表示没有 callback, 因为 POLL_CALLBACK = -2
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE
                ALOGD("%p ~ pollOnce - returning signalled identifier %d: "
                        "fd=%d, events=0x%x, data=%p",
                        this, ident, fd, events, data);
#endif
                if (outFd != nullptr) *outFd = fd;
                if (outEvents != nullptr) *outEvents = events;
                if (outData != nullptr) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
#if DEBUG_POLL_AND_WAKE
            ALOGD("%p ~ pollOnce - returning result %d", this, result);
#endif
            if (outFd != nullptr) *outFd = 0;
            if (outEvents != nullptr) *outEvents = 0;
            if (outData != nullptr) *outData = nullptr;
            return result;
        }
        // 再处理内部轮询
        result = pollInner(timeoutMillis);
    }
}
```

处理内部轮询调用 `Looper.pollInner(timeoutMillis)`。

```C++
int Looper::pollInner(int timeoutMillis) {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - waiting: timeoutMillis=%d", this, timeoutMillis);
#endif

    // 基于下个 message 到期时间对计时时间进行调整
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - next message in %" PRId64 "ns, adjusted timeout: timeoutMillis=%d",
                this, mNextMessageUptime - now, timeoutMillis);
#endif
    }

    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    // 即将处于 idle 状态
    mPolling = true;

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    int eventCount = epoll_wait(mEpollFd.get(), eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    // 不再处于 idle 状态
    mPolling = false;

    // 请求本队列的锁
    mLock.lock();

    // 在需要的时候重新建立 epoll fd
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();
        goto Done;
    }

    // 检查 epoll 错误
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        ALOGW("Poll failed with an unexpected error: %s", strerror(errno));
        result = POLL_ERROR;
        goto Done;
    }

    // 检查 epoll 是否超时
    if (eventCount == 0) {
#if DEBUG_POLL_AND_WAKE
        ALOGD("%p ~ pollOnce - timeout", this);
#endif
        result = POLL_TIMEOUT;
        goto Done;
    }

    // 处理所有事件
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ pollOnce - handling events from %d fds", this, eventCount);
#endif

    for (int i = 0; i < eventCount; i++) {
        const SequenceNumber seq = eventItems[i].data.u64;
        uint32_t epollEvents = eventItems[i].events;
        if (seq == WAKE_EVENT_FD_SEQ) {
            if (epollEvents & EPOLLIN) {
                awoken(); // 进行唤醒
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            const auto& request_it = mRequests.find(seq);
            if (request_it != mRequests.end()) {
                const auto& request = request_it->second;
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                // 处理 request，生成对应的 response 对象，push 到响应数组
                mResponses.push({.seq = seq, .events = events, .request = request});
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x for sequence number %" PRIu64
                      " that is no longer registered.",
                      epollEvents, seq);
            }
        }
    }
Done: ;

    // 调用相应的回调方法
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            {
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock(); // 我们会在这里释放锁，handler 会在执行 handleMessage 后才会被删除

#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
                ALOGD("%p ~ pollOnce - sending message: handler=%p, what=%d",
                        this, handler.get(), message.what);
#endif
                handler->handleMessage(message); 
            }

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK; // 发生回调
        } else {
            // 最后一个的消息是在队首的，它决定了下次唤醒时间
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    // 释放锁
    mLock.unlock();

    // 处理带有 Callback() 方法的 Response 事件，执行 Reponse 相应的回调方法
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
#if DEBUG_POLL_AND_WAKE || DEBUG_CALLBACKS
            ALOGD("%p ~ pollOnce - invoking fd event callback %p: fd=%d, events=0x%x, data=%p",
                    this, response.request.callback.get(), fd, events, data);
#endif
            // 处理回调方法，注意 fd 可能在 callback 中会被释放，需要注意之后对 fd 的回收
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                AutoMutex _l(mLock);
                // 这里在删除时，就考虑到了 fd 被 callback 释放的情况，这时 epoll fd 会被重建
                removeSequenceNumberLocked(response.seq);
            }

            response.request.callback.clear(); // 清除 repsonse 引用的回调方法
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

再看一下 `Looper.awoken()`：

```C++
void Looper::awoken() {
    uint64_t counter;
    // 不断读取管道数据，目的就是为了清空管道内容
    TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t)));
}
```

对于 `pollInner()`，其整体逻辑为：

1. 先调用 `epoll_wait()`，这是阻塞方法，用于等待事件发生或者超时；
2. 对于 `epoll_wait()` 返回，当且仅当以下3种情况出现：
	1. `POLL_ERROR`，发生错误，直接跳转到 Done；
    2. `POLL_TIMEOUT`，发生超时，直接跳转到 Done；
    3. 检测到管道有事件发生，则再根据情况做相应处理：
        - 如果是管道读端产生事件，则直接读取管道的数据；
        - 如果是其他事件，则处理 request，生成对应的 reponse 对象，push 到 reponse数组；
3. 进入 Done 标记位的代码段：
    1. 先处理 Native 的 Message，调用 Native 的 Handler 来处理该 Message;
    2. 再处理 Response 数组，`POLL_CALLBACK` 类型的事件；
## nativeWake()

调用链如下：

![[Native Handler Wake.png]]
`nativeWake()` 的调用位置为往消息队列添加 Message 时，需要根据 mBlocked 情况来决定是否需要调用 nativeWake。其通过 JNI 映射到了如下方法：

```C++
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}

void NativeMessageQueue::wake() {
    mLooper->wake();
}

void Looper::wake() {
#if DEBUG_POLL_AND_WAKE
    ALOGD("%p ~ wake", this);
#endif

    uint64_t inc = 1;
    // 向 mWakeEventFd 写入字符 1，唤醒该 fd，而写操作失败也会不断进行重试
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd.get(), &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            LOG_ALWAYS_FATAL("Could not write wake signal to fd %d (returned %zd): %s",
                             mWakeEventFd.get(), nWrite, strerror(errno));
        }
    }
}
```

## sendMessage

最核心的是 `sendMessageAtTime()`：

```C++
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
    size_t i = 0;
    { 
        // 请求锁
        AutoMutex _l(mLock);
        size_t messageCount = mMessageEnvelopes.size();
        // 找到 message 应该插入的位置 i
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }
        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);
        // 如果当前正在发送消息，那么不再调用 wake()，直接返回
        if (mSendingMessage) {
            return;
        }
    } // 释放锁
    // 当把消息加入到消息队列的头部时，需要唤醒 poll 循环。
    if (i == 0) {
        wake();
    }
}
```

# Native 层总结

![[Handler Native.png]]
- 红色虚线关系：Java 层和 Native 层的 `MessageQueue` 通过 JNI 建立关联，彼此之间能相互调用。
- 蓝色虚线关系：`Handler/Looper/Message` 这三大类 Java 层与 Native 层并没有任何的真正关联，只是分别在 Java 层和 Native 层的 handler 消息模型中具有相似的功能。都是彼此独立的，各自实现相应的逻辑。
- `WeakMessageHandler` 继承于 `MessageHandler` 类，`NativeMessageQueue` 继承于 `MessageQueue` 类。
- 消息处理流程是先处理 Native Message，再处理 Native Request，最后处理 Java Message。