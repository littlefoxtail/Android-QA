# OkHttp

OkHttp的内部实现通过一个责任链模式完成，将网络请求的各个阶段封装到各个链条中，实现各层解耦。


```kt
//RealCall
override fun execute(): Response {
    try {
        // 第1步
        client.dispatcher.executed(this)
        // 第2步
        return geResponseWithInterceptorChain()
    } finally {
        // 第3步
        client.dispatcher.finished(this)
    }
}
```

## 第一步Dispatcher

功能：

- 记录同步任务、异步任务及等待执行的异步任务。
- 线程池管理异步任务。
- 发起/取消网络请求API：execute、enqueue、cancel。

```kt
//RealCall
//execute的另一个重载
override fun execute(): Reponse {
    synchronized(this) {
      check(!executed) { "Already Executed" }
      executed = true
    }
    callStart()
    client.dispatcher.enqueue(AsyncCall(responseCallback))
}
```

```kt
//Dispatcher
private fun promoteAndExecute(): Boolean {
    synchronized(this) {
        val i = readyAsyncCalls.iterator()
        while (i.hasNext()) {
            val asyncCall = i.next()
            //阈值校验
            if (runningAsyncCalls.size >= this.maxRequests) break // Max capacity.
            if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue // Host max capacity.
            i.remove()
            asyncCall.callsPerHost.incrementAndGet()
            executableCalls.add(asyncCall)
            //移入runningAsynCalls列表
            runningAsyncCalls.add(asyncCall)
        }
        isRunning = runningCallsCount() > 0
    }

    for (i in 0 unitl executableCalls.size) {
        val asyncCall = executableCalls[i]
        asyncCall.executeOn(executorService)
    }
    return isRunning
}

```

## 第二步

```kt
inter
```