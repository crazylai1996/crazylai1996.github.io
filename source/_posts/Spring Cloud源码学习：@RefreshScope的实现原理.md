---
title: Spring Cloud源码学习：@RefreshScope的实现原理
date: 2024-11-03 13:01:02
tags: Spring Cloud
---

在 Spring Cloud 微服务开发中，我们都知道，要想在不重启应用的前提下，修改的配置能动态刷新并且生效，我们只需要在对应的类上加上 **@RefreshScope** 注解即可。

但最近一次项目开发过程中，我们却遇到了一个动态刷新不生效的问题，使用的场景如下：

```Java
@Component
@RefreshScope
public class ConfigUtil {

    private static String myKey;

    @Value("${mykey}")
    public void initMyKey(String myKey) {
        ConfigUtil.myKey = myKey;
    }

    public static String getMyKey() {
    	return myKey;
    }

}
```

通过 ConfigUtil#getMyKey 方法获取配置值，会发现修改配置后动态刷新并不生效。

本着“知根知底”的原则，我们决定先探究下 @RefreshScope 的实现原理。

<!--more-->

### 从@Scope说起 

 @Scope 是 Spring 中 Bean 的作用域，它主要有以下几种类型：

| 名称        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| singleton   | 默认类型，单例，表示 Spring 容器中只会创建该 bean 的一个实例 |
| prototype   | 每次调用都会创建一个新的 bean 实例                           |
| request     | 在 web 应用中使用，每个 http 请求都有一个 bean 实例          |
| session     | 在 web 应用中使用，每个 http 会话都有一个 bean 实例          |
| application | 在 web 应用中使用，全局共享                                  |

而 Spring Cloud 则在此基础上增加了 **refresh** 的作用域，包装为 **@RefreshScope** 注解 ，用于实现配置的动态刷新。

### @RefreshScope 做了什么

在 **@RefreshScope** 注解定义中：

```Java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Scope("refresh")
@Documented
public @interface RefreshScope {

   /**
    * @see Scope#proxyMode()
    * @return proxy mode
    */
   ScopedProxyMode proxyMode() default ScopedProxyMode.TARGET_CLASS;

}
```

我们注意到有个 proxyMode 属性，默认为 ScopedProxyMode.TARGET_CLASS ，这其实表明了 Spring 将对该类创建代理对象。



#### @RefreshScope 的解析

在 Spring Boot 对配置类进行扫描并生成 BeanDefinition 的过程中，首先会通过 **ScopedProxyUtils#createScopedProxy** 生成代理类的 BeanDefinition：

```Java
public static BeanDefinitionHolder createScopedProxy(BeanDefinitionHolder definition,
      BeanDefinitionRegistry registry, boolean proxyTargetClass) {

   //定义代理 bean 的名称为 bean 原名，并将目标 bean 的名称定义为 scopedTarget.${bean原名}
   String originalBeanName = definition.getBeanName();
   BeanDefinition targetDefinition = definition.getBeanDefinition();
   String targetBeanName = getTargetBeanName(originalBeanName);

   //定义代理 bean 的BeanDefinition 类型为 ScopedProxyFactoryBean
   // Create a scoped proxy definition for the original bean name,
   // "hiding" the target bean in an internal target definition.
   RootBeanDefinition proxyDefinition = new RootBeanDefinition(ScopedProxyFactoryBean.class);
   proxyDefinition.setDecoratedDefinition(new BeanDefinitionHolder(targetDefinition, targetBeanName));
   proxyDefinition.setOriginatingBeanDefinition(targetDefinition);
   proxyDefinition.setSource(definition.getSource());
   proxyDefinition.setRole(targetDefinition.getRole());

   //设置目标 bean 名称属性
   proxyDefinition.getPropertyValues().add("targetBeanName", targetBeanName);
   if (proxyTargetClass) {
      targetDefinition.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
      // ScopedProxyFactoryBean's "proxyTargetClass" default is TRUE, so we don't need to set it explicitly here.
   }
   else {
      proxyDefinition.getPropertyValues().add("proxyTargetClass", Boolean.FALSE);
   }

   // 将目标 bean 属性复制到代理 bean 定义
   // Copy autowire settings from original bean definition.
   proxyDefinition.setAutowireCandidate(targetDefinition.isAutowireCandidate());
   proxyDefinition.setPrimary(targetDefinition.isPrimary());
   if (targetDefinition instanceof AbstractBeanDefinition) {
      proxyDefinition.copyQualifiersFrom((AbstractBeanDefinition) targetDefinition);
   }

   // 目标 bean 定义为非首选 bean，并隐藏起来，不作为候选 bean 注入
   // The target bean should be ignored in favor of the scoped proxy.
   targetDefinition.setAutowireCandidate(false);
   targetDefinition.setPrimary(false);

   // 注册 目标 bean 定义，同时返回 代理 bean 定义 
   // Register the target bean as separate bean in the factory.
   registry.registerBeanDefinition(targetBeanName, targetDefinition);

   // Return the scoped proxy definition as primary bean definition
   // (potentially an inner bean).
   return new BeanDefinitionHolder(proxyDefinition, originalBeanName, definition.getAliases());
}
```

这里针对被 @Scope 修饰的 bean ，另外生成了一个类型为 **ScopedProxyFactoryBean** 的代理类 bean 定义，并注册到 Spring 容器，同时隐藏了对原 bean 修改为不参与注入，该代理 bean 定义保留了原 bean 的基本定义，可以看出，依赖原 bean 的地方在被注入依赖时使用的是该代理 bean 。



#### 创建代理对象

在得到代理 bean 的定义后，将首先经过 **RefreshScope** 的处理：

![image-20241103205114718](http://storage.laixiaoming.space/blog/image-20241103205114718.png)

RefreshScope 继承了 BeanDefinitionRegistryPostProcessor 和 BeanFactoryPostProcessor ,在 Spring Boot 实例化 bean 前，会先后经过这两种类型的处理器的扩展点处理。

**RefreshScope#postProcessBeanDefinitionRegistry** :

```Java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)
      throws BeansException {
   this.registry = registry;
   super.postProcessBeanDefinitionRegistry(registry);
}
```

**GenericScope#postProcessBeanDefinitionRegistry** ：

```Java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry)
      throws BeansException {
   for (String name : registry.getBeanDefinitionNames()) {
      BeanDefinition definition = registry.getBeanDefinition(name);
      if (definition instanceof RootBeanDefinition) {
         RootBeanDefinition root = (RootBeanDefinition) definition;
         if (root.getDecoratedDefinition() != null && root.hasBeanClass()
               && root.getBeanClass() == ScopedProxyFactoryBean.class) {
            if (getName().equals(root.getDecoratedDefinition().getBeanDefinition()
                  .getScope())) {
                //将代理对象类型设置为 LockedScopedProxyFactoryBean
               root.setBeanClass(LockedScopedProxyFactoryBean.class);
               root.getConstructorArgumentValues().addGenericArgumentValue(this);
               // surprising that a scoped proxy bean definition is not already
               // marked as synthetic?
               root.setSynthetic(true);
            }
         }
      }
   }
}
```

**GenericScope#postProcessBeanFactory** ：

```Java
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
      throws BeansException {
   this.beanFactory = beanFactory;
   //注册 RefreshScope 实现
   beanFactory.registerScope(this.name, this);
   setSerializationId(beanFactory);
}
```

postProcessBeanDefinitionRegistry 主要将代理 bean 的类型设置为 **LockedScopedProxyFactoryBean** , postProcessBeanFactory 则将 RefreshScope 实现注册到 Spring 容器。

LockedScopedProxyFactoryBean 继承了 ScopedProxyFactoryBean ，其本质是一个 FactoryBean ：

![image-20241103213436547](http://storage.laixiaoming.space/blog/image-20241103213436547.png)



在 ScopedProxyFactoryBean 实例化后，setBeanFactory 将被执行，**GenericScope.LockedScopedProxyFactoryBean#setBeanFactory** ：

```Java
@Override
public void setBeanFactory(BeanFactory beanFactory) {
   super.setBeanFactory(beanFactory);
   //获取代理对象
   Object proxy = getObject();
   if (proxy instanceof Advised) {
      Advised advised = (Advised) proxy;
      //将自身作为切面通知
      advised.addAdvice(0, this);
   }
}
```

**ScopedProxyFactoryBean#setBeanFactory** ：

```Java
@Override
public void setBeanFactory(BeanFactory beanFactory) {
   if (!(beanFactory instanceof ConfigurableBeanFactory)) {
      throw new IllegalStateException("Not running in a ConfigurableBeanFactory: " + beanFactory);
   }
   ConfigurableBeanFactory cbf = (ConfigurableBeanFactory) beanFactory;

   this.scopedTargetSource.setBeanFactory(beanFactory);

   //创建代理对象
   ProxyFactory pf = new ProxyFactory();
   pf.copyFrom(this);
   pf.setTargetSource(this.scopedTargetSource);

   Assert.notNull(this.targetBeanName, "Property 'targetBeanName' is required");
   Class<?> beanType = beanFactory.getType(this.targetBeanName);
   if (beanType == null) {
      throw new IllegalStateException("Cannot create scoped proxy for bean '" + this.targetBeanName +
            "': Target type could not be determined at the time of proxy creation.");
   }
   if (!isProxyTargetClass() || beanType.isInterface() || Modifier.isPrivate(beanType.getModifiers())) {
      pf.setInterfaces(ClassUtils.getAllInterfacesForClass(beanType, cbf.getBeanClassLoader()));
   }

   // Add an introduction that implements only the methods on ScopedObject.
   ScopedObject scopedObject = new DefaultScopedObject(cbf, this.scopedTargetSource.getTargetBeanName());
   pf.addAdvice(new DelegatingIntroductionInterceptor(scopedObject));

   // Add the AopInfrastructureBean marker to indicate that the scoped proxy
   // itself is not subject to auto-proxying! Only its target bean is.
   pf.addInterface(AopInfrastructureBean.class);

   this.proxy = pf.getProxy(cbf.getBeanClassLoader());
}
```

经过 setBeanFactory 处理后，代理对象被创建完成，同时 LockedScopedProxyFactoryBean 作为一个 FactoryBean ，在 getObject 方法中，也会返回该代理对象。

另外， LockedScopedProxyFactoryBean 自身也被作为切面通知添加到通知列表，在调用目标对象的方法时，将执行自身定义的 **GenericScope.LockedScopedProxyFactoryBean#invoke** 方法。



#### 目标对象的实例化

在 Spring Boot 启动后，会触发一次 目标对象的实例化，**RefreshScope#onApplicationEvent**：

```Java
public void onApplicationEvent(ContextRefreshedEvent event) {
    start(event);
}

public void start(ContextRefreshedEvent event) {
   if (event.getApplicationContext() == this.context && this.eager
         && this.registry != null) {
      eagerlyInitialize();
   }
}

private void eagerlyInitialize() {
   for (String name : this.context.getBeanDefinitionNames()) {
      BeanDefinition definition = this.registry.getBeanDefinition(name);
      if (this.getName().equals(definition.getScope())
            && !definition.isLazyInit()) {
         // 目标对象实例化
         Object bean = this.context.getBean(name);
         if (bean != null) {
            bean.getClass();
         }
      }
   }
}
```

同时，如果代理对象方法被调用时，也会触发目标对象的实例化，**GenericScope.LockedScopedProxyFactoryBean#invoke**：

```Java
@Override
public Object invoke(MethodInvocation invocation) throws Throwable {
   Method method = invocation.getMethod();
   if (AopUtils.isEqualsMethod(method) || AopUtils.isToStringMethod(method)
         || AopUtils.isHashCodeMethod(method)
         || isScopedObjectGetTargetObject(method)) {
      return invocation.proceed();
   }
   //获取代理对象
   Object proxy = getObject();
   ReadWriteLock readWriteLock = this.scope.getLock(this.targetBeanName);
   if (readWriteLock == null) {
      if (logger.isDebugEnabled()) {
         logger.debug("For bean with name [" + this.targetBeanName
               + "] there is no read write lock. Will create a new one to avoid NPE");
      }
      readWriteLock = new ReentrantReadWriteLock();
   }
   Lock lock = readWriteLock.readLock();
   lock.lock();
   try {
      if (proxy instanceof Advised) {
         Advised advised = (Advised) proxy;
         ReflectionUtils.makeAccessible(method);
         //目标对象实例化，同时调用目标对象方法
         return ReflectionUtils.invokeMethod(method,
               advised.getTargetSource().getTarget(),
               invocation.getArguments());
      }
      return invocation.proceed();
   }
   // see gh-349. Throw the original exception rather than the
   // UndeclaredThrowableException
   catch (UndeclaredThrowableException e) {
      throw e.getUndeclaredThrowable();
   }
   finally {
      lock.unlock();
   }
}
```

**SimpleBeanTargetSource#getTarget** ：

```Java
@Override
public Object getTarget() throws Exception {
   return getBeanFactory().getBean(getTargetBeanName());
}
```

熟悉的 getBean 方法，最终来到 **AbstractBeanFactory#doGetBean** ：

```Java
   if (mbd.isSingleton()) {
      //忽略....
   }
   else if (mbd.isPrototype()) {
      //忽略...
   }
   else {
      String scopeName = mbd.getScope();
      //获取 RefreshScope 实例
      final Scope scope = this.scopes.get(scopeName);
      if (scope == null) {
         throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
      }
      try {
         Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
               return createBean(beanName, mbd, args);
            }
            finally {
               afterPrototypeCreation(beanName);
            }
         });
         bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
      }
      catch (IllegalStateException ex) {
         //...
      }
   }
}
```

来到 RefreshScope ，**GenericScope#get** ：

```Java
public Object get(String name, ObjectFactory<?> objectFactory) {
   //放入 cache
   BeanLifecycleWrapper value = this.cache.put(name,
         new BeanLifecycleWrapper(name, objectFactory));
   this.locks.putIfAbsent(name, new ReentrantReadWriteLock());
   try {
      return value.getBean();
   }
   catch (RuntimeException e) {
      this.errors.put(name, e);
      throw e;
   }
}
```

这里将创建 bean 的流程作为 objectFactory ，然后封装为 BeanLifecycleWrapper 类型，将放入维护的 cache 缓存，最终调用的是 **GenericScope.BeanLifecycleWrapper#getBean** :

```Java
public Object getBean() {
   if (this.bean == null) {
      synchronized (this.name) {
         if (this.bean == null) {
            this.bean = this.objectFactory.getObject();
         }
      }
   }
   return this.bean;
}
```

在 bean 不为空的情况下，调用 objectFactory 创建一个新的对象，那什么情况下 bean 会为空呢？



### 配置的刷新过程

在配置变更时， RefreshEventListener 会接收到配置变更通知，同时调用 **ContextRefresher#refresh** ：

```Java
public synchronized Set<String> refresh() {
   //获取变更的key值
   Set<String> keys = refreshEnvironment();
   this.scope.refreshAll();
   return keys;
}
```

**RefreshScope#refreshAll** ：

```Java
public void refreshAll() {
   //销毁已创建的bean
   super.destroy();
   this.context.publishEvent(new RefreshScopeRefreshedEvent());
}
```

**GenericScope#destroy()** ：

```Java
public void destroy() {
   List<Throwable> errors = new ArrayList<Throwable>();
   // 获取所有的 BeanLifecycleWrapper 并清空 cache
   Collection<BeanLifecycleWrapper> wrappers = this.cache.clear();
   for (BeanLifecycleWrapper wrapper : wrappers) {
      try {
         Lock lock = this.locks.get(wrapper.getName()).writeLock();
         lock.lock();
         try {
            //逐个销毁
            wrapper.destroy();
         }
         finally {
            lock.unlock();
         }
      }
      catch (RuntimeException e) {
         errors.add(e);
      }
   }
   if (!errors.isEmpty()) {
      throw wrapIfNecessary(errors.get(0));
   }
   this.errors.clear();
}
```

**GenericScope.BeanLifecycleWrapper#destroy** :

```Java
public void destroy() {
   if (this.callback == null) {
      return;
   }
   synchronized (this.name) {
      Runnable callback = this.callback;
      if (callback != null) {
         callback.run();
      }
      // 将bean 实例置空
      this.callback = null;
      this.bean = null;
   }
}
```

可以看到，刷新实则是对缓存中的实例列表进行清空，当通过代理对象调用目标对象方法时，目标对象如果为空，将触发目标对象的重新实例化，从而实现了配置的动态刷新。



### 总结

回到文章开头的问题，原因就是被 @RefreshScope 修饰的类会动态生成代理对象，而静态方法的调用不会经过代理对象的处理，所以也不会触发 bean 的重新实例化。