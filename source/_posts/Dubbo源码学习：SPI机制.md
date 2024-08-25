---
title: Dubbo源码学习：SPI机制
date: 2024-06-23 12:52:15
tags: Dubbo
---

SPI全称是Service Provider Interface，是一种服务自发现机制，本质是将接口实现类的全限定名配置在文件中，在使用中，通过运行时加载，并动态地替换为具体的接口实现类。

<!--more-->

### JDK SPI

在开发中，SPI的应用中最让我们熟知的便是JDBC的使用，JDBC中定义了**java.sql.Driver**接口，当我们调用**DriverManager#getConnection方法时**，将触发**DriverManager**类的初始化，并利用SPI机制自动加载驱动类：

```Java
	static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
```

```Java
   private static void loadInitialDrivers() {
        //省略...
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
				//加载驱动类
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                //获取驱动器迭代器，类型为LazyIterator
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
                
                try{
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                // Do nothing
                }
                return null;
            }
        });

        //省略...
    }

```

在调用**ServiceLoader#load**时，将首先获取当前线程绑定的类加载器（ServiceLoader由启动类加载器加载，启动类加载器无法加载应用类代码）：

```Java
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
    }
```

随后将对迭代器**driversIterator**进行遍历，加载**META-INF/services/java.sql.Driver**路径下所有的文件，并按行读取 ：

```Java
private boolean hasNextService() {
    if (nextName != null) {
        return true;
    }
    if (configs == null) {
        try {
            //META-INF/services/java.sql.Driver
            String fullName = PREFIX + service.getName();
            if (loader == null)
                configs = ClassLoader.getSystemResources(fullName);
            else
                configs = loader.getResources(fullName);
        } catch (IOException x) {
            fail(service, "Error locating configuration files", x);
        }
    }
    while ((pending == null) || !pending.hasNext()) {
        if (!configs.hasMoreElements()) {
            return false;
        }
        pending = parse(service, configs.nextElement());
    }
    nextName = pending.next();
    return true;
}
```

回到**driversIterator.next()**的调用中，会触发具体驱动类的加载：

```
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
    	//加载驱动类
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
        if (!service.isAssignableFrom(c)) {
            fail(service,
                 "Provider " + cn  + " not a subtype");
        }
        try {
            S p = service.cast(c.newInstance())
            //缓存驱动类实例
            providers.put(cn, p);
            return p;
        } catch (Throwable x) {
            fail(service,
                 "Provider " + cn + " could not be instantiated",
                 x);
        }
        throw new Error();          // This cannot happen
}
```

以MySQL为例，加载的类为**com.mysql.cj.jdbc**，可以看到该段初始化代码中会通过**DriverManager#registerDriver**注册当前驱动类：

```Java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    static {
        try {
            DriverManager.registerDriver(new Driver());
        } catch (SQLException var1) {
            throw new RuntimeException("Can't register driver!");
        }
    }
}
```

在整个流程中，我们可以看出，JDK的SPI会一次性的加载并实例化，通过配置文件发现的所有的服务实现类，如果存在多个服务实现类，而我们只需要其中一个的话，则在一定程度上损耗了资源。

### Dubbo SPI实现

在Dubbo中，所有的组件组件都是由SPI进行加载，但Dubbo并示直接使用JDK提供的SPI机制，而是借鉴其核心思想，并对其进行了增强，实现了自身的一套SPI机制，其用法与JDK的类似，核心类是**ExtensionLoader**，配置文件也定义在**META-INF/**路径下，但Dubbo对配置文件具体分为了三类：

- META-INF/services/ 目录：该目录下的 SPI 配置文件用来兼容 JDK SPI 。
- META-INF/dubbo/ 目录：该目录用于存放用户自定义 SPI 配置文件。
- META-INF/dubbo/internal/ 目录：该目录用于存放 Dubbo 内部使用的 SPI 配置文件。

且配置文件的内容也不再是类名，而是K-V形式，key被称为扩展名，value则是扩展实现类，并通指定扩展名实现按需加载：

```
dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
```

#### 示例

定义扩展接口：

```Java
@SPI
public interface Animal {
    String name();
}
```

定义扩展实现：

```Java
public class Cat implements Animal{
    @Override
    public String name() {
        return "I'm a cat";
    }
}
```

```Java
public class Dog implements Animal{
    @Override
    public String name() {
        return "I'm a dog";
    }
}
```

配置文件，位置：META-INF/dubbo/org.apache.dubbo.mytest.Animal：

```Java
cat=org.apache.dubbo.mytest.Cat
dog=org.apache.dubbo.mytest.Dog
```

使用：

```Java
public class MyExtensionLoaderTest {

    @Test
    public void test() {
        ExtensionLoader<Animal> extensionLoader =
                ExtensionLoader.getExtensionLoader(Animal.class);
        Animal cat = extensionLoader.getExtension("cat");
        System.out.println(cat.name());
        Animal dog = extensionLoader.getExtension("dog");
        System.out.println(dog.name());
    }

}
```

#### 源码分析

##### 获取ExtensionLoader实例

```
    public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
		//省略...
        ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        if (loader == null) {
            EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
            loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
        }
        return loader;
    }
```

可以看出，通过EXTENSION_LOADERS缓存字段，保证一个扩展接口有且只有一个ExtensionLoader实例。

##### 获取扩展实例

```Java
    public T getExtension(String name, boolean wrap) {
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        //扩展对象持有器
        final Holder<Object> holder = getOrCreateHolder(name);
        Object instance = holder.get();
        //双检锁使用，创建扩展实例
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    instance = createExtension(name, wrap);
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

先检查缓存 ，没有则通过**createExtension**创建：

```Java
private T createExtension(String name, boolean wrap) {
	//加载所有扩展类，并通过扩展名称获取对应扩展类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null || unacceptableExceptions.contains(name)) {
        throw findException(name);
    }
    try {
    	//获取/创建扩展实例
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.getDeclaredConstructor().newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        
        //扩展实例依赖注入
        injectExtension(instance);

		//包装扩展实例（如果有的话）并返回
        if (wrap) {

            List<Class<?>> wrapperClassesList = new ArrayList<>();
            if (cachedWrapperClasses != null) {
                wrapperClassesList.addAll(cachedWrapperClasses);
                wrapperClassesList.sort(WrapperComparator.COMPARATOR);
                Collections.reverse(wrapperClassesList);
            }

            if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
                for (Class<?> wrapperClass : wrapperClassesList) {
                    Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
                    if (wrapper == null
                            || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), name))) {
                        instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                    }
                }
            }
        }
		//初始化实例
        initExtension(instance);
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                type + ") couldn't be instantiated: " + t.getMessage(), t);
    }
}
```

可以看到，**createExtension**主要包含了4个步骤：

1. 通过getExtensionClasses获取所有扩展实现类；
2. 获取/通过反射创建扩展实例；
3. 向扩展实例注入依赖；
4. 包装扩展实例；

###### 获取所有的扩展类

```Java
    private Map<String, Class<?>> getExtensionClasses() {
        //从缓存中获取扩展类
        Map<String, Class<?>> classes = cachedClasses.get();
        //双检锁使用，加载所有的扩展类
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

通过**loadExtensionClasses**加载扩展类：

```Java
  private Map<String, Class<?>> loadExtensionClasses() {
      	//缓存默认扩展名
        cacheDefaultExtensionName();

        Map<String, Class<?>> extensionClasses = new HashMap<>();
		//从不同目录加载扩展类
        for (LoadingStrategy strategy : strategies) {
            loadDirectory(extensionClasses, strategy.directory(), type.getName(), strategy.preferExtensionClassLoader(),
                    strategy.overridden(), strategy.excludedPackages());
            loadDirectory(extensionClasses, strategy.directory(), type.getName().replace("org.apache", "com.alibaba"),
                    strategy.preferExtensionClassLoader(), strategy.overridden(), strategy.excludedPackages());
        }

        return extensionClasses;
    }
```

这里首先解析了@SPI注解，获取到默认的扩展名并缓存下来，然后通过不同的策略加载扩展类，那这里的策略都有哪些呢，通过定位可以看到是通过**loadLoadingStrategies**获取的：

```Java
    private static LoadingStrategy[] loadLoadingStrategies() {
        return stream(load(LoadingStrategy.class).spliterator(), false)
                .sorted()
                .toArray(LoadingStrategy[]::new);
    }
```

继续深入可以发现，该策略是最终是通过JDK的SPI机制加载的，而通过配置文件可以看到主要包含了有一种策略类**org.apache.dubbo.common.extension.DubboInternalLoadingStrategy**、**org.apache.dubbo.common.extension.DubboLoadingStrategy**、**org.apache.dubbo.common.extension.ServicesLoadingStrategy**，而这3种策略类则分别对应了前端提到的3个配置文件的目录。

回到**loadDirectory**方法：

````Java
    private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type,
                               boolean extensionLoaderClassLoaderFirst, boolean overridden, String... excludedPackages) {
        String fileName = dir + type;
        try {
            Enumeration<java.net.URL> urls = null;
            ClassLoader classLoader = findClassLoader();

            // try to load from ExtensionLoader's ClassLoader first
            if (extensionLoaderClassLoaderFirst) {
                ClassLoader extensionLoaderClassLoader = ExtensionLoader.class.getClassLoader();
                if (ClassLoader.getSystemClassLoader() != extensionLoaderClassLoader) {
                    urls = extensionLoaderClassLoader.getResources(fileName);
                }
            }

            if (urls == null || !urls.hasMoreElements()) {
                if (classLoader != null) {
                    urls = classLoader.getResources(fileName);
                } else {
                    urls = ClassLoader.getSystemResources(fileName);
                }
            }

            if (urls != null) {
                while (urls.hasMoreElements()) {
                    java.net.URL resourceURL = urls.nextElement();
                    loadResource(extensionClasses, classLoader, resourceURL, overridden, excludedPackages);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
````

获取到了对应所有的文件资源，并通过**loadResource**加载：

```Java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader,
                          java.net.URL resourceURL, boolean overridden, String... excludedPackages) {
    try {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
            String line;
            String clazz = null;
            while ((line = reader.readLine()) != null) {
                //只取#前端内容
                final int ci = line.indexOf('#');
                if (ci >= 0) {
                    line = line.substring(0, ci);
                }
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        //分隔以获取扩展名及扩展类名
                        String name = null;
                        int i = line.indexOf('=');
                        if (i > 0) {
                            name = line.substring(0, i).trim();
                            clazz = line.substring(i + 1).trim();
                        } else {
                            clazz = line;
                        }
                        if (StringUtils.isNotEmpty(clazz) && !isExcluded(clazz, excludedPackages)) {
                            //加载扩展类
                            loadClass(extensionClasses, resourceURL, Class.forName(clazz, true, classLoader), name, overridden);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException(
                                "Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL +
                                        ", cause: " + t.getMessage(), t);
                        exceptions.put(line, e);
                    }
                }
            }
        }
    } catch (Throwable t) {
        logger.error("Exception occurred when loading extension class (interface: " +
                type + ", class file: " + resourceURL + ") in " + resourceURL, t);
    }
}
```

加载扩展类：

```Java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name,
                       boolean overridden) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + " is not subtype of interface.");
    }
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        //缓存自适应扩展类
        cacheAdaptiveClass(clazz, overridden);
    } else if (isWrapperClass(clazz)) {
        //缓存包装类
        cacheWrapperClass(clazz);
    } else {
        //普通扩展类
        clazz.getConstructor();
        //如果名称为空，则通过@Extension注解获取
        if (StringUtils.isEmpty(name)) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException(
                        "No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }
		//切分为多个扩展名
        String[] names = NAME_SEPARATOR.split(name);
        if (ArrayUtils.isNotEmpty(names)) {
            cacheActivateClass(clazz, names[0]);
            for (String n : names) {
                //添加到扩展类集合
                cacheName(clazz, n);
                saveInExtensionClass(extensionClasses, clazz, n, overridden);
            }
        }
    }
}
```

从代码分支上看，加载扩展类这里分为3种情况，分别为自适应扩展类、包装类及普通扩展类。

###### 依赖注入

创建扩展实例后，下一步则是向扩展实例注入依赖，具体逻辑是**injectExtension**：

```Java
private T injectExtension(T instance) {

    if (objectFactory == null) {
        return instance;
    }

    try {
        //遍历扩展类的所有方法
        for (Method method : instance.getClass().getMethods()) {
            //是否setter方法
            if (!isSetter(method)) {
                continue;
            }

            /*
             * Check {@link DisableInject} to see if we need autowire injection for this property
             */
            if (method.getAnnotation(DisableInject.class) != null) {
                continue;
            }
			//获取方法参数类型
            Class<?> pt = method.getParameterTypes()[0];
            if (ReflectUtils.isPrimitives(pt)) {
                continue;
            }

            /*
             * Check {@link Inject} to see if we need auto-injection for this property
             * {@link Inject#enable} == false will skip inject property phase
             * {@link Inject#InjectType#ByName} default inject by name
             */
            //获取setter属性名
            String property = getSetterProperty(method);
            Inject inject = method.getAnnotation(Inject.class);
            if (inject == null) {
                //注入依赖值
                injectValue(instance, method, pt, property);
            } else {
                if (!inject.enable()) {
                    continue;
                }

                if (inject.type() == Inject.InjectType.ByType) {
                    //按类型注入
                    injectValue(instance, method, pt, null);
                } else {
                    //按名称注入
                    injectValue(instance, method, pt, property);
                }
            }
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
    }
    return instance;
}
```

```Java
    private void injectValue(T instance, Method method, Class<?> pt, String property) {
        try {
            Object object = objectFactory.getExtension(pt, property);
            if (object != null) {
                method.invoke(instance, object);
            }
        } catch (Exception e) {
            logger.error("Failed to inject via method " + method.getName()
                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
        }
    }
```

可以看到，这里的依赖注入是通过解析set方法进行注入，可以通过类型或名称进行注入，具体的注入逻辑由**objectFactory**决定，通过深入查看发现此处**objectFactory**通过**ExtensionLoader#getAdaptiveExtension**方法获取得到，其类型为**AdaptiveExtensionFactory**，该类被称为自适应扩展类，而它也是通过SPI机制加载得到：

```
@Adaptive
public class AdaptiveExtensionFactory implements ExtensionFactory {

    private final List<ExtensionFactory> factories;

    public AdaptiveExtensionFactory() {
        ExtensionLoader<ExtensionFactory> loader = ExtensionLoader.getExtensionLoader(ExtensionFactory.class);
        List<ExtensionFactory> list = new ArrayList<ExtensionFactory>();
        for (String name : loader.getSupportedExtensions()) {
            list.add(loader.getExtension(name));
        }
        factories = Collections.unmodifiableList(list);
    }

    @Override
    public <T> T getExtension(Class<T> type, String name) {
    	//遍历所有ExtensionFactory，获取到依赖后便直接返回
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }

}
```

而**AdaptiveExtensionFactory**中维护了factories集合，在Spring环境下，该集合包括**SpiExtensionFactory**和**SpringExtensionFactory**。



##### 获取自适应扩展类

什么是自适应扩展类呢？在前文了解到，Dubbo的依赖注入有用到自适应扩展类，通过观察其实现，可以发现它实际上是一种代理类。在Dubbo中，可以通过**@Adaptive**注解标注扩展接口或接口方法，而Dubbo将自动为其生成具有代理功能的代码，并通过编译得到Class类，当调用自适应扩展类时，将通过方法中的URL参数决定具体调用的扩展实例。其关键代码在于**getAdaptiveExtension**：

```Java
public T getAdaptiveExtension() {
    //从缓存中获取自适应扩展类
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError != null) {
            throw new IllegalStateException("Failed to create adaptive instance: " +
                    createAdaptiveInstanceError.toString(),
                    createAdaptiveInstanceError);
        }
		//创建自适应扩展实例
        synchronized (cachedAdaptiveInstance) {
            instance = cachedAdaptiveInstance.get();
            if (instance == null) {
                try {
                    instance = createAdaptiveExtension();
                    cachedAdaptiveInstance.set(instance);
                } catch (Throwable t) {
                    createAdaptiveInstanceError = t;
                    throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                }
            }
        }
    }

    return (T) instance;
}
```

```Java
private T createAdaptiveExtension() {
    try {
        return injectExtension((T) getAdaptiveExtensionClass().newInstance());
    } catch (Exception e) {
        throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
    }
}
```

与创建普通的扩展实例相似，创建自适应扩展实例也分为3个步骤：

1. 获取自适应扩展类；
2. 创建自应用扩展实例；
3. 依赖注入；

我们具体看下**getAdaptiveExtensionClass**：

```
private Class<?> getAdaptiveExtensionClass() {
    getExtensionClasses();
    if (cachedAdaptiveClass != null) {
        return cachedAdaptiveClass;
    }
    return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

**getAdaptiveExtensionClass**同样包含了3个步骤：

1. 获取扩展类（与获取普通的扩展类的调用是同一个方法，这里会缓存**@Adaptive**注解标注的类为cachedAdaptiveClass）；
2. 缓存的自适应扩展类cachedAdaptiveClass不为空时，直接返回；
3. 创建自适应扩展类；

```Java
private Class<?> createAdaptiveExtensionClass() {
    //生成自适应扩展类的代码
    String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
    //获取编译器实现类
    ClassLoader classLoader = findClassLoader();
    org.apache.dubbo.common.compiler.Compiler compiler =
            ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
    //编译代码，并生成Class
    return compiler.compile(code, classLoader);
}
```

createAdaptiveExtensionClass用于生成自适应扩展类代码，并通过**org.apache.dubbo.common.compiler.Compiler**实例编码代码，得到Class类。通过查看其配置文件可知，目前存在**JdkCompiler**和**JavassistCompiler**两种编译器，而Dubbo默认使用**JavassistCompiler**。

那生成的自适应扩展类是什么样的呢，这里以**ProxyFactory**为例，使用arthas工具反编译得到：

```Java
public class ProxyFactory$Adaptive
implements ProxyFactory {
    public Object getProxy(Invoker invoker) throws RpcException {
        if (invoker == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        }
        URL uRL = invoker.getUrl();
        String string = uRL.getParameter("proxy", "javassist");
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (").append(uRL.toString()).append(") use keys([proxy])").toString());
        }
        ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(string);
        return proxyFactory.getProxy(invoker);
    }

    public Object getProxy(Invoker invoker, boolean bl) throws RpcException {
        if (invoker == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        }
        if (invoker.getUrl() == null) {
            throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
        }
        URL uRL = invoker.getUrl();
        String string = uRL.getParameter("proxy", "javassist");
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (").append(uRL.toString()).append(") use keys([proxy])").toString());
        }
        ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(string);
        return proxyFactory.getProxy(invoker, bl);
    }

    public Invoker getInvoker(Object object, Class clazz, URL uRL) throws RpcException {
        if (uRL == null) {
            throw new IllegalArgumentException("url == null");
        }
        URL uRL2 = uRL;
        String string = uRL2.getParameter("proxy", "javassist");
        if (string == null) {
            throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (").append(uRL2.toString()).append(") use keys([proxy])").toString());
        }
        ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getExtension(string);
        return proxyFactory.getInvoker(object, clazz, uRL);
    }
}
```

可以看到，自适应扩展类就是通过获取指定URL参数，来动态决定扩展实现类。



##### 自动包装类

回到**ExtensionLoader#loadClass**方法中：

```
    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name,
                           boolean overridden) throws NoSuchMethodException {
        //省略...
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            cacheAdaptiveClass(clazz, overridden);
            //是否包含拷贝构造函数
        } else if (isWrapperClass(clazz)) {
        	//缓存包装类
            cacheWrapperClass(clazz);
        } else {
			//省略...
        }
    }
```

那包装类是什么呢，通过查看cachedWrapperClasses的使用位置，可以看到，在**ExtensionLoader#createExtension**中：

```
if (wrap) {

    List<Class<?>> wrapperClassesList = new ArrayList<>();
    if (cachedWrapperClasses != null) {
        wrapperClassesList.addAll(cachedWrapperClasses);
        wrapperClassesList.sort(WrapperComparator.COMPARATOR);
        Collections.reverse(wrapperClassesList);
    }

    if (CollectionUtils.isNotEmpty(wrapperClassesList)) {
        for (Class<?> wrapperClass : wrapperClassesList) {
            Wrapper wrapper = wrapperClass.getAnnotation(Wrapper.class);
            if (wrapper == null
                    || (ArrayUtils.contains(wrapper.matches(), name) && !ArrayUtils.contains(wrapper.mismatches(), name))) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
    }
}
```

包装类是一种装饰类，其作用更多用于将扩展类的公共逻辑处理。



### 总结

本文简单介绍了 Java SPI 与 Dubbo SPI 用法，并通过源码分析了 Dubbo SPI 加载拓展类的过程，总的来说，

1. Dubbo实现了自己的一套SPI机制，可以按需进行加载；
2. 在Dubbo中可以通过**@Adaptive**注解创建自适应扩展类，根据传入的URL参数加载具体的扩展实现；
3. Dubbo可以对创建的扩展实例进行依赖注入，但只针对setter方法进行注入，目前存在两种注入实现，分别为**SpiExtensionFactory**和**SpringExtensionFactory**;
4. Dubbo可以通过包装类实现对扩展类的装饰；