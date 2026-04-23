---
title: Spring Boot源码学习：自动配置原理
date: 2023-05-02 20:23:31
tags: Spring Boot
---



## 前言

“约定优于配置”是Spring Boot倡导的一个思想，而其自动配置的特性则恰好体现了这一思想。而有了自动配置，不仅简化了Maven的依赖配置，更重要的是摆脱了以往使用Spring框架开发时，所必须编写的一堆繁琐的xml配置文件。

<!--more-->

> 以下内容基于Spring Boot 2.1.9.RELEASE版本



## 从@SpringBootApplication注解说起

我们都知道，一个Spring Boot主启动类上必须标注@SpringBootApplication注解，点开这个注解我们可以看到这是一个组合注解，其中关键的注解有@SpringBootConfiguration、@EnableAutoConfiguration和@ComponentScan这3个。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    //...
}
```



### @SpringBootConfiguration

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```

点开@SpringBootConfiguration，可以看到它被@Configuration标注，这表明了我们的启动类同时也是一个配置类，这一点需要注意，因为Spring Boot整个启动流程可以说都是围绕着我们的主启动类进行的。



### @EnableAutoConfiguration

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    //...
}
```

@Import注解的作用是将目标类作为Bean添加到IOC容器，通常用于将多个分散的配置类融合成一个更大的配置类。@Import支持导入**普通类**、**配置类**、**ImportSelector 的实现类**以及**ImportBeanDefinitionRegistrar 的实现类**。



#### AutoConfigurationImportSelector的作用

![AutoConfigurationImportSelector](http://storage.laixiaoming.space/blog/AutoConfigurationImportSelector.jpg)

AutoConfigurationImportSelector是DeferredImportSelector的实现类，而DeferredImportSelector继承了ImportSelector，而ImportSelector#selectImports方法正是用来获取需要实际导入到IOC容器的类名数组的方法。

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    //...
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
                                                                              annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

可以发现，在selectImports方法中，获取类名数组的关键方法就在于getAutoConfigurationEntry方法



##### AutoConfigurationImportSelector#getAutoConfigurationEntry

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
    //...
    //获取候选的自动配置类
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    //...
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

来到getCandidateConfigurations方法：

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
                                                                         getBeanClassLoader());
    //...
    return configurations;
}
```

里面调用了SpringFactoriesLoader的loadFactoryNames方法，并传入了**EnableAutoConfiguration.class**这样一个参数，继续深入可以发现在里面又调用了loadSpringFactories方法：



##### SpringFactoriesLoader#loadFactoryNames

```java
//...
Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
result = new LinkedMultiValueMap<>();
while (urls.hasMoreElements()) {
    URL url = urls.nextElement();
    UrlResource resource = new UrlResource(url);
    Properties properties = PropertiesLoaderUtils.loadProperties(resource);
    for (Map.Entry<?, ?> entry : properties.entrySet()) {
        String factoryClassName = ((String) entry.getKey()).trim();
        for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
            result.add(factoryClassName, factoryName.trim());
        }
    }
}
//...
return result;
```

通过代码可以看出，loadSpringFactories方法做的事情就是：

1. 扫描所有jar包路径下**META-INF/spring.factories**文件；
2. 以Properties的形式加载加载该文件，将将其收集到Map返回；

回到loadFactoryNames方法，在前面拿到了**META-INF/spring.factories**的Map内容后，会取出**EnableAutoConfiguration.class**对应的值，随后便将其添加到容器中。

那么**META-INF/spring.factories**都有着什么内容呢



##### ```META-INF/spring.factories```文件

这个文件可以在**spring-boot-autoconfiguration**包下找到，我在其中截取了部分**EnableAutoConfiguration.class**内容，不难看到，这些都是我们在日常开发中常见的配置类，这其实也解释了，为什么即便我们在项目中没有显示添加任何配置，而只要我们添加了对应的starter依赖，Spring Boot便会帮助我们创建对应的Bean：

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,\
org.springframework.boot.autoconfigure.cache.CacheAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration
#...
```



#### @AutoConfigurationPackage

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

还是@Import注解，点进AutoConfigurationPackages.Registrar可以发现其手动注册了一个**BasePackages.class**的Bean，并把主启动类的成员设置到**BasePackages**对象成员里；

那这样做有什么用呢？其实AutoConfigurationPackages类上的注释已经告诉了我们答案，它是设计来给其他模块或组件用的，相当于提供给了其他模块或组件获取根包的一个入口：

```java
 Class for storing auto-configuration packages for reference later (e.g. by JPA entity
 scanner)
```



### @ComponentScan

开启组件扫描，可以指定扫描路径，在不指定的情况下，会扫描当前配置类所在包及子包的所有组件，这也解释了，为什么我们在使用Spring Boot的时候，启动类要放在最外层。

到这里，不知道读者是否会有疑问，启动类被@Configuration修饰，它既然也是一个配置类，那么它是什么时候被注册到IOC容器的呢？

通过dubug启动过程，其实不难发现是在SpringApplication#prepareContext方法进行的：

```java
//...
//获取主启动类位置
Set<Object> sources = getAllSources();
Assert.notEmpty(sources, "Sources must not be empty");
//加载
load(context, sources.toArray(new Object[0]));
//...
```



## 总结

1. Spring Boot通过 AutoConfigurationImportSelector，并扫描所有jar包目录中**META-INF/spring.factories**配置文件，并加载```org.springframework.boot.autoconfigure.EnableutoConfiguration```配置项中对应的自动配置类，这也是自动配置生效的原理；
2. 主启动类也是一个配置类，而**@ComponentScan**会默认扫描当前配置类所在包及子包的所有组件；







