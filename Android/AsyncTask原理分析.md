# AsyncTask原理分析

### 为什么要用AsyncTask？

1. 主线程不能执行耗时操作，当Activity中某操作超过5s或在BroadCast Recevier和Service中超过10s就会ANR异常(Application Not Response)
2. 只有UI线程才能更新UI，在ViewRootImpl中的checkThread()方法中就指明了。

```java
    void checkThread() {
        if (mThread != Thread.currentThread()) {
            throw new CalledFromWrongThreadException(
                    "Only the original thread that created a view hierarchy can touch its views.");
        }
    }
```

### AsyncTask提供的方法

AsyncTask内部维护了一个Handler和一个线程池，在线程池中处理耗时任务，任务完成后通过Handler发送回主线程更新UI。它为我们提供以下几个方法：

```java
//这是抽象方法，必须实现，只有它运行在后台，也就是非主线程，执行后台任务
String doInBackground(Params... params)  
    
//以下方法都执行在主线程，后台线程与主线程的信息交互是由Handler实现的
void onPreExecute() //在任务执行之前，也就是doInBackground之前执行，用于一些准备工作
void onProgressUpdate(Progress... values) //表示任务进度更新
void onPostExecute(Result result) //任务完成后执行，参数即为后台任务的返回结果
void onCancelled() //任务被取消后调用
```

### AsyncTask的局限性

1. 在Android4.1之前AsyncTask必须在主线程加载，类加载相关知识可以参考我的文章[java虚拟类加载机制]([https://www.theaze.cn/2019/04/29/java%E8%99%9A%E6%8B%9F%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/](https://www.theaze.cn/2019/04/29/java虚拟类加载机制/))。4,1之后的版本在ActivityThread中的main方法里会自动加载AsyncTask
2. AsyncTask对象必须在主线程中创建
3. AsyncTask对象的execute方法只能在主线程调用且只能调用一次
4. 非静态的AsyncTasK内部类会可能会导致内存泄漏

### AsyncTask原理分析

AsyncTask是从调用execute方法开始的，让我们看看execute做了什么：

```java
    public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
```

execute其实是调用了executeOnExecutor方法，并将sDefaultExecutor作为executeOnExecutor的第一个默认参数，查看源码我们发现它其实是静态内部类SerialExecutor的对象，那么我们再来看看类SerialExecutor是什么：

```java
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

分析源码我们发现这其实是一个任务缓冲队列，真正执行任务的线程池是THREAD_POOL_EXECUTOR，当线程池中没有正在执行中的任务时，下一个任务才会开始，所以AsyncTask是串行执行任务的。

我们再分析THREAD_POOL_EXECUTOR,线程池的相关知识请参考我的[Java线程池]([https://www.theaze.cn/2019/05/21/java%E7%BA%BF%E7%A8%8B%E6%B1%A0/](https://www.theaze.cn/2019/05/21/java线程池/))

```java
     private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
     private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
     private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
     private static final int KEEP_ALIVE_SECONDS = 30;
     private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);
	 public static final Executor THREAD_POOL_EXECUTOR;
     static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

分析可以看出这是一个**核心线程池**大小为最大为4、最小为2；**最大线程池**大小为CPU的两倍+1，**线程存活时间**为30s，**排队策略**是无界阻塞队列的线程池。

**为什么AsyncTask必须在主线程加载？**

我们注意到线程池所有的参数和对象都是静态的，所以需要保证类的加载必须在主线程，否则，THREAD_POOL_EXECUTOR.execute(mActive)这一语句就不是在主线程运行的了。

**AsyncTask对象必须在主线程中创建且execute方法只能在主线程调用**

因为Handler要把任务消息返回到调用它的线程来修改UI，如果不是主线程调用它，那么就会抛出不在UI线程才能更新UI的异常。

**那么为什么execute只能调用一次，但还要用线程池呢？**

execute方法只能调用一次只限于同一对象，AsyncTask的不同实例，也就是不同的对象都可以调用execute方法，而且前面提到所有的参数和对象都是静态的，那么同一进程中所有AsyncTask对象调用的后台线程都会在同一线程池中运行。

我们再看一下executeOnExecutor方法的源码

```java
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

在executeOnExecutor执行前会判断当前的状态，如果是已经开始或已经结束时会抛出异常。如果是首次调用那么就把Status设置为RUNNING，FINISHED状态是在结束时设置的

```java
    private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result); //任务取消后执行
        } else {
            onPostExecute(result); //任务完成后执行，参数即为后台任务的返回结果
        }
        mStatus = Status.FINISHED;
    }
```

这就是为什么execute只能调用一次。

**为什么非静态的AsyncTask内部类可能会导致内存泄漏？**

因为非静态内部类会隐性持有一个外部类的引用，Activity占用的会无法正常释放，而导致内存泄漏，具体可以参考我的[常见android内存泄漏原理及解决方法]([https://www.theaze.cn/2019/05/24/%E5%B8%B8%E8%A7%81android%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E5%8E%9F%E7%90%86%E5%8F%8A%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95/](https://www.theaze.cn/2019/05/24/常见android内存泄漏原理及解决方法/))。

