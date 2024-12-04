---
title: Dubbo源码学习：集群容错
date: 2024-12-04 15:32:52
tags: Dubbo

---



### 前言

在微服务架构中，为了避免单点故障，一个服务通常会部署多个实例，而对于服务消费者而言，则需要考虑如何选择一个可用的实例，以及调用失败的情况如何处理。在 Dubbo 中，针对这些问题，Dubbo 也提供了多种集群容错策略。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本



### 集群容错

Cluster 接口是 Dubbo 对一组服务提供者进行抽象的结果，该接口定义了如何从多个服务提供者中选择一个或多个提供者进行调用，并处理调用过程中可能出现的异常。Dubbo 内置了几种常见的集群容错策略，包括：

- Failover Cluster：失败自动切换
- Failfast Cluster：快速失败
- Failsafe Cluster：失败安全
- Failback Cluster：失败自动恢复
- Forking Cluster：并行调用
- Broadcast Cluster：广播调用



### 源码实现

在每个 Cluster 接口实现中，都会创建对应的 Cluster Invoker 对象，而调用的逻辑实现由 Cluster Invoker 提供，这些 Cluster Invoker 者继承了 AbstractClusterInvoker ：

```Java
@Override
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();

    // binding attachments into invocation.
    Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
    }
	// 获取服务提供者列表
    List<Invoker<T>> invokers = list(invocation);
    // 获取负载均衡器
    LoadBalance loadbalance = initLoadBalance(invokers, invocation);
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    // 调用，由子类实现
    return doInvoke(invocation, invokers, loadbalance);
}
```

我们直接看一下其子类 doInvoke 方法的实现。

#### FailoverClusterInvoker

```Java
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyInvokers = invokers;
    checkInvokers(copyInvokers, invocation);
    String methodName = RpcUtils.getMethodName(invocation);
    //获取重试次数
    int len = calculateInvokeTimes(methodName);
    // retry loop.
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    // 循环调用，失败时则进行重试
    for (int i = 0; i < len; i++) {
        //Reselect before retry to avoid a change of candidate `invokers`.
        //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
        // 重试时重新获取最新的服务提供者列表
        if (i > 0) {
            checkWhetherDestroyed();
            copyInvokers = list(invocation);
            // check again
            checkInvokers(copyInvokers, invocation);
        }
        // 通过负载均衡器选中其中一个服务提供者
        Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
        // 添加到调用过的服务列表
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List) invoked);
        try {
            Result result = invoker.invoke(invocation);
            if (le != null && logger.isWarnEnabled()) {
                //省略...
            }
            return result;
        } catch (RpcException e) {
            if (e.isBiz()) { // biz exception.
                throw e;
            }
            le = e;
        } catch (Throwable e) {
            le = new RpcException(e.getMessage(), e);
        } finally {
            providers.add(invoker.getUrl().getAddress());
        }
    }
    // 重试失败则抛出异常
    throw new RpcException(le.getCode(), "Failed to invoke the method "
            + methodName + " in the service " + getInterface().getName()
            + ". Tried " + len + " times of the providers " + providers
            + " (" + providers.size() + "/" + copyInvokers.size()
            + ") from the registry " + directory.getUrl().getAddress()
            + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
            + Version.getVersion() + ". Last error is: "
            + le.getMessage(), le.getCause() != null ? le.getCause() : le);
}

private int calculateInvokeTimes(String methodName) {
    // 默认重试次数为2次，可以通过配置retries参数调整。
    int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1;
    RpcContext rpcContext = RpcContext.getContext();
    Object retry = rpcContext.getObjectAttachment(RETRIES_KEY);
    if (null != retry && retry instanceof Number) {
        len = ((Number) retry).intValue() + 1;
        rpcContext.removeAttachment(RETRIES_KEY);
    }
    if (len <= 0) {
        len = 1;
    }

    return len;
}
```

- FailoverClusterInvoker 在调用失败时，会自动切换到另一个服务提供者进行重试；

- 默认重试次数为2次，可以通过配置retries参数调整。

适用场景：适用于对请求响应时间要求不高，但对成功率有较高要求的场景。



#### FailfastClusterInvoker

```Java
@Override
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
    try {
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
            throw (RpcException) e;
        }
        throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0,
                "Failfast invoke providers " + invoker.getUrl() + " " + loadbalance.getClass().getSimpleName()
                        + " select from all providers " + invokers + " for service " + getInterface().getName()
                        + " method " + invocation.getMethodName() + " on consumer " + NetUtils.getLocalHost()
                        + " use dubbo version " + Version.getVersion()
                        + ", but no luck to perform the invocation. Last error is: " + e.getMessage(),
                e.getCause() != null ? e.getCause() : e);
    }
}
```

FailfastClusterInvoker 只会发起一次调用，如果调用失败则立即抛出异常，不再重试。

适用场景：适用于对响应时间有一定要求的场景。



#### FailsafeClusterInvoker

```Java
@Override
public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        logger.error("Failsafe ignore exception: " + e.getMessage(), e);
        return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
    }
}
```

FailsafeClusterInvoker 同样也只会发起一次调用，它跟 FailfastClusterInvoker 的区别在于，如果调用失败了，只会记录日志，而不会抛出异常。

适用场景：适用于对调用结果一致性要求不高场景。



#### FailbackClusterInvoker

```Java
@Override
protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    Invoker<T> invoker = null;
    try {
        checkInvokers(invokers, invocation);
        invoker = select(loadbalance, invocation, invokers, null);
        return invoker.invoke(invocation);
    } catch (Throwable e) {
        logger.error("Failback to invoke method " + invocation.getMethodName() + ", wait for retry in background. Ignored exception: "
                + e.getMessage() + ", ", e);
        // 调用失败时，添加到失败队列
        addFailed(loadbalance, invocation, invokers, invoker);
        // 返回空结果
        return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
    }
}

private void addFailed(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, Invoker<T> lastInvoker) {
    if (failTimer == null) {
        synchronized (this) {
            if (failTimer == null) {
                failTimer = new HashedWheelTimer(
                    new NamedThreadFactory("failback-cluster-timer", true),
                    1,
                    TimeUnit.SECONDS, 32, failbackTasks);
            }
        }
    }
    // 创建延迟执行任务
    RetryTimerTask retryTimerTask = new RetryTimerTask(loadbalance, invocation, invokers, lastInvoker, retries, RETRY_FAILED_PERIOD);
    try {
        // 5 秒后执行
        failTimer.newTimeout(retryTimerTask, RETRY_FAILED_PERIOD, TimeUnit.SECONDS);
    } catch (Throwable e) {
        logger.error("Failback background works error,invocation->" + invocation + ", exception: " + e.getMessage());
    }
}
```

FailbackClusterInvoker 在调用失败时，会立即返回空结果，同时将此次调用添加到失败列表，并在 5 秒后进行重试，重试次数可以通过 retries 指定。

适用场景：适用于对调用结果一致性要求比较高的，且对响应时间要求不高的场景。



#### ForkingClusterInvoker

```Java
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    try {
        checkInvokers(invokers, invocation);
        final List<Invoker<T>> selected;
        // 并行调用的服务数，通过forks参数配置，默认为2
        final int forks = getUrl().getParameter(FORKS_KEY, DEFAULT_FORKS);
        // 超时
        final int timeout = getUrl().getParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
        // 并行调用数大于服务提供者数量时，调用所有服务提供者
        if (forks <= 0 || forks >= invokers.size()) {
            selected = invokers;
        } else {
            // 通过负载均衡器选中并行调用的服务提供者
            selected = new ArrayList<>(forks);
            while (selected.size() < forks) {
                Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                if (!selected.contains(invoker)) {
                    //Avoid add the same invoker several times.
                    selected.add(invoker);
                }
            }
        }
        RpcContext.getContext().setInvokers((List) selected);
        final AtomicInteger count = new AtomicInteger();
        final BlockingQueue<Object> ref = new LinkedBlockingQueue<>();
        for (final Invoker<T> invoker : selected) {
            // 通过线程池异步发起调用
            executor.execute(() -> {
                try {
                    Result result = invoker.invoke(invocation);
                    // 调用结果放入结果队列
                    ref.offer(result);
                } catch (Throwable e) {
                    int value = count.incrementAndGet();
                    // 如果全都失败了，将异常放入结果队列
                    if (value >= selected.size()) {
                        ref.offer(e);
                    }
                }
            });
        }
        try {
            // 阻塞等待结果
            Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
            if (ret instanceof Throwable) {
                Throwable e = (Throwable) ret;
                throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0, "Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
            }
            return (Result) ret;
        } catch (InterruptedException e) {
            throw new RpcException("Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e);
        }
    } finally {
        // clear attachments which is binding to current thread.
        RpcContext.getContext().clearAttachments();
    }
}
```

ForkingClusterInvoker 会通过线程池异步同时调用多个服务提供者，返回第一个调用成功的结果，如果全部失败，则返回失败。

适用场景：适用于对响应时间要求比较高，且服务提供者数量比较多的场景。



#### BroadcastClusterInvoker

```Java
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    checkInvokers(invokers, invocation);
    RpcContext.getContext().setInvokers((List) invokers);
    RpcException exception = null;
    Result result = null;
    URL url = getUrl();
    // The value range of broadcast.fail.threshold must be 0～100.
    // 100 means that an exception will be thrown last, and 0 means that as long as an exception occurs, it will be thrown.
    // see https://github.com/apache/dubbo/pull/7174
    // 失败的阈值，如果失败次数达到阈值，将停止继续调用并直接返回失败
    int broadcastFailPercent = url.getParameter(BROADCAST_FAIL_PERCENT_KEY, MAX_BROADCAST_FAIL_PERCENT);

    if (broadcastFailPercent < MIN_BROADCAST_FAIL_PERCENT || broadcastFailPercent > MAX_BROADCAST_FAIL_PERCENT) {
        logger.info(String.format("The value corresponding to the broadcast.fail.percent parameter must be between 0 and 100. " +
                "The current setting is %s, which is reset to 100.", broadcastFailPercent));
        broadcastFailPercent = MAX_BROADCAST_FAIL_PERCENT;
    }

    int failThresholdIndex = invokers.size() * broadcastFailPercent / MAX_BROADCAST_FAIL_PERCENT;
    // 记录调用失败的次数
    int failIndex = 0;
    // 向所有的服务提供者发起调用
    for (Invoker<T> invoker : invokers) {
        try {
            result = invoker.invoke(invocation);
            if (null != result && result.hasException()) {
                Throwable resultException = result.getException();
                if (null != resultException) {
                    exception = getRpcException(result.getException());
                    logger.warn(exception.getMessage(), exception);
                    if (failIndex == failThresholdIndex) {
                        break;
                    } else {
                        failIndex++;
                    }
                }
            }
        } catch (Throwable e) {
            exception = getRpcException(e);
            logger.warn(exception.getMessage(), exception);
            if (failIndex == failThresholdIndex) {
                break;
            } else {
                failIndex++;
            }
        }
    }

    if (exception != null) {
        if (failIndex == failThresholdIndex) {
            logger.debug(
                    String.format("The number of BroadcastCluster call failures has reached the threshold %s", failThresholdIndex));
        } else {
            logger.debug(String.format("The number of BroadcastCluster call failures has not reached the threshold %s, fail size is %s",
                    failIndex));
        }
        throw exception;
    }

    return result;
}
```

BroadcastClusterInvoker 会调用所有的服务提供者，如果有其中一个调用失败，便返回失败。

适用场景：适用于发布/订阅的场景。



### 总结

本文通过源码简单介绍了Dubbo提供的多种集群容错策略及其适用场景，在实际使用中，合理选择和配置这些策略可以有效提高系统的稳定性和可用性。



