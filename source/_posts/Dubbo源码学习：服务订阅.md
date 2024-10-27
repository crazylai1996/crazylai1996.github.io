---
title: Dubbo源码学习：服务订阅
date: 2024-09-28 11:49:43
tags: Dubbo

---



在前一篇 [Dubbo源码学习：服务引用](https://laixiaoming.space/2024/07/13/Dubbo%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%EF%BC%9A%E6%9C%8D%E5%8A%A1%E5%BC%95%E7%94%A8/#more) 中我们了解到，在Dubbo服务消费端，Invoker对象具有远程调用的功能，但服务消费端是如何感知服务端的地址呢？在实际使用时，同一个服务提供者往往具有多个实例，在服务提供者实例上下线或实例数量发生变更时，服务消费端会如何做出相应的更新？

在深入了解之前，我们需要先了解下服务目录的概念。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本



### 服务目录Directory

服务目录是服务提供者的集合，包含了多个Invoker，其中存储了与服务提供者有关的信息，通过服务目录，服务消费者可以获取到服务提供者的地址等信息。服务目录分为**StaticDirectory**和**DynamicDirectory**，其类继承体系图如下：

![image-20240928154310365](http://storage.laixiaoming.space/blog/image-20240928154310365.png)

1. **StaticDirectory** 是静态服务目录，其服务提供者列表是静态的，在创建完成之后不会在运行期间发生变化。
2. **DynamicDirectory** 是动态服务目录，其维护的提供者列表是动态变化的。动态服务目录实现了 **NotifyListener** 接口，在创建后向注册中心订阅服务提供者的变化信息，当收到来自注册中心的服务提供者变更通知后，会根据变更内容更新其中维护的服务提供者列表。
3. 动态服务目录有两种：**RegistryDirectory** 和 **ServiceDiscoveryRegistryDirectory** 。**RegistryDirectory** 用于记录和监听接口级服务提供者，而 **ServiceDiscoveryRegistryDirectory** 则用来记录和监听应用级服务提供者。

本文主要以 **RegistryDirectory** 为例进行深入了解。



### 服务订阅

在前一篇 [Dubbo源码学习：服务引用](https://laixiaoming.space/2024/07/13/Dubbo%E6%BA%90%E7%A0%81%E5%AD%A6%E4%B9%A0%EF%BC%9A%E6%9C%8D%E5%8A%A1%E5%BC%95%E7%94%A8/#more) 中我们了解到，远程调用的Invoker是通过 **InterfaceCompatibleRegistryProtocol##getInvoker** 创建的：

```Java
public <T> ClusterInvoker<T> getInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
    //创建RegistryDirectory实例
    DynamicDirectory<T> directory = new RegistryDirectory<>(type, url);
    return doCreateInvoker(directory, cluster, registry, type);
}

```

**RegistryProtocol#doCreateInvoker** ：

```Java
protected <T> ClusterInvoker<T> doCreateInvoker(DynamicDirectory<T> directory, Cluster cluster, Registry registry, Class<T> type) {
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    // all attributes of REFER_KEY
    Map<String, String> parameters = new HashMap<String, String>(directory.getConsumerUrl().getParameters());
    URL urlToRegistry = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (directory.isShouldRegister()) {
        directory.setRegisteredConsumerUrl(urlToRegistry);
        registry.register(directory.getRegisteredConsumerUrl());
    }
    directory.buildRouterChain(urlToRegistry);
    //订阅服务变更通知
    directory.subscribe(toSubscribeUrl(urlToRegistry));
	//cluster类型为MockClusterWrapper
    return (ClusterInvoker<T>) cluster.join(directory);
}
```

**MockClusterWrapper#join**：

```Java
public class MockClusterWrapper implements Cluster {
	//FailoverCluster实现类
    private Cluster cluster;

    public MockClusterWrapper(Cluster cluster) {
        this.cluster = cluster;
    }

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
    	//返回MockClusterInvoker实例
        return new MockClusterInvoker<T>(directory,
                this.cluster.join(directory));
    }

}
```

**AbstractCluster#join**：

```Java
public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
	//doJoin返回FailoverClusterInvoker实例
    return buildClusterInterceptors(doJoin(directory), directory.getUrl().getParameter(REFERENCE_INTERCEPTOR_KEY));
}
```



在创建 **RegistryDirectory** 实例后，则通过 **RegistryDirectory#subscribe** 订阅服务提供方变更通知：

```Java
public void subscribe(URL url) {
    setConsumerUrl(url);
    //将当前RegistryDirectory作为配置监听器注册
    CONSUMER_CONFIGURATION_LISTENER.addNotifyListener(this);
    referenceConfigurationListener = new ReferenceConfigurationListener(this, url);
    //订阅服务变更通知
    registry.subscribe(url, this);
}
```

深入subscribe方法，这里以 Nacos 注册中心为例，来到 **NacosRegistry#doSubscribe** ：

```Java
public void doSubscribe(final URL url, final NotifyListener listener) {
	//获取服务名，这里为了兼容旧版本服务，同一个服务会生成两个服务名
    Set<String> serviceNames = getServiceNames(url, listener);

    //将新旧版本的同一个服务名关联起来，以便于后续一起处理
    if (isServiceNamesWithCompatibleMode(url)) {
        for (String serviceName : serviceNames) {
            NacosInstanceManageUtil.setCorrespondingServiceNames(serviceName, serviceNames);
        }
    }

    doSubscribe(url, listener, serviceNames);
}

```

```Java
private void doSubscribe(final URL url, final NotifyListener listener, final Set<String> serviceNames) {
    execute(namingService -> {
        //服务名称兼容模式
        if (isServiceNamesWithCompatibleMode(url)) {
            List<Instance> allCorrespondingInstanceList = Lists.newArrayList();

            /**
             * Get all instances with serviceNames to avoid instance overwrite and but with empty instance mentioned
             * in https://github.com/apache/dubbo/issues/5885 and https://github.com/apache/dubbo/issues/5899
             *
             * namingService.getAllInstances with {@link org.apache.dubbo.registry.support.AbstractRegistry#registryUrl}
             * default {@link DEFAULT_GROUP}
             *
             * in https://github.com/apache/dubbo/issues/5978
             */
            for (String serviceName : serviceNames) {
                //从nacos获取所有已有实例
                List<Instance> instances = namingService.getAllInstances(serviceName,
                        getUrl().getParameter(GROUP_KEY, Constants.DEFAULT_GROUP));
                NacosInstanceManageUtil.initOrRefreshServiceInstanceList(serviceName, instances);
                allCorrespondingInstanceList.addAll(instances);
            }
            //以当前获取到的所有实例通知到directory进行刷新
            notifySubscriber(url, listener, allCorrespondingInstanceList);
            for (String serviceName : serviceNames) {
                //订阅服务变更
                subscribeEventListener(serviceName, url, listener);
            }
        } else {
            List<Instance> instances = new LinkedList<>();
            for (String serviceName : serviceNames) {
                instances.addAll(namingService.getAllInstances(serviceName
                        , getUrl().getParameter(GROUP_KEY, Constants.DEFAULT_GROUP)));
                notifySubscriber(url, listener, instances);
                subscribeEventListener(serviceName, url, listener);
            }
        }

    });
}
```

可以看到，服务订阅主要做了以下3个事：

1. 从注册中心获取所有实例；
2. 以获取到的所有实例，通知并刷新（初始化）服务目录；
3. 订阅服务变更通知。



### 接收服务变更通知

当注册中心的服务配置变更时，将通过 **NotifyListener#notify** 方法接口通知，而 **NacosDirectory** 实现了该接口：

```Java
public synchronized void notify(List<URL> urls) {
    //按照category分成configurators、routers、providers三类
    Map<String, List<URL>> categoryUrls = urls.stream()
            .filter(Objects::nonNull)
            .filter(this::isValidCategory)
            .filter(this::isNotCompatibleFor26x)
            .collect(Collectors.groupingBy(this::judgeCategory));
	//取configurators类型的URL，并转换成Configurator对象
    List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
    this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

    //获取routers类型的URL，并转成Router对象，添加到RouterChain中
    List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
    toRouters(routerURLs).ifPresent(this::addRouters);

    // 获取providers类型的URL，调用refreshOverrideAndInvoker方法进行处理
    List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
    /**
     * 3.x added for extend URL address
     */
    ExtensionLoader<AddressListener> addressListenerExtensionLoader = ExtensionLoader.getExtensionLoader(AddressListener.class);
    List<AddressListener> supportedListeners = addressListenerExtensionLoader.getActivateExtension(getUrl(), (String[]) null);
    if (supportedListeners != null && !supportedListeners.isEmpty()) {
        for (AddressListener addressListener : supportedListeners) {
            providerURLs = addressListener.notify(providerURLs, getConsumerUrl(), this);
        }
    }
    refreshOverrideAndInvoker(providerURLs);
}
```

在 **RegistryDirectory#notify** 方法中，首先会按照 category 将 URL 分成 configurators、routers、providers 三类，并分别对不同类型的 URL 进行处理：

1. 将 configurators 类型的 URL 转化为 Configurator，保存到 configurators 字段中；
2. 将 router 类型的 URL 转化为 Router，并添加到 routerChain ；
3. 将 provider 类型的 URL 通过refreshOverrideAndInvoker方法进行刷新。

**RegistryDirectory#refreshOverrideAndInvoker** ：

```Java
private synchronized void refreshOverrideAndInvoker(List<URL> urls) {
    // mock zookeeper://xxx?mock=return null
    overrideDirectoryUrl();
    refreshInvoker(urls);
}
```

这里主要看 **refreshInvoker** 方法：

```Java
private void refreshInvoker(List<URL> invokerUrls) {
    Assert.notNull(invokerUrls, "invokerUrls should not be null");
	//invokerUrls长度为1，并且协议为empty，则销毁所有invoker
    if (invokerUrls.size() == 1
            && invokerUrls.get(0) != null
            && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
        //标记为禁止访问
        this.forbidden = true; // Forbid to access
        //销毁所有invoker实例
        this.invokers = Collections.emptyList();
        routerChain.setInvokers(this.invokers);
        destroyAllInvokers(); // Close all invokers
    } else {
        Map<URL, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
        if (invokerUrls == Collections.<URL>emptyList()) {
            invokerUrls = new ArrayList<>();
        }
        //如果invokerUrls为空，并且cachedInvokerUrls不为空，则使用cachedInvokerUrls
        if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
            invokerUrls.addAll(this.cachedInvokerUrls);
        } else {
            //缓存invokerUrls
            this.cachedInvokerUrls = new HashSet<>();
            this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
        }
        if (invokerUrls.isEmpty()) {
            return;
        }
        this.forbidden = false; // Allow to access
        //将url转换为invoker实例
        Map<URL, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

        /**
         * If the calculation is wrong, it is not processed.
         *
         * 1. The protocol configured by the client is inconsistent with the protocol of the server.
         *    eg: consumer protocol = dubbo, provider only has other protocol services(rest).
         * 2. The registration center is not robust and pushes illegal specification data.
         *
         */
        if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
            logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                    .toString()));
            return;
        }

        //更新服务目录中的invoker列表
        List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
        // pre-route and build cache, notice that route cache should build on original Invoker list.
        // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
        routerChain.setInvokers(newInvokers);
        this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
        this.urlInvokerMap = newUrlInvokerMap;

        //销毁无用的invoker
        // Close the unused Invoker
        destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap);

    }

    // notify invokers refreshed
    this.invokersChanged();
}
```

其核心逻辑在于 **toInvokers** 方法，该方法用于将 invokerUrls 转换为 invoker实例集合：

```Java
private Map<URL, Invoker<T>> toInvokers(List<URL> urls) {
    Map<URL, Invoker<T>> newUrlInvokerMap = new ConcurrentHashMap<>();
    if (CollectionUtils.isEmpty(urls)) {
        return newUrlInvokerMap;
    }
    Set<URL> keys = new HashSet<>();
    //获取消费端支持的协议
    String queryProtocols = this.queryMap.get(PROTOCOL_KEY);
    for (URL providerUrl : urls) {
        // If protocol is configured at the reference side, only the matching protocol is selected
        if (queryProtocols != null && queryProtocols.length() > 0) {
            boolean accept = false;
            String[] acceptProtocols = queryProtocols.split(",");
            for (String acceptProtocol : acceptProtocols) {
                if (providerUrl.getProtocol().equals(acceptProtocol)) {
                    accept = true;
                    break;
                }
            }
            //是否支持消费端协议，不支持则忽略
            if (!accept) {
                continue;
            }
        }
        //忽略empty协议的URL
        if (EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
            continue;
        }
        //通过SPI的方式检测消费端是否存在对应的扩展实现
        if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
            logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() +
                    " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() +
                    " to consumer " + NetUtils.getLocalHost() + ", supported protocol: " +
                    ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
            continue;
        }
        //合并URL，按一定优先级合并配置
        URL url = mergeUrl(providerUrl);

        if (keys.contains(url)) { // Repeated url
            continue;
        }
        keys.add(url);
        // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
        Map<URL, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
        Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(url);
        if (invoker == null) { // Not in the cache, refer again
            try {
                boolean enabled = true;
                //根据URL参数决定是否创建invoker
                if (url.hasParameter(DISABLED_KEY)) {
                    enabled = !url.getParameter(DISABLED_KEY, false);
                } else {
                    enabled = url.getParameter(ENABLED_KEY, true);
                }
                //通过Protocol#refer方法创建invoker实例
                if (enabled) {
                    invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl);
                }
            } catch (Throwable t) {
                logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
            }
            if (invoker != null) { // Put new invoker in cache
                newUrlInvokerMap.put(url, invoker);
            }
        } else {
            newUrlInvokerMap.put(url, invoker);
        }
    }
    keys.clear();
    return newUrlInvokerMap;
}
```

总的来说， **refreshInvoker** 在刷新invoker列表过程中：

1. 校验 invokerUrls 中的 URL 协议是否为 “empty” ，是则代表该服务的实例数为0，此时将销毁所有已有的 invoker 实例，并将该服务标记为禁止访问；
2. 否则，则缓存 invokerUrls ，并将 invokerUrls 转换为 invoker 实例列表：
   1. 对 URL 进行检测，过滤消费端不支持的 URL ；
   2. 合并 URL 配置；
   3. 根据具体协议，通过Protocol#refer方法创建invoker实例；
3. 将转换后的 invoker 实例列表更新到服务目录的 invoker 实例列表；
4. 销毁旧的无用的 invoker 实例。



在创建 invoker 实例时，protocol 实例类型为自适应扩展实现类，而 url 协议类型为 dubbo ，可知最终使用的是 DubboProtocol实例，但通过 debug 会发现，会先经过多个 Protocol 的包装类（其中便包括用于构建Filter拦截器链的 ProtocolFilterWrapper 包装类）处理过后，最终才到 DubboProtocol ：

**AbstractProtocol#refer** ：

```java
@Override
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
}
```

这里有一个点值得注意的是，在使用 protocolBindingRefer 生成 DubboInvoker 实例后，会将 DubboInovker 实例包装为 **AsyncToSyncInvoker** 实例，实际上 Dubbo 的调用是天然被设计为异步的，而该 Invoker 实例的作用则是将异步结果转化同步。



**DubboProtocol#protocolBindingRefer** ：

```java
@Override
public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);

    // 创建dubbo invoker
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);

    return invoker;
}
```

**DubboProtocol#getClients** ：

```java
private ExchangeClient[] getClients(URL url) {
    // 是否共享连接
    int connections = url.getParameter(CONNECTIONS_KEY, 0);
    // if not configured, connection is shared, otherwise, one connection for one service
    if (connections == 0) {
        /*
         * The xml configuration should have a higher priority than properties.
         */
        String shareConnectionsStr = url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
        //获取连接数
        connections = Integer.parseInt(StringUtils.isBlank(shareConnectionsStr) ? ConfigUtils.getProperty(SHARE_CONNECTIONS_KEY,
                DEFAULT_SHARE_CONNECTIONS) : shareConnectionsStr);
        return getSharedClient(url, connections).toArray(new ExchangeClient[0]);
    } else {
        //初始化新的客户端
        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            clients[i] = initClient(url);
        }
        return clients;
    }

}
```

这里会先根据 connections 数量决定是获取共享客户端实例还是创建新的客户端实例，获取到客户端连接实例后，将其封装后创建 **DubboInvoker** 实例并返回。



### 总结

1. 本文简单介绍了服务目录的概念，以及**StaticDirectory**  和 **DynamicDirectory**  两种服务目录类型的区别；
2. 以 Nacos 作为注册中心为例，针对 **RegistryDirectory** 的源码作了一定的学习，包括服务的订阅过程、服务变更的通知处理流程等；
3. 服务变更后将刷新内部维护的 invoker 列表，将根据实际配置初始连接或使用共享连接。

 