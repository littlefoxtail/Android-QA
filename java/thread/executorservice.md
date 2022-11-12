# ExecutorService

![线程池时序图](/img/线程池时序图.png)

## shutdown

shutdown和awaitTermination为接口ExecutorService定义的两个方法，一般情况配合使用来关闭线程池

1. shutdown方法：平滑的关闭ExecutorService，当此方法被调用时停止接收新的任务并且等待已经提交的任务（包含提交正在执行和提交未执行）执行完成。当所有提交任务执行完毕，线程池即被关闭。当线程池调用该方法时，线程池的状态立刻变成SHUTDOWN。此时不能再往线程池中添加任何任务，否则将会抛出RejectedExecutionException。但是，线程池不会立刻退出，直到添加到线程池中的任务都已经处理完成，才会退出。

    - 将线程池设置为`SHUTDOWN`，并不会立即停止：
        - 停止接收外部submit的任务
        - 内部正在跑的的任务和队列里等待的任务，会执行完
        - 等到第二步完成后，才真正停止
2. shutdownNow()
    线程池的状态立刻变成STOP状态，并试图停止所有正在执行的线程，不再处理还在池队列中等待的任务，当然，它会返回那些未执行的任务。
    它试图终止线程的方法是通过调用Thread.interrupt()方法来实现的，但这种方法作用有限，如果线程中没有sleep、wait、Condition、定时锁等应用，interrupt方法无法终端当前线程。所以，shutdownNow()并不代表线程池就一定立即退出，它可能必须要等待素有正在执行的任务都执行完成了才能退出。
    - 将线程池状态置为`STOP`。企图立即停止，事实上不一定：
        - 跟`shutdown()`一样，先停止接收外部提交的任务
        - 忽略队列里等待的任务
        - 尝试将正在跑的任务`interrupt`中断
        - 返回未执行的任务列表

    <!-- * shutDownNow方法会返回未完成的任务队列中的任务列表
    * advanceRunState方法中传入的是STOP，而不是SHUTDOWN。 -->

3. awaitTermination:一个是timeout即超时时间，另一个是unit即时间单位。这个方法会使线程等待timeout时长，当超过timeout时间后，会监测ExecutorService是否已经关闭。该方法不会更改线程池状态，一般与shutdown配合
    - 当前线程阻塞，直到
        - 等所有已提交的任务(包括正在跑和队列中等待的)执行完
        - 或者等超时时间到
        - 或者线程被中断，抛出`InterruptedException`

然后返回true（shutdown请求后所有任务执行完毕）或false（已超时)，实验发现，shuntdown()和awaitTermination()效果差不多，方法执行之后，都要等到提交的任务全部执行完才停。

```java
public void shutdown() {
    final ReentrantLock mainlock = this.mainlock;
    mainlock.lock();
    try {
        //Check if caller can modify out threads
        checkShutdownAccess();
        // 设置线程池状态为SHUTDOWN状态
        advanceRunState(SHUTDOWN);
// 一个一个中断线程
        interruptIdleWorkers();
    } finally {
        mainlock.unlock();
    }
    tryTerminate(); //里面是一个看起来像死循环的东西，但是有return
}
```

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt(); //代码2
                } catch (SecurityException ignore) {

                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

代码2：针对在线程池关闭前，有部分工作线程就一直在等待要处理的任务，也就是说工作线程空闲，调用interrupt()进行中断

getTask

```java

try {
    Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
        if (r != null)
            return r;
        timeOut = true;
} catch (InterruptedException retry) {
    timeOut = false;
}
```

## shutdown和shutdownNow的区别:

从字面意思就能理解，shutdownNow()能立即停止线程池，正在跑的和正在等待的任务都停下了。这样做立即生效，但是风险也比较大；
shutdown()只是关闭了提交通道，用submit()是无效的；而内部该怎么跑还是怎么跑，跑完再停。

## shutdown和awaitTermination()

shutdown()后，不能再提交新的任务进去；但是awaitTermination()后，可以继续提交
awaitTermination()是阻塞的，返回结果是线程池是否已停止(true/false)；shutdown不阻塞

# 清理工作
主要做三件事：
1. 维护Worker线程结束后的线程池状态，如移除Worker集合
2. 检测线程池是否满足TIDYING状态，满足则调整状态触发terminated
3. 线程池状态为RUNNING或SHUTDOWN
	1. 任务执行异常引起的worker线程死亡
	2. 线程数量为0，且任务队列不为空
	3. 若不允许核心线程超时回收，线程数量少于核心线程时

processWorkerExit 方法是为将要终结的Worker做一次清理和数据记录工作。
```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
//因为抛出用户异常导致线程终结，直接使工作线程数-1
	if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
		decrementWorkerCount();

	final ReentrantLock mainLock = this.mainLock;
	mainLock.lock();
	try {
	//全局的已完成任务记录加上此将要终结的Worker中的已完成任务数
		completedTaskCount += w.completedTasks;
		//工作线程集合中移除此将要终结的Worker
		workers.remove(w);
	} finally {
		mainLock.unlock();
	}
//尝试进入tidying状态
	tryTerminate();

	int c = ctl.get();
	//若线程池为Running或shutdown
	if (runStateLessThan(c, STOP)) {
	//若由于任务执行异常引起则直接跳过，创建新的Worker代替
		if (!completedAbruptly) {
		//若允许核心线程超时回收，则最低线程数量为0，否则为核心线程数
			int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
			
			if (min == 0 && ! workQueue.isEmpty())
				min = 1;
			if (workerCountOf(c) >= min)
				return; // replacement not needed
		}
		addWorker(null, false);
	}
}

```

## 总结

- 优雅的关闭，用shutdown()
- 想立马关闭，并得到未执行任务列表，用shutdownNow()
- 优雅的关闭，并允许关闭声明后新任务能提交，用awaitTermination() (得配合shutdown)

```text
关闭功能 【从强到弱】 依次是：shuntdownNow() > shutdown() > awaitTermination()
```
