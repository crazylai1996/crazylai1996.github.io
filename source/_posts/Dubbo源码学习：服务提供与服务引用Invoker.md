---
title: Dubbo源码学习：服务提供与服务引用Invoker
date: 2024-08-20 23:16:43
tags: Dubbo
---

### 前言

在Dubbo中，Invoker无处不在，它是Dubbo中非常重要的一个模型，不管在服务提供端或者服务引用端均有它的身影。

> Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本

### 服务提供 Invoker

在服务提供方中，Invoker是由**ProxyFactory**创建的，Dubbo默认使用的ProxyFactory实现类为**JavassistProxyFactory**：

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

可以看到**JavassistProxyFactory**首先创建了Wrapper类，然后重写了**AbstractProxyInvoker**的**doInvoke**方法，并将调用转发给了Wrapper实例。我们直接看下Wrapper类是如何创建的，深入可发现其关键方法在于**Wrapper#makeWrapper**，这里我们直接通过arthas工具反编译生成的Wrapper类的内容：

```Java
public class Wrapper1
extends Wrapper
implements ClassGenerator.DC {
    //公有字段、getter、setter字段名
    public static String[] pns;
    //公有字段、getter、setter字段名及类型
    public static Map pts;
    //所有方法名称
    public static String[] mns;
    //当前类方法名称
    public static String[] dmns;
    public static Class[] mts0;

    @Override
    public String[] getPropertyNames() {
        return pns;
    }

    @Override
    public boolean hasProperty(String string) {
        return pts.containsKey(string);
    }

    public Class getPropertyType(String string) {
        return (Class)pts.get(string);
    }

    @Override
    public String[] getMethodNames() {
        return mns;
    }

    @Override
    public String[] getDeclaredMethodNames() {
        return dmns;
    }

    @Override
    public void setPropertyValue(Object object, String string, Object object2) {
        try {
            HelloServiceImpl helloServiceImpl = (HelloServiceImpl)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        if (string.equals("foo")) {
            helloServiceImpl.foo = (String)object2;
            return;
        }
        throw new NoSuchPropertyException(new StringBuffer().append("Not found property \"").append(string).append("\" field or setter method in class laixiaoming.service.HelloServiceImpl.").toString());
    }

    @Override
    public Object getPropertyValue(Object object, String string) {
        HelloServiceImpl helloServiceImpl;
        try {
            helloServiceImpl = (HelloServiceImpl)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        if (string.equals("foo")) {
            return helloServiceImpl.foo;
        }
        throw new NoSuchPropertyException(new StringBuffer().append("Not found property \"").append(string).append("\" field or getter method in class laixiaoming.service.HelloServiceImpl.").toString());
    }

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

查看其实现可以知道，服务提供类的Invoker主要提供3个方法，其中**setPropertyValue**用于设置服务提供类的属性值，**getPropertyValue**用于获取服务提供类的属性值，**invokeMethod**方则用于调用服务提供类的方法。

### 服务引用 Invoker

在服务引用过程中，Invoker由Protocol的**refer**方法创建而来，用于发起远程调用，在只存在一个注册中心的情况下，从传入的Url为***registry://***，可知由**InterfaceCompatibleRegistryProtocol**提供实现，**InterfaceCompatibleRegistryProtocol**继承自**RegistryProtocol**，查看**InterfaceCompatibleRegistryProtocol**可以发现，**InterfaceCompatibleRegistryProtocol**没有重写refer方法，而是重写了一些getInoker方法：

```Java
    @Override
    public <T> ClusterInvoker<T> getInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
        DynamicDirectory<T> directory = new RegistryDirectory<>(type, url);
        return doCreateInvoker(directory, cluster, registry, type);
    }

    @Override
    public <T> ClusterInvoker<T> getServiceDiscoveryInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
        try {
            registry = registryFactory.getRegistry(super.getRegistryUrl(url));
        } catch (IllegalStateException e) {
            String protocol = url.getProtocol();
            logger.warn(protocol + " do not support service discovery, automatically switch to interface-level service discovery.");
            registry = AbstractRegistryFactory.getDefaultNopRegistryIfNotSupportServiceDiscovery();
        }

        DynamicDirectory<T> directory = new ServiceDiscoveryRegistryDirectory<>(type, url);
        return doCreateInvoker(directory, cluster, registry, type);
    }

    @Override
    protected <T> ClusterInvoker<T> getMigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry,
                                                        Class<T> type, URL url, URL consumerUrl) {
//        ClusterInvoker<T> invoker = getInvoker(cluster, registry, type, url);
        return new MigrationInvoker<T>(registryProtocol, cluster, registry, type, url, consumerUrl);
    }
```

而以上这3种Invoker，则分别用于：

1. getInvoker为接口级Invoker;
2. getServiceDiscoveryInvoker为应用级Invoker；
3. getMigrationInvoker则同时兼容了接口级及应用级Invoker；

对于接口级Invoker及应用级的Invoker，在创建对应的服务目录Directory后，都是由**RegistryProtocol#doCreateInvoker**创建而来：

```
protected <T> ClusterInvoker<T> doCreateInvoker(DynamicDirectory<T> directory, Cluster cluster, Registry registry, Class<T> type) {
	//设置注册中心和协议
    directory.setRegistry(registry);
    directory.setProtocol(protocol);
    // all attributes of REFER_KEY
    Map<String, String> parameters = new HashMap<String, String>(directory.getConsumerUrl().getParameters());
    //生成服务消费者链接
    URL urlToRegistry = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
    if (directory.isShouldRegister()) {
        directory.setRegisteredConsumerUrl(urlToRegistry);
        registry.register(directory.getRegisteredConsumerUrl());
    }
    directory.buildRouterChain(urlToRegistry);
    //服务订阅
    directory.subscribe(toSubscribeUrl(urlToRegistry));
    //生成Invoker
    return (ClusterInvoker<T>) cluster.join(directory);
}
```

回到**RegistryProtocol#refer**方法：

```Java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
	//转换协议头，比如nacos://
    url = getRegistryUrl(url);
    //获取注册中心实例
    Registry registry = getRegistry(url);
    if (RegistryService.class.equals(type)) {
        return proxyFactory.getInvoker((T) registry, type, url);
    }

	//将url查询字符串转化为map
    // group="a,b" or group="*"
    Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
    String group = qs.get(GROUP_KEY);
    if (group != null && group.length() > 0) {
        if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
            return doRefer(Cluster.getCluster(MergeableCluster.NAME), registry, type, url, qs);
        }
    }
	//获取集群容错模式，默认failover，此处经过包装为MockClusterWrapper
    Cluster cluster = Cluster.getCluster(qs.get(CLUSTER_KEY));
    return doRefer(cluster, registry, type, url, qs);
}

    protected <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url, Map<String, String> parameters) {
        URL consumerUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        //生成MigrationInvoker
        ClusterInvoker<T> migrationInvoker = getMigrationInvoker(this, cluster, registry, type, url, consumerUrl);
        return interceptInvoker(migrationInvoker, url, consumerUrl);
    }

```

可以看到这里实际创建的是MigrationInvoker，在得到MigrationInvoker后，一路跟踪来到**MigrationRuleListener#onRefer**：

```
@Override
public synchronized void onRefer(RegistryProtocol registryProtocol, ClusterInvoker<?> invoker, URL url) {
    MigrationInvoker<?> migrationInvoker = (MigrationInvoker<?>) invoker;

    MigrationRuleHandler<?> migrationListener = new MigrationRuleHandler<>(migrationInvoker);
    listeners.add(migrationListener);

    migrationListener.doMigrate(rawRule);
}
```

传入规则信息，进入到**MigrationRuleHandler#doMigrate**：

```Java
public void doMigrate(String rawRule) {
    MigrationRule rule = MigrationRule.parse(rawRule);

    if (null != currentStep && currentStep.equals(rule.getStep())) {
        if (logger.isInfoEnabled()) {
            logger.info("Migration step is not change. rule.getStep is " + currentStep.name());
        }
        return;
    } else {
        currentStep = rule.getStep();
    }

    migrationInvoker.setMigrationRule(rule);

    if (migrationInvoker.isMigrationMultiRegistry()) {
        if (migrationInvoker.isServiceInvoker()) {
            migrationInvoker.refreshServiceDiscoveryInvoker();
        } else {
            migrationInvoker.refreshInterfaceInvoker();
        }
    } else {
        switch (rule.getStep()) {
            //应用级优先
            case APPLICATION_FIRST:
                migrationInvoker.migrateToServiceDiscoveryInvoker(false);
                break;
            //应用级
            case FORCE_APPLICATION:
                migrationInvoker.migrateToServiceDiscoveryInvoker(true);
                break;
            //接口级
            case FORCE_INTERFACE:
            default:
                migrationInvoker.fallbackToInterfaceInvoker();
        }
    }
}
```

这里根据不同规则，来到**MigrationRuleHandler#migrateToServiceDiscoveryInvoker**，刷新创建对应的接口级或应用级Invoker:

```Java
public synchronized void migrateToServiceDiscoveryInvoker(boolean forceMigrate) {
    if (!forceMigrate) {
   		//应用级及接口级服务发现双订阅
        refreshServiceDiscoveryInvoker();
        refreshInterfaceInvoker();
        setListener(invoker, () -> {
            this.compareAddresses(serviceDiscoveryInvoker, invoker);
        });
        setListener(serviceDiscoveryInvoker, () -> {
            this.compareAddresses(serviceDiscoveryInvoker, invoker);
        });
    } else {
    	//应用级服务发现
        refreshServiceDiscoveryInvoker();
        setListener(serviceDiscoveryInvoker, () -> {
            this.destroyInterfaceInvoker(this.invoker);
        });
    }
}
```

Invoker 创建完毕后，最后通过**ProxyFactory #getProxy**创建服务代理：

```Java
    private static final ProxyFactory PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
	//省略...
    return (T) PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic));

```

这里的PROXY_FACTORY是一个自适应扩展点，根据url的proxy参数使用使用不同的代理实现，默认使用**JavassistProxyFactory**：

```Java
@Override
@SuppressWarnings("unchecked")
public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
    return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}
```

那最终生成的代理类是什么样的呢，这里通过arthas反编译得到：

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

可以看到，对Dubbo接口的调用，都会转发给**InvokerInvocationHandler**，最终调用到对应的invoker。