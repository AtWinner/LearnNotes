不同形式的线程虽然都是线程，但是它们仍然具有不同的特性和实用场景：
- AsyncTask封装了线程池和Handler，它主要是为了方便开发者在子线程中更新UI
- HandlerThread是一种具有消息循环的线程，在它的内部可以使用Handler
- IntentService是一个服务，系统对其进行了封装使其可以更方便的执行后台任务，IntentService内部采用HandlerThread来执行任务，当任务执行完毕后IntentService会自动退出

从任务执行的角度来看，IntentService的作用很像一个后台线程，但是IntentService是一种服务，它不容易被系统杀死从而可以尽量保证任务的执行，而如果是一个后台线程，哟郁郁这个时候进程中没有活动的四大组件，那么这个进程的优先级就会非常低，会很容易被系统杀死。

在操作系统中，线程是操作系统调度的最小单位，同时线程又是一种受限的系统资源，即线程不可能无限制地产生，并且线程的创建和销毁都会有相应的开销。当系统中存在大量的线程时，系统会通过时间片轮转的方式调度每个线程，因此线程不可能做到绝对的秉性，除非线程数量小于等于CPU的核心数，一般来说这是不可能的。如果一个进程中频繁创建和销毁线程，这显然不是高效的做法，正确的方法是采用线程池，一个线程池会缓存一定数量的线程，通过线程池就可以避免因为频繁创建和销毁线程所带来的系统开销。Android的线程池来源于Java，主要是通过Executor来派生特定类型的线程池，不同类型的线程池又具有各自不同的特性。

# Android中的线程形态
## AsyncTask
系统提供了AsyncTask，AsyncTask经过了几次修改，导致了对于不同的API版本AsyncTask具有不同的表现，尤其是多任务的并发执行上。AsyncTask是一个轻量级的异步任务类，它可以在线程池中执行后台，然后把执行的进度和最终结果传递给主线程并在主线程中更新UI，从实现上来说，AsyncTask封装了Thread和Handler，通过AsyncTask可以更加方便的执行后台任务以及在主线程中访问UI，但是AsyncTask并不适合进行特别耗时的后台任务，对于特别耗时的任务来说，建议使用线程池。

AsyncTask是一个抽象的泛型类，它提供了Params、Progress和Result这三个泛型参数，其中Params表示参数的类型，Progress表示后台任务的执行进度的类型，而Result则表示后台任务的返回结果的类型，如果AsyncTask确实不需要传递具体的参数，那么这三个泛型参数可以用Void来代替。AsyncTask这个类的声明如下所示。

``` java
public abstract class AsyncTask<Params, Progress, Result>
```
AsyncTask提供了4个核心方法
1. onPreExecute()在主线程中执行，在异步任务执行之前，此方法会被调用，一般可以用于做一下准备工作。
2. doInBackground(Params... params)在线程池中执行，此方法用于执行异步任务，params参数表示异步任务的输入参数。在此方法中可以通过publishProgress方法来更新任务的进度，publishProgress方法会调用onProgressUpdate方法。另外此方法需要返回计算结果给onPostExecute方法。
3. onProgressUpdate(Progress... values)在主线程中执行，当后台任务的执行进度发生改变时，此方法会被调用。
4. onPostExecute(Result result)在主线程中执行，在异步任务执行之后，此方法会被调用，其中result参数是后台任务的返回值，即DoInBackground的返回值。

AsyncTask在具体使用的过程中也有一些条件限制，主要有以下几点：
1. AsyncTask的类必须在主线程中加载，这就意味着第一次访问AsyncTask必须发生在主线程，当然这个过程在Android4.1以及以上版本中已经被系统自动完成。在Android5.0的源码中，可以查看ActivityThread的main方法，它会调用AsyncTask的init方法，这就满足了AsyncTask的类必须在主线程中进行加载这个条件了。至于为什么必须要满足这个条件。
2. AsyncTask的对象必须在主线程中创建
3. execute方法必须在UI线程调用
4. 不要在程序中直接调用onPreExecute、doInBackground、onProgressUpdate、onPostExecute方法
5. 一个onPostExecute对象只能执行一次，即只能调用一次execute方法，窦泽会报运行时异常
6. 在Android1.6之前，AsyncTask是串行执行任务的，Android1.6的时候AsyncTask开始采用线程池里处理并行任务，但是从Android3.0开始，为了避免AsyncTask所带来的并发错误，AsyncTask又采用一个线程来串行执行任务。尽管如此，在Android3.0以后以及后续的版本中，我们仍然可以通过AsyncTask的executeOnExecutor方法来并行执行任务。

## AsyncTask的工作原理

``` java
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

@MainThread
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
在上面的代码中，sDefaultExecutor实际上师一个串行的线程池，一个进程中所有AsyncTask全部在这个串行的线程池中排队执行，在executeOnExecutor方法中，AsyncTask的onPreExecute方法最先执行，最后线程池开始执行，然后线程池开始执行。下面分析线程池的执行过程

``` java
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
从SerialExecutor的实现可以分析AsyncTask的排队执行的过程。首先系统会把AsyncTask的Params参数封装为FutureTask对象，FutureTask是一个并发类，在这里它冲当了Runnable的作用。接着这个FutureTask就会交给SerialExecutor的execue方法去处理，SerialExecutor的execute方法首先会吧FutureTask对象插入到任务队列mTasks中，如果这个时候没有正在活动的AsyncTask任务，那么就会调用SerialExecutor的scheduleNext方法来执行下一个AsyncTask任务。同时当一个AsyncTask任务执行完后，AsyncTask会继续执行其他任务直到所有的任务都被执行为止，从这一点可以看出，在默认的情况下，AsyncTask是串行执行的。

AsyncTask中有两个线程池（SerialExecutor和THREAD_POOL_EXECUTOR）和一个Handler（InternalHandler），其中线程池SerialExecutor用于任务的排毒，而线程池THREAD_POOL_EXECUTOR用于真正的执行任务，InternalHandler用于将执行环境从线程池切换到主线程，其本质任然是线程的调度过程。在AsyncTask的构造方法中有那么一段代码，由于FutureTask的run方法会调用mWork的call方法，因此mWork的call方法最终会再线程池中执行。
``` java
mWorker = new WorkerRunnable<Params, Result>() {
    public Result call() throws Exception {
        mTaskInvoked.set(true);
        Result result = null;
        try {
            Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
            //noinspection unchecked
            result = doInBackground(mParams);
            Binder.flushPendingCommands();
        } catch (Throwable tr) {
            mCancelled.set(true);
            throw tr;
        } finally {
            postResult(result);
        }
        return result;
    }
};
```
在mWork的call方法中，首先将mTaskInvoked设为true，表示当前任务已经被调用过了，然后执行AsyncTask的doInBackground方法，接着将返回值传递给postResult方法，postResult实现如下：

``` java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}

private static Handler getMainHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}
```
postResult会通过getHandler()发送一个MESSAGE_POST_RESULT的消息，这个handler的定义如下：

``` java
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
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
可以发现sHandler是一个静态Handler对象，为了能够将执行环境切换到主线程，这就要求sHandler这个对象必须在主线程中创建。由于静态成员会在加载类的时候进行初始化，因此这就变相的要求AsyncTask的类必须在主线程中加载，否则同一个线程中AsyncTask都将无法正常工作。sHandler收到MESSAGE_POST_DEFAULT这个消息后会调用AsyncTask的finish方法。

``` java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```
finish方法的逻辑比较简单，如果AsyncTask被取消了，那么就调用onCancelled方法，否则会调用onPostExecute方法，可以看到doInBackground的返回结果会传递给onPostExecute方法。

## HandlerThread
HandlerThread继承自Thread，它是一种可以使用Handler的Thread，它的实现也很简单，在run方法中通过Looper.prepare()创建消息队列，并且通过Looper.loop()开启消息循环，这样在实际的使用中就允许在HandlerThread中创建Handler了。
``` java
@Override
public void run() {
    mTid = Process.myTid();
    Looper.prepare();
    synchronized (this) {
        mLooper = Looper.myLooper();
        notifyAll();
    }
    Process.setThreadPriority(mPriority);
    onLooperPrepared();
    Looper.loop();
    mTid = -1;
}
```

从HandlerThread的实现来看，它和普通的Thread有显著的不同之处。普通Thread主要用于run方法中执行了一个耗时任务，而HandlerThread在内部创建了一个消息队列，外界需要通过Handler的消息方式来通知HandlerThread执行一个具体的任务。HandlerThread是一个很有用的类，它在Android中的一个具体的使用场景是IntentService。由于HandlerThread的run方法是一个无限循环，因此当明确不需要HandlerThread时，可以通过quit或者quitSafely方法来终止这个线程，这是一种良好的变成习惯。

# Android中的线程池
线程池的好处：

- 重用线程池中的线程，避免因为线程的创建和销毁所带来的性能开销；
- 能有效控制线程池的最大并发数，避免大量的线程之间因互相抢占系统资源而导致系统阻塞；
- 能够对线程进行简单的管理，并提供定时执行以及制定间隔循环执行等功能。

Android中的线程池的概念来源于Java中的Executor，Executor是一个接口，真正实现线程池的是ThreadPoolExecutor。ThreadPoolExecutor提供了一系列参数来配置线程池，通过不同的参数可以创建不同的线程池，从线程池的功能特性上来说，Android线程池主要分为4类，这4类线程池可以通过Executors所提供的工厂方法来得到。

## ThreadPoolExecutor
ThreadPoolExecutor的构造方法提供了多个参数：

``` java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory)
```

#### corePoolSize
线程池的核心线程数，默认情况下，核心线程会线程池中一直存活，即使它们处于闲置状态。如果京ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true，那么闲置的核心线程在等待新任务到来时会有超时策略，这个时间间隔由keepAliveTime所指定，当等待时间超过keepAliveTime所指定的时长后，核心线程就会被终止。

#### maximumPoolSize
线程池所能容纳的最大线程数，当活动线程数达到这个数值后，后续的任务将会被阻塞。

#### keepAliveTime
非核心线程闲置时的超市市场，超过这个时长，非核心线程就会被回收。当ThreadPoolExecutor的allowCoreThreadTimeOut属性设置为true时，keepAliveTime同样会作用于核心线程。

#### unit
用于指定keepAliveTime参数的时间单位，这是一个枚举，常用的有TimeUnit.MILLISECONDS（毫秒）、TimeUnit.SECOND（秒）以及TimeUnit.MINUTE（分钟）等。

#### workQueue
线程池中的任务队列，通过线程池的execute方法提交的Runnable对象会存储在这个参数中。

#### threadFactory
线程工厂，为线程池提供创建新线程的功能。threadFactory是一个接口，它只有一个方法：Thread newThread(Runnable r)。

除了上面的参数，ThreadPoolExecutor还有一个不常用的参数，RejectedExecutionHandler handler。当线程池无法执行新任务时，可能是由于任务队列已满或者无法成功执行任务，这个时候ThreadPoolExecutor会调用handler的rejectedExecution方法来通知调用者，默认情况下，rejectedExecution会直接抛出一个RejectedExecutionExecption。


ThreadPoolExecutor执行任务时大致遵循如下规则：
1. 如果线程池中的线程数量未达到核心线程的数量，那么会直接启动一个新的核心线程执行任务。
2. 如果线程池中的数量已经达到或者超过核心线程的数量，那么任务会被插入到任务队列中排队等待执行。
3. 如果步骤2无法将任务插入到任务队列中，这往往是由于任务排队已满，这个时候如果线程数量未达到线程池规定的最大值，那么会立刻启动一个非核心线程来执行任务。
4. 如果步骤3中线程数量已经达到线程池规定的最大值，那么久拒绝执行此任务，ThreadPoolExecutor会调用RejectExecutionHandler的rejectedExecution方法来通知调用者。

ThreadPoolExecutor的参数配置在AsyncTask中有明显的提现，下面是AsyncTask中的线程池的配置情况：

``` java
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
// We want at least 2 threads and at most 4 threads in the core pool,
// preferring to have 1 less than the CPU count to avoid saturating
// the CPU with background work
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final int KEEP_ALIVE_SECONDS = 30;

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
public static final Executor THREAD_POOL_EXECUTOR;

static {
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
            sPoolWorkQueue, sThreadFactory);
    threadPoolExecutor.allowCoreThreadTimeOut(true);
    THREAD_POOL_EXECUTOR = threadPoolExecutor;
}
```

从上面的代码可以知道，AsyncTask对THREAD_POOL_EXECUTOR这个线程进行了配置，配置之后的线程池规格如下：
- 核心线程数量等于CPU核心数+1
- 线程池的最大线程数为CPU核心数的2倍+1
- 核心线程数无超时机制，非核心线程在闲置时的超时时间为1秒
- 任务队列的容量是128

## 线程池的分类
Android中最常见的四类具有不同特性的线程池，它们都是直接或者间接的通过配置ThreadPoolExecutor来实现自己的功能特性，这四类线程池分别是FixedThreadPool、CacheThreadPool、ScheduledThreadPool以及SingleThreadExecutor。

#### FixedThreadPool
通过Executors的newFixedThreadPool方法来创建，它是一种线程数量固定的线程池，当线程处于空闲状态时，它们并不会被回收，除非线程池被关闭。当所有的线程都处于活动状态的时候，新任务都会处于等待状态，直到有线程空闲出来。由于FixedThreadPool只有核心线程并且这些核心线程不会被回收，这意味着它能够更加快速的响应外界的请求。newFixedThreadPool方法的实现如下，可以发现FixedThreadPool中只有核心线程并且这些核心线程没有超时机制，另外任务队列也没有大小限制的。

``` java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

#### CacheThreadPool
通过Executors的newCachedThreadPool方法来创建。它是一种线程数量不定的线程池，它只有非核心线程，并且其最大线程数为Integer.MAX_VALUE。由于Integer.MAX_VALUE是一个很大的数，实际上就相当于最大线程数可以任意大。当线程池中的线程都处于活动状态时，线程池会创建新的线程来处理新任务，否则就会利用空闲的线程来处理新任务。线程池中的线程都有超时机制，这个超时时长为60秒，超过60秒闲置线程就会被回收。和FixedThreadPool不同的是，CachedThreadPool的任务队列其实相当于一个空集合，这将导致任何任务都会被立即执行，因为这种场景下的SynchronousQueue是无法插入任务的。SynchronousQueue是一个非常特殊的队列，在很多情况下可以把它简单理解为一个无法储存元素的队列。从CachedThreadPool的特性来看，这类线程池比较适合执行大量的耗时较少的任务。当整个线程池都处于闲置状态的时候，线程池中的线程都会超时而被停止，这个时候CachedThreadPool之中实际上是没有任何线程的，它几乎不占用任何系统资源的。newCachedThreadPool方法实现如下：
``` java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

#### ScheduledThreadPool
通过Executors的newScheduledThreadPool方法来创建，它的核心线程数量是固定的，而非核心线程数是没有限制的，并且当非核心线程闲置时会被立刻回收。ScheduledThreadPool这类线程池主要用于执行定时任务和具有固定周期的重复任务，newScheduledThreadPool方法的实现如下：

``` java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}

public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
}
```

#### SingleThreadExecutor
通过Executors的newSingleThreadExecutor方法来创建。这类线程池内部只有一个核心线程，它确保所有的任务都在同一个线程中按顺序执行。SingleThreadExecutor的意义在于统一所有的外界任务到一个线程中，这使得在这些任务之间不需要处理线程同步的问题。newSingleThreadExecutor方法实现如下：
``` java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
