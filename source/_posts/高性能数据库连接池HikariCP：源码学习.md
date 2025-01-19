---
title: 高性能数据库连接池HikariCP：源码学习
date: 2025-01-18 14:19:03
tags: HikariCP
---



### 前言

数据库连接池是在项目开发中广泛应用的一项技术，它用于分配、管理、释放数据库连接，允许应用程序复用现有的数据库连接，而不是重新建立一个。

为什么使用数据库连接池呢？首先数据库连接的建立是一种耗时长、性能低、代价高的一个操作，频繁地建立和关闭连接会影响系统的性能，特别是高并发的情况下。另外，数据库同时支持的连接总数也是有限的，而连接池的出现则很好地避免了以上问题。

HikariCP数据库连接池是号称性能最好的一个连接池，也是Spring Boot 2.0 选择的默认数据库连接池，“快速、轻量、可靠”是其显著的特点。

本文结合 HikariCP 数据库连接池的源码，一起来学习下。

<!--more-->

> 以下内容基于 HikariCP 4.0.3版本



### HikariCP 为什么这么快

#### ConcurrentBag：更好的并发集合类实现

ConcurrentBag 是 HikarkCP 专门为连接池设计的一个无锁并发集合类。它主要包含几个属性：

- sharedList：类型为 CopyOnWriteArrayList，存放着状态为使用中、未使用和保留三种状态的资源对象；
- threadList：类型为 ThreadLocal<List<Object>>，存放着当前线程归还的资源对象；
- handoffQueue：类型为 SynchronousQueue，无容量队列，用于给等待的线程提供资源；
- listener：通知连接池新建连接资源。



添加资源对象，ConcurrentBag#add：

```Java
public void add(final T bagEntry)
{
   if (closed) {
      LOGGER.info("ConcurrentBag has been closed, ignoring add()");
      throw new IllegalStateException("ConcurrentBag has been closed, ignoring add()");
   }
	
   // 添加新资源到 sharedList
   sharedList.add(bagEntry);

   // 如果存在等待的线程，则直接交给等待线程
   // spin until a thread takes it or none are waiting
   while (waiters.get() > 0 && bagEntry.getState() == STATE_NOT_IN_USE && !handoffQueue.offer(bagEntry)) {
      Thread.yield();
   }
}
```



删除资源对象，ConcurrentBag#remove：

```Java
public boolean remove(final T bagEntry)
{
   // 如果不是使用中，或被预定状态，则返回失败
   if (!bagEntry.compareAndSet(STATE_IN_USE, STATE_REMOVED) && !bagEntry.compareAndSet(STATE_RESERVED, STATE_REMOVED) && !closed) {
      LOGGER.warn("Attempt to remove an object from the bag that was not borrowed or reserved: {}", bagEntry);
      return false;
   }
   // 从 sharedList 中移除
   final boolean removed = sharedList.remove(bagEntry);
   if (!removed && !closed) {
      LOGGER.warn("Attempt to remove an object from the bag that does not exist: {}", bagEntry);
   }
   // 从 threadList 中移除
   threadList.get().remove(bagEntry);

   return removed;
}
```

remove 仅在 borrow 和 resreve 方法中被使用，前者是在获取资源对象后通过探活发现连接已失效时，将其关闭，后者用于在对空闲连接清理的时候。



归还资源对象，ConcurrentBag#requite：

```Java
public void requite(final T bagEntry)
{
   // 将资源重新设置为可用
   bagEntry.setState(STATE_NOT_IN_USE);

   // 存在等待线程，则将资源直接提供给等待线程
   for (int i = 0; waiters.get() > 0; i++) {
      if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) {
         return;
      }
      else if ((i & 0xff) == 0xff) {
         parkNanos(MICROSECONDS.toNanos(10));
      }
      else {
         Thread.yield();
      }
   }

   // 否则，将资源存入线程本地
   final List<Object> threadLocalList = threadList.get();
   if (threadLocalList.size() < 50) {
      threadLocalList.add(weakThreadLocals ? new WeakReference<>(bagEntry) : bagEntry);
   }
}
```



获取资源对象，ConcurrentBag#borrow：

```Java
public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException
{
   // 优先从当前线程本地的 threadList 中获取
   // Try the thread-local list first
   final List<Object> list = threadList.get();
   for (int i = list.size() - 1; i >= 0; i--) {
      final Object entry = list.remove(i);
      @SuppressWarnings("unchecked")
      final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
      // CAS 获取
      if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
         return bagEntry;
      }
   }

   // 等待获取资源的线程数+1
   // Otherwise, scan the shared list ... then poll the handoff queue
   final int waiting = waiters.incrementAndGet();
   try {
      // 从 sharedList 中获取
      for (T bagEntry : sharedList) {
         // cas 乐观锁获取
         if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            // If we may have stolen another waiter's connection, request another bag add.
            // 如果从 sharedList 中获取到了资源，但是存在等待线程，则是抢占了其他线程的资源，需要再申请补充一个资源
             if (waiting > 1) {
               listener.addBagItem(waiting - 1);
            }
            return bagEntry;
         }
      }

      // 如果 sharedList 中没有获取到资源，则申请补充一个资源
      listener.addBagItem(waiting);

      // 通过 handoffQueue 等待资源投放
      timeout = timeUnit.toNanos(timeout);
      do {
         final long start = currentTime();
         final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
         if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            return bagEntry;
         }

         timeout -= elapsedNanos(start);
      } while (timeout > 10_000);

      return null;
   }
   finally {
      // 等待获取资源的线程数-1
      waiters.decrementAndGet();
   }
}
```

在获取资源时：

1. 首先尝试从线程本地 threadList 中获取属于当前线程的资源对象，如果有，则返回；

2. 如果线程本地中没有可用资源，则再次从共享的 sharedList 中获取，sharedList 类型是 CopyOnWriteArrayList ，它的读取方法也是无锁的，适合读多写少的场景；
3. 如果 sharedList 中也没有可用的资源，则进行等待。

在从 threadList 中获取资源时，同样通过 CAS 去获取，初看可能会感到奇怪，既然是线程本地资源，为什么还需要通过 CAS 去控制并发呢？实际上 HikariCP 数据库连接池中所有资源都存在于 sharedList 中，线程本地存储中的资源和 sharedList 中的资源可能指向同一个， 所以需要用CAS方法防止并发情况下的重复获取。

可以看到，不管是新增、删除，还是获取资源，都是通过 CAS 去实现的，而这也正是 HikariCP 数据库连接池速度快的主要原因。



#### 使用 FastList 替代 ArrayList

FastList 是一个 List 接口的精简实现，只实现了接口中必要的几个方法。相比于 ArrayList，FastList 

- **去掉了索引范围检查**：FastList 的 get 方法比 ArrayList 少一个 `rangeCheck(index)` ，这行代码的作用是范围检查，在性能上的追求可以说是极致了；
- **尾部删除**：FastList 的 remove 方法是从尾部开始扫描的，而 ArrayList 是从头部开始扫描的。因为Connection的打开和关闭顺序通常是相反的，从尾部开始扫描可以更快的找到元素。另外，FastList 的根据下标删除方法也去掉索引范围检查。



#### fastPathPool 和 pool

在 HikariDataSource 中，存在两个 HikariPool 的引用：

```Java
private final HikariPool fastPathPool;
private volatile HikariPool pool;
```

一个 final 修饰，另外一个则被 volatile 修饰，在获取连接时：

```Java
public Connection getConnection() throws SQLException
{
   if (isClosed()) {
      throw new SQLException("HikariDataSource " + this + " has been closed.");
   }
   // 优先通过 fastPathPool 获取
   if (fastPathPool != null) {
      return fastPathPool.getConnection();
   }

   // See http://en.wikipedia.org/wiki/Double-checked_locking#Usage_in_Java
   HikariPool result = pool;
   // 双检锁
   if (result == null) {
      synchronized (this) {
         result = pool;
         if (result == null) {
            validate();
            LOGGER.info("{} - Starting...", getPoolName());
            try {
               pool = result = new HikariPool(this);
               this.seal();
            }
            catch (PoolInitializationException pie) {
               if (pie.getCause() instanceof SQLException) {
                  throw (SQLException) pie.getCause();
               }
               else {
                  throw pie;
               }
            }
            LOGGER.info("{} - Start completed.", getPoolName());
         }
      }
   }

   return result.getConnection();
}
```

优先通过 fastPathPool 获取。这两个有什么区别呢？

从性能方面看，被 volatile 修饰的变量，相较于 final 变量，由于需要保证线程间的内存可见性，会多许多额外的操作，在效率上自然也比不过 final 变量。

另外，查看其引用，可以看到它有两种初始化方式：

1. 无参构造方法：

   ```Java
   public HikariDataSource()
   {
      super();
      fastPathPool = null;
   }
   ```

​	在该情况下，fastPathPool 引用为 null，而 pool 的初始化则是在 getConnection 方法中，采用懒加载的方式。

2. 有参构造方法：

   ```Java
   public HikariDataSource(HikariConfig configuration)
   {
      configuration.validate();
      configuration.copyStateTo(this);
   
      LOGGER.info("{} - Starting...", configuration.getPoolName());
      pool = fastPathPool = new HikariPool(this);
      LOGGER.info("{} - Start completed.", configuration.getPoolName());
   
      this.seal();
   }
   ```

   此时，fastPathPool 和 pool 都是指向同一个数据源。所以在使用 HikariCP 数据源时，更推荐使用有参构造方法。

   

### 连接管理

HikariCP 数据库连接池的连接管理通过 HikariPool 完成。



#### 获取连接

```Java
public Connection getConnection(final long hardTimeout) throws SQLException
{
   suspendResumeLock.acquire();
   final long startTime = currentTime();

   try {
      long timeout = hardTimeout;
      do {
         // 从 ConcurrentBag 中借用连接
         PoolEntry poolEntry = connectionBag.borrow(timeout, MILLISECONDS);
         // 超时了，跳出循环，抛出异常
         if (poolEntry == null) {
            break; // We timed out... break and throw exception
         }

         final long now = currentTime();
         // 连接的有效性检查
         if (poolEntry.isMarkedEvicted() || (elapsedMillis(poolEntry.lastAccessed, now) > aliveBypassWindowMs && !isConnectionAlive(poolEntry.connection))) {
            // 关闭失效连接
            closeConnection(poolEntry, poolEntry.isMarkedEvicted() ? EVICTED_CONNECTION_MESSAGE : DEAD_CONNECTION_MESSAGE);
            timeout = hardTimeout - elapsedMillis(startTime);
         }
         else {
            // 对获取到的连接对象，生成代理对象并返回
            metricsTracker.recordBorrowStats(poolEntry, startTime);
            return poolEntry.createProxyConnection(leakTaskFactory.schedule(poolEntry), now);
         }
      } while (timeout > 0L);

      metricsTracker.recordBorrowTimeoutStats(startTime);
      throw createTimeoutException(startTime);
   }
   catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      throw new SQLException(poolName + " - Interrupted during connection acquisition", e);
   }
   finally {
      suspendResumeLock.release();
   }
}
```



#### 创建连接

在获取不到连接资源时，会通过 HikariPool#addBagItem 通知 HikariPool 创建连接资源：

```
public void addBagItem(final int waiting)
{
   // 是否超过连接池大小
   final boolean shouldAdd = waiting - addConnectionQueueReadOnlyView.size() >= 0; // Yes, >= is intentional.
   if (shouldAdd) {
      addConnectionExecutor.submit(poolEntryCreator);
   }
   else {
      logger.debug("{} - Add connection elided, waiting {}, queue {}", poolName, waiting, addConnectionQueueReadOnlyView.size());
   }
}
```

提交了一个异步任务，来到 HikariPool.PoolEntryCreator#call：

```Java
public Boolean call()
{
   long sleepBackoff = 250L;
   // 连接池状态正常并且需求创建连接时
   while (poolState == POOL_NORMAL && shouldCreateAnotherConnection()) {
      final PoolEntry poolEntry = createPoolEntry();
      if (poolEntry != null) {
         // 将资源对象添加到 ConcurrentBag 对象中的 sharedList 中
         connectionBag.add(poolEntry);
         logger.debug("{} - Added connection {}", poolName, poolEntry.connection);
         if (loggingPrefix != null) {
            logPoolState(loggingPrefix);
         }
         return Boolean.TRUE;
      }

      // 休眠一定时间，然后重试
      // failed to get connection from db, sleep and retry
      if (loggingPrefix != null) logger.debug("{} - Connection add failed, sleeping with backoff: {}ms", poolName, sleepBackoff);
      quietlySleep(sleepBackoff);
      sleepBackoff = Math.min(SECONDS.toMillis(10), Math.min(connectionTimeout, (long) (sleepBackoff * 1.5)));
   }

   // Pool is suspended or shutdown or at max size
   return Boolean.FALSE;
}
```



#### 归还连接

在使用完连接后进行释放时，通过 ProxyConnection#close：

```Java
public final void close() throws SQLException
{
   // 关闭 statements
   // Closing statements can cause connection eviction, so this must run before the conditional below
   closeStatements();

   if (delegate != ClosedConnection.CLOSED_CONNECTION) {
      leakTask.cancel();

      try {
         // 存在未提交的事务，并且未开启自动提交，则进行回滚
         if (isCommitStateDirty && !isAutoCommit) {
            delegate.rollback();
            lastAccess = currentTime();
            LOGGER.debug("{} - Executed rollback on connection {} due to dirty commit state on close().", poolEntry.getPoolName(), delegate);
         }

         if (dirtyBits != 0) {
            poolEntry.resetConnectionState(this, dirtyBits);
            lastAccess = currentTime();
         }

         delegate.clearWarnings();
      }
      catch (SQLException e) {
         // when connections are aborted, exceptions are often thrown that should not reach the application
         if (!poolEntry.isMarkedEvicted()) {
            throw checkException(e);
         }
      }
      finally {
         delegate = ClosedConnection.CLOSED_CONNECTION;
         // 回收连接
         poolEntry.recycle(lastAccess);
      }
   }
}
```

最终在回收连接时，是通过 ConcurrentBag#requite 方法完成的。



### 总结

HikariCP 数据库连接池对数据结构的使用可谓是“知人善用”，ConcurrentBag 和 FastList 都非常适合资源的池化分配，前者其通过 CAS 替换了传统的重量级锁，并通过 ThreadLocal 将资源本地化，减少了共享资源的竞争；另外就是“细微之处见真章”，虽然可能都是一些不被人关注和在乎的小优化，但累加起来对性能却能有很大的提升。

