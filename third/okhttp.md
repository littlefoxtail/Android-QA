# OkHttp
- 支持HTTP/2、允许链接到同一主机的请求公用一个Socket
- 如果HTTP/2不可用，通过连接池减少请求的延迟
- 通过GZIP压缩减少传输数据的大小
- 通过缓存避免了网络重复请求

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
# 责任链和拦截器
真正核心的是它的拦截器。它不仅负责OkHttp的核心功能。而且还提供了一些用户自定义的功能。
interceptor是一个接口，只定义了一个方法intercept(Chain chain)和一个内部接口。
```kotlin
  @Throws(IOException::class)
  internal fun getResponseWithInterceptorChain(): Response {
    // Build a full stack of interceptors.
    val interceptors = mutableListOf<Interceptor>()
    interceptors += client.interceptors
    interceptors += RetryAndFollowUpInterceptor(client)
    interceptors += BridgeInterceptor(client.cookieJar)
    interceptors += CacheInterceptor(client.cache)
    interceptors += ConnectInterceptor
    if (!forWebSocket) {
      interceptors += client.networkInterceptors
    }
    interceptors += CallServerInterceptor(forWebSocket)

    val chain = RealInterceptorChain(
      call = this,
      interceptors = interceptors,
      index = 0,
      exchange = null,
      request = originalRequest,
      connectTimeoutMillis = client.connectTimeoutMillis,
      readTimeoutMillis = client.readTimeoutMillis,
      writeTimeoutMillis = client.writeTimeoutMillis
    )

    var calledNoMoreExchanges = false
    try {
      val response = chain.proceed(originalRequest)
      if (isCanceled()) {
        response.closeQuietly()
        throw IOException("Canceled")
      }
      return response
    } catch (e: IOException) {
      calledNoMoreExchanges = true
      throw noMoreExchanges(e) as Throwable
    } finally {
      if (!calledNoMoreExchanges) {
        noMoreExchanges(null)
      }
    }
  }

```
将所有的interceptor作为一个集合，并创建一个RealInterceptorChain对象，然后执行它的proceed方法。
```kotlin
@Throws(IOException::class)  
override fun proceed(request: Request): Response {  
  check(index < interceptors.size)  
  
  calls++  
  
  if (exchange != null) {  
    check(exchange.finder.routePlanner.sameHostAndPort(request.url)) {  
      "network interceptor ${interceptors[index - 1]} must retain the same host and port"    }  
    check(calls == 1) {  
      "network interceptor ${interceptors[index - 1]} must call proceed() exactly once"    }  
  }  
  
  // Call the next interceptor in the chain.  
  val next = copy(index = index + 1, request = request)  
  val interceptor = interceptors[index]  
  
  @Suppress("USELESS_ELVIS")  
  val response = interceptor.intercept(next) ?: throw NullPointerException(  
      "interceptor $interceptor returned null")  
  
  if (exchange != null) {  
    check(index + 1 >= interceptors.size || next.calls == 1) {  
      "network interceptor $interceptor must call proceed() exactly once"    }  
  }  
  
  return response  
}
```