---
title: Dubbo源码学习：服务导出
date: 2024-07-06 14:49:43
tags: Dubbo
---

我们都知道，在Dubbo微服务项目开发中，只需要通过**@EnableDubbo**和**@DubboService**注解就可以把服务注册到注册中心，那整个过程是怎么发生的呢？

<!--more-->

> 以下内容基于Dubbo 2.7.12版本



### 从@EnableDubbo说起

```Java
//省略...
@EnableDubboConfig
@DubboComponentScan
public @interface EnableDubbo {
	//省略...
}
```

**@EnableDubbo**主要包含了**@EnableDubboConfig**和**@DubboComponentScan**，我们重点看下**@DubboComponentScan**：

```Java
//省略...
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {
	//省略...
}
```

这是一个熟悉的**@Import**注解，它导入了一个**DubboComponentScanRegistrar**类：

```Java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		//获取元数据扫描路径
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
		//注册ServiceClassPostProcessor
        registerServiceClassPostProcessor(packagesToScan, registry);

		//注册通用bean
        // @since 2.7.6 Register the common beans
        registerCommonBeans(registry);
    }

    private void registerServiceClassPostProcessor(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceClassPostProcessor.class);
        builder.addConstructorArgValue(packagesToScan);
        builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
        BeanDefinitionReaderUtils.registerWithGeneratedName(beanDefinition, registry);

    }
    //省略...
}
```

可以看到，**DubboComponentScanRegistrar**最终又将**ServiceClassPostProcessor**注册到了IOC容器：

![image-20240630164051353](http://storage.laixiaoming.space/blog/image-20240630164051353.png)

**ServiceClassPostProcessor**实现了**BeanDefinitionRegistryPostProcessor**接口，可以通过实现**postProcessBeanDefinitionRegistry**新增bean定义，并将其注册到IOC容器：

```Java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

    //注册一个监听器
    registerInfrastructureBean(registry, DubboBootstrapApplicationListener.BEAN_NAME, DubboBootstrapApplicationListener.class);

    //获取扫描路径
    Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

    if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
        //注册ServiceBean
        registerServiceBeans(resolvedPackagesToScan, registry);
    } else {
        //省略...
    }

}
```

来到**registerServiceBeans**：

```Java
private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

    DubboClassPathBeanDefinitionScanner scanner =
            new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

    BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

    scanner.setBeanNameGenerator(beanNameGenerator);

    // 扫描@DubboService、@Service、@com.alibaba.dubbo.config.annotation.Service注解
    serviceAnnotationTypes.forEach(annotationType -> {
        scanner.addIncludeFilter(new AnnotationTypeFilter(annotationType));
    });

    for (String packageToScan : packagesToScan) {

        // Registers @Service Bean first
        scanner.scan(packageToScan);

        // 扫描并获取所有满足条件的BeanDefinition
        Set<BeanDefinitionHolder> beanDefinitionHolders =
                findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

        if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

            for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                //注册ServiceBean
                registerServiceBean(beanDefinitionHolder, registry, scanner);
            }
			//省略...

        } else {
			//省略...
        }

    }

}
```

至此，我们知道了，Dubbo启动后会通过扫描**@DubboService**等注解，并注册**ServiceBean**到IOC容器，除此之外，未发现Dubbo服务启动和注册相关的逻辑。

![ServiceBean类继承关系图](http://storage.laixiaoming.space/blog/image-20240630212039912.png)

这里有一个值得注意的点是，在**AbstractConfig**类中，有一个**addIntoConfigManager**初始化方法，会将当前配置类添加到全局的配置管理器：

```Java
@PostConstruct
public void addIntoConfigManager() {
    ApplicationModel.getConfigManager().addConfig(this);
}
```

回到**postProcessBeanDefinitionRegistry**，我们还发现它注册了一个**DubboBootstrapApplicationListener**监听器，查看其实现：

```Java
    @Override
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

    private void onContextRefreshedEvent(ContextRefreshedEvent event) {
        dubboBootstrap.start();
    }

```

可以看出，Dubbo服务的启动是在 Spring 容器发布刷新事件后，由**DubboBootstrapApplicationListener**监听器触发的。

### 服务导出

来到**DubboBootstrap#start**方法：

```Java
public DubboBootstrap start() {
    //乐观锁，避免重复启动
    if (started.compareAndSet(false, true)) {
        destroyed.set(false);
        ready.set(false);
        //初始化
        initialize();
        if (logger.isInfoEnabled()) {
            logger.info(NAME + " is starting...");
        }
        // 服务导出
        exportServices();

        // Not only provider register
        if (!isOnlyRegisterProvider() || hasExportedServices()) {
            //导出元数据服务
            exportMetadataService();
            //注册本地服务实例
            registerServiceInstance();
        }

        referServices();
		//省略...
    return this;
}
```

我们重点看下服务的导出，**DubboBootstrap#exportServices**方法：

```Java
private void exportServices() {
    //遍历需要导出的服务
    configManager.getServices().forEach(sc -> {
        // TODO, compatible with ServiceConfig.export()
        ServiceConfig serviceConfig = (ServiceConfig) sc;yn
        serviceConfig.setBootstrap(this);

        //异步导出
        if (exportAsync) {
            ExecutorService executor = executorRepository.getServiceExporterExecutor();
            Future<?> future = executor.submit(() -> {
                try {
                    exportService(serviceConfig);
                } catch (Throwable t) {
                    logger.error("export async catch error : " + t.getMessage(), t);
                }
            });
            asyncExportingFutures.add(future);
        } else {
            exportService(serviceConfig);
        }
    });
}
```

来到**ServiceConfig#export**方法：

```Java
public synchronized void export() {
    //检查bootstrap是否初始化
    if (bootstrap == null) {
        bootstrap = DubboBootstrap.getInstance();
        // compatible with api call.
        if (null != this.getRegistry()) {
            bootstrap.registries(this.getRegistries());
        }
        bootstrap.initialize();
    }
	//检查相关配置
    checkAndUpdateSubConfigs();

    //初始化元数据
    initServiceMetadata(provider);
    serviceMetadata.setServiceType(getInterfaceClass());
    serviceMetadata.setTarget(getRef());
    serviceMetadata.generateServiceKey();

    //是否导出
    if (!shouldExport()) {
        return;
    }

    //是否延迟导出
    if (shouldDelay()) {
        DELAY_EXPORT_EXECUTOR.schedule(() -> {
            try {
                // Delay export server should print stack trace if there are exception occur.
                this.doExport();
            } catch (Exception e) {
                logger.error("delay export server occur exception, please check it.", e);
            }
        }, getDelay(), TimeUnit.MILLISECONDS);
    } else {
        doExport();
    }

    exported();
}
```

**ServiceConfig#doExportUrls**

```Java
private void doExportUrls() {
    //获取服务仓库，缓存provider
    ServiceRepository repository = ApplicationModel.getServiceRepository();
    ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
    repository.registerProvider(
            getUniqueServiceName(),
            ref,
            serviceDescriptor,
            this,
            serviceMetadata
    );

    //获取服务注册中心，协议为registry://
    List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);

    //在每个协议下注册服务
    int protocolConfigNum = protocols.size();
    for (ProtocolConfig protocolConfig : protocols) {
        String pathKey = URL.buildKey(getContextPath(protocolConfig)
                .map(p -> p + "/" + path)
                .orElse(path), group, version);
        // In case user specified path, register service one more time to map it to path.
        repository.registerService(pathKey, interfaceClass);
        doExportUrlsFor1Protocol(protocolConfig, registryURLs, protocolConfigNum);
    }
}
```

在这里，Dubbo允许我们使用不同的协议导出服务，同时向多个注册中心注册服务。

进入到**ServiceConfig#doExportUrlsFor1Protocol**：

```Java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs, int protocolConfigNum) {
    //转义名默认为dubbo
    String name = protocolConfig.getName();
    if (StringUtils.isEmpty(name)) {
        name = DUBBO;
    }

    //收集URL参数
    Map<String, String> map = new HashMap<String, String>();
    map.put(SIDE_KEY, PROVIDER_SIDE);

    ServiceConfig.appendRuntimeParameters(map);
    AbstractConfig.appendParameters(map, getMetrics());
    AbstractConfig.appendParameters(map, getApplication());
    AbstractConfig.appendParameters(map, getModule());
    // remove 'default.' prefix for configs from ProviderConfig
    // appendParameters(map, provider, Constants.DEFAULT_KEY);
    AbstractConfig.appendParameters(map, provider);
    AbstractConfig.appendParameters(map, protocolConfig);
    AbstractConfig.appendParameters(map, this);
    MetadataReportConfig metadataReportConfig = getMetadataReportConfig();
    if (metadataReportConfig != null && metadataReportConfig.isValid()) {
        map.putIfAbsent(METADATA_KEY, REMOTE_METADATA_STORAGE_TYPE);
    }
    //解析收集org.apache.dubbo.config.annotation.Method注解参数
    if (CollectionUtils.isNotEmpty(getMethods())) {
        for (MethodConfig method : getMethods()) {
			//省略...
        } // end of methods for
    }

    //是否泛化调用
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(GENERIC_KEY, generic);
        map.put(METHODS_KEY, ANY_VALUE);
    } else {
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put(REVISION_KEY, revision);
        }

       	//获取接口方法并添加到URL参数
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            logger.warn("No method found in service interface " + interfaceClass.getName());
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
            map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }

    //添加token到参数map
    if (ConfigUtils.isEmpty(token) && provider != null) {
        token = provider.getToken();
    }

    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(TOKEN_KEY, token);
        }
    }
    //init serviceMetadata attachments
    serviceMetadata.getAttachments().putAll(map);

    //获取主机和端口，得到最终的URL，如dubbo://192.168.1.1:20880/x.y.z.HelloService?a=b
    String host = findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = findConfigedPorts(protocolConfig, name, map, protocolConfigNum);
    URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

    // 添加额外参数
    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(SCOPE_KEY);
    // don't export when none is configured
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {
		//是否本地导出
        // export to local if the config is not remote (export to remote only when config is remote)
        if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        //是否远程导出
        // export to remote if the config is not local (export to local only when config is local)
        if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
            if (CollectionUtils.isNotEmpty(registryURLs)) {
                for (URL registryURL : registryURLs) {
                    if (SERVICE_REGISTRY_PROTOCOL.equals(registryURL.getProtocol())) {
                        url = url.addParameterIfAbsent(REGISTRY_TYPE_KEY, SERVICE_REGISTRY_TYPE);
                    }

                    //if protocol is only injvm ,not register
                    if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                        continue;
                    }
                    
                    url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                    //加载监视器链接并追加
                    URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                    if (monitorUrl != null) {
                        url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                    }
                    if (logger.isInfoEnabled()) {
                        if (url.getParameter(REGISTER_KEY, true)) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " +
                                    registryURL);
                        } else {
                            logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                        }
                    }

                    //是否使用自定义动态代理
                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(PROXY_KEY);
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                    }

                    //生成代理类invoker
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass,
                            registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
					
                    //导出服务，并生成exporter
                    Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
                //不存在注册中心，仅导出exporter
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
                //协议为dubbo://，使用DubboProtocol导出
                Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                exporters.add(exporter);
            }

            MetadataUtils.publishServiceDefinition(url);
        }
    }
    this.urls.add(url);
}
```

可以看到，这里主要做了几件事：

1. 构建参数map，组装URL对象，这里需要注意的是，这里的 URL 不是**java.net.URL**，而是**com.alibaba.dubbo.common.URL**；
2. 根据scope参数导出服务，同时生成代理Invoker：
   - 为none时，不导出服务
   - 不为remote时，导出到本地
   - 不为local时，导出到远程

#### 创建Invoker

Invoker是由**ProxyFactory**创建的，Dubbo默认使用的ProxyFactory实现类为**JavassistProxyFactory**：

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



#### 导出服务到本地

```Java
private void exportLocal(URL url) {
    URL local = URLBuilder.from(url)
            .setProtocol(LOCAL_PROTOCOL)
            .setHost(LOCALHOST_VALUE)
            .setPort(0)
            .build();
    Exporter<?> exporter = PROTOCOL.export(
            PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local));
    exporters.add(exporter);
    logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
}
```

导出服务到本地时，先创建Invoker，然后调用**Protocol#export**导出，此时URL协议头为**injvm**，使用的Protocol实现为**InjvmProtocol**：

```Java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
}
```

这里将Invoker包装为Exporter后就返回了。



#### 导出服务到远程

```Java
Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
```

```Java
private static final Protocol PROTOCOL = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();
```

根据**registryURL**可判断这里调用的是**RegistryProtocol**：

```Java
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
    //获取注册地址
    URL registryUrl = getRegistryUrl(originInvoker);
    //获取服务地址，即传递进来的export参数，如dubbo://...
    // url to export locally
    URL providerUrl = getProviderUrl(originInvoker);

    //向注册中心进行订阅 override 数据
    // Subscribe the override data
    // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call
    //  the same service. Because the subscribed is cached key with the name of the service, it causes the
    //  subscription information to cover.
    final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
    final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
    overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);

    providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
    //导出到provider
    // export invoker
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

    // url to registry
    //获取注册中心实例
    final Registry registry = getRegistry(originInvoker);
    final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);

    // decide if we need to delay publish
    boolean register = providerUrl.getParameter(REGISTER_KEY, true);
    if (register) {
        //注册
        registry.register(registeredProviderUrl);
    }

    // register stated url on provider model
    registerStatedUrl(registryUrl, registeredProviderUrl, register);


    exporter.setRegisterUrl(registeredProviderUrl);
    exporter.setSubscribeUrl(overrideSubscribeUrl);

    // Deprecated! Subscribe to override rules in 2.6.x or before.
    registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

    notifyExport(exporter);
    //Ensure that a new exporter instance is returned every time export
    return new DestroyableExporter<>(exporter);
}
```

来到**RegistryProtocol#doLocalExport**：

```
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
    String key = getCacheKey(originInvoker);

    return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
        Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
        return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
    });
}
```

这里的运行时协议是dubbo，协议为dubbo://，则protocol实例则是**DubboProtocol**：

```Java
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
    URL url = invoker.getUrl();

    //获取服务标识
    // export service.
    String key = serviceKey(url);
    //创建DubboExporter并放入缓存
    DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
    exporterMap.put(key, exporter);

    //export an stub service for dispatching event
    Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
    Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
    if (isStubSupportEvent && !isCallbackservice) {
        String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
        if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
            if (logger.isWarnEnabled()) {
                logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                        "], has set stubproxy support event ,but no stub methods founded."));
            }

        }
    }

    //启动本地服务器
    openServer(url);
    optimizeSerialization(url);

    return exporter;
}
```

启动服务器之后，就是注册服务。



#### 服务注册

获取注册中心实例：

```Java
protected Registry getRegistry(final Invoker<?> originInvoker) {
    URL registryUrl = getRegistryUrl(originInvoker);
    return getRegistry(registryUrl);
}
```

由于这里使用的注册中心为nacos，此时registryUrl的协议被转换为了nacos：

```Java
protected Registry getRegistry(URL url) {
    try {
        return registryFactory.getRegistry(url);
    } catch (Throwable t) {
        LOGGER.error(t.getMessage(), t);
        throw t;
    }
}
```

这里registryFactory为**RegistryFactory**的自适应扩展类，由**SpiExtensionFactory**注入，最终获取到的RegistryFactory实例类型为**ListenerRegistryWrapper**，由包装类**RegistryFactoryWrapper#getRegistry**返回，registryFactory类型为**NacosRegistryFactory**，getResistry获取到的Registry类型为**NacosRegistry**：

```
@Override
public Registry getRegistry(URL url) {
    return new ListenerRegistryWrapper(registryFactory.getRegistry(url),
            Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(RegistryServiceListener.class)
                    .getActivateExtension(url, "registry.listeners")));
}
```

**RegistryFactoryWrapper#register**方法:

```Java
@Override
public void register(URL url) {
    try {
        registry.register(url);
    } finally {
        if (CollectionUtils.isNotEmpty(listeners)) {
            RuntimeException exception = null;
            for (RegistryServiceListener listener : listeners) {
                if (listener != null) {
                    try {
                        listener.onRegister(url);
                    } catch (RuntimeException t) {
                        logger.error(t.getMessage(), t);
                        exception = t;
                    }
                }
            }
            if (exception != null) {
                throw exception;
            }
        }
    }
}
```

最终来到**NacosRegistry#doRegister**，向Nacos注册服务：

```Java
public void doRegister(URL url) {
    //获取服务名
    final String serviceName = getServiceName(url);

    //服务实例信息
    final Instance instance = createInstance(url);
    /**
     *  namingService.registerInstance with {@link org.apache.dubbo.registry.support.AbstractRegistry#registryUrl}
     *  default {@link DEFAULT_GROUP}
     *
     * in https://github.com/apache/dubbo/issues/5978
     */
    execute(namingService -> namingService.registerInstance(serviceName,
            getUrl().getParameter(GROUP_KEY, Constants.DEFAULT_GROUP), instance));
}
```



### 总结

本文简单分析了，Dubbo服务的启动和注册过程，包括：

1. **@EnableDubbo**注解的原理及Dubbo的启动入口；
2. Dubbo服务导出的过程：包括URL组装、服务的导出及本地服务的启动；
3. 在没有注册中心时，根据dubbo://协议，使用DubboProtocol导出；存在注册中心的情况下，先根据registry://协议使用RegistryProtocol导出，再使用传递进来的export参数，使用DubboProtocol进行导出；
4. 远程服务的情况下，创建注册中心实例，进行服务注册（以Nacos为例）。

