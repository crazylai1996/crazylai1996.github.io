---
title: Spring Boot源码学习：事务生效原理
date: 2024-04-13 13:50:43
tags: Spring Boot

---



### 前言

我们都知道，在使用Spring进行项目开发时，开启事务只需要在类或者上标注一个**@Transactional**注解即可实现，那么这个是怎么做到的呢？

> 以下内容基于Spring Boot 2.1.9.RELEASE版本

探究其实现原理之前，我们先来看一个**@EnableTransactionManagement**注解。

<!--more-->

### 从@EnableTransactionManagement说起

```Java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {

	boolean proxyTargetClass() default false;

	AdviceMode mode() default AdviceMode.PROXY;

	int order() default Ordered.LOWEST_PRECEDENCE;

}

```

可以看到，该注解导入了**TransactionManagementConfigurationSelector**类：

```
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}

	//...

}
```

而在默认的PROXY模式下，**TransactionManagementConfigurationSelector**类注册了**AutoProxyRegistrar**和**ProxyTransactionManagementConfiguration**类：



#### AutoProxyRegistrar

通过深入**AutoProxyRegistrar**可以看到，该类最终又调用了**AopConfigUtils#registerAutoProxyCreatorIfNecessary**注册了**InfrastructureAdvisorAutoProxyCreator**：

```
public abstract class AopConfigUtils {
	private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>(3);

	static {
		// Set up the escalation list...
		APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
	}
	//...
	@Nullable
	public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
		return registerAutoProxyCreatorIfNecessary(registry, null);
	}
	
	public static BeanDefinition registerAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
	}
	
	@Nullable
	private static BeanDefinition registerOrEscalateApcAsRequired(
			Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

		//如果该bean已存在，将根据APC_PRIORITY_LIST中定义的优先级进行加载/替换
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
				int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
				int requiredPriority = findPriorityForClass(cls);
				if (currentPriority < requiredPriority) {
					apcDefinition.setBeanClassName(cls.getName());
				}
			}
			return null;
		}

		RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
		beanDefinition.setSource(source);
		beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
		return beanDefinition;
	}
}
```

那么，**InfrastructureAdvisorAutoProxyCreator**有什么作用呢

```
/**
 * Auto-proxy creator that considers infrastructure Advisor beans only,
 * ignoring any application-defined Advisors.
 *
 * @author Juergen Hoeller
 * @since 2.0.7
 */
@SuppressWarnings("serial")
public class InfrastructureAdvisorAutoProxyCreator extends AbstractAdvisorAutoProxyCreator {

	@Nullable
	private ConfigurableListableBeanFactory beanFactory;


	@Override
	protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		super.initBeanFactory(beanFactory);
		this.beanFactory = beanFactory;
	}

	@Override
	protected boolean isEligibleAdvisorBean(String beanName) {
		return (this.beanFactory != null && this.beanFactory.containsBeanDefinition(beanName) &&
				this.beanFactory.getBeanDefinition(beanName).getRole() == BeanDefinition.ROLE_INFRASTRUCTURE);
	}

}
```

从代码注释中可以看出，这是一个代理对象创建器。

了解过**@EnableAspectJAutoProxy**注解原理的可能会发现，这俩走的是同一个注册方法，而对于同时开启**EnableAspectJAutoProxy**和**EnableTransactionManagement**的情况下，在注册的时候会根据定义的优先级**AnnotationAwareAspectJAutoProxyCreator > AspectJAwareAdvisorAutoProxyCreator > InfrastructureAdvisorAutoProxyCreator**，保留其中一个，通过对比其实现可以知道，**AnnotationAwareAspectJAutoProxyCreator**具有的功能更全面。

#### ProxyTransactionManagementConfiguration

```
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		advisor.setAdvice(transactionInterceptor());
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}

}
```

**ProxyTransactionManagementConfiguration**注册了3个bean。

##### transactionAdvisor

```
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

	@Nullable
	private TransactionAttributeSource transactionAttributeSource;

	private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
		@Override
		@Nullable
		protected TransactionAttributeSource getTransactionAttributeSource() {
			return transactionAttributeSource;
		}
	};


	/**
	 * Set the transaction attribute source which is used to find transaction
	 * attributes. This should usually be identical to the source reference
	 * set on the transaction interceptor itself.
	 * @see TransactionInterceptor#setTransactionAttributeSource
	 */
	public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
		this.transactionAttributeSource = transactionAttributeSource;
	}

	/**
	 * Set the {@link ClassFilter} to use for this pointcut.
	 * Default is {@link ClassFilter#TRUE}.
	 */
	public void setClassFilter(ClassFilter classFilter) {
		this.pointcut.setClassFilter(classFilter);
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

}
```

```Java
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {

	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		if (TransactionalProxy.class.isAssignableFrom(targetClass) ||
				PlatformTransactionManager.class.isAssignableFrom(targetClass) ||
				PersistenceExceptionTranslator.class.isAssignableFrom(targetClass)) {
			return false;
		}
		TransactionAttributeSource tas = getTransactionAttributeSource();
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}
}
```

从**BeanFactoryTransactionAttributeSourceAdvisor**的类继承关系图可以看出，它是一个事务增强器，并且其切入点实现依赖于**TransactionAttributeSource**，在这里的具体实现是**AnnotationTransactionAttributeSource**，其关键的方法是**TransactionAttributeSource#getTransactionAttribute**。

##### AnnotationTransactionAttributeSource

```
/**
This class reads Spring's JDK 1.5+ Transactional annotation and exposes corresponding transaction attributes to Spring's transaction infrastructure. Also supports JTA 1.2's javax.transaction.Transactional and EJB3's javax.ejb.TransactionAttribute annotation (if present). This class may also serve as base class for a custom TransactionAttributeSource, or get customized through TransactionAnnotationParser strategies.
**/
public class AnnotationTransactionAttributeSource extends AbstractFallbackTransactionAttributeSource
		implements Serializable {
		//...
}
```

从该类的注释可以看出，该类主要用于解析**@Transactional**注解，并将其属性提供给事务框架使用，而从其构造函数可以知道，它使用了**SpringTransactionAnnotationParser**作为事务属性配置的解析器。



##### transactionInterceptor

![image-20240413174621857](http://storage.laixiaoming.space/blog/image-20240413174621857.png)

从该类的继承关系图可以看出，**TransactionInterceptor**实现了**MethodInterceptor**，是一个方法拦截器，不验证猜出事务是行为是在这个类中实现的，接下来我们具体看下**TransactionInterceptor** 到底做了什么。



### TransactionInterceptor工作原理

**invoke**是**MethodInterceptor**的核心方法，其主要逻辑在**invokeWithinTransaction**体现：

```Java
	@Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
```



#### invokeWithinTransaction事务整体流程

```
	@Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		TransactionAttributeSource tas = getTransactionAttributeSource();
		//获取@Transaction属性配置
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		//获取事务管理器
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			//创建事务
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);

			Object retVal;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				//回滚事务
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			//提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			//...
		}
	}

```

可以看出，这里使用的是环绕通知处理器，体现了事务创建-提交-回滚的整个轮廓。



#### createTransactionIfNecessary创建事务

```Java
	protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				//获取事务状态
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
		
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```

##### 事务的传播机制

```Java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
	Object transaction = doGetTransaction();

	// Cache debug flag to avoid repeated checks.
	boolean debugEnabled = logger.isDebugEnabled();

	if (definition == null) {
		// Use defaults if no transaction definition given.
		definition = new DefaultTransactionDefinition();
	}
	
	//是否存在当前事务
	if (isExistingTransaction(transaction)) {
		// Existing transaction found -> check propagation behavior to find out how to behave.
		return handleExistingTransaction(definition, transaction, debugEnabled);
	}

	// Check definition settings for new transaction.
	if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
		throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
	}

	//当前事务不存在，则根据事务传播行为做不同处理
	// No existing transaction found -> check propagation behavior to find out how to proceed.
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
		throw new IllegalTransactionStateException(
				"No existing transaction found for transaction marked with propagation 'mandatory'");
	}
	else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
			definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		//挂起当前事务，当前事务为不存在，不需要挂起
		SuspendedResourcesHolder suspendedResources = suspend(null);
		if (debugEnabled) {
			logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
		}
		try {
			boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
			DefaultTransactionStatus status = newTransactionStatus(
					definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
			doBegin(transaction, definition);
			prepareSynchronization(status, definition);
			return status;
		}
		catch (RuntimeException | Error ex) {
			resume(null, suspendedResources);
			throw ex;
		}
	}
	else {
		// Create "empty" transaction: no actual transaction, but potentially synchronization.
		if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
			logger.warn("Custom isolation level specified but no actual transaction initiated; " +
					"isolation level will effectively be ignored: " + definition);
		}
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
	}
}
```

从getTransaction代码处理上看，在不存在当前事务的情况下，事务传播行为：

1. 是**MANDATORY**类型时，则直接抛出异常，因为此时还没有事务
2. 是**REQUIRED**、**REQUIRES_NEW**、**NESTED**类型时，则创建一个新的事务
3. 其余情况，返回空事务

而在存在当前事务的情况下，**handleExistingTransaction**方法将被执行：

```Java
private TransactionStatus handleExistingTransaction(
		TransactionDefinition definition, Object transaction, boolean debugEnabled)
		throws TransactionException {
	//传播行为是NEVER，则不允许执行
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
		throw new IllegalTransactionStateException(
				"Existing transaction found for transaction marked with propagation 'never'");
	}
	//传播行为是NOT_SUPPORTED，挂起当前事务
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
		if (debugEnabled) {
			logger.debug("Suspending current transaction");
		}
		Object suspendedResources = suspend(transaction);
		boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
		return prepareTransactionStatus(
				definition, null, false, newSynchronization, debugEnabled, suspendedResources);
	}
	//传播行为是REQUIRED_NEW，挂起当前事务，并新建一个事务
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
		if (debugEnabled) {
			logger.debug("Suspending current transaction, creating new transaction with name [" +
					definition.getName() + "]");
		}
		SuspendedResourcesHolder suspendedResources = suspend(transaction);
		try {
			boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
			DefaultTransactionStatus status = newTransactionStatus(
					definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
			doBegin(transaction, definition);
			prepareSynchronization(status, definition);
			return status;
		}
		catch (RuntimeException | Error beginEx) {
			resumeAfterBeginException(transaction, suspendedResources, beginEx);
			throw beginEx;
		}
	}
	//传播行为是NESTED，创建一个保存点
	if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
		if (!isNestedTransactionAllowed()) {
			throw new NestedTransactionNotSupportedException(
					"Transaction manager does not allow nested transactions by default - " +
					"specify 'nestedTransactionAllowed' property with value 'true'");
		}
		if (debugEnabled) {
			logger.debug("Creating nested transaction with name [" + definition.getName() + "]");
		}
		if (useSavepointForNestedTransaction()) {
			// Create savepoint within existing Spring-managed transaction,
			// through the SavepointManager API implemented by TransactionStatus.
			// Usually uses JDBC 3.0 savepoints. Never activates Spring synchronization.
			DefaultTransactionStatus status =
					prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
			status.createAndHoldSavepoint();
			return status;
		}
		else {
			// Nested transaction through nested begin and commit/rollback calls.
			// Usually only for JTA: Spring synchronization might get activated here
			// in case of a pre-existing JTA transaction.
			boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
			DefaultTransactionStatus status = newTransactionStatus(
					definition, transaction, true, newSynchronization, debugEnabled, null);
			doBegin(transaction, definition);
			prepareSynchronization(status, definition);
			return status;
		}
	}

	// Assumably PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
	if (debugEnabled) {
		logger.debug("Participating in existing transaction");
	}
	if (isValidateExistingTransaction()) {
		if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
			Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
			if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
				Constants isoConstants = DefaultTransactionDefinition.constants;
				throw new IllegalTransactionStateException("Participating transaction with definition [" +
						definition + "] specifies isolation level which is incompatible with existing transaction: " +
						(currentIsolationLevel != null ?
								isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
								"(unknown)"));
			}
		}
		if (!definition.isReadOnly()) {
			if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
				throw new IllegalTransactionStateException("Participating transaction with definition [" +
						definition + "] is not marked as read-only but existing transaction is");
			}
		}
	}
	boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
	return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
}
```

##### 事务的挂起和恢复

```Java
protected final SuspendedResourcesHolder suspend(@Nullable Object transaction) throws TransactionException {
	if (TransactionSynchronizationManager.isSynchronizationActive()) {
		//记录当前的事务同步管理器信息
		List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
		try {
			Object suspendedResources = null;
			if (transaction != null) {
				//挂起当前事务
				suspendedResources = doSuspend(transaction);
			}
			String name = TransactionSynchronizationManager.getCurrentTransactionName();
			TransactionSynchronizationManager.setCurrentTransactionName(null);
			boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
			TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
			Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
			TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
			boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
			TransactionSynchronizationManager.setActualTransactionActive(false);
			return new SuspendedResourcesHolder(
					suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
		}
		catch (RuntimeException | Error ex) {
			// doSuspend failed - original transaction is still active...
			doResumeSynchronization(suspendedSynchronizations);
			throw ex;
		}
	}
	else if (transaction != null) {
		// Transaction active but no synchronization active.
		Object suspendedResources = doSuspend(transaction);
		return new SuspendedResourcesHolder(suspendedResources);
	}
	else {
		// Neither transaction nor synchronization active.
		return null;
	}
}
```



```Java
protected Object doSuspend(Object transaction) {
	DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
	txObject.setConnectionHolder(null);
	return TransactionSynchronizationManager.unbindResource(obtainDataSource());
}
```



```Java
public static Object unbindResource(Object key) throws IllegalStateException {
	Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
	Object value = doUnbindResource(actualKey);
	if (value == null) {
		throw new IllegalStateException(
				"No value for key [" + actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
	}
	return value;
}
```



```Java
private static Object doUnbindResource(Object actualKey) {
	Map<Object, Object> map = resources.get();
	if (map == null) {
		return null;
	}
	Object value = map.remove(actualKey);
	// Remove entire ThreadLocal if empty...
	if (map.isEmpty()) {
		resources.remove();
	}
	// Transparently suppress a ResourceHolder that was marked as void...
	if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
		value = null;
	}
	if (value != null && logger.isTraceEnabled()) {
		logger.trace("Removed value [" + value + "] for key [" + actualKey + "] from thread [" +
				Thread.currentThread().getName() + "]");
	}
	return value;
}
```

可以看到，事务的挂起实际是对当前事务的状态信息和数据源连接对象封装到**SuspendedResourcesHolder**对象中，并将线程本地变量保存的事务状态清除。

那什么时候进行恢复呢？通过查看代码可以发现，在事务提交或者回滚后，org.springframework.transaction.support.AbstractPlatformTransactionManager#cleanupAfterCompletion，会进行挂起事务的恢复：

```Java
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
	status.setCompleted();
	if (status.isNewSynchronization()) {
		TransactionSynchronizationManager.clear();
	}
	if (status.isNewTransaction()) {
		doCleanupAfterCompletion(status.getTransaction());
	}
	if (status.getSuspendedResources() != null) {
		if (status.isDebug()) {
			logger.debug("Resuming suspended transaction after completion of inner transaction");
		}
		Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
		resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
	}
}
```



```Java
protected final void resume(@Nullable Object transaction, @Nullable SuspendedResourcesHolder resourcesHolder)
		throws TransactionException {

	if (resourcesHolder != null) {
		Object suspendedResources = resourcesHolder.suspendedResources;
		if (suspendedResources != null) {
			doResume(transaction, suspendedResources);
		}
		List<TransactionSynchronization> suspendedSynchronizations = resourcesHolder.suspendedSynchronizations;
		if (suspendedSynchronizations != null) {
			TransactionSynchronizationManager.setActualTransactionActive(resourcesHolder.wasActive);
			TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(resourcesHolder.isolationLevel);
			TransactionSynchronizationManager.setCurrentTransactionReadOnly(resourcesHolder.readOnly);
			TransactionSynchronizationManager.setCurrentTransactionName(resourcesHolder.name);
			doResumeSynchronization(suspendedSynchronizations);
		}
	}
}
```

### 总结

1. Spring事务也是通过AOP实现，相关逻辑由**BeanFactoryTransactionAttributeSourceAdvisor**、**TransactionAttributeSource**、**TransactionInterceptor**实现；
2. 事务的挂起与恢复，实际是通过线程本地变量对当前事务信息的保存、清除和恢复；