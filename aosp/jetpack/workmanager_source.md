# 源码分析

## 初始化

初始化执行器：

```java
//Schedulers
/**
 * 主要是为了创建调度者集合
 * 如果版本号>=23，返回SystemJobScheduler，内部主要使用JobScheduler完成调度
 * 如果手机支持GCM，则返回GcmScheduler调度者，国内基本告别了
 * 其他情况下返回SystemAlarmScheduler，内部使用AlarmManager实现原理
 */
static Scheduler createBestAvailableBackgroundScheduler(..) {
    Scheduler scheduler;
    boolean enableFirebaseJobService = false;
    boolean enableSystemAlarmService = false;

    if (Build.VERSION.SDK_INT >= 23) {
        scheduler = new SystemJobScheduler(context, workManager);
        setComponentEnable(..);
    } else {
        try {
            scheduler = tryCreateFirebaseJobScheduler(context);
            enableFirebaseJobService = true;
        } catch (Exception e) {
            schduler = new SystemAlarmScheduler(context);
            enableSystemAlarmService = true;
        }
    }
    return scheduler;
}
```

## 执行阶段

```java
//WorkContinuationImpl
public void enqueue() {
    if (!mEnqueued) {
        EnqueueRunnable runnable = new EnqueueRunnable(this);
        mWorkManagerImpl.getTaskExecutor()
                .executeOnBackgroundThread(runnable));
        mOperation = runnable.getOperation();
    }
    return mOperation;
}
```

## 调度执行

```java
//EnqueueRunnable
public void run() {
    boolean needsScheduling = addToDatabase();
    if (needsScheduling) {
        scheduleWorkInBackground();
    }
}

public void scheduleWorkInBackgournd() {
    WorkManagerImpl  workManager = mWorkContinuation.getWorkManagerImpl();
    Schedulers.schedule(
        workManager.getConfiguration(),
        workManager.getWorkDatabase(),
        workManager.getSchedulers());
}
```

判断时候有约束条件。没用则调用`startWork`执行任务。有则将任务入集合

```java
// GreedyScheduler
public synchronized void schedule(WorkSpec ... workSpecs) {
    for (WorkSpec workSpec: workSpecs) {
        if (workSpec.state = WorkInfo.State.ENQUEUED
                && !workSpec.isPeriodic()
                && workSpec.initialDelay == 0L
                && !workSpec.isBackedOff()) {
            if (workSpec.hasConstraints()) {
                if (Build.VERSION.SDK_INT < 24
                    || !workSpec.constraints.hasContentUriTriggers()) {
                    constrainedWorkSpec.add(workSpec);
                    constriainedWorkSpecIds.add(workSpec.id);
                }
            } else {
                mWorkManagerImpl.startWork(workSpec.id);
            }
        }
    }

    synchronized (mLock) {
        if (!constrainedWorkSpecs.isEmpty()) {
             mConstrainedWorkSpecs.addAll(constrainedWorkSpecs);
            mWorkConstraintsTracker.replace(mConstrainedWorkSpecs);
        }
    }
}
```

WorkerWrapper：

1. 反射机制获取到ListenableWorker对象。其中Worker类继承自ListenableWorker类
2. 调用ListenableWorker.startWork，实际上调用Worker类的startWork方法
3. 在Worker类的startWork方法中又会调用doWork方法，

```java
// WorkerWrapper
private void runWorker() {
    if (mWorker == null) {
        mWorker = mConfigutation.getWorkerFactory().createWorkerWithDefaultFallback(mAppContext, mWorkSpec.workerClassName, params);


        mWorkTaskExecutor.getMainThreadExecutor()
                .execute(new Runnable() {
                    public void run() {
                        mInnerFuture = mWorker.startWork();
                        future.setFuture(mInnerFuture);
                    }
                })
    }
}

// WorkerFactory
public final ListenableWorker createWorkerWithDefaultFallback(..) {
    worker = createWorker(appContext, workerClassName, workerParamsters);
    if (worker != null) {
        return worker;
    }

    Class<? extends ListenableWorker> clazz;
    try {
        clazz = Class.forName(workerClassName).asSubclass(ListenableWorker.class)
    } catch (ClassNotFoundException e) {
        return null;
    }

    try {
        Constructor<? extends ListenableWorker> constructor =
                clazz.getDeclaredConstructor(Context.class, WorkerParameters.class);
        worker = constructor.newInstance(
                appContext,
                workerParameters);
        return worker;
    } catch (Exception e) {
        Logger.get().error(TAG, "Could not instantiate " + workerClassName, e);
    }
    return null;

}
```

## 总结

1. Worker：指定我们需要执行的任务。WorkManager API包含一个抽象的Worker类WorkManagerImpl，需要继承这个类并且在这里执行工作。
2. WorkRequest：代表一个单独的任务。一个WorkRequest对象指定哪个Work类应该执行该任务，而且，还可以向WorkRequest对象添加详细信息，指定任务运行的环境等。每个WorkRequest都有一个自动生成的唯一ID，可以使用该ID来执行诸如取消排队任务或者获取任务状态等内容。WorkRequest是一个抽象，在代码中，需要使用它的直接子类，OneTimeWorkRequest或PeriodicWorkRequest。
3. WorkRequest.Builder：用于创建WorkRequest对象的辅助。
4. Constraints：指定任务在何时运行（例如，“仅在连接到网络时”）。我们可以通过Constraints.Builder 来创建Constraints对象，并在创建WorkRequest之前，将 Constraints 对象传递给 WorkRequest.Builder。
5. Constraints：指定任务在何时运行（例如，“仅在连接到网络时”）。我们可以通过Constraints.Builder 来创建Constraints对象，并在创建WorkRequest之前，将 Constraints 对象传递给 WorkRequest.Builder。
6. WorkStatus：包含有关特定任务的信息。WorkManager 为每个 WorkRequest 对象提供一个LiveData，LiveData持有一个WorkStatus对象，通过观察LiveData，我们可以确定任务的当前状态，并在任务完成后获取返回的任何值。

