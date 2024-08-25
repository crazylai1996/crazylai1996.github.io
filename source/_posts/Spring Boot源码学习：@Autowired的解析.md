---
title: Spring Boot源码学习：@Autowired的解析
date: 2023-05-18 08:49:43
tags: Spring Boot
---



## 前言

@Autowired是开发中常用的注解，我们可以使用@Autowired将其标记在构造函数、成员变量、setter方法上，并由Spring自动完成依赖注入的工作。但是，这个过程是怎么完成的呢？

<!--more-->

理解这个之前，我们需要先了解一下**BeanPostProcessor**概念。

> 以下内容基于Spring Boot 2.1.9.RELEASE版本



### BeanPostProcessor是什么

```java
/**
 * Factory hook that allows for custom modification of new bean instances,
 * e.g. checking for marker interfaces or wrapping them with proxies.
 *
 * <p>ApplicationContexts can autodetect BeanPostProcessor beans in their
 * bean definitions and apply them to any beans subsequently created.
 * Plain bean factories allow for programmatic registration of post-processors,
 * applying to all beans created through this factory.
 *
 * <p>Typically, post-processors that populate beans via marker interfaces
 * or the like will implement {@link #postProcessBeforeInitialization},
 * while post-processors that wrap beans with proxies will normally
 * implement {@link #postProcessAfterInitialization}.
 *
 */
public interface BeanPostProcessor {
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
    @Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```

由注释我们可以看到，我们可以通过**BeanPostProcessor**修改bean的实例，例如检查bean的接口或将将bean包装为代理对象。接口中提供了两个方法，通常情况下，**postProcessBeforeInitialization**被用于填充bean的属性，而**postProcessAfterInitialization**用于返回bean的代理对象，另外需要注意的是，如果该方法返回了null，则表示后续的BeanPostProcessors不会再被调用。

了解**BeanPostProcessor**，我们不禁猜测，@Autowire注解会不会也是由**BeanPostProcessor**解析和注入的呢，没错，它正是由**BeanPostProcessor**的实现类**AutowiredAnnotationBeanPostProcessor**处理的。

## 理解AutowiredAnnotationBeanPostProcessor

![AutowiredAnnotationBeanPostProcessor](http://storage.laixiaoming.space/blog/AutowiredAnnotationBeanPostProcessor.jpg)

但查看**AutowiredAnnotationBeanPostProcessor**的类继承关系图我们会发现，**AutowiredAnnotationBeanPostProcessor**

并不是直接实现**BeanPostProcessor**，而是实现自**BeanPostProcessor**的子接口，**MergedBeanDefinitionPostProcessor**和**SmartInstantiationAwareBeanPostProcessor**。

回到**AutowiredAnnotationBeanPostProcessor**的源码，我们看到它主要有实现了几个关键的方法：determineCandidateConstructors、postProcessMergedBeanDefinition、postProcessProperties，我们逐个看。

### determineCandidateConstructors

这个方法是**SmartInstantiationAwareBeanPostProcessor**接口中定义的，从方法注释我们可以知道，该方法用于确定给定bean的构造函数，这里我们不展开细说。

### postProcessMergedBeanDefinition

这个方法是**MergedBeanDefinitionPostProcessor**接口中定义的，查看其实现：

```java
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
    metadata.checkConfigMembers(beanDefinition);
}
```

#### findAutowiringMetadata

该方法作用是构建Autowired的元数据，深入发现，其核心是调用了org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#buildAutowiringMetadata：

```java
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
    List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
    Class<?> targetClass = clazz;

    do {
        final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
		//解析类所有属性字段，将其封装为AutowiredFieldElement
        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            //解析该字段上的@Autowired注解，查看该方法可以发现，该处理器不仅解析了@Autowired注解，还有@Value及@Inject注解
            AnnotationAttributes ann = findAutowiredAnnotation(field);
            if (ann != null) {
                //忽略static字段
                if (Modifier.isStatic(field.getModifiers())) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Autowired annotation is not supported on static fields: " + field);
                    }
                    return;
                }
                boolean required = determineRequiredStatus(ann);
                currElements.add(new AutowiredFieldElement(field, required));
            }
        });

        //解析类方法，将其封装为AutowiredMethodElement
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                return;
            }
            //解析该方法上的@Autowire注解
            AnnotationAttributes ann = findAutowiredAnnotation(bridgedMethod);
            if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                //忽略静态方法
                if (Modifier.isStatic(method.getModifiers())) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Autowired annotation is not supported on static methods: " + method);
                    }
                    return;
                }
                //忽略无参方法
                if (method.getParameterCount() == 0) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Autowired annotation should only be used on methods with parameters: " +
                                    method);
                    }
                }
                boolean required = determineRequiredStatus(ann);
                PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                currElements.add(new AutowiredMethodElement(method, required, pd));
            }
        });

        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    }
    //循环解析父类
    while (targetClass != null && targetClass != Object.class);

    return new InjectionMetadata(clazz, elements);
}
```

可以看出，buildAutowiringMetadata主要是收集被@Autowired修饰的字段以及方法。

#### checkConfigMembers

```java
public void checkConfigMembers(RootBeanDefinition beanDefinition) {
    Set<InjectedElement> checkedElements = new LinkedHashSet<>(this.injectedElements.size());
    for (InjectedElement element : this.injectedElements) {
        Member member = element.getMember();
        if (!beanDefinition.isExternallyManagedConfigMember(member)) {
            beanDefinition.registerExternallyManagedConfigMember(member);
            checkedElements.add(element);
            if (logger.isTraceEnabled()) {
                logger.trace("Registered injected element on class [" + this.targetClass.getName() + "]: " + element);
            }
        }
    }
    this.checkedElements = checkedElements;
}
```

这里会缓存已经检验过的注入点，但是这个作用是什么，查看checkConfigMembers的调用位置，发现除了被 用于**AutowiredAnnotationBeanPostProcessor**这个处理器外，还在**CommonAnnotationBeanPostProcessor**中出现，而**CommonAnnotationBeanPostProcessor**是用于解析@Resource注解的，不难看出，这里主要是避免重复注入的问题。



### postProcessProperties

```java
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
 	//省略...
    //注入
    metadata.inject(bean, beanName, pvs);
    //省略...
    return pvs;
}
```

```java
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Collection<InjectedElement> checkedElements = this.checkedElements;
    Collection<InjectedElement> elementsToIterate =
        (checkedElements != null ? checkedElements : this.injectedElements);
    if (!elementsToIterate.isEmpty()) {
        for (InjectedElement element : elementsToIterate) {
            element.inject(target, beanName, pvs);
        }
    }
}
```

拿到了前面解析过程得到的@Autowired元数据后，进行值的注入，我们以字段的注入为例，查看其注入流程，来到AutowiredFieldElement#inject方法：

```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
    Field field = (Field) this.member;
    Object value;
    //从缓存中读取值
    if (this.cached) {
        value = resolvedCachedArgument(beanName, this.cachedFieldValue);
    }
    else {
        DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
        desc.setContainingClass(bean.getClass());
        Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
        Assert.state(beanFactory != null, "No BeanFactory available");
        TypeConverter typeConverter = beanFactory.getTypeConverter();
        try {
            //从beanFactory中解析依赖的bean
            value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
        }
        catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
        }
        synchronized (this) {
            if (!this.cached) {
                if (value != null || this.required) {
                    this.cachedFieldValue = desc;
                    //注册依赖关系
                    registerDependentBeans(beanName, autowiredBeanNames);
                    if (autowiredBeanNames.size() == 1) {
                        String autowiredBeanName = autowiredBeanNames.iterator().next();
                        if (beanFactory.containsBean(autowiredBeanName) &&
                            beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
                            this.cachedFieldValue = new ShortcutDependencyDescriptor(
                                desc, autowiredBeanName, field.getType());
                        }
                    }
                }
                else {
                    this.cachedFieldValue = null;
                }
                this.cached = true;
            }
        }
    }
    if (value != null) {
        //通过反射设置依赖值
        ReflectionUtils.makeAccessible(field);
        field.set(bean, value);
    }
}
}
```

其中关键是使用DefaultListableBeanFactory#resolveDependency解析依赖值，查看其实现，主要逻辑在DefaultListableBeanFactory#doResolveDependency：

```java
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        Object shortcut = descriptor.resolveShortcut(this);
        if (shortcut != null) {
            return shortcut;
        }

        //处理@Value注解，并获取对应值
        Class<?> type = descriptor.getDependencyType();
        Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
        if (value != null) {
            if (value instanceof String) {
                String strVal = resolveEmbeddedValue((String) value);
                BeanDefinition bd = (beanName != null && containsBean(beanName) ?
                                     getMergedBeanDefinition(beanName) : null);
                value = evaluateBeanDefinitionString(strVal, bd);
            }
            TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
            try {
                return converter.convertIfNecessary(value, type, descriptor.getTypeDescriptor());
            }
            catch (UnsupportedOperationException ex) {
                // A custom TypeConverter which does not support TypeDescriptor resolution...
                return (descriptor.getField() != null ?
                        converter.convertIfNecessary(value, type, descriptor.getField()) :
                        converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
            }
        }

        //处理集合bean的注入
        Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
        if (multipleBeans != null) {
            return multipleBeans;
        }

        //处理单个bean的注入
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
        if (matchingBeans.isEmpty()) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            return null;
        }

        String autowiredBeanName;
        Object instanceCandidate;

        //如果获取到的bean有多个，将根据@Primary及@Priority确定最优的一个
        if (matchingBeans.size() > 1) {
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
                }
                else {
                    // In case of an optional Collection/Map, silently ignore a non-unique case:
                    // possibly it was meant to be an empty collection of multiple regular beans
                    // (before 4.3 in particular when we didn't even look for collection beans).
                    return null;
                }
            }
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        }
        else {
            // We have exactly one match.
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }

        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }
        if (instanceCandidate instanceof Class) {
            //根据beanName，从beanFactory获取bean实例
            instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
        }
        Object result = instanceCandidate;
        if (result instanceof NullBean) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            result = null;
        }
        if (!ClassUtils.isAssignableValue(type, result)) {
            throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
        }
        return result;
    }
    finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```

了解了**AutowiredAnnotationBeanPostProcessor**的处理流程，那该处理器是在什么时候被调用的呢？



### AutowiredAnnotationBeanPostProcessor的调用时机

查看其方法调用位置，不难看出：

```
AbstractAutowireCapableBeanFactory#createBean
	->AbstractAutowireCapableBeanFactory#doCreateBean
        ->AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors
        	->MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition
        ->AbstractAutowireCapableBeanFactory#populateBean
            ->InstantiationAwareBeanPostProcessor#postProcessProperties
```



## 总结

1. @Autowired、@Value等是由**AutowiredAnnotationBeanPostProcessor**处理的；
2. **AutowiredAnnotationBeanPostProcessor**的postProcessMergedBeanDefinition方法用于解析构建@Autowired元数据，而postProcessProperties方法则用于@Autowired解析后值的注入；