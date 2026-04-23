---
title: Spring Boot源码学习：AOP代理对象的创建
date: 2023-06-14 23:17:31
tags: Spring Boot
---



## 前言

AOP是Spring中的核心功能之一，使用AOP，可以让关注点代码与业务代码分离，并且动态地添加和删除在切面上的逻辑而不影响原来的执行代码，从而可以在不修改源代码的情况下，实现对功能的增强。

AOP的应用场景很多，日志记录、性能监控、事务管理等都可以通过AOP去实现。

AOP的原理就是动态代理，在 Spring 中，存在两种实现机制， JDK 动态代理以及 CGLIB 动态代理。

在Spring Boot中，AOP可以通过**@EnableAspectJAutoProxy**注解开启，那该注解是怎么起作用的呢，代理对象又是如何被创建的呢？

<!--more-->

> 以下内容基于Spring Boot 2.1.9.RELEASE版本



## @EnableAspectJAutoProxy

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

	/**
	 * Indicate whether subclass-based (CGLIB) proxies are to be created as opposed
	 * to standard Java interface-based proxies. The default is {@code false}.
	 */
	boolean proxyTargetClass() default false;

	/**
	 * Indicate that the proxy should be exposed by the AOP framework as a {@code ThreadLocal}
	 * for retrieval via the {@link org.springframework.aop.framework.AopContext} class.
	 * Off by default, i.e. no guarantees that {@code AopContext} access will work.
	 * @since 4.3.1
	 */
	boolean exposeProxy() default false;

}
```

该注解@Import了**AspectJAutoProxyRegistrar**这个类：

```java
/**
 * Registers an {@link org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator
 * AnnotationAwareAspectJAutoProxyCreator} against the current {@link BeanDefinitionRegistry}
 * as appropriate based on a given @{@link EnableAspectJAutoProxy} annotation.
 */
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	/**
	 * Register, escalate, and configure the AspectJ auto proxy creator based on the value
	 * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
	 * {@code @Configuration} class.
	 */
	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        //注册AnnotationAwareAspectJAutoProxyCreator
		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy != null) {
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}

}
```

由类注释可以看出，**AspectJAutoProxyRegistrar**的作用是向IOC容器注册**AnnotationAwareAspectJAutoProxyCreator**。那么**AnnotationAwareAspectJAutoProxyCreator**是什么呢?



## AnnotationAwareAspectJAutoProxyCreator

我们先来看下**AnnotationAwareAspectJAutoProxyCreator**的类继承关系图：

![AnnotationAwareAspectJAutoProxyCreator](http://storage.laixiaoming.space/blog/AnnotationAwareAspectJAutoProxyCreator.jpg)

可以发现，**AnnotationAwareAspectJAutoProxyCreator**：

1. 实现了**BeanPostProcessor **接口，其关键方法**postProcessBeforeInitialization**和**postProcessAfterInitialization**将会在bean创建过程中的初始化流程中**AbstractAutowireCapableBeanFactory#initializeBean**被调用，而AOP代理代理对象也是通过**postProcessAfterInitialization**得到；
2. 实现了**InstantiationAwareBeanPostProcessor**接口，其关键方法**postProcessBeforeInstantiation**会在bean实例化前尝试被调用；
3. 实现了**SmartInstantiationAwareBeanPostProcessor**接口，其关键方法**getEarlyBeanReference**将会作为提前暴露代理对象的入口放入bean的三级缓存中；

在本文中，我们主要关注**postProcessAfterInitialization **方法，该方法在**AbstractAutoProxyCreator**中被实现。



## AbstractAutoProxyCreator#postProcessAfterInitialization

```java
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
            return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}
```

## earlyProxyReferences

查看其引用，可以发现只在**getEarlyBeanReference**中被使用，而**getEarlyBeanReference**是在循环引用发生的情况下被调用，结合这里的判断不难看出，这里主要是防止代理对象的重复生成。



### wrapIfNecessary

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //判断当前bean是否在targetSourcedBeans中存在，存在则表示已处理过
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    //是否无需处理
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }
    //isInfrastructureClass判断是否基础设施类型，这里过滤了Advice、PointCut、Advisor、AopInfrastructureBean类型，
    //以及被@Advice标注的类
    //shouldSkip判断是否需要跳过该bean
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    //获取能对该bean进行增强的切面
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        //创建代理对象
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}
```



#### shouldSkip

```java
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
    //加载所有的增强器，并将其缓存
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    for (Advisor advisor : candidateAdvisors) {
        //跳过增强器类
        if (advisor instanceof AspectJPointcutAdvisor &&
            ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
            return true;
        }
    }
    //调用父类实现，如果是原始类型，则跳过
    return super.shouldSkip(beanClass, beanName);
}
```



```java
protected List<Advisor> findCandidateAdvisors() {
    //调用父类实现，加载Advisor类型的bean
    List<Advisor> advisors = super.findCandidateAdvisors();
    if (this.aspectJAdvisorsBuilder != null) {
        //加载被@Aspect标注的类，
        //并将@Around、@Before、@After、@AfterReturning、@AfterThrowing标注的方法等构建为Advisor并返回
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}
```



#### getAdvicesAndAdvisorsForBean

```java
protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
	//找到符合条件的增强器
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}
```



```java
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //加载所有的增加器，该方法在shouldSkip中调用过一次，再次调用将从缓存中获取
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    //过滤出匹配到的增强器，这里会根据切入点（@Pointcut）表达式去匹配该bean
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}
```



#### createProxy

```java
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
                             @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
    }
    //创建代理工厂
    ProxyFactory proxyFactory = new ProxyFactory();
    //从当前类复制配置
    proxyFactory.copyFrom(this);

    if (!proxyFactory.isProxyTargetClass()) {
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    //组合所有增强器，并将其放入代理工厂
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    proxyFactory.addAdvisors(advisors);
    proxyFactory.setTargetSource(targetSource);
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }

    //通过代理工厂创建代理对象
    return proxyFactory.getProxy(getProxyClassLoader());
}
```



#### getProxy

创建代理对象：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}
```



```java
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
        activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}
```



```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                                         "Either an interface or a target is required for proxy creation.");
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```

最终来到DefaultAopProxyFactory，然后是我们熟悉的逻辑：

1. 如果目标对象有接口，用JDK动态代理；
2. 如果没有接口，用CGLIB动态代理；



## 总结

1. AOP代理对象的创建通过**AnnotationAwareAspectJAutoProxyCreator**实现，该类是**BeanPostProcessor**的子类，并且在不存在循环依赖的情况下，是在**postProcessAfterInitialization**方法中创建的；
2. **postProcessAfterInitialization**中，会匹配出所有和当前Bean相关的增强器，并最终根据实际情况判断是使用JDK动态代理，还是CGLIB动态代理；

