---
title: Kafka源码学习：生产者消息收集器
date: 2025-01-02 21:06:13
tags: Kafka

---



### 前言

前面我们提到，当通过 KafkaProducer#send 方法发送消息时，消息会先被写入到消息收集器—— **RecordAccumulator** 中缓存，当缓存的消息满足一定的条件后，将由 Sender 线程获取批量消息并将其发往 Kafka ，这样的好处是能够减少网络请求的次数，从而提升网络吞量。本篇文章一起来学习下 **RecordAccumulator** 的内部实现。



<!--more-->

> 以下内容基于Kafka 2.5.1版本



### append 方法

```Java
public RecordAppendResult append(TopicPartition tp,
                                 long timestamp,
                                 byte[] key,
                                 byte[] value,
                                 Header[] headers,
                                 Callback callback,
                                 long maxTimeToBlock,
                                 boolean abortOnNewBatch,
                                 long nowMs) throws InterruptedException {
    // We keep track of the number of appending thread to make sure we do not miss batches in
    // abortIncompleteBatches().
    // 记录追回消息的线程数
    appendsInProgress.incrementAndGet();
    ByteBuffer buffer = null;
    if (headers == null) headers = Record.EMPTY_HEADERS;
    try {
        // check if we have an in-progress batch
        // 获取对应主题分区的双端队列
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);
        // 同步锁
        synchronized (dq) {
            if (closed)
                throw new KafkaException("Producer closed while send in progress");
            // 尝试追加消息
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq, nowMs);	  // 添加成功则返回
            if (appendResult != null)
                return appendResult;
        }

        // we don't have an in-progress record batch try to allocate a new batch
        // 是否创建新的消息批次
        if (abortOnNewBatch) {
            // Return a result that will cause another call to append.
            return new RecordAppendResult(null, false, false, true);
        }

        byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
        // 计算批次大小，取批次大小和消息大小的最大值
        int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
        log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
        // 申请内存空间 buffer
        buffer = free.allocate(size, maxTimeToBlock);

        // Update the current time in case the buffer allocation blocked above.
        nowMs = time.milliseconds();
        // 再次加锁
        synchronized (dq) {
            // Need to check if producer is closed again after grabbing the dequeue lock.
            if (closed)
                throw new KafkaException("Producer closed while send in progress");
			// 尝试追加消息
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq, nowMs);
            if (appendResult != null) {
                // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                return appendResult;
            }

            // 构建新的消息批次 ProducerBatch
            MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, nowMs);
            FutureRecordMetadata future = Objects.requireNonNull(batch.tryAppend(timestamp, key, value, headers,
                    callback, nowMs));
			// 将消息批次放入双端队列
            dq.addLast(batch);
            incomplete.add(batch);

            // Don't deallocate this buffer in the finally block as it's being used in the record batch
            // 新申请的内存 buffer 置空
            buffer = null;
            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true, false);
        }
    } finally {
        // 如果 buffer 不为空，表明该新申请的这块 buffer 没有用上，需要释放掉
        if (buffer != null)
            free.deallocate(buffer);
        appendsInProgress.decrementAndGet();
    }
}
```

整体流程如下：

1. 获取或创建新的双端队列：

   ```Java
   private Deque<ProducerBatch> getOrCreateDeque(TopicPartition tp) {
       Deque<ProducerBatch> d = this.batches.get(tp);
       if (d != null)
           return d;
       d = new ArrayDeque<>();
       Deque<ProducerBatch> previous = this.batches.putIfAbsent(tp, d);
       if (previous == null)
           return d;
       else
           return previous;
   }
   ```

   其中 batches 类型为 CopyOnWriteMap ，线程安全类，存放了主题到消息批次队列的映射。

2. 尝试往队列中追加数据，此时会取队列的最后一个消息批次，并通过 ProducerBatch#tryAppend 进行追加：

   ```Java
   public FutureRecordMetadata tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers, Callback callback, long now) {
       // 是否超过 buffer 限制
       if (!recordsBuilder.hasRoomFor(timestamp, key, value, headers)) {
           return null;
       } else {
           // 往 buffer 追加消息
           Long checksum = this.recordsBuilder.append(timestamp, key, value, headers);
           this.maxRecordSize = Math.max(this.maxRecordSize, AbstractRecords.estimateSizeInBytesUpperBound(magic(),
                   recordsBuilder.compressionType(), key, value, headers));
           this.lastAppendTime = now;
           FutureRecordMetadata future = new FutureRecordMetadata(this.produceFuture, this.recordCount,
                                                                  timestamp, checksum,
                                                                  key == null ? -1 : key.length,
                                                                  value == null ? -1 : value.length,
                                                                  Time.SYSTEM);
           // we have to keep every future returned to the users in case the batch needs to be
           // split to several new batches and resent.
           thunks.add(new Thunk(callback, future));
           this.recordCount++;
           return future;
       }
   }
   ```

3. 计算消息批次大小（取批次大小和消息大小的最大值），申请新的内存空间 buffer；
4. 再次尝试添加消息；
5. 使用申请到的内存，构建新的消息批次，将消息追加到消息批次，同时将消息批次添加到主题分区对应的队列；
6. 释放无用的 buffer。

这里有几点值得思考：

1. 为什么对同一个锁对象，前后加了再次锁，能不能合并成一个呢？

   首先，锁的粒度控制得越小，锁竞争造成的开销也就越小；另外，申请新的内在空间 buffer 是比较耗时操作，在并发情况下，生产者的消息体也并不都是一样的。如果合并成一个同步锁，假使生产者1在发送消息时，因为消息太大而需要申请新的内存空间，而此时的消息批次剩余的内在空间却能满足生产者2发送的消息时，由于生产者1先进来，会新构建一个消息批次，后来的生产者也会使用新构建的消息批次追加消息，而可能导致空间没能很好的利用。

2. 流程的最后，有一个内存释放的操作，这个有作用是什么呢？

   同样是并发情况下，如果两个生产者在第一次尝试添加消息失败时，会构建新的消息批次，在先到来的生产者1构建完成后并将消息追加进去，生产者2这里会先再次尝试追加，如果追回成功，那么此时生产者2则不需要构建一个新的消息批次，申请的内存也自然用不上了，所以需要及时地释放。



### 内存池 BufferPool

前面我们提到，构建一个新的 ProducerBatch 前会先申请一块内存，而内存的来源就是内存池，也叫 BufferPool。

我们都知道线程池、连接池等将资源池化的最大作用就是复用，BufferPool 也不例外，其通过对内存块的复用，可以避免频繁的创建内存和销毁内存，从而避免频繁地 GC。那 BufferPool 是怎么实现的呢，首先来看下其申请内存的方法实现。

 #### allocate

```Java
public ByteBuffer allocate(int size, long maxTimeToBlockMs) throws InterruptedException {
    // 申请的内存大小是否超过总大小，默认是 32M
    if (size > this.totalMemory)
        throw new IllegalArgumentException("Attempt to allocate " + size
                                           + " bytes, but there is a hard limit of "
                                           + this.totalMemory
                                           + " on memory allocations.");

    ByteBuffer buffer = null;
    // 加锁
    this.lock.lock();

    if (this.closed) {
        this.lock.unlock();
        throw new KafkaException("Producer closed while allocating memory");
    }

    try {
        // check if we have a free buffer of the right size pooled
        // 申请的大小如果等于池化内存块的大小（默认为 16k ，也为消息批次的默认大小），并且有可用的内存池，则直接从池子取出返回
        if (size == poolableSize && !this.free.isEmpty())
            return this.free.pollFirst();

        // now check if the request is immediately satisfiable with the
        // memory on hand or if we need to block
        // 池化空间内存大小
        int freeListSize = freeSize() * this.poolableSize;
        // 如果非池化可用空间加上池化可用空间能满足目标空间
        if (this.nonPooledAvailableMemory + freeListSize >= size) {
            // we have enough unallocated or pooled memory to immediately
            // satisfy the request, but need to allocate the buffer
            // 释放池化空间到 nonPooledAvailableMemory，以确保能够满足目标空间
            freeUp(size);
            this.nonPooledAvailableMemory -= size;
        } else {
            // we are out of memory and will have to block
            // 当前 BufferPool 空间不足以满足目标空间时，需要阻塞等待可用
            int accumulated = 0;
            Condition moreMemory = this.lock.newCondition();
            try {
                // 最长等待时间
                long remainingTimeToBlockNs = TimeUnit.MILLISECONDS.toNanos(maxTimeToBlockMs);
                this.waiters.addLast(moreMemory);
                // loop over and over until we have a buffer or have reserved
                // enough memory to allocate one
                // 没有足够的空间就一直循环
                while (accumulated < size) {
                    long startWaitNs = time.nanoseconds();
                    long timeNs;
                    boolean waitingTimeElapsed;
                    try {
                        waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                    } finally {
                        long endWaitNs = time.nanoseconds();
                        timeNs = Math.max(0L, endWaitNs - startWaitNs);
                        recordWaitTime(timeNs);
                    }

                    if (this.closed)
                        throw new KafkaException("Producer closed while allocating memory");

                    if (waitingTimeElapsed) {
                        throw new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");
                    }

                    remainingTimeToBlockNs -= timeNs;

                    // check if we can satisfy this request from the free list,
                    // otherwise allocate memory
                    // 目标空间是池化内存大小，并且存在可用的内存块，则直接使用
                    if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                        // just grab a buffer from the free list
                        buffer = this.free.pollFirst();
                        accumulated = size;
                    } else {
                        // we'll need to allocate memory, but we may only get
                        // part of what we need on this iteration
                        // 继续释放池化空间到 nonPooledAvailableMemory
                        freeUp(size - accumulated);
                        int got = (int) Math.min(size - accumulated, this.nonPooledAvailableMemory);
                        this.nonPooledAvailableMemory -= got;
                        accumulated += got;
                    }
                }
                // Don't reclaim memory on throwable since nothing was thrown
                accumulated = 0;
            } finally {
                // When this loop was not able to successfully terminate don't loose available memory
                this.nonPooledAvailableMemory += accumulated;
                this.waiters.remove(moreMemory);
            }
        }
    } finally {
        // signal any additional waiters if there is more memory left
        // over for them
        try {
            if (!(this.nonPooledAvailableMemory == 0 && this.free.isEmpty()) && !this.waiters.isEmpty())
                this.waiters.peekFirst().signal();
        } finally {
            // Another finally... otherwise find bugs complains
            lock.unlock();
        }
    }

    if (buffer == null)
        // 从非池化空间 nonPooledAvailableMemory 分配内存
        return safeAllocateByteBuffer(size);
    else
        return buffer;
}
```

首先需要了解的是，BufferPool 的可用空间分为两个部分：

1. 池化内存可用区域 free（ByteBuffer 队列）：初始值为空，由固定大小 poolableSize（等于消息批次大小）的 ByteBuffer 组成；
2. 非池化可用内存大小 nonPooledAvailableMemory：初始值等于总内存大小 totalMemory；

当申请内存，按以下流程分配：

1. 如果申请的目标空间等池化内存大小 poolableSize，则优先从池化内存可用区 free 分配；
2. 如果池化区没有可用内存块，或者目标空间大小不等于池化内存大小，则从非池空间区域分配；
   1. 如果剩余空间（池化内存大小+非池化内存大小）能满足目标空间，则释放池化内存块，同时增加非池化内存大小，最后从非池化内存区分配；
   2. 否则，循环阻塞等待释放足够的可用空间出来；
3. 返回分配到的内存块 ByteBuffer。



#### deallocate

从 BufferPool 中分配的 ByteBuffer 内存块在使用完之后，可以通过 deallocate 方法释放：

```Java
public void deallocate(ByteBuffer buffer, int size) {
    lock.lock();
    try {
        // 如果待释放的 ByteBuffer 大小为 poolableSize ,则放入池化内存区
        if (size == this.poolableSize && size == buffer.capacity()) {
            buffer.clear();
            this.free.add(buffer);
        } else {
            // 否则放入非池化内存区 nonPooledAvailableMemory ，ByteBuffer 就等待自动回收了
            this.nonPooledAvailableMemory += size;
        }
        Condition moreMem = this.waiters.peekFirst();
        if (moreMem != null)
            moreMem.signal();
    } finally {
        lock.unlock();
    }
}
```

总的来说，释放的 ByteBuffer 大小如果满足池化大小，则放入池化内存区域重复利用，否则增加非池化内存大小，而释放的 ByteBuffer 则等待 JVM 的 GC 进行自动回收。



### 总结

本文针对 Kafka 客户端的 RecordAccumulator 消息收集器的 append 添加消息方法的流程作了简单的分析学习，又借此引出了 BufferPool 内存池，并对申请内存和释放内存作了简单的分析学习，了解到 BufferPool 包括两个部分 free 和nonPooledAvailableMemory，而能够复用的是 free 这一部分。