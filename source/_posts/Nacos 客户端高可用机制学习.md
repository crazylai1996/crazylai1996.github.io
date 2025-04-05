---
title: Nacos 客户端高可用机制学习
date: 2025-04-04 22:07:03
tags: Nacos
---

### 前言

在微服务架构中，服务注册和发现是核心组件之一，而 Nacos 能作为目前主流的注册中心，不仅仅是因为其功能特性、开源、成熟度等，更重要的是它在可用性方面所作的保障，确保了系统能在复杂的场景下稳定运行。本文就 Nacos 在客户端实现的一系列高可用机制，并结合相关源码一起学习下。

<!--more-->

> 以下内容基于 nacos-client 1.4.2版本



### 本地缓存

先来思考一个问题，假如我们实际使用 Dubbo 作为微服务框架，服务在运行过程中，Nacos 服务端宕机了，此时 Dubbo 服务还能正常调用其他服务吗？这个问题大多数人都能回答上来，答案是“可以”，因为 Dubbo 的消费端实际缓存了一份服务提供者列表，而只有当服务提供者列表发生上下线时，才会推送到服务消费端进行更新，所以此时服务依然能正常调用。但如果此时服务发生了重启呢，服务还能正常调用吗？答案也是可以，这就涉及到 Nacos 客户端的缓存机制了。

**ServiceInfo** 是 Nacos 定义的服务注册信息类，客户端获取到的服务列表通过一个 serviceInfoMap 在内存进行维护：

```Java
// com.alibaba.nacos.client.naming.core.HostReactor
private final Map<String, ServiceInfo> serviceInfoMap;
```



#### 初始化

在客户端初始化时，如果启用了“加载缓存文件”开关（loadCacheAtStart），则从磁盘读取指定的缓存目录，并初始化内存中的 serviceInfoMap ，该开关可以通过传入 namingLoadCacheAtStart 配置进行指定：

```Java
// com.alibaba.nacos.client.naming.core.HostReactor
public HostReactor(NamingProxy serverProxy, BeatReactor beatReactor, String cacheDir, boolean loadCacheAtStart,
        boolean pushEmptyProtection, int pollingThreadCount) {
    // init executorService
    //...
    
    this.beatReactor = beatReactor;
    this.serverProxy = serverProxy;
    this.cacheDir = cacheDir;
    // 是否加载磁盘缓存
    if (loadCacheAtStart) {
        this.serviceInfoMap = new ConcurrentHashMap<String, ServiceInfo>(DiskCache.read(this.cacheDir));
    } else {
        this.serviceInfoMap = new ConcurrentHashMap<String, ServiceInfo>(16);
    }
    //...
}
```



#### 更新

在 Nacos 中，客户端通过 “主动查询“ + “被动通知”两种方式感知到服务列表的变化，在变化时会同时更新 serviceInfoMap 。

主动查询通过定时请求服务端的方式实现：

```Java
// com.alibaba.nacos.client.naming.core.HostReactor.UpdateTask
@Override
public void run() {
    // 查询间隔为 1s
    long delayTime = DEFAULT_DELAY;
    
    try {
        ServiceInfo serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));
        // 缓存中的服务为空时，更新一次
        if (serviceObj == null) {
            updateService(serviceName, clusters);
            return;
        }
        
        // 缓存的时间小于上次更新时间时
        if (serviceObj.getLastRefTime() <= lastRefTime) {
            updateService(serviceName, clusters);
            serviceObj = serviceInfoMap.get(ServiceInfo.getKey(serviceName, clusters));
        } else {
            // if serviceName already updated by push, we should not override it
            // since the push data may be different from pull through force push
            refreshOnly(serviceName, clusters);
        }
        
        lastRefTime = serviceObj.getLastRefTime();
        
        //...
    } catch (Throwable e) {
        incFailCount();
        NAMING_LOGGER.warn("[NA] failed to update serviceName: " + serviceName, e);
    } finally {
        executor.schedule(this, Math.min(delayTime << failCount, DEFAULT_DELAY * 60), TimeUnit.MILLISECONDS);
    }
}
```



被动通知则通过 udp 推送实现：

```Java
// com.alibaba.nacos.client.naming.core.PushReceiver
@Override
public void run() {
    while (!closed) {
        try {
            
            // byte[] is initialized with 0 full filled by default
            byte[] buffer = new byte[UDP_MSS];
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
            // 接收到服务端的通知时
            udpSocket.receive(packet);
            
            String json = new String(IoUtils.tryDecompress(packet.getData()), UTF_8).trim();
            NAMING_LOGGER.info("received push data: " + json + " from " + packet.getAddress().toString());
            
            PushPacket pushPacket = JacksonUtils.toObj(json, PushPacket.class);
            String ack;
            if ("dom".equals(pushPacket.type) || "service".equals(pushPacket.type)) {
                hostReactor.processServiceJson(pushPacket.data);
                //...
            } else if ("dump".equals(pushPacket.type)) {
                //...
            } else {
                //...
            }
            
            udpSocket.send(new DatagramPacket(ack.getBytes(UTF_8), ack.getBytes(UTF_8).length,
                    packet.getSocketAddress()));
        } catch (Exception e) {
            if (closed) {
                return;
            }
            NAMING_LOGGER.error("[NA] error while receiving push data", e);
        }
    }
}
```



不管是 “主动查询“，还是“被动通知”，最终都是通过 **com.alibaba.nacos.client.naming.core.HostReactor#processServiceJson** 进行处理，更新服务列表缓存 serviceInfoMap ：

```Java
public ServiceInfo processServiceJson(String json) {
    ServiceInfo serviceInfo = JacksonUtils.toObj(json, ServiceInfo.class);
    String serviceKey = serviceInfo.getKey();
    if (serviceKey == null) {
        return null;
    }
    ServiceInfo oldService = serviceInfoMap.get(serviceKey);
    
    if (pushEmptyProtection && !serviceInfo.validate()) {
        //empty or error push, just ignore
        return oldService;
    }
    
    boolean changed = false;
    
    if (oldService != null) {
        
        //...
        // 更新服务列表缓存
        serviceInfoMap.put(serviceInfo.getKey(), serviceInfo);
        
        // 判断新获取的服务信息较于缓存中的服务信息是否产生了变化
        Map<String, Instance> oldHostMap = new HashMap<String, Instance>(oldService.getHosts().size());
        for (Instance host : oldService.getHosts()) {
            oldHostMap.put(host.toInetAddr(), host);
        }
        
        Map<String, Instance> newHostMap = new HashMap<String, Instance>(serviceInfo.getHosts().size());
        for (Instance host : serviceInfo.getHosts()) {
            newHostMap.put(host.toInetAddr(), host);
        }
        
        Set<Instance> modHosts = new HashSet<Instance>();
        Set<Instance> newHosts = new HashSet<Instance>();
        Set<Instance> remvHosts = new HashSet<Instance>();
        
        List<Map.Entry<String, Instance>> newServiceHosts = new ArrayList<Map.Entry<String, Instance>>(
                newHostMap.entrySet());
        for (Map.Entry<String, Instance> entry : newServiceHosts) {
            Instance host = entry.getValue();
            String key = entry.getKey();
            if (oldHostMap.containsKey(key) && !StringUtils
                    .equals(host.toString(), oldHostMap.get(key).toString())) {
                modHosts.add(host);
                continue;
            }
            
            if (!oldHostMap.containsKey(key)) {
                newHosts.add(host);
            }
        }
        
        for (Map.Entry<String, Instance> entry : oldHostMap.entrySet()) {
            Instance host = entry.getValue();
            String key = entry.getKey();
            if (newHostMap.containsKey(key)) {
                continue;
            }
            
            if (!newHostMap.containsKey(key)) {
                remvHosts.add(host);
            }
            
        }
        
        //...
        
        serviceInfo.setJsonFromServer(json);
        
        if (newHosts.size() > 0 || remvHosts.size() > 0 || modHosts.size() > 0) {
            // 推送服务变更事件
            NotifyCenter.publishEvent(new InstancesChangeEvent(serviceInfo.getName(), serviceInfo.getGroupName(),
                    serviceInfo.getClusters(), serviceInfo.getHosts()));
            // 将缓存持久化一份到磁盘
            DiskCache.write(serviceInfo, cacheDir);
        }
        
    } else {
        changed = true;
        NAMING_LOGGER.info("init new ips(" + serviceInfo.ipCount() + ") service: " + serviceInfo.getKey() + " -> "
                + JacksonUtils.toJson(serviceInfo.getHosts()));
        serviceInfoMap.put(serviceInfo.getKey(), serviceInfo);
        // 推送服务变更事件
        NotifyCenter.publishEvent(new InstancesChangeEvent(serviceInfo.getName(), serviceInfo.getGroupName(),
                serviceInfo.getClusters(), serviceInfo.getHosts()));
        serviceInfo.setJsonFromServer(json);
        // 将缓存持久化一份到磁盘
        DiskCache.write(serviceInfo, cacheDir);
    }
    
    //...
    
    return serviceInfo;
}
```



#### 持久化

在 processServiceJson 方法中，Nacos 客户端在更新的 serviceInfoMap 服务缓存时，会同时 **com.alibaba.nacos.client.naming.cache.DiskCache#write** 持久化一份到本地文件，其默认的存储路径为：{user.home}/nacos/ ，可以通过 "JM.SNAPSHOT.PATH" 参数指定。



### failover 容灾

另外，我们注意到，除了缓存的持久化目录（cacheDir），同时存在于一个 "{cacheDir}/failover" 目录，存入的同样是服务列表文件信息，这是为什么呢？这其实时 Nacos 客户端的另一机制，用于本地容灾处理。

容灾一般有两种使用场景：

1. 在 Nacos 服务端发布的时候，可以打开容灾开关，Nacos 客户端将只使用本地容灾数据，这样 Nacos 服务的数据抖动或者数据错误都不会影响客户端，我们可以在 Nacos 服务端升级完成并且数据验证没问题之后再关闭容灾；
2. 在 Nacos 运行期间，如果出现接口不可用或者数据异常时，可以打开容灾开关，减少服务受影响的窗口，等 Nacos 服务端恢复后再关闭容灾。

在 Nacos 客户端中，容灾的处理逻辑集中在 **FailoverReactor** 类，Nacos 客户端的所有查询调用会先经过 FailoverReactor ，如果容灾开关打开，将直接使用容灾数据：

```Java
// com.alibaba.nacos.client.naming.core.HostReactor#getServiceInfo
public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {
    
    // 是不开启容灾，是的话则使用容灾数据
    String key = ServiceInfo.getKey(serviceName, clusters);
    if (failoverReactor.isFailoverSwitch()) {
        return failoverReactor.getService(key);
    }
    
    //...
}
```



FailoverReactor 在初始化时，会开启 2 个定时任务：

```Java
public void init() {
    
    // 开关检测，在打开开关的情况下，将从磁盘加载服务数据到内存，每 5 秒执行
    executorService.scheduleWithFixedDelay(new SwitchRefresher(), 0L, 5000L, TimeUnit.MILLISECONDS);
    
    // 备份服务数据到磁盘, 初始化延迟 30 分钟, 每 24 小时执行一次
    executorService.scheduleWithFixedDelay(new DiskFileWriter(), 30, DAY_PERIOD_MINUTES, TimeUnit.MINUTES);
    
    // backup file on startup if failover directory is empty.
    // 启动后备份一次服务数据，延迟 10 秒
    executorService.schedule(new Runnable() {
        @Override
        public void run() {
            try {
                File cacheDir = new File(failoverDir);
                
                if (!cacheDir.exists() && !cacheDir.mkdirs()) {
                    throw new IllegalStateException("failed to create cache dir: " + failoverDir);
                }
                
                File[] files = cacheDir.listFiles();
                if (files == null || files.length <= 0) {
                    new DiskFileWriter().run();
                }
            } catch (Throwable e) {
                NAMING_LOGGER.error("[NA] failed to backup file on startup.", e);
            }
            
        }
    }, 10000L, TimeUnit.MILLISECONDS);
}
```



容灾开关检测任务：

```Java
// com.alibaba.nacos.client.naming.backups.FailoverReactor.SwitchRefresher
class SwitchRefresher implements Runnable {
    
    long lastModifiedMillis = 0L;
    
    @Override
    public void run() {
        try {
            // 开关文件是否存在，文件名为 “00-00---000-VIPSRV_FAILOVER_SWITCH-000---00-00”
            File switchFile = new File(failoverDir + UtilAndComs.FAILOVER_SWITCH);
            if (!switchFile.exists()) {
                switchParams.put("failover-mode", "false");
                NAMING_LOGGER.debug("failover switch is not found, " + switchFile.getName());
                return;
            }
            
            // 检查开关文件是否变更
            long modified = switchFile.lastModified();
            
            if (lastModifiedMillis < modified) {
                lastModifiedMillis = modified;
                String failover = ConcurrentDiskUtil.getFileContent(failoverDir + UtilAndComs.FAILOVER_SWITCH,
                        Charset.defaultCharset().toString());
                if (!StringUtils.isEmpty(failover)) {
                    String[] lines = failover.split(DiskCache.getLineSeparator());
                    
                    for (String line : lines) {
                        String line1 = line.trim();
                        // 开关文件内容为 “1”时，则为打开，此时加载磁盘服务数据到内存
                        if ("1".equals(line1)) {
                            switchParams.put("failover-mode", "true");
                            NAMING_LOGGER.info("failover-mode is on");
                            new FailoverFileReader().run();
                        } else if ("0".equals(line1)) {
                            switchParams.put("failover-mode", "false");
                            NAMING_LOGGER.info("failover-mode is off");
                        }
                    }
                } else {
                    switchParams.put("failover-mode", "false");
                }
            }
            
        } catch (Throwable e) {
            NAMING_LOGGER.error("[NA] failed to read failover switch.", e);
        }
    }
}
```

该任务主要用于检测容灾开关是否打开，检测的方法是检查容灾目录下是否存在文件名为“00-00---000-VIPSRV_FAILOVER_SWITCH-000---00-00” 的开关文件，存在且文件内容为“1”时，则认为是打开状态，同时加载容灾目录下的服务数据到内存。



#### 客户端重试

当 Nacos 客户端与服务端通信失败时，会进行多次重试。

```Java
public String reqApi(String api, Map<String, String> params, Map<String, String> body, List<String> servers,
        String method) throws NacosException {
    
    params.put(CommonParams.NAMESPACE_ID, getNamespaceId());
    
    if (CollectionUtils.isEmpty(servers) && StringUtils.isBlank(nacosDomain)) {
        throw new NacosException(NacosException.INVALID_PARAM, "no server available");
    }
    
    NacosException exception = new NacosException();
    // 当只配置了一个 Nacos 服务端地址时，进行一定次数的重试
    if (StringUtils.isNotBlank(nacosDomain)) {
        for (int i = 0; i < maxRetry; i++) {
            try {
                return callServer(api, params, body, nacosDomain, method);
            } catch (NacosException e) {
                exception = e;
                if (NAMING_LOGGER.isDebugEnabled()) {
                    NAMING_LOGGER.debug("request {} failed.", nacosDomain, e);
                }
            }
        }
    } else {
        // 当有多个 Nacos 服务端地址时，对不同地址进行重试
        Random random = new Random(System.currentTimeMillis());
        int index = random.nextInt(servers.size());
        
        for (int i = 0; i < servers.size(); i++) {
            String server = servers.get(index);
            try {
                return callServer(api, params, body, server, method);
            } catch (NacosException e) {
                exception = e;
                if (NAMING_LOGGER.isDebugEnabled()) {
                    NAMING_LOGGER.debug("request {} failed.", server, e);
                }
            }
            index = (index + 1) % servers.size();
        }
    }
    
    //...
    
}
```

重试分为两种情况：

1. 当只配置了一个 Nacos 服务端地址时，进行一定次数的重试；
2. 当有多个 Nacos 服务端地址时，对不同地址进行重试。



### 总结

本文针对 Nacos 客户端如何保障高可用进行了简单的总结学习。总的来说，Nacos 客户端通过多种机制（如本地缓存、failover 容灾、重试等）来保障高可用性。这些机制共同作用，确保了在 Nacos 服务端出现故障或网络异常时，客户端仍能正常运行并提供服务。