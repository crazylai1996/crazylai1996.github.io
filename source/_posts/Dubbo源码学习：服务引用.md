---
title: Dubbo源码学习：服务引用
date: 2024-07-13 15:30:25
tags: Dubbo
---



在上一篇中我们学习了Dubbo服务导出的过程，在开发中，如果我们需要引用一个服务的话，只需要在成员或方法上标注**@DubboReference**注解即可，那它内部是如何实现的呢，我们一起来看下。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本

### @DubboReference注解的解析

在通过**@EnableDubbo**注解导入的**DubboComponentScanRegistrar**：

```Java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

        registerServiceClassPostProcessor(packagesToScan, registry);

        // @since 2.7.6 Register the common beans
        registerCommonBeans(registry);
    }
    //省略...
}
```

在**DubboBeanUtils#registerCommonBeans**中：

```Java
public static void registerCommonBeans(BeanDefinitionRegistry registry) {

    // Since 2.5.7 Register @Reference Annotation Bean Processor as an infrastructure Bean
    registerInfrastructureBean(registry, ReferenceAnnotationBeanPostProcessor.BEAN_NAME,
            ReferenceAnnotationBeanPostProcessor.class);

	//省略...
}
```

转到**ReferenceAnnotationBeanPostProcessor**，查看类继承关系图：

![image-20240713190631268](http://storage.laixiaoming.space/blog/image-20240713190631268.png)

可发现这是一个**BeanPostProcessor**的实现类，并实现了**InstantiationAwareBeanPostProcessor**和**MergedBeanDefinitionPostProcessor**，了解过@Autowired的注入原理的看到这里应该都会有一种熟悉感，回到其代码实现可以看到其父类**AbstractAnnotationBeanPostProcessor**实现了**postProcessPropertyValues**，不难看出，该方法也是**@DubboReference**解析的入口：

```Java
public PropertyValues postProcessPropertyValues(
        PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
    //解析注解元数据
    InjectionMetadata metadata = findInjectionMetadata(beanName, bean.getClass(), pvs);
    try {
        //注入
        metadata.inject(bean, beanName, pvs);
    } catch (BeanCreationException ex) {
        throw ex;
    } catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of @" + getAnnotationType().getSimpleName()
                + " dependencies is failed", ex);
    }
    return pvs;
}
```

**AbstractAnnotationBeanPostProcessor#findInjectionMetadata**主要是解析并获取成员或方法上的**@DubboReference**注解，封装并得到**AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata**类型对象：

```Java
private AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata buildAnnotatedMetadata(final Class<?> beanClass) {
    //解析要成员上的@DubboReference注解
    Collection<AbstractAnnotationBeanPostProcessor.AnnotatedFieldElement> fieldElements = findFieldAnnotationMetadata(beanClass);
    //解析要方法上的@DubboReference注解
    Collection<AbstractAnnotationBeanPostProcessor.AnnotatedMethodElement> methodElements = findAnnotatedMethodMetadata(beanClass);
    return new AbstractAnnotationBeanPostProcessor.AnnotatedInjectionMetadata(beanClass, fieldElements, methodElements);
}
```

然后统一调用**InjectionMetadata#inject**方法进行注入，继续追踪则来到了**ReferenceAnnotationBeanPostProcessor#doGetInjectedBean**：

```Java
protected Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                   InjectionMetadata.InjectedElement injectedElement) throws Exception {
    //生成ServiceBean名称
    /**
     * The name of bean that annotated Dubbo's {@link Service @Service} in local Spring {@link ApplicationContext}
     */
    String referencedBeanName = buildReferencedBeanName(attributes, injectedType);

    //生成ReferenceBean名称
    /**
     * The name of bean that is declared by {@link Reference @Reference} annotation injection
     */
    String referenceBeanName = getReferenceBeanName(attributes, injectedType);

    referencedBeanNameIdx.computeIfAbsent(referencedBeanName, k -> new TreeSet<String>()).add(referenceBeanName);

    //生成ReferenceBean
    ReferenceBean referenceBean = buildReferenceBeanIfAbsent(referenceBeanName, attributes, injectedType);

    //是否本地Service
    boolean localServiceBean = isLocalServiceBean(referencedBeanName, referenceBean, attributes);

    //确保待注入服务已经导出
    prepareReferenceBean(referencedBeanName, referenceBean, localServiceBean);

    //注册ReferenceBean到Spring容器
    registerReferenceBean(referencedBeanName, referenceBean, localServiceBean, referenceBeanName);

    cacheInjectedReferenceBean(referenceBean, injectedElement);

    //对ReferenceBean应用其他bean后置处理器
    return getBeanFactory().applyBeanPostProcessorsAfterInitialization(referenceBean.get(), referenceBeanName);
}
```

这里需要注意的是，**buildReferencedBeanName**生成的是ServiceBean（服务提供者）的名称，生成的规则遵循**ServiceBean:interfaceClassName:version:group**，用于后续判断待注入的服务是否可以由本地提供，而**getReferenceBeanName**生成的是待注入的bean的名称：

```Java
public String build() {
    StringBuilder beanNameBuilder = new StringBuilder("ServiceBean");
    // Required
    append(beanNameBuilder, interfaceClassName);
    // Optional
    append(beanNameBuilder, version);
    append(beanNameBuilder, group);
    // Build and remove last ":"
    String rawBeanName = beanNameBuilder.toString();
    // Resolve placeholders
    return environment.resolvePlaceholders(rawBeanName);
}
```

如果是待注入的服务是本地暴露的服务，直接从spring容器里获取，并以referenceBeanName为名注册一个别名：

```Java
private void registerReferenceBean(String referencedBeanName, ReferenceBean referenceBean,
                                   boolean localServiceBean, String beanName) {

    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	/是否本地ServiceBean
    if (localServiceBean) {  // If @Service bean is local one
        /**
         * Get  the @Service's BeanDefinition from {@link BeanFactory}
         * Refer to {@link ServiceClassPostProcessor#buildServiceBeanDefinition}
         */
        AbstractBeanDefinition beanDefinition = (AbstractBeanDefinition) beanFactory.getBeanDefinition(referencedBeanName);
        RuntimeBeanReference runtimeBeanReference = (RuntimeBeanReference) beanDefinition.getPropertyValues().get("ref");
        // The name of bean annotated @Service
        String serviceBeanName = runtimeBeanReference.getBeanName();
        // register Alias rather than a new bean name, in order to reduce duplicated beans
        beanFactory.registerAlias(serviceBeanName, beanName);
    } else { // Remote @Service Bean
        if (!beanFactory.containsBean(beanName)) {
            beanFactory.registerSingleton(beanName, referenceBean);
        }
    }
}
```

如果由远程提供服务的话，则注册ReferenceBean，后续通过**ReferenceConfig#get**方法获取实际的对象并调用其他的bean后置处理器。

### ReferenceBean的生成

我们注意到，**ReferenceBean**实现了**FactoryBean**接口，实际注册的bean由**getObject**方法提供，查看其实现可以知道**getObject**中也调用了**ReferenceConfig#get**方法：

```Java
public synchronized T get() {
    if (destroyed) {
        throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
    }
    if (ref == null) {
        init();
    }
    return ref;
}
```

这里判断接口代理对象是否存在，不存在则进行初始化。

来到**init**方法，我们重点关注下其中的**createProxy**方法：

```Java
private T createProxy(Map<String, String> map) {
    //是否本地服务引用
    if (shouldJvmRefer(map)) {
        URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
        invoker = REF_PROTOCOL.refer(interfaceClass, url);
        if (logger.isInfoEnabled()) {
            logger.info("Using injvm service " + interfaceClass.getName());
        }
    } else {
        urls.clear();
        //配置了url属性，可能是点对点调用，也可能是写的注册中心的url
        if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
            String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (StringUtils.isEmpty(url.getPath())) {
                        url = url.setPath(interfaceName);
                    }
                    //如果是registry协议，说明连接的是注册中心，就设置refer参数到url
                    if (UrlUtils.isRegistry(url)) {
                        urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        //参数合并
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else { // assemble URL from register center's configuration
            // if protocols not injvm checkRegistry
            if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                //加载注册中心url
                checkRegistry();
                List<URL> us = ConfigValidationUtils.loadRegistries(this, false);
                if (CollectionUtils.isNotEmpty(us)) {
                    for (URL u : us) {
                        URL monitorUrl = ConfigValidationUtils.loadMonitor(this, u);
                        if (monitorUrl != null) {
                            map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                        }
                        //添加refer参数到url中
                        urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                    }
                }
                if (urls.isEmpty()) {
                    throw new IllegalStateException(
                            "No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() +
                                    " use dubbo version " + Version.getVersion() +
                                    ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                }
            }
        }
		//单个注册中心或服务提供者
        if (urls.size() == 1) {
            invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
            //获取所有的 Invoker
            for (URL url : urls) {
                Invoker<?> referInvoker = REF_PROTOCOL.refer(interfaceClass, url);
                if (shouldCheck()) {
                    if (referInvoker.isAvailable()) {
                        invokers.add(referInvoker);
                    } else {
                        referInvoker.destroy();
                    }
                } else {
                    invokers.add(referInvoker);
                }

                if (UrlUtils.isRegistry(url)) {
                    registryURL = url; // use last registry url
                }
            }

            if (shouldCheck() && invokers.size() == 0) {
                throw new IllegalStateException("Failed to check the status of the service "
                        + interfaceName
                        + ". No provider available for the service "
                        + (group == null ? "" : group + "/")
                        + interfaceName +
                        (version == null ? "" : ":" + version)
                        + " from the multi registry cluster"
                        + " use dubbo version " + Version.getVersion());
            }

            if (registryURL != null) { // registry url is available
                //对有注册中心的Cluster 只用ZoneAwareCluster
                // for multi-subscription scenario, use 'zone-aware' policy by default
                String cluster = registryURL.getParameter(CLUSTER_KEY, ZoneAwareCluster.NAME);
                // The invoker wrap sequence would be: ZoneAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, routing happens here) -> Invoker
                invoker = Cluster.getCluster(cluster, false).join(new StaticDirectory(registryURL, invokers));
            } else { // not a registry url, must be direct invoke.
                String cluster = CollectionUtils.isNotEmpty(invokers)
                        ?
                        (invokers.get(0).getUrl() != null ? invokers.get(0).getUrl().getParameter(CLUSTER_KEY, ZoneAwareCluster.NAME) :
                                Cluster.DEFAULT)
                        : Cluster.DEFAULT;
                invoker = Cluster.getCluster(cluster).join(new StaticDirectory(invokers));
            }
        }
    }

    if (logger.isInfoEnabled()) {
        logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
    }

    URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
    MetadataUtils.publishServiceDefinition(consumerURL);

    //创建服务代理
    // create service proxy
    return (T) PROXY_FACTORY.getProxy(invoker, ProtocolUtils.isGeneric(generic));
}
```

这个方法主要是根据本地调用或远程调用创建不同的Invoker实例，然后创建服务代理对象。



#### 本地服务调用Invoker

本地调用时，URL协议为injvm，Protocol实现类为**InjvmProtocol**，其refer方法由AbstractProtocol提供：

```Java
public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
    return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
}
```

**InjvmProtocol#protocolBindingRefer**：

```Java
public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
    return new InjvmInvoker<T>(serviceType, url, url.getServiceKey(), exporterMap);
}
```

这里得到**InjvmInvoker**实例后，又将其包装成**AsyncToSyncInvoker**返回。



#### 远程服务调用Invoker

在只存在一个注册中心的情况下，从传入的Url为***registry://***，可知由**InterfaceCompatibleRegistryProtocol**提供实现，**InterfaceCompatibleRegistryProtocol**继承自**RegistryProtocol**，查看**InterfaceCompatibleRegistryProtocol**可以发现，**InterfaceCompatibleRegistryProtocol**没有重写refer方法，而是重写了一些getInoker方法：

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



#### 创建服务代理类

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



### 总结

1. **@DubboReference**的注解由**ReferenceAnnotationBeanPostProcessor**解析并注入，注入的对象是通过**ReferenceBean#get**根据实际情况生成的Invoker实例；
2. **ReferenceBean#get**获取到的对象是代理过的Invoker实例，而Invoker实例由**Protocol#refer**得到：
   1. 若是，则调用**InjvmProtocol**的refer方法生成Invoker实例；
   2. 否则读取直连配置项或注册中心url，并将读取到的url存储到urls中，此时Invoker由**InterfaceCompatibleRegistryProtocol**生成：
      1. 若urls元素数量为1，则直接通过Protocol自适应拓展类构建 Invoker实例接口；
      2. 若urls元素数量大于1，即存在多个注册中心或服务直连url，此时先根据url构建Invoker，然后再通过Cluster合并多个Invoker，

3. 生成对应的Invoker实例后，再创建服务代理实例。

