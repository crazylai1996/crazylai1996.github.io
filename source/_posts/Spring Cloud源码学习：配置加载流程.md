---
title: Spring Cloud源码学习：配置加载流程
date: 2024-11-16 12:46:12
tags: Spring Cloud

---



在 Spring Boot/Cloud 中，配置是应用环境的一部分，而环境有一个专属的类 —— “Environment”。

Environment 是 Spring 对环境的抽象，它封装了应用运行时的所有配置属性，这些属性可以来自不同的来源，如本地配置文件、系统属性、环境变量、命令行参数、配置中心等，而 Environment 则提供了一个统一的接口来访问这些属性。

那这些不同来源的配置文件是如何加载并生效的呢？本文一起来看看。

<!--more-->



### 创建 Environment

进入 SpringApplication.run 方法，来到 **org.springframework.boot.SpringApplication#prepareEnvironment** ：

```Java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
      ApplicationArguments applicationArguments) {
   // 根据当前应用环境类型，创建对应的 Environment 实例，此处实例类型为 StandardServletEnvironment
   ConfigurableEnvironment environment = getOrCreateEnvironment();
   // 将启动参数绑定到 environment 中
   configureEnvironment(environment, applicationArguments.getSourceArgs());
   ConfigurationPropertySources.attach(environment);
   // 发布环境准备完毕事件
   listeners.environmentPrepared(environment);
   bindToSpringApplication(environment);
   if (!this.isCustomEnvironment) {
      environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
            deduceEnvironmentClass());
   }
   ConfigurationPropertySources.attach(environment);
   return environment;
}
```

这里在创建好 Environment 实例后，使用 SpringApplicationRunListener 发布了一个广播事件，此时只有一个 EventPublishingRunListener 实例，直接来到其 environmentPrepared 方法：

```Java
@Override
public void environmentPrepared(ConfigurableEnvironment environment) {
   this.initialMulticaster
         .multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
}
```

![image-20241116160613820](http://storage.laixiaoming.space/blog/image-20241116160613820.png)



### 加载bootstrap本地配置文件

广播事件会回调所有适配的 ApplicationListener 监听器，调用其 onApplicationEvent 方法，这里第一个被调用的是 **BootstrapApplicationListener#onApplicationEvent** ：

```Java
@Override
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
   ConfigurableEnvironment environment = event.getEnvironment();
   // 默认执行，除非显示关闭
   if (!environment.getProperty("spring.cloud.bootstrap.enabled", Boolean.class,
         true)) {
      return;
   }
   // don't listen to events in a bootstrap context
   // 如果当前配置源已经包含了 "bootstrap" ，则忽略不重复处理
   if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
      return;
   }
   ConfigurableApplicationContext context = null;
   String configName = environment
         .resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
   // 是否存在父容器上下文的初始化器，通过 debug 发现没有
   for (ApplicationContextInitializer<?> initializer : event.getSpringApplication()
         .getInitializers()) {
      if (initializer instanceof ParentContextApplicationContextInitializer) {
         context = findBootstrapContext(
               (ParentContextApplicationContextInitializer) initializer,
               configName);
      }
   }
   if (context == null) {
      // 创建 bootstrap 级别的应用上下文
      context = bootstrapServiceContext(environment, event.getSpringApplication(),
            configName);
      event.getSpringApplication()
            .addListeners(new CloseContextOnFailureApplicationListener(context));
   }

   apply(context, event.getSpringApplication(), environment);
}
```

#### 构建新的内置 SpringApplication

来到 **BootstrapApplicationListener#bootstrapServiceContext** ：

```Java
private ConfigurableApplicationContext bootstrapServiceContext(
      ConfigurableEnvironment environment, final SpringApplication application,
      String configName) {
   // 创建 bootstrap 级别的 environment
   StandardEnvironment bootstrapEnvironment = new StandardEnvironment();
   // 将初始化生成的一些 PropertySource 移除，包括 systemProperties(JVM级的环境变量) 和 systemEnvironment(操作系统级的环境变量)
   MutablePropertySources bootstrapProperties = bootstrapEnvironment
         .getPropertySources();
   for (PropertySource<?> source : bootstrapProperties) {
      bootstrapProperties.remove(source.getName());
   }
   // 是否存在外部配置文件
   String configLocation = environment
         .resolvePlaceholders("${spring.cloud.bootstrap.location:}");
   String configAdditionalLocation = environment
         .resolvePlaceholders("${spring.cloud.bootstrap.additional-location:}");
   // 初始化容器参数
   Map<String, Object> bootstrapMap = new HashMap<>();
   bootstrapMap.put("spring.config.name", configName);
   // if an app (or test) uses spring.main.web-application-type=reactive, bootstrap
   // will fail
   // force the environment to use none, because if though it is set below in the
   // builder
   // the environment overrides it
   bootstrapMap.put("spring.main.web-application-type", "none");
   if (StringUtils.hasText(configLocation)) {
      bootstrapMap.put("spring.config.location", configLocation);
   }
   if (StringUtils.hasText(configAdditionalLocation)) {
      bootstrapMap.put("spring.config.additional-location",
            configAdditionalLocation);
   }
   // 将 environment 的配置添加到新增的 bootstrapEnvironment
   bootstrapProperties.addFirst(
         new MapPropertySource(BOOTSTRAP_PROPERTY_SOURCE_NAME, bootstrapMap));
   for (PropertySource<?> source : environment.getPropertySources()) {
      if (source instanceof StubPropertySource) {
         continue;
      }
      bootstrapProperties.addLast(source);
   }
   // 构建了一个新的 SpringApplication
   SpringApplicationBuilder builder = new SpringApplicationBuilder()
         .profiles(environment.getActiveProfiles()).bannerMode(Mode.OFF)
         .environment(bootstrapEnvironment)
         // Don't use the default properties in this builder
         .registerShutdownHook(false).logStartupInfo(false)
         .web(WebApplicationType.NONE);
   final SpringApplication builderApplication = builder.application();
   if (builderApplication.getMainApplicationClass() == null) {
      // gh_425:
      // SpringApplication cannot deduce the MainApplicationClass here
      // if it is booted from SpringBootServletInitializer due to the
      // absense of the "main" method in stackTraces.
      // But luckily this method's second parameter "application" here
      // carries the real MainApplicationClass which has been explicitly
      // set by SpringBootServletInitializer itself already.
      builder.main(application.getMainApplicationClass());
   }
   if (environment.getPropertySources().contains("refreshArgs")) {
      // If we are doing a context refresh, really we only want to refresh the
      // Environment, and there are some toxic listeners (like the
      // LoggingApplicationListener) that affect global static state, so we need a
      // way to switch those off.
      builderApplication
            .setListeners(filterListeners(builderApplication.getListeners()));
   }
   // 添加 BootstrapImportSelectorConfiguration 配置类
   builder.sources(BootstrapImportSelectorConfiguration.class);
   // 调用 SpringApplication 的run方法
   final ConfigurableApplicationContext context = builder.run();
   // gh-214 using spring.application.name=bootstrap to set the context id via
   // `ContextIdApplicationContextInitializer` prevents apps from getting the actual
   // spring.application.name
   // during the bootstrap phase.
   context.setId("bootstrap");
   // Make the bootstrap context a parent of the app context
   addAncestorInitializer(application, context);
   // It only has properties in it now that we don't want in the parent so remove
   // it (and it will be added back later)
   bootstrapProperties.remove(BOOTSTRAP_PROPERTY_SOURCE_NAME);
   // 配置合并
   mergeDefaultProperties(environment.getPropertySources(), bootstrapProperties);
   return context;
}
```

可以看到，这里面构造了一个新的 SpringApplication，新的 SpringApplication 有几点值得关注：

1. 激活的 profile 与原 SpringApplication 保持一致；
2. 使用了新构建的 environment ，并添加了一个 "bootstrap" 属性源；
3. WebApplicationType 设置为了 NONE ，说明该 SpringApplication 不会创建 Web 容器；
4. 添加了一个 BootstrapImportSelectorConfiguration 配置类；



#### BootstrapImportSelectorConfiguration 配置类

```Java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
   ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
   // Use names and ensure unique to protect against duplicates
   // 加载 BootstrapConfiguration 配置项类
   List<String> names = new ArrayList<>(SpringFactoriesLoader
         .loadFactoryNames(BootstrapConfiguration.class, classLoader));
   names.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(
         this.environment.getProperty("spring.cloud.bootstrap.sources", ""))));

   List<OrderedAnnotatedElement> elements = new ArrayList<>();
   for (String name : names) {
      try {
         elements.add(
               new OrderedAnnotatedElement(this.metadataReaderFactory, name));
      }
      catch (IOException e) {
         continue;
      }
   }
   AnnotationAwareOrderComparator.sort(elements);

   String[] classNames = elements.stream().map(e -> e.name).toArray(String[]::new);

   return classNames;
}
```

该配置类使用 Spring 中的 SPI ，加载了 **BootstrapConfiguration.class** 配置的组件，通过 debug 发现，这里加载的类（使用 Nacos 配置中心）有：

- com.alibaba.cloud.nacos.NacosConfigBootstrapConfiguration
- org.springframework.cloud.bootstrap.config.PropertySourceBootstrapConfiguration
- org.springframework.cloud.bootstrap.encrypt.EncryptionBootstrapConfiguration
- org.springframework.cloud.autoconfigure.ConfigurationPropertiesRebinderAutoConfiguration
- org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration

这些配置都与配置中心有关，值得关注的一点是，**NacosConfigBootstrapConfiguration** 也是在 bootstrap SpringApplication 中创建的，也说明了 Nacos 配置中心的地址的配置需要配置在 bootstrap 配置文件，而不能在 application 配置文件。



#### 启动内置 SpringApplication

新启动的 SpringApplication 与原 SpringApplication 的流程基本一致。

首先是又一次来到 **BootstrapApplicationListener#onApplicationEvent** :

```Java
public void onApplicationEvent(ApplicationEnvironmentPreparedEvent event) {
   //省略...
   // don't listen to events in a bootstrap context
   if (environment.getPropertySources().contains(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
      return;
   }
   //省略...
}
```

不同的是，由于当前已存在 "bootstrap" 属性源，该监听器的后续操作也不会继续执行。

我们直接看 **ConfigFileApplicationListener#onApplicationEvent** 。



#### ConfigFileApplicationListener 的监听动作

```Java
@Override
public void onApplicationEvent(ApplicationEvent event) {
   if (event instanceof ApplicationEnvironmentPreparedEvent) {
      onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
   }
   if (event instanceof ApplicationPreparedEvent) {
      onApplicationPreparedEvent(event);
   }
}
```

当前是 ApplicationEnvironmentPreparedEvent 事件，来到 onApplicationEnvironmentPreparedEvent ：

```Java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
   List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
   postProcessors.add(this);
   AnnotationAwareOrderComparator.sort(postProcessors);
   for (EnvironmentPostProcessor postProcessor : postProcessors) {
      postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
   }
}
```

这里出现了一个 EnvironmentPostProcessor ，它是运行环境的后置处理器，而 ConfigFileApplicationListener 本身也是一个 EnvironmentPostProcessor ,来到 **ConfigFileApplicationListener#postProcessEnvironment** ：

```Java
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
   addPropertySources(environment, application.getResourceLoader());
}
```

```Java
protected void addPropertySources(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
   RandomValuePropertySource.addToEnvironment(environment);
   new Loader(environment, resourceLoader).load();
}
```

启动了一个 Loader 加载器，用于加载配置源：

```Java
public void load() {
   this.profiles = new LinkedList<>();
   this.processedProfiles = new LinkedList<>();
   this.activatedProfiles = false;
   this.loaded = new LinkedHashMap<>();
   // 初始化 profiles 集合，通过debug 发现有两个元素，一个是 null，另一个当前激活的比如 dev
   initializeProfiles();
   while (!this.profiles.isEmpty()) {
      Profile profile = this.profiles.poll();
      if (profile != null && !profile.isDefaultProfile()) {
         addProfileToEnvironment(profile.getName());
      }
      load(profile, this::getPositiveProfileFilter, addToLoaded(MutablePropertySources::addLast, false));
      this.processedProfiles.add(profile);
   }
   resetEnvironmentProfiles(this.processedProfiles);
   load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
   addLoadedPropertySources();
}
```

确定要加载的 profile 后，继续深入来到 load 方法：

```Java
private void load(Profile profile, DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
   //从配置文件所在目录中逐个扫描
    getSearchLocations().forEach((location) -> {
      boolean isFolder = location.endsWith("/");
      Set<String> names = isFolder ? getSearchNames() : NO_SEARCH_NAMES;
      names.forEach((name) -> load(location, name, profile, filterFactory, consumer));
   });
}

private Set<String> getSearchNames() {
   //是否有 spring.config.name 属性，有则返回，通过 debug 发现此时值为 bootstrap
   if (this.environment.containsProperty(CONFIG_NAME_PROPERTY)) {
      String property = this.environment.getProperty(CONFIG_NAME_PROPERTY);
      return asResolvedSet(property, null);
   }
   return asResolvedSet(ConfigFileApplicationListener.this.names, DEFAULT_NAMES);
}
```

获取配置文件所在位置，逐个扫描该位置下的配置文件，扫描的目录包括 file:./config/、file:./、classpath:/config/、classpath:/，这里有一个点需要注意的是，这里会按顺序进行扫描，获取到的配置将追加到 MutablePropertySources 的末尾，也直接影响了配置获取的优先级：

![image-20241116173530035](http://storage.laixiaoming.space/blog/image-20241116173530035.png)

结合 getSearchNames 方法可知，该内置 SpringApplication 的作用之一便是加载 bootstrap 级别的配置文件到 Environment。

#### 合并配置

在加载完 bootstrap 配置文件后， 再通过 **mergeDefaultProperties** 将 bootstrap 的配置合并到原 application 的 environment 中。

回到 **BootstrapApplicationListener#onApplicationEvent** ，临到创建好的 bootstrap 的 ApplicationContext 后，同时也将 bootstrap 中的创建的 ApplicationContextInitializer 注册到原 application 的上下文，其中也包括 **PropertySourceBootstrapConfiguration**：

```Java
private void apply(ConfigurableApplicationContext context,
      SpringApplication application, ConfigurableEnvironment environment) {
   @SuppressWarnings("rawtypes")
   List<ApplicationContextInitializer> initializers = getOrderedBeansOfType(context,
         ApplicationContextInitializer.class);
   application.addInitializers(initializers
         .toArray(new ApplicationContextInitializer[initializers.size()]));
   addBootstrapDecryptInitializer(application);
}
```

至此，该 bootstrap 内置 SpringApplication 也执行完成了。



### 加载 application 本地配置文件

回到外层 application 的创建流程中，在BootstrapApplicationListener 监听器执行完成后，将再一次执行 **ConfigFileApplicationListener#onApplicationEvent** ，但此时不同的是，加载的配置文件名称不再是 "bootstrap" 了，而是 "application" ：

![image-20241116194509525](http://storage.laixiaoming.space/blog/image-20241116194509525.png)



### Nacos配置文件的加载

在前面我们了解到，在创建运行 bootstrap 内置 SpringApplication 后，会注册一系列 ApplicationContextInitializer ，其中就包括 **PropertySourceBootstrapConfiguration** ，那它是什么时候被调用的呢

通过 debug ，来到 **SpringApplication#prepareContext** ：

```Java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
      SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
   //...
   applyInitializers(context);
   //...
}
protected void applyInitializers(ConfigurableApplicationContext context) {
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
                                                                        ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

**PropertySourceBootstrapConfiguration#initialize** :

```Java
public void initialize(ConfigurableApplicationContext applicationContext) {
   List<PropertySource<?>> composite = new ArrayList<>();
   AnnotationAwareOrderComparator.sort(this.propertySourceLocators);
   boolean empty = true;
   ConfigurableEnvironment environment = applicationContext.getEnvironment();
   // 使用 PropertySourceLocator 拉取远程配置
   for (PropertySourceLocator locator : this.propertySourceLocators) {
      Collection<PropertySource<?>> source = locator.locateCollection(environment);
      if (source == null || source.size() == 0) {
         continue;
      }
      List<PropertySource<?>> sourceList = new ArrayList<>();
      for (PropertySource<?> p : source) {
         sourceList.add(new BootstrapPropertySource<>(p));
      }
      logger.info("Located property source: " + sourceList);
      composite.addAll(sourceList);
      empty = false;
   }
   if (!empty) {
      // 合并远程配置
      MutablePropertySources propertySources = environment.getPropertySources();
      String logConfig = environment.resolvePlaceholders("${logging.config:}");
      LogFile logFile = LogFile.get(environment);
      for (PropertySource<?> p : environment.getPropertySources()) {
         if (p.getName().startsWith(BOOTSTRAP_PROPERTY_SOURCE_NAME)) {
            propertySources.remove(p.getName());
         }
      }
      insertPropertySources(propertySources, composite);
      reinitializeLoggingSystem(environment, logConfig, logFile);
      setLogLevels(applicationContext, environment);
      handleIncludedProfiles(environment);
   }
}
```

在使用 Nacos 配置中心时，将使用 **NacosPropertySourceLocator#locate** 进行配置的拉取，这里不深入了解。



### 总结

本文从源码的角度简单的分析了 Spring Boot/Cloud 下的配置加载流程：

1. Spring Boot 在启动时，通过 **ApplicationEnvironmentPreparedEvent** 事件触发 **BootstrapApplicationListener** 监听器的执行，该监听器：
   1.  构建一个新的内置 SpringApplication 并且启动；
   2.  该bootstrap 级 SpringApplication 通过 **ConfigFileApplicationListener ** 加载 bootstrap 配置文件，加载完成后合并到外部的 SpringApplication 的 Environment；
   3.  同时也将 **PropertySourceBootstrapConfiguration** 初始化器注册到外部 SpringApplication 中；
2.  加载 application 配置文件，同样也是由 **ConfigFileApplicationListener ** 完成；
3. 通过 **PropertySourceBootstrapConfiguration** 加载远程配置；



