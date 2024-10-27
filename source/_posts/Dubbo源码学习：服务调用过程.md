---
title: Dubbo源码学习：服务调用过程
date: 2024-10-26 12:00:00
tags: Dubbo

---

在之前的文章中，我们了解到了 Dubbo 服务导出和服务引用的过程。本篇文章一起来看下服务远程调用的过程。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本



### 服务消费端

在服务引用一篇中，我们了解到服务引用对象是通过动态代理对象生成的，而远程调用也通过该代理对象发起：

```Java
public class proxy0
implements ClassGenerator.DC,
EchoService,
Destroyable,
HelloService {
    public static Method[] methods;
    private InvocationHandler handler;

    @Override
    public Object $echo(Object object) {
        Object[] objectArray = new Object[]{object};
        Object object2 = this.handler.invoke(this, methods[0], objectArray);
        return object2;
    }

    public String sayHello() {
        Object[] objectArray = new Object[]{};
        Object object = this.handler.invoke(this, methods[1], objectArray);
        return (String)object;
    }

    @Override
    public void $destroy() {
        Object[] objectArray = new Object[]{};
        Object object = this.handler.invoke(this, methods[2], objectArray);
    }

    public proxy0() {
    }

    public proxy0(InvocationHandler invocationHandler) {
        this.handler = invocationHandler;
    }
}
```

该代理类实现了服务提供接口的方法，并将请求都转发到了 **InvocationHandler#invoke** 方法：

```Java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    //Object方法直接本地调用
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(invoker, args);
    }
    String methodName = method.getName();
    Class<?>[] parameterTypes = method.getParameterTypes();
    //toString、hashCode、equals方法直接本地调用
    if (parameterTypes.length == 0) {
        if ("toString".equals(methodName)) {
            return invoker.toString();
            //$destroy方法销毁当前invoker
        } else if ("$destroy".equals(methodName)) {
            invoker.destroy();
            return null;
        } else if ("hashCode".equals(methodName)) {
            return invoker.hashCode();
        }
    } else if (parameterTypes.length == 1 && "equals".equals(methodName)) {
        return invoker.equals(args[0]);
    }
    //将方法及参数封装为RpcInvocation
    RpcInvocation rpcInvocation = new RpcInvocation(method, invoker.getInterface().getName(), protocolServiceKey, args);
    String serviceKey = invoker.getUrl().getServiceKey();
    rpcInvocation.setTargetServiceUniqueName(serviceKey);

    // invoker.getUrl() returns consumer url.
    RpcContext.setRpcContext(invoker.getUrl());

    if (consumerModel != null) {
        rpcInvocation.put(Constants.CONSUMER_MODEL, consumerModel);
        rpcInvocation.put(Constants.METHOD_MODEL, consumerModel.getMethodModel(method));
    }
	//调用invoker的invoker此处invoker类型为MigrationInvoker
    return invoker.invoke(rpcInvocation).recreate();
}
```

这里针对一些不需要远程调用的一些方法，使用本地调用。继续来到 **MigrationInvoker#invoke** ：

```Java
@Override
public Result invoke(Invocation invocation) throws RpcException {
	//检查应用级服务invoker是否可用，不可用时使用接口级服务发现invoker
    if (!checkInvokerAvailable(serviceDiscoveryInvoker)) {
        if (logger.isDebugEnabled()) {
            logger.debug("Using interface addresses to handle invocation, interface " + type.getName() + ", total address size " + (invoker.getDirectory().getAllInvokers() == null ? "is null" : invoker.getDirectory().getAllInvokers().size()));
        }
        //invoker类型为MockClusterInvoker
        return invoker.invoke(invocation);
    }
	
    if (!checkInvokerAvailable(invoker)) {
        if (logger.isDebugEnabled()) {
            logger.debug("Using instance addresses to handle invocation, interface " + type.getName() + ", total address size " + (serviceDiscoveryInvoker.getDirectory().getAllInvokers() == null ? " is null " : serviceDiscoveryInvoker.getDirectory().getAllInvokers().size()));
        }
        return serviceDiscoveryInvoker.invoke(invocation);
    }

    return currentAvailableInvoker.invoke(invocation);
}
```

这里将优先使用应用级服务发现invoker，应用级服务发现invoker不可用时则使用接口级服务发现invoker。来到 **MockClusterInvoker#invoke** ：

```Java
@Override
public Result invoke(Invocation invocation) throws RpcException {
    Result result = null;

    //获取mock参数
    String value = getUrl().getMethodParameter(invocation.getMethodName(), MOCK_KEY, Boolean.FALSE.toString()).trim();
    if (value.length() == 0 || "false".equalsIgnoreCase(value)) {
        //mock参数未配置或为false时，转发请求至内部 invoker，此时 invoker 类型为 InterceptorInvokerNode
        result = this.invoker.invoke(invocation);
    } else if (value.startsWith("force")) {
        if (logger.isWarnEnabled()) {
            logger.warn("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + getUrl());
        }
        //mock值为force，则强制走mock处理逻辑
        result = doMockInvoke(invocation, null);
    } else {
        //fail-mock
        //mock值为fail时，则调用内部invoker，并在异常时进行mock处理
        try {
            result = this.invoker.invoke(invocation);

            //fix:#4585
            //区分是业务异常，或者Rpc异常，业务异常则不走mock处理
            if(result.getException() != null && result.getException() instanceof RpcException){
                RpcException rpcException= (RpcException)result.getException();
                if(rpcException.isBiz()){
                    throw  rpcException;
                }else {
                    result = doMockInvoke(invocation, rpcException);
                }
            }

        } catch (RpcException e) {
            if (e.isBiz()) {
                throw e;
            }

            if (logger.isWarnEnabled()) {
                logger.warn("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + getUrl(), e);
            }
            result = doMockInvoke(invocation, e);
        }
    }
    return result;
}
```

这里出现了mock参数的判断及处理，有3各取值情况（false、force、fail），这其实是Dubbo远程调用的mock机制，通过它我们可以实现服务降级，或者用于测试过程中的模拟调用。具体的doMockInvoke实现处理，不在这里具体展开。继续来到 **AbstractCluster.InterceptorInvokerNode#invoke** ：

```Java
@Override
public Result invoke(Invocation invocation) throws RpcException {
    Result asyncResult;
    try {
        //cluster拦截器的前置处理，此处类型默认为ConsumerContextClusterInterceptor
        interceptor.before(next, invocation);
        //拦截并执行目标方法
        asyncResult = interceptor.intercept(next, invocation);
    } catch (Exception e) {
        // onError callback
        if (interceptor instanceof ClusterInterceptor.Listener) {
            ClusterInterceptor.Listener listener = (ClusterInterceptor.Listener) interceptor;
            listener.onError(e, clusterInvoker, invocation);
        }
        throw e;
    } finally {
        //拦截器的后置处理
        interceptor.after(next, invocation);
    }
    return asyncResult.whenCompleteWithContext((r, t) -> {
        // onResponse callback
        if (interceptor instanceof ClusterInterceptor.Listener) {
            ClusterInterceptor.Listener listener = (ClusterInterceptor.Listener) interceptor;
            if (t == null) {
                listener.onMessage(r, clusterInvoker, invocation);
            } else {
                listener.onError(t, clusterInvoker, invocation);
            }
        }
    });
}
```

这里主要是ClusterInterceptor的处理：

1. 调用 ClusterInterceptor 的前置处理；
2. 调用 ClusterInterceptor 的 intercept 方法，该方法将调用 AbstractClusterInvoker 的 invoke 方法；
3. 调用 ClusterInterceptor 的后置处理。

来到 **AbstractClusterInvoker#invoke** ：

```Java
@Override
public Result invoke(final Invocation invocation) throws RpcException {
    checkWhetherDestroyed();
	// 绑定attachments到 invocation
    // binding attachments into invocation.
    Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
    if (contextAttachments != null && contextAttachments.size() != 0) {
        ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
    }
	// 获取invoker列表
    List<Invoker<T>> invokers = list(invocation);
    // 获取负载均衡器，默认为 RandomLoadBalance 
    LoadBalance loadbalance = initLoadBalance(invokers, invocation);
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
    // doInvoke
    return doInvoke(invocation, invokers, loadbalance);
}
```

将attachments参数 添加到 invocation，并通过 Directory#list 方法获取 invoker 列表，以及加载并获取负载均衡器，最后调用 **FailoverClusterInvoker#doInvoke** 方法：

```Java
@Override
@SuppressWarnings({"unchecked", "rawtypes"})
public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
    List<Invoker<T>> copyInvokers = invokers;
    // 判空
    checkInvokers(copyInvokers, invocation);
    String methodName = RpcUtils.getMethodName(invocation);
    // 获取重试次数
    int len = calculateInvokeTimes(methodName);
    // retry loop.
    RpcException le = null; // last exception.
    List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
    Set<String> providers = new HashSet<String>(len);
    for (int i = 0; i < len; i++) {
        //Reselect before retry to avoid a change of candidate `invokers`.
        //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
        if (i > 0) {
            // 在重试时，重新获取一次最新的 invoker 列表，一定程度上确保可用
            checkWhetherDestroyed();
            copyInvokers = list(invocation);
            // check again
            checkInvokers(copyInvokers, invocation);
        }
        // 通过负载均衡器选中 invoker
        Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
        // 记录调用过的invoker 列表
        invoked.add(invoker);
        RpcContext.getContext().setInvokers((List) invoked);
        try {
            // 调用目标invoker
            Result result = invoker.invoke(invocation);
            if (le != null && logger.isWarnEnabled()) {
                //...
            }
            return result;
        } catch (RpcException e) {
            // 业务异常不进行重试
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
    //...
}
```

 在该方法中，首先获取重试次数，以在调用 invoker 失败的时候进行重试操作，在选中具体的 invoker 实例时：

```Java
protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation,
                            List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
	
    if (CollectionUtils.isEmpty(invokers)) {
        return null;
    }
    String methodName = invocation == null ? StringUtils.EMPTY_STRING : invocation.getMethodName();

    // 是否粘滞调用
    boolean sticky = invokers.get(0).getUrl()
            .getMethodParameter(methodName, CLUSTER_STICKY_KEY, DEFAULT_CLUSTER_STICKY);

    //ignore overloaded method
    // stickyInvoker 不可用时置空
    if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
        stickyInvoker = null;
    }
    //ignore concurrency problem
    // 开启了粘滞调用，并且 stickyInvoker 不为空时，如果之前没尝试调用过该 stickyInvoker，则检查是否可用，可用则直接返回
    if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
        if (availablecheck && stickyInvoker.isAvailable()) {
            return stickyInvoker;
        }
    }

    Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);
	
    // 将选中 invoker 赋值为 stickyInvoker
    if (sticky) {
        stickyInvoker = invoker;
    }
    return invoker;
}
```

这里包含了粘滞调用的处理，粘滞调用是指消费端在确保 invoker 可用时，将尽可能使用同一个 invoker 进行调用。

```Java
private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation,
                            List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {

    if (CollectionUtils.isEmpty(invokers)) {
        return null;
    }
    if (invokers.size() == 1) {
        return invokers.get(0);
    }
    // 通过 loadbalance 选择 invoker 实例
    Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

    //If the `invoker` is in the  `selected` or invoker is unavailable && availablecheck is true, reselect.
    if ((selected != null && selected.contains(invoker))
            || (!invoker.isAvailable() && getUrl() != null && availablecheck)) {
        try {
            // 如果选中 invoker 实例不可用，则进行重选
            Invoker<T> rInvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
            if (rInvoker != null) {
                invoker = rInvoker;
            } else {
                //Check the index of current selected invoker, if it's not the last one, choose the one at index+1.
                int index = invokers.indexOf(invoker);
                try {
                    //Avoid collision
                    invoker = invokers.get((index + 1) % invokers.size());
                } catch (Exception e) {
                    logger.warn(e.getMessage() + " may because invokers list dynamic change, ignore.", e);
                }
            }
        } catch (Throwable t) {
            logger.error("cluster reselect fail reason is :" + t.getMessage() + " if can not solve, you can set cluster.availablecheck=false in url", t);
        }
    }
    return invoker;
}
```

这里主要做了两件事：

1. 通过 LoadBalance 选择 invoker 实例；

2. 如果选出来的 invoker 不可用，则使用 reselect 进行重选。

在选中 invoker 实例后，回到 **FailoverClusterInvoker#doInvoke** ，将进行 invoker 的调用。在经过Dubbo Filter链的调用后，来到**AsyncToSyncInvoker** ：

```
@Override
public Result invoke(Invocation invocation) throws RpcException {
    Result asyncResult = invoker.invoke(invocation);

    try {
        //如果是同步调用，则阻塞等待结果
        if (InvokeMode.SYNC == ((RpcInvocation) invocation).getInvokeMode()) {
            /**
             * NOTICE!
             * must call {@link java.util.concurrent.CompletableFuture#get(long, TimeUnit)} because
             * {@link java.util.concurrent.CompletableFuture#get()} was proved to have serious performance drop.
             */
            asyncResult.get(Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
        }
    } catch (InterruptedException e) {
        throw new RpcException("Interrupted unexpectedly while waiting for remote result to return!  method: " +
                invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
    } catch (ExecutionException e) {
       //...
    } catch (Throwable e) {
        throw new RpcException(e.getMessage(), e);
    }
    return asyncResult;
}
```

这是一个将异步调用转换为同步调用的 Invoker ，值得注意的是，在 Dubbo 中的远程调用都是以异步方式进行调用的。

来到 DubboInvoker ：

```Java
@Override
public Result invoke(Invocation inv) throws RpcException {
    // if invoker is destroyed due to address refresh from registry, let's allow the current invoke to proceed
    if (destroyed.get()) {
        //...
    }
    RpcInvocation invocation = (RpcInvocation) inv;
    invocation.setInvoker(this);
    if (CollectionUtils.isNotEmptyMap(attachment)) {
        invocation.addObjectAttachmentsIfAbsent(attachment);
    }

    //设置attachment
    Map<String, Object> contextAttachments = RpcContext.getContext().getObjectAttachments();
    if (CollectionUtils.isNotEmptyMap(contextAttachments)) {
        /**
         * invocation.addAttachmentsIfAbsent(context){@link RpcInvocation#addAttachmentsIfAbsent(Map)}should not be used here,
         * because the {@link RpcContext#setAttachment(String, String)} is passed in the Filter when the call is triggered
         * by the built-in retry mechanism of the Dubbo. The attachment to update RpcContext will no longer work, which is
         * a mistake in most cases (for example, through Filter to RpcContext output traceId and spanId and other information).
         */
        invocation.addObjectAttachments(contextAttachments);
    }

    invocation.setInvokeMode(RpcUtils.getInvokeMode(url, invocation));
    RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

    Byte serializationId = CodecSupport.getIDByName(getUrl().getParameter(SERIALIZATION_KEY, DEFAULT_REMOTING_SERIALIZATION));
    if (serializationId != null) {
        invocation.put(SERIALIZATION_ID_KEY, serializationId);
    }

    AsyncRpcResult asyncResult;
    try {
        // 调用Dubbo#doInvoke
        asyncResult = (AsyncRpcResult) doInvoke(invocation);
    } catch (InvocationTargetException e) { // biz exception
        //...
    } catch (RpcException e) {
        //...
    } catch (Throwable e) {
        asyncResult = AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
    }
    RpcContext.getContext().setFuture(new FutureAdapter(asyncResult.getResponseFuture()));
    return asyncResult;
}
```

这里主要是将 attachment 信息添加到 invocation ，然后来到 **DubboInvoker#doInvoke** 方法：

```Java
@Override
protected Result doInvoke(final Invocation invocation) throws Throwable {
    RpcInvocation inv = (RpcInvocation) invocation;
    final String methodName = RpcUtils.getMethodName(invocation);
    //设置path 和版本信息到attachment中
    inv.setAttachment(PATH_KEY, getUrl().getPath());
    inv.setAttachment(VERSION_KEY, version);

    //获取客户端连接实例
    ExchangeClient currentClient;
    if (clients.length == 1) {
        currentClient = clients[0];
    } else {
        //轮询获取一个客户端
        currentClient = clients[index.getAndIncrement() % clients.length];
    }
    try {
        //是否单向通信
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        int timeout = calculateTimeout(invocation, methodName);
        invocation.put(TIMEOUT_KEY, timeout);
        if (isOneway) {
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            // 单身通信时忽略返回结果
            currentClient.send(inv, isSent);
            return AsyncRpcResult.newDefaultAsyncResult(invocation);
        } else {
            // 默认异步调用，返回异步结果
            ExecutorService executor = getCallbackExecutor(getUrl(), inv);
            // 发起请求
            CompletableFuture<AppResponse> appResponseFuture =
                    currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);
            // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
            FutureContext.getContext().setCompatibleFuture(appResponseFuture);
            AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
            result.setExecutor(executor);
            return result;
        }
    } catch (TimeoutException e) {
        //...
    } catch (RemotingException e) {
        //...
    }
}
```

这里首先以轮询的方式获取到客户端，然后发起远程请求，结合 **AsyncToSyncInvoker** 的处理逻辑可以看出，在非单向调用的情况下，Dubbo 的调用返回结果都是异步结果，对于同步的调用，调用结果由内部 **AsyncToSyncInvoker**  阻塞并获取调用结果返回，而对于异步调用则返回 **AsyncRpcResult** ，最终结果的获取时机则交由用户决定。



来到请求的发起，**HeaderExchangeChannel#request** ：

```Java
@Override
public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
    if (closed) {
        throw new RemotingException(this.getLocalAddress(), null,
                "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
    }
    // create request.
    // 构建Request对象
    Request req = new Request();
    req.setVersion(Version.getProtocolVersion());
    req.setTwoWay(true);
    req.setData(request);
    // 构建DefaultFuture
    DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
    try {
        //调用 NettyClient 发送请求
        channel.send(req);
    } catch (RemotingException e) {
        future.cancel();
        throw e;
    }
    return future;
}
```

首先是构建 **Request** 请求对象，然后通过 **DefaultFuture#newFuture**  构造一个 **DefaultFuture** ：

```Java
public static DefaultFuture newFuture(Channel channel, Request request, int timeout, ExecutorService executor) {
    final DefaultFuture future = new DefaultFuture(channel, request, timeout);
    future.setExecutor(executor);
    // ThreadlessExecutor needs to hold the waiting future in case of circuit return.
    if (executor instanceof ThreadlessExecutor) {
        ((ThreadlessExecutor) executor).setWaitingFuture(future);
    }
    // timeout check
    timeoutCheck(future);
    return future;
}
```

```Java
private DefaultFuture(Channel channel, Request request, int timeout) {
    this.channel = channel;
    this.request = request;
    //获取请求 id
    this.id = request.getId();
    this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
    // put into waiting map.
    //存储 <请求id, DefaultFuture>的映射关系到 FUTURES 中
    FUTURES.put(id, this);
    CHANNELS.put(id, channel);
}
```

在 **DefaultFuture** 的构造函数中，可以看到，在 DefaultFuture 中，维护了 **请求id ->DefaultFuture** 的映射关系，而这个是 Dubbo 实现异步的关键。

当该次调用接收到结果时，通过这个映射关系可以直接找到 **DefaultFuture** 对象，然后将调用结果填充回去，整个异步调用的请和结果的关联在此处得以体现，**DefaultFuture#received**：

```Java
public static void received(Channel channel, Response response, boolean timeout) {
    try {
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            Timeout t = future.timeoutCheckTask;
            if (!timeout) {
                // decrease Time
                t.cancel();
            }
            future.doReceived(response);
        } else {
            //...
        }
    } finally {
        CHANNELS.remove(response.getId());
    }
}
```



拿到 Request 请求对象后，最后发送请求，Dubbo 默认使用 Netty作为底层通信框架，此时 channel 实例类型为 NettyClient ，即通过 **NettyClient#send** 方法发送请求，后续就是根据 Dubbo 协议对该请求对象进行编码然后将请求发送出去。



关于 Dubbo消费端请求的发起调用，本篇就简单看到这里，这里附上一个调用的关键路径：

```
InvokerInvocationHandler#invoke
MigrationInvoker#invoke
MockClusterInvoker#invoke
AbstractCluster$InterceptorInvokerNode#invoke
(ConsumerContextClusterInterceptor)ClusterInterceptor#intercept
AbstractClusterInvoker#invoke -> FailoverClusterInvoker#doInvoke
FilterNode#invoke（Filter链执行）
AsyncToSyncInvoker#invoke
AbstractInvoker#invoke -> DubboInvoker#invoke
ReferenceCountExchangeClient#request
HeaderExchangeClient#request
HeaderExchangeChannel#request
NettyClient#send
```



### 服务提供端

前面我们提到，Dubbo 使用 Netty 作为底层通信框架，当 Netty 接收到新请求后，首先会根据 Dubbo 协议通过解码器进行 **Request** 请求对象的解码，在得到 Request 请求对象后，接着便是调用服务了，我们直接看下 **ChannelEventRunnable#run** ：

```Java
@Override
public void run() {
    //处理接收事件，包括请求和响应事件
    if (state == ChannelState.RECEIVED) {
        try {
            handler.received(channel, message);
        } catch (Exception e) {
            logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                    + ", message is " + message, e);
        }
    } else {
        switch (state) {
        //处理建立连接事件
        case CONNECTED:
            try {
                handler.connected(channel);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
            }
            break;
        //处理断开连接事件
        case DISCONNECTED:
            try {
                handler.disconnected(channel);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel, e);
            }
            break;
        //处理发送事件
        case SENT:
            try {
                handler.sent(channel, message);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                        + ", message is " + message, e);
            }
            break;
        //异常捕获
        case CAUGHT:
            try {
                handler.caught(channel, exception);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                        + ", message is: " + message + ", exception is " + exception, e);
            }
            break;
        default:
            logger.warn("unknown state: " + state + ", message is " + message);
        }
    }

}
```

这里针对不同类型的事件，调用 **DecodeHandler** 的不同方法，对于接收请求，则来到 **DecodeHandler#received** ：

```Java
@Override
public void received(Channel channel, Object message) throws RemotingException {
    if (message instanceof Decodeable) {
        // 对 Decodeable 对象进行解码
        decode(message);
    }

    if (message instanceof Request) {
        // 对 Request 对象的 data 进行解码
        decode(((Request) message).getData());
    }

    if (message instanceof Response) {
        // 对 Response 的 result 进行解码
        decode(((Response) message).getResult());
    }

    handler.received(channel, message);
}
```

解码后， Request 对象传递到 **HeaderExchangeHandler#received** ：

```Java
@Override
public void received(Channel channel, Object message) throws RemotingException {
    final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    if (message instanceof Request) {
        // 处理请求对象
        Request request = (Request) message;
        if (request.isEvent()) {
            // 处理事件
            handlerEvent(channel, request);
        } else {
            // 双向通信
            if (request.isTwoWay()) {
                handleRequest(exchangeChannel, request);
            } else {
                // 单向通信时，直接调用 handler 处理，忽略响应
                handler.received(exchangeChannel, request.getData());
            }
        }
    // 处理响应请求
    } else if (message instanceof Response) {
        handleResponse(channel, (Response) message);
    } else if (message instanceof String) {
        if (isClientSide(channel)) {
            Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
            logger.error(e.getMessage(), e);
        } else {
            String echo = handler.telnet(channel, (String) message);
            if (echo != null && echo.length() > 0) {
                channel.send(echo);
            }
        }
    } else {
        handler.received(exchangeChannel, message);
    }
}
```

这里区分了双向通信和单向通信，对于双向通信，则先来到 **HeaderExchangeHandler#handleRequest** :

```Java
void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
    Response res = new Response(req.getId(), req.getVersion());
    //检测请求是否合法，非法则返回 BAD_REQUEST
    if (req.isBroken()) {
        Object data = req.getData();

        String msg;
        if (data == null) {
            msg = null;
        } else if (data instanceof Throwable) {
            msg = StringUtils.toString((Throwable) data);
        } else {
            msg = data.toString();
        }
        res.setErrorMessage("Fail to decode request due to: " + msg);
        res.setStatus(Response.BAD_REQUEST);

        channel.send(res);
        return;
    }
    // 获取 data 信息，也就是 RpcInvocation 对象
    Object msg = req.getData();
    try {
        // 继续传递，ExchangeHandlerAdapter#reply ，该实现类为 DubboProtocol 内部匿名实现类对象
        CompletionStage<Object> future = handler.reply(channel, msg);
        future.whenComplete((appResult, t) -> {
            try {
                // 设置响应状态码 
                if (t == null) {
                    res.setStatus(Response.OK);
                    res.setResult(appResult);
                } else {
                    res.setStatus(Response.SERVICE_ERROR);
                    res.setErrorMessage(StringUtils.toString(t));
                }
                // 往客户端回写响应结果
                channel.send(res);
            } catch (RemotingException e) {
                logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
            }
        });
    } catch (Throwable e) {
        res.setStatus(Response.SERVICE_ERROR);
        res.setErrorMessage(StringUtils.toString(e));
        channel.send(res);
    }
}
```

来到 **DubboProtocol$ExchangeHandlerAdapter#reply** ：

```Java
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
    @Override
    public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {

        if (!(message instanceof Invocation)) {
            throw new RemotingException(channel, "Unsupported request: "
                    + (message == null ? null : (message.getClass().getName() + ": " + message))
                    + ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress());
        }

        Invocation inv = (Invocation) message;
        // 获取 invoker 实例
        Invoker<?> invoker = getInvoker(channel, inv);
        // need to consider backward-compatibility if it's a callback
        if (Boolean.TRUE.toString().equals(inv.getObjectAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
            String methodsStr = invoker.getUrl().getParameters().get("methods");
            boolean hasMethod = false;
            if (methodsStr == null || !methodsStr.contains(",")) {
                hasMethod = inv.getMethodName().equals(methodsStr);
            } else {
                String[] methods = methodsStr.split(",");
                for (String method : methods) {
                    if (inv.getMethodName().equals(method)) {
                        hasMethod = true;
                        break;
                    }
                }
            }
            if (!hasMethod) {
                logger.warn(new IllegalStateException("The methodName " + inv.getMethodName()
                        + " not found in callback service interface ,invoke will be ignored."
                        + " please update the api interface. url is:"
                        + invoker.getUrl()) + " ,invocation is :" + inv);
                return null;
            }
        }
        RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
        // 通过 invoker 调用服务
        Result result = invoker.invoke(inv);
        return result.thenApply(Function.identity());
    }
    //...
}
```

这里首先通过 **DubboProtocol#getInvoker** 是获取对应的 invoker 实例：

```Java
Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
    boolean isCallBackServiceInvoke = false;
    boolean isStubServiceInvoke = false;
    int port = channel.getLocalAddress().getPort();
    String path = (String) inv.getObjectAttachments().get(PATH_KEY);

    // if it's callback service on client side
    isStubServiceInvoke = Boolean.TRUE.toString().equals(inv.getObjectAttachments().get(STUB_EVENT_KEY));
    if (isStubServiceInvoke) {
        port = channel.getRemoteAddress().getPort();
    }

    //callback
    isCallBackServiceInvoke = isClientSide(channel) && !isStubServiceInvoke;
    if (isCallBackServiceInvoke) {
        path += "." + inv.getObjectAttachments().get(CALLBACK_SERVICE_KEY);
        inv.getObjectAttachments().put(IS_CALLBACK_SERVICE_INVOKE, Boolean.TRUE.toString());
    }

    // 获取服务 serviceKey ，通过 serviceKey 获取 DubboProtocol 中维护的 exporter 实例
    String serviceKey = serviceKey(
            port,
            path,
            (String) inv.getObjectAttachments().get(VERSION_KEY),
            (String) inv.getObjectAttachments().get(GROUP_KEY)
    );
    DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

    if (exporter == null) {
        throw new RemotingException(channel,
                "Not found exported service: " + serviceKey + " in " + exporterMap.keySet() + ", may be version or group mismatch " +
                        ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress() +
                        ", message:" + getInvocationWithoutData(inv));
    }

    // 返回 invoker 实例
    return exporter.getInvoker();
}
```

然后调用 **Invoker#invoke** 方法，我们在服务导出中了解到，服务导出生成的 invoker 实例实际是动态代理对象，在经过服务提供端拦截器链的处理后，会调用该代理对象的 invoke 方法：

```Java
public class JavassistProxyFactory extends AbstractProxyFactory {

	//省略...

    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        //创建Wrapper类实例
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        //覆写doInvoke方法
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

}
```

而 invoke 方法又通过 Wrapper 对象，使用反射调用具体的业务实现方法：

```Java
public class Wrapper1
extends Wrapper
implements ClassGenerator.DC {
    //省略...

    public Object invokeMethod(Object object, String string, Class[] classArray, Object[] objectArray) throws InvocationTargetException {
        HelloServiceImpl helloServiceImpl;
        try {
            helloServiceImpl = (HelloServiceImpl)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        try {
            if ("sayHello".equals(string) && classArray.length == 0) {
                return helloServiceImpl.sayHello();
            }
        }
        catch (Throwable throwable) {
            throw new InvocationTargetException(throwable);
        }
        throw new NoSuchMethodException(new StringBuffer().append("Not found method \"").append(string).append("\" in class laixiaoming.service.HelloServiceImpl.").toString());
    }
}
```

在拿到服务调用返回后，将其封装为 Response 对象，最后通过 **HeaderExchangeChannel#send** 回写 Response 对象到调用方。 

关于服务提供端的处理链路，这里附上一个调用中的关键的路径：

```
ChannelEventRunnable#run
DecodeHandler#received
HeaderExchangeHandler#received
HeaderExchangeHandler#handleRequest
DubboProtocol$ExchangeHandlerAdapter#reply
AbstractProxyInvoker#invoke
Wrapper1#invokeMethod
HelloServiceImpl#sayHello
```



### 调用结果的处理

在服务提供方将 Response 对象返回时，同样有一个编码的过程，在服务消费方接收到响应数据后，也有一个解码的过程，最终得到 Response 对象。过程我们直接看下 **HeaderExchangeHandler#received** 中的处理：

```Java
@Override
public void received(Channel channel, Object message) throws RemotingException {
    final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
    if (message instanceof Request) {
        //省略...
    } else if (message instanceof Response) {
        handleResponse(channel, (Response) message);
    } else if (message instanceof String) {
        //省略...
    } else {
        handler.received(exchangeChannel, message);
    }
}
```

 **HeaderExchangeHandler#handleResponse** ：

```Java
static void handleResponse(Channel channel, Response response) throws RemotingException {
    if (response != null && !response.isHeartbeat()) {
        DefaultFuture.received(channel, response);
    }
}
```

来到 **DefaultFuture#received** ：

```Java
public static void received(Channel channel, Response response) {
    received(channel, response, false);
}

public static void received(Channel channel, Response response, boolean timeout) {
    try {
        //根据请求id 获取到关联的 DefaultFuture对象
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            Timeout t = future.timeoutCheckTask;
            if (!timeout) {
                // decrease Time
                t.cancel();
            }
            future.doReceived(response);
        } else {
            logger.warn("The timeout response finally returned at "
                    + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                    + ", response status is " + response.getStatus()
                    + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                    + " -> " + channel.getRemoteAddress()) + ", please check provider side for detailed result.");
        }
    } finally {
        CHANNELS.remove(response.getId());
    }
}
```

通过请求 id 获取到对应的 DefaultFuture 对象后，**DefaultFuture#doReceived** ：

```Java
private void doReceived(Response res) {
    if (res == null) {
        throw new IllegalStateException("response cannot be null");
    }
    if (res.getStatus() == Response.OK) {
        this.complete(res.getResult());
    } else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
        this.completeExceptionally(new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage()));
    } else {
        this.completeExceptionally(new RemotingException(channel, res.getErrorMessage()));
    }

    // the result is returning, but the caller thread may still waiting
    // to avoid endless waiting for whatever reason, notify caller thread to return.
    if (executor != null && executor instanceof ThreadlessExecutor) {
        ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
        if (threadlessExecutor.isWaiting()) {
            threadlessExecutor.notifyReturn(new IllegalStateException("The result has returned, but the biz thread is still waiting" +
                    " which is not an expected state, interrupt the thread manually by returning an exception."));
        }
    }
}
```

最后将响应结果设置到 Future 中，消费端用户线程被唤起，Future#get 就可以获取到调用结果了。至此，一次 dubbo 调用就大致结束了。



### 小结

本文主要就 Dubbo 调用的整个过程，从源码的角度作了简单分析，包括：服务消费端的请求发起、服务提供端处理请求的处理、服务提供端响应的返回、以及服务消费端响应的处理。