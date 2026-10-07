---
title: "OkHttp源码-Dispatcher"
pubDatetime: 2024-08-26T22:51:04+08:00
slug: okhttp-dispatcher
tags: ["Android", "OkHttp"]
description: "Dispatcher 使用 ExecutorService 来执行网络请求。其核心功能包括："
---


> [!NOTE]
> 基于 OkHttp 4.12.0
> 完整代码：[Dispatcher.kt](https://github.com/square/okhttp/blob/parent-4.12.0/okhttp/src/main/kotlin/okhttp3/Dispatcher.kt)

### 一、功能概述

`Dispatcher` 使用 `ExecutorService` 来执行网络请求。其核心功能包括：

1. **请求队列管理:**  维护 `readyAsyncCalls` 队列来存储待执行的异步请求，并根据最大请求数 `maxRequests` 和域名并发限制 `maxRequestsPerHost` 进行调度。
2. **并发控制:** 通过 `maxRequests` 和 `maxRequestsPerHost` 限制最大并发请求数和每个域名最大并发请求数，避免资源过度竞争。
3. **线程池管理:** 使用 `ExecutorService` 来执行异步请求，并可选择自定义线程池。
4. **空闲状态回调:** 提供 `idleCallback` 回调函数，在调度器空闲时触发，方便进行资源清理等操作。

### 二、源码解析

#### 1. 构造函数与属性

`Dispatcher` 提供了两个构造函数：

- `Dispatcher()`: 默认构造函数。

- `Dispatcher(executorService: ExecutorService)`: 使用自定义的线程池。

主要属性包括：

- `maxRequests`: 最大并发请求数，默认为 **64**。超过该限制之后的请求会继续在 `readyAsyncCalls` 中排队。

- `maxRequestsPerHost`: 每个域名最大并发请求数，默认为 **5**。多个域名可能解析到同一IP，所以针对单个IP地址的并发请求可能会超过此限制，Websocket 的连接则不计入该限制，超过该限制之后的请求会继续在 `readyAsyncCalls` 中排队。

- `idleCallback`: 调度器空闲时的回调函数。每个请求 `finish` 后检查 `runningAsyncCalls.size + runningSyncCalls.size == 0` 才会调用
  
- `executorServiceOrNull`: 线程池，默认使用 `SynchronousQueue` 作为队列， **60s** 超时，该队列容量为 0，无法缓存任何东西，一旦有 Call 直接创建线程来处理。

- `readyAsyncCalls`: 待执行的异步请求队列。

- `runningAsyncCalls`: 正在执行的异步请求队列。

- `runningSyncCalls`: 正在执行的同步请求队列。

#### 2. 请求入队与调度

- `enqueue(call: AsyncCall)` 方法负责将异步请求加入待执行队列 `readyAsyncCalls`，并调用 `promoteAndExecute()` 方法尝试调度执行。

  ```kotlin file="Dispatcher.kt"
  internal fun enqueue(call: AsyncCall) {
    synchronized(this) {
      // 加入待执行队列
      readyAsyncCalls.add(call)
      
      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      // 共享同一 Host 并发的计数器
      if (!call.call.forWebSocket) {
        val existingCall = findExistingCallWithHost(call.host)
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall)
      }
    }
    promoteAndExecute()
  }
  ```

- `promoteAndExecute()` 方法是调度请求的核心逻辑，它会将符合条件的请求从 `readyAsyncCalls` 转移到 `runningAsyncCalls`，并在线程池中执行。

  ```kotlin file="Dispatcher.kt"
  private fun promoteAndExecute(): Boolean {
    // ...
    val executableCalls = mutableListOf<AsyncCall>()
    val isRunning: Boolean
    synchronized(this) {
      val i = readyAsyncCalls.iterator()
      while (i.hasNext()) {
        val asyncCall = i.next()
  
        // 检查是否超过最大并发请求数和 Host 最大并发请求数
        if (runningAsyncCalls.size >= this.maxRequests) break 
        if (asyncCall.callsPerHost.get() >= this.maxRequestsPerHost) continue 
  
        i.remove()
        // 增加 域名 并发数
        asyncCall.callsPerHost.incrementAndGet()
        // 将符合条件的请求从 readyAsyncCalls 转移到 runningAsyncCalls
        executableCalls.add(asyncCall)
        runningAsyncCalls.add(asyncCall)
      }
      isRunning = runningCallsCount() > 0
    }
  
    // 在线程池中执行请求
    for (i in 0 until executableCalls.size) {
      val asyncCall = executableCalls[i]
      asyncCall.executeOn(executorService)
    }

    return isRunning
  }
  ```

- `finished(call: AsyncCall)` 方法在异步请求执行完后将 `callsPerHost` 计数器 -1 并且将 `call` 从 `runningAsyncCalls` 队列中移除。

  ```kotlin file="Dispatcher.kt"
  /** Used by [AsyncCall.run] to signal completion. */
  internal fun finished(call: AsyncCall) {
    call.callsPerHost.decrementAndGet()
    finished(runningAsyncCalls, call)
  }
  ```

#### 3. 并发控制

`promoteAndExecute()` 方法通过以下逻辑进行并发控制：

- 检查当前正在执行的异步请求数量是否超过 `maxRequests`，如果超过则不再调度新的请求。
- 检查目标Host当前正在执行的请求数量是否超过 `maxRequestsPerHost`，如果超过则跳过该请求。

#### 4. 同步请求的处理

- `executed(call: RealCall)` 方法负责将同步请求加入队列 `runningSyncCalls`，表示该请求正在执行，该队列中的请求并不会使用 `executorService` 执行，而是由 `RealCall.execute()` 方法的调用线程执行，

  ```kotlin file="Dispatcher.kt"
  /** Used by [Call.execute] to signal it is in-flight. */
  @Synchronized internal fun executed(call: RealCall) {
    runningSyncCalls.add(call)
  }
  ```

- `finished(call: RealCall)` 方法分别用于标记同步请求结束，并维护 `runningSyncCalls` 队列。

  ```kotlin file="Dispatcher.kt"
  /** Used by [Call.execute] to signal completion. */
  internal fun finished(call: RealCall) {
    finished(runningSyncCalls, call)
  }
  ```

#### 5. 空闲状态回调

在 `finished()` 方法中，当所有请求都执行完毕后，会判断是否需要执行 `idleCallback` 回调函数。

```kotlin file="Dispatcher.kt"
/** Used by [Call.execute] to signal it is in-flight. */
private fun <T> finished(calls: Deque<T>, call: T) {
  val idleCallback: Runnable?
  synchronized(this) {
    if (!calls.remove(call)) throw AssertionError("Call wasn't in-flight!")
    idleCallback = this.idleCallback
  }

  val isRunning = promoteAndExecute()

  if (!isRunning && idleCallback != null) {
    idleCallback.run()
  }
}
```
