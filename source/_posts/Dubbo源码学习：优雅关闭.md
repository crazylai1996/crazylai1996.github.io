---
title: Dubbo源码学习：优雅关闭
date: 2025-01-12 15:09:50
tags: Dubbo

---



### 前言

在微服务架构中，能快速更新和迭代是其最大的特点之一，而在重启或更新的过程中，为了确保现有请求的完整性以及最小化对业务的影响，“优雅关闭”就显得十分重要了。

本文将结合源码一起来学习下在 Dubbo 中，“优雅关闭”是如何实现的。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本



### Shutdown Hook

在 Linux 中，终止一个线程通常使用 ``kill [参数] [PID]`` 完成，其常用的信号选项有：

1. ``kill pid``，等同于 ``kill -15 pid``，向指定进程发送 SIGTERM 信号；
2. ``kill -9 pid`` ，向指定进程发送 SIGKILL 立即终止信号；
3. ``kill -2 pid`` ，向指定进程发送 SIGINT 中断信号，等同于 ``CTRL + C`` ，用于结束前台进程。

而在 Java 中，JVM 提供了 Shutdown Hook 钩子，在 JVM 接收到 SIGTERM 信号通知时，将会触发已注册的 Shutdown Hook 关闭钩子的执行，而注册关闭钩子函数则可以通过 ``Java.Runtime.addShutdownHook``完成，该方法接收一个 ``java.lang.Thread`` 参数，执行时将启动所有的钩子线程。 



### Dubbo 的优雅关闭

Dubbo 的优雅关闭同样了依赖于 Shutdown Hook，这也表明了只有在对 Java 进程发送 SIGTERM （``kill pid``）信号时，优雅关闭才会生效。

#### DubboShutdownHook

在 Dubbo 启动时，将在 DubboBootstrap 中的构造方法中注册了 **DubboShutdownHook**：

```Java
private DubboBootstrap() {
    // ... 省略

	// 注册DubboShutdownHook
    DubboShutdownHook.getDubboShutdownHook().register();
    // 添加DubboShutdownHook回调函数
    ShutdownHookCallbacks.INSTANCE.addCallback(DubboBootstrap.this::destroy);
}
```

在收到进程关闭信号时，将启动对应的钩子线程，执行 run 方法：

```Java
public void run() {
    if (logger.isInfoEnabled()) {
        logger.info("Run shutdown hook now.");
    }

    callback();
    doDestroy();
}
```

在 callback 方法中，会执行所有注册的回调方法，包含 DubboBootstrap#destroy ：

```Java
public void destroy() {
    if (destroyLock.tryLock()) {
        try {
            if (started.compareAndSet(true, false)
                    && destroyed.compareAndSet(false, true)) {
				// 注销移除当前应用级服务实例在注册中心的记录
                unregisterServiceInstance();
                // 取消导出元数据服务
                unexportMetadataService();
                // 取消服务导出，注销在注册中心注册的记录
                unexportServices();
                // 取消服务引用
                unreferServices();
				//注册中心
                destroyRegistries();
				// 销毁应用级注册中心
                destroyServiceDiscoveries();
                destroyExecutorRepository();
                // 清除应用配置类以及相关应用模型
                clear();
                shutdown();
                release();
                ExtensionLoader<DubboBootstrapStartStopListener> exts = getExtensionLoader(DubboBootstrapStartStopListener.class);
                exts.getSupportedExtensionInstances().forEach(ext -> ext.onStop(this));
            }
			// hook 执行
            DubboShutdownHook.destroyAll();
        } finally {
            destroyLock.unlock();
        }
    }
}
```

在这里，我们主要学习下取消服务导出、销毁注册中心、以及关闭 Protocol 这几个流程。



#### 取消服务导出

取消服务导出的流程包含在 unexportServices 方法中：

```Java
private void unexportServices() {
    // 遍历所有已导出的服务，调用其 unexport 方法
    exportedServices.forEach((serviceName, sc) -> {
        configManager.removeConfig(sc);
        sc.unexport();
    });

    asyncExportingFutures.forEach(future -> {
        if (!future.isDone()) {
            future.cancel(true);
        }
    });
    asyncExportingFutures.clear();
    exportedServices.clear();
}
```

它会针对每一个已导出的服务，调用其ServiceConfig#unexport 方法：

```
public void unexport() {
    if (!exported) {
        return;
    }
    if (unexported) {
        return;
    }
    if (!exporters.isEmpty()) {
    	// 调用 Exporter#unexport 方法
        for (Exporter<?> exporter : exporters) {
            try {
                exporter.unexport();
            } catch (Throwable t) {
                logger.warn("Unexpected error occurred when unexport " + exporter, t);
            }
        }
        exporters.clear();
    }
    unexported = true;

    // dispatch a ServiceConfigUnExportedEvent since 2.7.4
    dispatch(new ServiceConfigUnexportedEvent(this));
}
```

最终来到 RegistryProtocol.ExporterChangeableWrapper#unexport：

```Java
public void unexport() {
    if (!unexported.compareAndSet(false,true)) {
        return;
    }

    String key = getCacheKey(this.originInvoker);
    bounds.remove(key);

    // 获取注册中心实例
    Registry registry = RegistryProtocol.this.getRegistry(originInvoker);
    try {
        // 向注销下线该服务
        registry.unregister(registerUrl);
    } catch (Throwable t) {
        LOGGER.warn(t.getMessage(), t);
    }
    try {
        // 取消订阅该服务
        NotifyListener listener = RegistryProtocol.this.overrideListeners.remove(subscribeUrl);
        registry.unsubscribe(subscribeUrl, listener);
        ExtensionLoader.getExtensionLoader(GovernanceRuleRepository.class).getDefaultExtension()
                .removeListener(subscribeUrl.getServiceKey() + CONFIGURATORS_SUFFIX,
                        serviceConfigurationListeners.get(subscribeUrl.getServiceKey()));
    } catch (Throwable t) {
        LOGGER.warn(t.getMessage(), t);
    }

    executor.submit(() -> {
        try {
            // 获取关闭等待时长，默认 10 ms
            int timeout = ConfigurationUtils.getServerShutdownTimeout();
            if (timeout > 0) {
                LOGGER.info("Waiting " + timeout + "ms for registry to notify all consumers before unexport. " +
                        "Usually, this is called when you use dubbo API");
                Thread.sleep(timeout);
            }
            // 销毁 exporter 实例
            exporter.unexport();
        } catch (Throwable t) {
            LOGGER.warn(t.getMessage(), t);
        }
    });
}
```

可以看到，取消服务导出，主要：

1. 首先向注册中心注销当前服务，以 Nacos 注册中心为例，会下线当前服务，同时停掉与注册的心跳任务，然后取消订阅；
2. 在等待 10 s 后（等待时间可以通过 dubbo.service.shutdown.wait 配置），销毁对应的 exporter 实例（异步执行）。

这里为什么会有个等待时间呢？在向注册中心下线注销服务后，下线通知通知到服务消费端往往存在一定时间的延迟，在这期间新的请求还是有可能发往该服务提供者，考虑到这种情况，加入等待机制能尽可能地将影响降至最小。当然，等待时长也需考虑实际的情况，太短可能会对业务有影响，太长则影响发布的时长。



#### 销毁注册中心

在 destroyRegistries 方法中，包含了注销注册中心的逻辑，AbstractRegistryFactory#destroyAll：

```Java
public static void destroyAll() {
    // 是不销毁
    if (!destroyed.compareAndSet(false, true)) {
        return;
    }

    if (LOGGER.isInfoEnabled()) {
        LOGGER.info("Close all registries " + getRegistries());
    }
    // Lock up the registry shutdown process
    LOCK.lock();
    try {
        // 获取所有的注册中心，逐个销毁
        for (Registry registry : getRegistries()) {
            try {
                registry.destroy();
            } catch (Throwable e) {
                LOGGER.error(e.getMessage(), e);
            }
        }
        REGISTRIES.clear();
    } finally {
        // Release the lock
        LOCK.unlock();
    }
}
```

以 nacos 注册中心为例，会来到 org.apache.dubbo.registry.support.AbstractRegistry#destroy：

```Java
public void destroy() {
    if (logger.isInfoEnabled()) {
        logger.info("Destroy registry:" + getUrl());
    }
    // 已注册的地址
    Set<URL> destroyRegistered = new HashSet<>(getRegistered());
    if (!destroyRegistered.isEmpty()) {
        for (URL url : new HashSet<>(destroyRegistered)) {
            if (url.getParameter(DYNAMIC_KEY, true)) {
                try {
                    unregister(url);
                    if (logger.isInfoEnabled()) {
                        logger.info("Destroy unregister url " + url);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to unregister url " + url + " to registry " + getUrl() + " on destroy, cause: " + t.getMessage(), t);
                }
            }
        }
    }
    // 订阅地址
    Map<URL, Set<NotifyListener>> destroySubscribed = new HashMap<>(getSubscribed());
    if (!destroySubscribed.isEmpty()) {
        for (Map.Entry<URL, Set<NotifyListener>> entry : destroySubscribed.entrySet()) {
            URL url = entry.getKey();
            for (NotifyListener listener : entry.getValue()) {
                try {
                    unsubscribe(url, listener);
                    if (logger.isInfoEnabled()) {
                        logger.info("Destroy unsubscribe url " + url);
                    }
                } catch (Throwable t) {
                    logger.warn("Failed to unsubscribe url " + url + " to registry " + getUrl() + " on destroy, cause: " + t.getMessage(), t);
                }
            }
        }
    }
    AbstractRegistryFactory.removeDestroyedRegistry(this);
}
```

这里做的事情主要有两件：

1. 移除内存中已经注册的服务；
2. 是取消所有服务订阅。



#### 关闭 Protocol

在 DubboShutdownHook.destroyAll 方法中，包含了销毁注册中心和关闭 Protocol 两个事情，其中销毁注册中心在前面执行过，不会重复执行，我们直接来看下 Protocol 的关闭：

```Java
public static void destroyProtocols() {
    ExtensionLoader<Protocol> loader = ExtensionLoader.getExtensionLoader(Protocol.class);
    for (String protocolName : loader.getLoadedExtensions()) {
        try {
            // 加载协议
            Protocol protocol = loader.getLoadedExtension(protocolName);
            if (protocol != null) {
                protocol.destroy();
            }
        } catch (Throwable t) {
            logger.warn(t.getMessage(), t);
        }
    }
}
```

在运行时，loader.getLoadedExtension(protocolName) 会返回两种 Protocol：InjvmProtocol 和 DubboProtocol，而 DubboProtocol 用于提供网络间的调用，我们主要来看下 DubboProtocol#destroy：

```Java
public void destroy() {
    for (String key : new ArrayList<>(serverMap.keySet())) {
        ProtocolServer protocolServer = serverMap.remove(key);

        if (protocolServer == null) {
            continue;
        }

        // 获取服务提供端 server
        RemotingServer server = protocolServer.getRemotingServer();

        try {
            if (logger.isInfoEnabled()) {
                logger.info("Close dubbo server: " + server.getLocalAddress());
            }

            // 传入关闭时间
            server.close(ConfigurationUtils.getServerShutdownTimeout());

        } catch (Throwable t) {
            logger.warn(t.getMessage(), t);
        }
    }

    // 获取服务消费端 client
    for (String key : new ArrayList<>(referenceClientMap.keySet())) {
        Object clients = referenceClientMap.remove(key);
        if (clients instanceof List) {
            List<ReferenceCountExchangeClient> typedClients = (List<ReferenceCountExchangeClient>) clients;

            if (CollectionUtils.isEmpty(typedClients)) {
                continue;
            }

            for (ReferenceCountExchangeClient client : typedClients) {
                closeReferenceCountExchangeClient(client);
            }
        }
    }

    super.destroy();
}
```

包含了两部分逻辑：server 和 client，server 端对应了当前应用的服务提供者，而 client 对应当前应用的服务消费者。

##### 关闭 server 端

server 端的关闭会来到 HeaderExchangeServer#close(int)：

```Java
public void close(final int timeout) {
    startClose();
    if (timeout > 0) {
        final long max = timeout;
        final long start = System.currentTimeMillis();
        // 发送 READ_ONLY 事件，通知服务消费端不接收新请求
        if (getUrl().getParameter(Constants.CHANNEL_SEND_READONLYEVENT_KEY, true)) {
            sendChannelReadOnlyEvent();
        }
        // 等待
        while (isRunning() && System.currentTimeMillis() - start < max) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage(), e);
            }
        }
    }
    // 停掉连接关闭任务
    doClose();
    server.close(timeout);
}
```

首先会向服务消费端发送 READ_ONLY 事件，服务消费端在接收到此事件后不会再调用该服务提供者。前面在销毁注册中心时，实际已经向注册中心下线服务了，这里为什么又通知一次呢？这其实还是延迟的考虑，这样可以进一步降低注册中心延迟带来的影响。

来到 AbstractServer#close(int)：

```Java
public void close(int timeout) {
    // 优雅关闭线程池
    ExecutorUtil.gracefulShutdown(executor, timeout);
    // 关闭底层通信服务
    close();
}
```

关闭业务线程池，等待特定时间后关闭，主要是考虑尽可能将将已有任务执行完。

最后是关闭底通信服务 NettyServer。



##### 关闭 client 端

client 端的关闭逻辑会首先来到 HeaderExchangeClient#close(int)：

```Java
public void close(int timeout) {
    // Mark the client into the closure process
    startClose();
    // 关闭心跳和重连定时任务
    doClose();
    // 关闭
    channel.close(timeout);
}
```

来到 HeaderExchangeChannel#close(int)：

```Java
public void close(int timeout) {
    if (closed) {
        return;
    }
    closed = true;
    if (timeout > 0) {
        long start = System.currentTimeMillis();
        // 是否存在未收到响应的请求
        while (DefaultFuture.hasFuture(channel)
                && System.currentTimeMillis() - start < timeout) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                logger.warn(e.getMessage(), e);
            }
        }
    }
    close();
}
```

关闭 client 时，如果存在未收到响应的请求，并且等待的时长小于超时时长，将进行等待。这里主要是尽可能的保证了发送出去但未收到响应的请求能在关闭前处理完。



#### 在 Spring 容器下的兼容

在 Spring 环境下，由于 Spring 也有着自己的 ShutdownHook，而 DubboShutdownHook 的执行与 Spring 的不在同一个线程内，因此执行的顺序无人保证。假设 Spring 先销毁了，其容器内的 bean 也将被销毁，如果 Dubbo 依赖了 Spring 中的 bean（如数据源等），即使此时 Dubbo 的 DubboShutdownHook 未执行，此时也无法正常使用，从而对业务也会产生中断。针对这种情况，Dubbo 也做了兼容的处理。

如果当前是 Spring 环境，SpringExtensionFactory 会被加载，并通过 SpringExtensionFactory#addApplicationContext：

```Java
public static void addApplicationContext(ApplicationContext context) {
    CONTEXTS.add(context);
    if (context instanceof ConfigurableApplicationContext) {
        // 注册 Spring 自带的 ShutdownHook
        ((ConfigurableApplicationContext) context).registerShutdownHook();
        // see https://github.com/apache/dubbo/issues/7093
        // 移除 Dubbo 的 ShutdownHook
        DubboShutdownHook.getDubboShutdownHook().unregister();
    }
}
```

将 Dubbo 的 ShutdownHook 移除，同时新增了 DubboBootstrapApplicationListener ，增加了对 Spring 容器关闭事件的处理：

```Java
public void onApplicationContextEvent(ApplicationContextEvent event) {
    if (DubboBootstrapStartStopListenerSpringAdapter.applicationContext == null) {
        DubboBootstrapStartStopListenerSpringAdapter.applicationContext = event.getApplicationContext();
    }
    if (event instanceof ContextRefreshedEvent) {
        onContextRefreshedEvent((ContextRefreshedEvent) event);
    } else if (event instanceof ContextClosedEvent) {
        onContextClosedEvent((ContextClosedEvent) event);
    }
}
private void onContextClosedEvent(ContextClosedEvent event) {
    DubboShutdownHook.getDubboShutdownHook().run();
}
```

在 ContextClosedEvent 事件处理逻辑中，调用了 DubboShutdownHook  的处理方法。



### 总结

总的来说，Dubbo 的优雅关闭主要分为几个部分：取消服务导出、销毁注册中心、以及关闭 Protocol 。主要确保了：

1. 服务提供者向注册中心注销该服务，同时通知到服务消费者，将不再接收新的请求；

2. 对于服务提供者，能够尽可能地在停止之前完成所有正在进行的请求的处理；

3. 对于服务消费者，已经发出的服务请求，会等待响应返回再关闭。