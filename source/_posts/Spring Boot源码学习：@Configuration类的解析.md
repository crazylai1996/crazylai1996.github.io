---
title: Spring Boot源码学习：@Configuration类的解析
date: 2023-05-04 23:34:20
tags: Spring Boot
---



## 前言

使用@Configuration注解可以为一个类声明为配置类，一个配置类声明了一个或多个@Bean方法，这些方法返回值将作为bean定义注册到Spring IoC容器中，并允许在配置类中通过调用同一类中的其他@Bean方法来定义bean之间的依赖关系。

<!--more-->

> 以下内容基于Spring Boot 2.1.9.RELEASE版本



## @Configuration类解析流程

Spring Boot是如何解析@Configuration配置类的呢？要弄懂这个过程，首先我们需要先了解一个BeanFactoryPostProcessor的概念。

### BeanFactoryPostProcessor是什么

```java
public interface BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}

```

BeanFactoryPostProcessor是Spring 提供的一个接口，从代码注释中我们可以了解到，我们可以利用BeanFactoryPostProcessor实现对内部bean工厂进行修改，允许我们通过它完成bean定义或者属性的修改，另外需要注意的一点是，BeanFactoryPostProcessor的执行时机是在BeanFactory标准初始化后，并且是在Bean实例化之前。

而@Configuration配置类的解析正是由BeanFactoryPostProcessor的实现类**ConfigurationClassPostProcessor**处理的：

![image-20220511233922833](http://storage.laixiaoming.space/blog/image-20220511233922833.png)

由ConfigurationClassPostProcessor的类继承关系图，我们可以发现ConfigurationClassPostProcessor并不直接实现BeanFactoryPostProcessor接口，而是实现了BeanFactoryPostProcessor的子接口BeanDefinitionRegistryPostProcessor，那么BeanDefinitionRegistryPostProcessor是什么呢

```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

BeanDefinitionRegistryPostProcessor继承自BeanFactoryPostProcessor，它额外定义了一个方法，通过这个方法参数BeanDefinitionRegistry我们可以看出，它与BeanFactoryPostProcessor的区别在于，我们可以通过这个方法新增Bean定义，另外，它的执行时机是在BeanFactoryPostProcessor之前。

回到ConfigurationClassPostProcessor，这个BeanDefinitionRegistryPostProcessor的实现类到底做了哪些事情呢，我们一起来看下它的实现：

```java
@Override
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    //...
    processConfigBeanDefinitions(registry);
}
```

ConfigurationClassPostProcessor#processConfigBeanDefinitions：

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    String[] candidateNames = registry.getBeanDefinitionNames();

    //填充配置类的full或lite属性
    for (String beanName : candidateNames) {
        BeanDefinition beanDef = registry.getBeanDefinition(beanName);
        if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
            ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
            }
        }
        else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
            configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
        }
    }

    //省略...

    // 配置类解析器
    ConfigurationClassParser parser = new ConfigurationClassParser(
        this.metadataReaderFactory, this.problemReporter, this.environment,
        this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
        //解析配置类（处理@ComponentScan，@Import，@Bean方法等注解），但这里实际上只会把@ComponentScan注解扫描的类注册为BeanDefinition
        parser.parse(candidates);
        parser.validate();

        Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        // Read the model and create bean definitions based on its content
        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                registry, this.sourceExtractor, this.resourceLoader, this.environment,
                this.importBeanNameGenerator, parser.getImportRegistry());
        }
        //这里会解析上一步parse操作中获取到的ConfigurationClass，比如由@Import生成的配置类、或标注了@Bean注解的方法等注册为BeanDefinition
        this.reader.loadBeanDefinitions(configClasses);
        alreadyParsed.addAll(configClasses);

        candidates.clear();
        //判断经过loadBeanDefinitions处理后，是否有新增配置类，有则继续解析
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
            Set<String> alreadyParsedClasses = new HashSet<>();
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                        !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    //省略...
}
```



### full和lite的区别

ConfigurationClassUtils#checkConfigurationClassCandidate：

```java
public static boolean checkConfigurationClassCandidate(
			BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

    //省略...
    if (isFullConfigurationCandidate(metadata)) {
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
    }
    else if (isLiteConfigurationCandidate(metadata)) {
        beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
    }
    else {
        return false;
    }

    // It's a full or lite configuration candidate... Let's determine the order value, if any.
    Integer order = getOrder(metadata);
    if (order != null) {
        beanDef.setAttribute(ORDER_ATTRIBUTE, order);
    }

    return true;
}
```

ConfigurationClassUtils#isFullConfigurationCandidate：

```java
public static boolean isFullConfigurationCandidate(AnnotationMetadata metadata) {
    return metadata.isAnnotated(Configuration.class.getName());
}
```

ConfigurationClassUtils#isLiteConfigurationCandidate：

```java
private static final Set<String> candidateIndicators = new HashSet<>(8);

static {
    candidateIndicators.add(Component.class.getName());
    candidateIndicators.add(ComponentScan.class.getName());
    candidateIndicators.add(Import.class.getName());
    candidateIndicators.add(ImportResource.class.getName());
}

public static boolean isLiteConfigurationCandidate(AnnotationMetadata metadata) {
    // Do not consider an interface or an annotation...
    if (metadata.isInterface()) {
        return false;
    }

    // Any of the typical annotations found?
    for (String indicator : candidateIndicators) {
        if (metadata.isAnnotated(indicator)) {
            return true;
        }
    }

    // Finally, let's look for @Bean methods...
    try {
        return metadata.hasAnnotatedMethods(Bean.class.getName());
    }
    catch (Throwable ex) {
        if (logger.isDebugEnabled()) {
            logger.debug("Failed to introspect @Bean methods on class [" + metadata.getClassName() + "]: " + ex);
        }
        return false;
    }
}
```

从源码中我们可以知道：

**full：**@Configuration标注的类；

**lite：**@Component、@ComponentScan、@Import、@ImportResource或者存在@Bean方法的类；

那么问题来了，区分full和lite有什么作用呢？

通过查找ConfigurationClassUtils#isFullConfigurationClass的调用位置，我们可以定位到ConfigurationClassPostProcessor#enhanceConfigurationClasses：

```java
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
		Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
		for (String beanName : beanFactory.getBeanDefinitionNames()) {
			BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
            //判断是否full配置类
			if (ConfigurationClassUtils.isFullConfigurationClass(beanDef)) {
				//省略...
				configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
			}
		}
		//省略...

		ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
		for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
			AbstractBeanDefinition beanDef = entry.getValue();
			// If a @Configuration class gets proxied, always proxy the target class
			beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			try {
				// Set enhanced subclass of the user-specified bean class
				Class<?> configClass = beanDef.resolveBeanClass(this.beanClassLoader);
				if (configClass != null) {
                    //对full配置类进行增强
					Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
					if (configClass != enhancedClass) {
						//省略...
						beanDef.setBeanClass(enhancedClass);
					}
				}
			}
			catch (Throwable ex) {
				//省略...
			}
		}
	}
```

Spring Boot会对full配置类进行增强，那么为什么要对其进行增加呢？想必大家都看到过以下这种用法，这实际上就是为了支持@Bean方法能够在同一配置类相互调用。

```java
@Configuration
public class MyConfig {
    @Bean
    public A a() {
    }
    @Bean
    public B b() {
        return new B(a());
    }
}
```



### parser.pase()

通过查看其调用链路，定位到具体处理解析代码processConfigurationClass：

```java
ConfigurationClassParser#parse(java.util.Set<org.springframework.beans.factory.config.BeanDefinitionHolder>)
->
ConfigurationClassParse#parse(org.springframework.core.type.AnnotationMetadata, java.lang.String)
->
ConfigurationClassParser#processConfigurationClass
```

```java
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
		//省略...
		//递归处理配置类及其父类
		SourceClass sourceClass = asSourceClass(configClass);
		do {
			sourceClass = doProcessConfigurationClass(configClass, sourceClass);
		}
		while (sourceClass != null);

		this.configurationClasses.put(configClass, configClass);
	}
```

来到doProcessConfigurationClass：

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {

    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // 处理内部类，如果内部类也是一个配置类，将会递归去解析
        processMemberClasses(configClass, sourceClass);
    }

    // 处理@PropertySource注解
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(), PropertySources.class,
        org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            //省略...
        }
    }

    //处理@ComponentScan注解，扫描指定包下的所有.class，这里也是一个递归解析的过程
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
        sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
        !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    //处理@Import注解
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    //处理@ImportResource注解
    AnnotationAttributes importResource =
        AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    //处理@Bean方法
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    //处理接口默认方法
    processInterfaces(configClass, sourceClass);

    //获取父类并返回，用以递归解析
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java") &&
            !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```



### this.reader.loadBeanDefinitions()

定位到具体处理逻辑，ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass：

```java
private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

    //省略...
    if (configClass.isImported()) {
        //对于被导入的类，将其注册为BeanDefinition
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        //将@Bean方法，注册为BeanDefinition
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }

    //处理@ImportResource
    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    //处理@Import注解中，导入的是ImportBeanDefinitionRegistrar接口实现类，调用其registerBeanDefinitions方法
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```



## 总结

1. 我们可以通过实现BeanFactoryPostProcessor对bean进行定义或修改，而Spring Boot是也是通过其实现类ConfigurationClassPostProcessor进行配置类的解析；
2. 配置类分为full和lite，区别为前者会被增强处理；
3. 我们常用的@ComponentScan包扫描，@Import注解，@Bean注解方法等也都是通过ConfigurationClassPostProcessor处理的；



