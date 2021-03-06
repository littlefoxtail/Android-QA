# WorkManager

WorkManager是什么？官方给的解释是：它对可延期任务操作非常简单，同时稳定性非常强，对于异步任务，即使App退出运行或者设备重启，它都能够很好的保证任务的顺利执行。
对于平常的使用，如果一个后台任务在执行的过程中，app突然退出或者手机断网，这时后台任务将直接终止。

## Types of Background Work

通常来说，可以把所有的后台任务按照任务的紧迫性（是马上需要执行的任务/还是可以缓期执行的任务）和任务的重要性（是确保一定要被执行/还是最好能够执行的任务）来划分。通常来说对于非确保一定要执行的任务，无论时间是否紧迫，我们都可以使用`ThreadPool`来完成这个任务。对于那么比较重要的又时间紧迫的任务，一般会使用`Foreground Service`来完成这个操作。那些希望确保可以被执行但是又可以接受延期执行的任务。这些任务可以使用JobScheduler/JobDispatcher/AlarmManager/BroadcastReceivers来完成。

## JobScheduler

通过JobScheduler来创建一个Job，一旦所设的条件达到，就会执行该Job，但JobScheduler是在API21加入的，同时在API21&22有一个系统Bug
这意味着它只能在API23及以上的版本

```java
if (Build.VERSION.SDK_INT >= 23) {
    //use JobScheduler
}
```

23以下版本

## AlarmManager & BroadcastReceiver

这时对于API23以下，可以使用AlarmManager来进行任务的执行，同时结合BoradcastReceiver来进行任务的条件监听，例如网络的连接状态、设备的启动等。
看到这里是不是开始头大了呢，我们开始的目的只是想做一个稳定性的后台任务，最后发现居然还要进行版本兼容。兼容性与实现性进一步加大。
那么有没有统一的实现方式呢？当然有，它就是WorkManager，它的核心原理使用的就是上面所分析的结合体。
他会结合版本自动使用最佳的实现方式，同时还会提供额外的便利操作，例如状态监听、链式请求等等。

WorkManager的使用，我将其分为以下几步：

1. 构建Work
2. 配置WorkRequest
3. 添加到WorkContinuation中
4. 获取响应结果

## 构建Work

WorkManager每一个任务都是由Work构成，所以Work是任务具体执行的核心所在。

## 配置WorkRequest

1. OneTimeWorkRequest 只是执行一次任务
2. PeriodicWorkRequest 重复执行的任务请求

```java
OneTimeWorkRequest.Builder(BlurWorker.class);
```

```java
OneTimeWorkRequest save = new OneTimeWorkRequest.Builder(SaveImageToFileWorker.class)
.setConstraints(constraints)
.addTag(TAG_OUTPUT)
.build();
```

### OneTimeWorkRequest

作用于一次性任务，即任务只执行一次，一旦执行完就自动结束。

### PeriodicWorkRequest

可以周期性的执行任务

### 添加到WorkContinuation中

已经将WorkRequest配置好了，剩下的是加入到work工作链中进行执行。

```java
// Add WorkRequest to Cleanup temporary images
WorkContinuation continuation = mWorkManager
        .beginUniqueWork(IMAGE_MANIPULATION_WORK_NAME,
                ExistingWorkPolicy.REPLACE,
                OneTimeWorkRequest.from(CleanupWorker.class));

continuation = continuation.then(save);

// Actually start the work
continuation.enqueue();
```

## 取消任务

```java
void cancelWork() {
    mWorkManager.cancelUniqueWork(IMAGE_MANIPULATION_WORK_NAME);
}
```

对于已经正在运行或完成的任务是无法取消任务的。

## 获取结果

```java
private LiveData<List<WorkInfo>> mSavedWorkInfo;


// This transformation makes sure that whenever the current work Id changes the WorkInfo
        // the UI is listening to changes
mSavedWorkInfo = mWorkManager.getWorkInfosByTagLiveData(TAG_OUTPUT);
```

## 任务约束Constraints WorkManager

允许我们指定任务执行的环境，比如网络已经连接、电量充足时等，在满足条件的情况下任务才会执行。

1. 使用Constraints.Builder()创建并配置Constraints对象，可以指定上述任务运行时间时的约束
2. 创建Worker时调用setConstraints指定约束条件


[源码实现](workmanager_source.md)