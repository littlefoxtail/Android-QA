# TaskManager

```java
// 启动阶段的线程池管理工具
//TaskManager
public void enqueue(Task task) {
    TaskRequest request = new TaskRequest.Builder(task)
        build();
    enqueue(request);
}

public void enqueue(TaskRequest baseTask) {
    schedulerManager.schedule(baseTask);
}

```

## 调度器

```java
//SchedulerManager
public void schedule(TaskRequest taskRequest) {
    List<TaskRequest> mPermittedTaskRequests = new ArrayList<>();
    handleTaskRequest(taskRequest, mPermittedTaskRequests);
    if (!mPermittedTaskRequests.isEmpty()) {
        mTaskPermitsTracker.updatePermitTaskList(mPermittedTaskRequests, true);
    }
}

//SchedulerManager
private void handleTaskRequest(final TaskRequest taskRequest, final List<TaskRequest> mPermittedTaskRequests) {
    if (taskRquest.getState() == State.ENQUEUED) {
        //是加入队列任务的话， 默认就是它
        int delayTime = taskRequest.getDelayTime();
        if (delayTime == 0 || delayTime == Integer.MAX_VALUE) {
            taskRequest.setDelayTime(0);//[避免依赖状态改变后，再重新进入等待队列的bug]
            doTask(taskRequest, mPermittedTaskRequests, delaytTime == Integer.MAX_VALUE);
        } else {
            // 只有对一次post 执行的时候，会调用执行这里;
            taskRequest.setDelayTime(0);//[避免依赖状态改变后，再重新进入等待队列的bug]
            taskContainer.add(taskRequest);

            if (DebugLog.isDebug() && taskRequest.getRequestId() == 0 && TaskManager.enableDebugCheckCrash) {
                throw new IllegalStateException("task request id cant be 0" + taskRequest.getTask().getName());
            }
            workHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    if (taskContainer.remove(taskRequest)) { //[otherwise been taken by others]
                        doTask(taskRequest, mPermittedTaskRequests, false);
                    }
                }
            }, delaytTime);
        }
    }
}

//SchedulerManager
private void doTask(TaskRequest taskRequest, List<TaskRequest> mPermittedTaskRequests, boolean isPending) {
    if (taskRequest.hasPermits()) {//添加进入权限等待队列
        taskContainer.add(taskRequest);
    } else if (taskRequest.hasDependantTasks()) {
        taskContainer.add(taskRequest);
        Task mTask = task.getTask();
        int ids[] = taskRequest.getDependantTaskIds();
        if (ids != null && mTask != null) {
            for (int taskId : ids) {
                if (DebugLog.isDebug()) {
                //这里还利用了一段debug代码，来检查任务的循环依赖问题
                }
                taskRecords.addSuccessorForTask(mTask, taskID);

            }
        } else {
            DebugLog.e(TAG, "there might have bugs :  has dependantTasks , but has no ids");
            taskRequest.setExecuter(executor);
        }

    } else if (isPending) {
        taskContainer.add(taskRequest);
    } else {
        // task 或得到exceuter资源
        TaskRecorder.enqueue(taskRequest);
        taskRequest.setExecuter(executor);
    }
}

```

```java
// TaskRequest
private void setExecuter(TaskExecutor executer) {
    int size = getTaskSize();
    this.executor = executor;
    //将TaskRequest放入队列中
    executer.enqueue(this)
    if (mTasks != null && size > 0) {
        if (getTaskSize() == 1) {
            TaskWrapper wrapper = new TaskWrapper(this, 0);
            Task task = getTask();
            task.setWrapper(wrapper);
            if (!isRunningOnMainThread()) {
                executer.executeOnBackgroundThread(wrapper, task.getThreadPriority());
            }
        } else {
            for (int i = size - 1; i >= 0; i--) {
                Task task2 = this.mTask[i];
                if (task2 != null) {
                    TaskWrapper wrapper2 = new TaskWrapper(this, i);
                    task2.setWrapper(wrapper2);
                    if (i == 0) {
                        this.syncTaskWrapper = wrapper2;
                        wrapper2.run()；
                    } else {
                        executer.executeOnBackgroundThread(wrapper2, ThreadPriority.FLEXABLE)
                    }
                }
            }

        }
    }

}
```


```java
//TaskManagerExecutor
public void executeOnBackgroundThread(Runnable runnable, int priority) {
    handleNormalPriority(runnable);
    handleHighPriority(runnable);
    handleLowPriority(runnable);
}
```

Task包装类的执行代码：

```java
//TaskWrapper
public void run() {
    if (this.isCanceled) {
        return;
    }
    do {
        int taskId;
        do {
            taskId = this.taskIndex;
            Task task = this.mTaskRequest.getTaskAt(this.taskIndex);

            int stateCheck = mTaskRequest.onTaskStateChange(taskIndex, Task.STATE_RUNNING);

            if (stateCheck < 0 || (taskRun && stateCheck == Task.STATE_TAKEN)) {
                taskRun = false;
                task.setWrapper(this);
                task.doBeforeTask();
                task.doTask();
                task.doAfterTask();
                mTaskRequest.onTaskStateChange(taskId, Task.STATE_FINISHED);
            } else {
                mTaskRequest.requestNextIdle(this);
            }
        } while(taskId != taskIndex && !isCanceled);
    } while((mTaskRequest = fetchPendingRequest()) != null)

}
```

```java
//TaskRequest
public int onTaskStateChange(int index, int newState) {
    setState(State.RUNNING);
}
```
