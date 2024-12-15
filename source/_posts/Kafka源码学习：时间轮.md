---
title: Kafka源码学习：时间轮
date: 2024-12-15 13:30:16
tags: Kafka

---



### 前言

在 Kafka 中，有许多地方都需要用到延时操作，如延时生产、延时拉取等。Kafka 作为一个高性能的消息队列，并没有使用 JDK 中自带的 DelayQueue 来实现延时的功能，而是基于时间轮的概念实现了一个更高效的数据结构。

<!--more-->

> 以下内容基于Kafka 2.5.1版本

### 时间轮简介

时间轮将时间抽象为一个环形结构，底层由数组实现，数组中的每个元素都存放了一个定时任务列表（TimerTaskList），TimerTaskList 是一个环形的双向链表，链表中的每一个节点（TimerTaskEntry）都存放着一个定时任务（TimerTask）。

时间轮有多个时间格，数组中的每个元素对应着一个时间格（bucket），每个时间格代表当前时间轮的基本时间跨度（tickMs），时间轮的时间格个数是固定的，可用 wheelSize 表示，那么可以算出每个时间轮的总时间跨度则为 interval = wheelSize * tickMs。时间轮还有一个表盘指针（currentTime），用来表示时间轮当前所处的时间，同时也表示指向的时间格任务到期需要执行，随着时间的推移，这个指针也会不断推进。

![kafka时间轮结构](http://storage.laixiaoming.space/blog/kafka时间轮结构.png)



如图中所示，假设时间格为 1 ms，一个时间轮为 8 格，则整个时间轮的最大延时时间为 8ms，那如果延时时间大于该时间轮所能表示的最大时间跨度怎么办呢？

实际上 Kafka 中的时间轮为多层级时间轮，当任务的延时时间超过当前时间轮的最大时间跨度时，会尝试将该任务添加到上层时间轮，与当前时间轮不同的是，上层时间轮的基本时间跨度（tickMs）为当前时间轮的最大时间跨度，即 8ms，此时上层的总时间跨度为 8ms * 8 = 64 ms。



### 时间轮的实现

#### TimerTask

```scala
trait TimerTask extends Runnable {
  // 定时任务超时时间
  val delayMs: Long // timestamp in millisecond
  // 关联TimerTaskEntry
  private[this] var timerTaskEntry: TimerTaskEntry = null
  // 取消定时任务
  def cancel(): Unit = {
    synchronized {
      if (timerTaskEntry != null) timerTaskEntry.remove()
      timerTaskEntry = null
    }
  }
  // 关联TimerTaskEntry
  private[timer] def setTimerTaskEntry(entry: TimerTaskEntry): Unit = {
    synchronized {
      // if this timerTask is already held by an existing timer task entry,
      // we will remove such an entry first.
      if (timerTaskEntry != null && timerTaskEntry != entry)
        timerTaskEntry.remove()

      timerTaskEntry = entry
    }
  }
  // 获取关联的TimerTaskEntry
  private[timer] def getTimerTaskEntry(): TimerTaskEntry = {
    timerTaskEntry
  }

}
```

每个 TimerTask 都关联了一个 TimerTaskEntry，通过 TimerTask 也可以知道当前任务在 TimerTaskList 中存放的位置。



#### TimerTaskEntry

```scals
private[timer] class TimerTaskEntry(val timerTask: TimerTask, val expirationMs: Long) extends Ordered[TimerTaskEntry] {

  // 当前 bucket 对应的链表
  @volatile
  var list: TimerTaskList = null
  // 后指针
  var next: TimerTaskEntry = null
  // 前指针
  var prev: TimerTaskEntry = null

  // if this timerTask is already held by an existing timer task entry,
  // setTimerTaskEntry will remove it.
  if (timerTask != null) timerTask.setTimerTaskEntry(this)

  // 当前任务是不被取消
  def cancelled: Boolean = {
    timerTask.getTimerTaskEntry != this
  }

  // 从链表中移除
  def remove(): Unit = {
    var currentList = list
    // If remove is called when another thread is moving the entry from a task entry list to another,
    // this may fail to remove the entry due to the change of value of list. Thus, we retry until the list becomes null.
    // In a rare case, this thread sees null and exits the loop, but the other thread insert the entry to another list later.
    while (currentList != null) {
      currentList.remove(this)
      currentList = list
    }
  }

  // ...
}
```

这里需要注意的是 TimerTaskList 被 volatile 修饰，因为在 Kafka 中，当上层时间轮剩余时间小于基本时间跨度（tickMs），又没到执行时间时，就会将该任务重新添加到下层时间轮中，最终由下层时间轮推进执行，因此，这里才有需要保证线程之间的内在可见性。



#### TimerTaskList

```scala
private[timer] class TimerTaskList(taskCounter: AtomicInteger) extends Delayed {

  // TimerTaskList forms a doubly linked cyclic list using a dummy root entry
  // root.next points to the head
  // root.prev points to the tail
  // dummyNode ，简化边界条件
  private[this] val root = new TimerTaskEntry(null, -1)
  root.next = root
  root.prev = root

  // TimerTaskList 的过期时间
  private[this] val expiration = new AtomicLong(-1L)
  
  // Set the bucket's expiration time
  // Returns true if the expiration time is changed
  // 设置过期时间
  def setExpiration(expirationMs: Long): Boolean = {
    expiration.getAndSet(expirationMs) != expirationMs
  }

  // Get the bucket's expiration time
  def getExpiration(): Long = {
    expiration.get()
  }

  def getDelay(unit: TimeUnit): Long = {
    unit.convert(max(getExpiration - Time.SYSTEM.hiResClockMs, 0), TimeUnit.MILLISECONDS)
  }

  def compareTo(d: Delayed): Int = {

    val other = d.asInstanceOf[TimerTaskList]

    if(getExpiration < other.getExpiration) -1
    else if(getExpiration > other.getExpiration) 1
    else 0
  }
    
  //...
}
```

TimerTaskList 实现了 Delayed 接口，这里因为在 Kafka 中，时间轮的推进是通过 DelayQueue 进行的，每个 TimerTaskList 都会添加到 DelayQueue 。在设置过期时间时，会对新旧值进行判断，因为 bucket 是可以重用的，只有更新过期时间成功后，才会将该 bucket 重新添加到 DelayQueue。



#### TimingWheel

```scala
@nonthreadsafe
private[timer] class TimingWheel(tickMs: Long, wheelSize: Int, startMs: Long, taskCounter: AtomicInteger, queue: DelayQueue[TimerTaskList]) {
  // 时间轮的总时间跨度
  private[this] val interval = tickMs * wheelSize
  // 时间格
  private[this] val buckets = Array.tabulate[TimerTaskList](wheelSize) { _ => new TimerTaskList(taskCounter) }
  // 当前时间，取小于当前时间的，最大基本时间跨度的整数倍
  private[this] var currentTime = startMs - (startMs % tickMs) // rounding down to multiple of tickMs

  // 上层时间轮
  // overflowWheel can potentially be updated and read by two concurrent threads through add().
  // Therefore, it needs to be volatile due to the issue of Double-Checked Locking pattern with JVM
  @volatile private[this] var overflowWheel: TimingWheel = null

  //...
}
```

##### 添加延时任务

```scala
  // 添加延时任务
  def add(timerTaskEntry: TimerTaskEntry): Boolean = {
    val expiration = timerTaskEntry.expirationMs
    // 任务是否取消
    if (timerTaskEntry.cancelled) {
      // Cancelled
      false
    } else if (expiration < currentTime + tickMs) {
      // 任务过期
      // Already expired
      false
    } else if (expiration < currentTime + interval) {
      // Put in its own bucket
      // 添加任务到当前时间轮
      val virtualId = expiration / tickMs
      // 定位存放的 bucket
      val bucket = buckets((virtualId % wheelSize.toLong).toInt)
      bucket.add(timerTaskEntry)

      // 设置过期时间，此时的过期时间也设置为 tickMs 时间格的整数倍
      // Set the bucket expiration time
      if (bucket.setExpiration(virtualId * tickMs)) {
        // The bucket needs to be enqueued because it was an expired bucket
        // We only need to enqueue the bucket when its expiration time has changed, i.e. the wheel has advanced
        // and the previous buckets gets reused; further calls to set the expiration within the same wheel cycle
        // will pass in the same value and hence return false, thus the bucket with the same expiration will not
        // be enqueued multiple times.
        // 添加到 DelayQueue
        queue.offer(bucket)
      }
      true
    } else {
      // Out of the interval. Put it into the parent timer
      if (overflowWheel == null) addOverflowWheel()
      // 添加任务到上层时间轮
      overflowWheel.add(timerTaskEntry)
    }
  }
  
  // 创建上层时间轮，上层时间轮的基本时间跨度为当前时间轮的总时间跨度
  private[this] def addOverflowWheel(): Unit = {
    synchronized {
      if (overflowWheel == null) {
        overflowWheel = new TimingWheel(
          tickMs = interval,
          wheelSize = wheelSize,
          startMs = currentTime,
          taskCounter = taskCounter,
          queue
        )
      }
    }
  }
```



##### 当前时间的推进

前面我们提到，时间轮的推进是通过 DelayQueue 协助完成的，每一个 TimerTaskList 都会被添加到 DelayQueue ，并根据过期时间进行排序，Kafka 会用一个后台线程来获取 DelayQueue 中到期的任务列表，这个线程为 ExpiredOperationReaper：

```scala
/**
 * A background reaper to expire delayed operations that have timed out
 */
private class ExpiredOperationReaper extends ShutdownableThread(
  "ExpirationReaper-%d-%s".format(brokerId, purgatoryName),
  false) {

  override def doWork(): Unit = {
    advanceClock(200L)
  }
}
```

SystemTimer#advanceClock：

```scala
def advanceClock(timeoutMs: Long): Boolean = {
  // 获取到期的任务
  var bucket = delayQueue.poll(timeoutMs, TimeUnit.MILLISECONDS)
  if (bucket != null) {
    writeLock.lock()
    try {
      while (bucket != null) {
        // 驱动时间轮
        timingWheel.advanceClock(bucket.getExpiration())
        // 执行到期任务
        bucket.flush(reinsert)
        // 继续获取到期任务，直到为空
        bucket = delayQueue.poll()
      }
    } finally {
      writeLock.unlock()
    }
    true
  } else {
    false
  }
}
```

TimerTaskList#flush：

```scala
def flush(f: (TimerTaskEntry)=>Unit): Unit = {
  synchronized {
    // 遍历链表
    var head = root.next
    while (head ne root) {
      // 移除
      remove(head)
      // 到期任务执行或降级时间轮
      f(head)
      // 处理下一个节点
      head = root.next
    }
    expiration.set(-1L)
  }
}
```

SystemTimer#addTimerTaskEntry：

```scala
private def addTimerTaskEntry(timerTaskEntry: TimerTaskEntry): Unit = {
  // 将任务重新添加到时间轮
  if (!timingWheel.add(timerTaskEntry)) {
    // Already expired or cancelled
    if (!timerTaskEntry.cancelled)
      // 执行到期任务
      taskExecutor.submit(timerTaskEntry.timerTask)
  }
}

private[this] val reinsert = (timerTaskEntry: TimerTaskEntry) => addTimerTaskEntry(timerTaskEntry)
```

随着时间的推进，到期任务将尝试重新添加到时间轮，此时有两种情况：

- 任务到期，或任务被取消，如果任务未取消，则执行到期任务；
- 任务未到期，任务被重新添加到下级时间轮（时间轮降级）；



### 总结

Kafka 的延时任务实现实际上没有完全抛弃 DelayQueue ，一方面采用了 环形数组 + 双向链接的数据结构，使得延时任务的插入和删除操作能达到 O(1) 的时间复杂度；另一方面，由 DelayQueue 负表时间的推进，只有 bucket 任务到期才会返回任务结果，有效减少了“空推进”的问题，并且同一个 bucket 的多个任务在 DelayQueue 也只会入队一次，避免了 DelayQueue 的开销占用过高。时间轮与 DelayQueue 两者相铺相成，能高效地管理延时任务。

