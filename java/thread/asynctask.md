# AsyncTask

## 结论解读

1. API16以前，必须在主线程加载AsyncTask， API16以后就不用了

    ```java
    private static class InternalHandler extends Handler {
        public InternalHandler() {
            super(Looper.getMainLooper());
        }
    }
    ```

    之前的AsyncTask加载时，是得到当前加载线程的Handler，而最新代码中，总是得到UI线程的Looper来创建和UI交code互的Handler。

2. 多次调用一个AsyncTask对象会出现异常。如果要处理多个后台任务，需要创建多个AsyncTask并执行execute()

    ```java
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:")
            }
        }

    }
    ```

3. API 4-11默认是AsyncTask任务并发执行，API11后默认是顺序执行，必须等一个任务结束才能执行下一个。但是可以通过executeOnExecutor(AsyncTask.THREAD\_POOL\_EXECUTOR)来进行并行执行任务，在并行执行任务时，有最大执行个数的限制

    ```java
    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS, sPoolWorkQueuee, sThreadFactory)
        THREAD_POOL_EXEUCTOR = threadPoolExecutor;
    }
    ```

    THREAD_POOL_EXECUTOR是默认的并行执行任务的线程池，BlockingQueue队列的长度是128.8核手机为例，其核心线程数是9个，最大线程是17，所能最大加入的任务是128+17=145，如果超出这个任务数，就会报出RejectExecutionException异常。

4. AsyncTask需要在UI线程调用execute()函数
    调用execute()函数时调用了onPreExecute()，而onPreExecute()需要在UI线程中执行，所以AsyncTask需要在UI线程调用execute()函数。

## AsyncTask执行原理

```java
mWorker = new WorkerRunnable<Params, Result>() {
    public Result call() throws Exception {
        Result result = doInBackground(mParams);
        Binder.flushPendingCommands();
        postResult(result);
        return result;
    }
}

mFuture = new FutureTask<Result>(mWorker) {
    protected void done() {
        postResultIfNotINvoked(get());
    }
}
```

## AsynTask的黑暗面

### 使用不当带来的问题

1. 生命周期和内存泄露
    “当Activity结束或者退出应用时AsyncTask会一直执行doInBackground()方法直到方法执行结束，这可能会导致在onPostExecute时view不存在而导致崩溃溃，以及可能的内存泄露”。

    如果退出Activity时候AsyncTask仍在执行，以上所述的确会发生，这些问题需要由使用者来解决而不是AsyncTask来解决。
    应该在相应的生命周期如onDestory调用cancel取消任务。
2. cancel不能正常取消的问题
    调用cancel终止AsyncTask原理是对执行异步任务的线程调用interrupt函数。每个线程内部都有一个boolean型变量表示线程的终端状态，true代表线程处于中断状态，false表示未处于中断状态。
    Thread.interrupt()只在Object.wait(),Object.join(),Object.sleep()几个方法会主动抛出InterruptedException异常，从而结束阻塞状态。而在其他的时候，只是通过设置了Thread的一个标志位信息，需要程序自我进行处理

    如果AsyncTask后台任务从未做中断的处理肯定会一直执行这个线程。所以需要自己在doInbackground里进行中断处理。这个缺陷也应该是Thread类的缺陷，因为要用到线程狐狸异步任务。
3. Activity意外重启，状态消失问题
    当用户旋转屏幕的时候，Activity就会重新启动，如果之前有AsynTask正在异步加载处理数据，那么之前的数据就会消失。用其他方式进行请求同样会发生这个问题。

### 实际存在的问题

1. 并行串行问题
    源码文档描述：
    “AsyncTasks should ideally be userd for short operations (a few seconds at the most)”
    尽量执行一个短时间的任务。
    串行执行共用的AsyncTask是一个线程池。因为是顺序执行，导致你调用execute()可能没法立刻执行，也可能就执行不了，如果执行一个很耗时的任务，那么同一进程内的AsyncTask在这之后调用execute的都将无法执行。

    执行一个任务通过调用executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR)通过AsyncTask的THREAD_POOL_EXECUTOR线程池或者传入其他的线程池来立刻执行任务，但THREAD_POOL_EXECUTOR有最大并发数的限制

2. 错误处理问题
    AsyncTask没有对发生的一些异常进行处理，你只能在doInBackground里进行一些判断，但之外的一些异常情况你都无法了解

3. 多个任务的管理问题
    如果需要多个后台任务，需要新建多个AsyncTask来执行任务，在需要退出的时候你需要对每一个都进行一定的处理来避免内存泄露以及UI问题。
