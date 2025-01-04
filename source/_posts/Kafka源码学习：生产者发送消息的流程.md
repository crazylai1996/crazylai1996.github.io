---
title: Kafka源码学习：生产者发送消息的流程
date: 2024-12-30 22:03:06
tags: Kafka
---



我们都知道，在使用 Kakfa 客户端发送消息时，只需要指定主题和消息的内容，再调用发送方法即可。那发送方法中具体包含了哪些逻辑呢，本文结合源码一起来看下。

```Java
// 构建消息
ProducerRecord<String, String> record = new ProducerRecord<>(topic, "value");
// 发送
producer.send(record);
```

<!--more-->

> 以下内容基于Kafka 2.5.1版本

### 构建消息对象

实际上，Kafka 中的消息对象并不只有主题和消息内容两个属性：

```Java
public class ProducerRecord<K, V> {
	// 主题
    private final String topic;
    // 分区
    private final Integer partition;
    // 消息头部
    private final Headers headers;
    // 消息 key
    private final K key;
    // 消息内容
    private final V value;
    // 消息的时间
    private final Long timestamp;
    //...
}
```

其中 key 用来指定消息的键，可以用于计算分区号并且让消息发往特定的分区，同一个 key 的消息会被发往同一个分区，另外，由于 Kafka 可以保证同一个分区中的消息是有序的，相同 key 的消息的消费在一定程度上也能保证有序。



### 发送消息

构建消息后，接下来就是发送了：

```Java
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
    return send(record, null);
}
```

从 send 方法的返回值不难看出，该方法是一个异步方法，如果希望使用同步发送，则可以在拿到返回值后使用 **Future#get** 方法等发送结果。

发送结果是一个 RecordMetadata 对象：

```Java
public final class RecordMetadata {

    /**
     * Partition value for record without partition assigned
     */
    public static final int UNKNOWN_PARTITION = -1;
	// 消息的偏移量
    private final long offset;
    // The timestamp of the message.
    // If LogAppendTime is used for the topic, the timestamp will be the timestamp returned by the broker.
    // If CreateTime is used for the topic, the timestamp is the timestamp in the corresponding ProducerRecord if the
    // user provided one. Otherwise, it will be the producer local time when the producer record was handed to the
    // producer.
    // 时间戳
    private final long timestamp;
    // key 大小 
    private final int serializedKeySize;
    // 消息大小
    private final int serializedValueSize;
    // 主题和分区
    private final TopicPartition topicPartition;

    private volatile Long checksum;
    // ...
}
```

在 RecordMetadata 中记录了消息的元数据信息，如消息的主题、分区、分区中消息的偏移量、时间戳等。

另外，Kafka 也提供了具有回调参数的 send 方法：

```Java
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    // intercept the record, which can be potentially modified; this method does not throw exceptions
    // 生产者拦截器，发送前执行onSend方法
    ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

回调函数包含两个参数：

```Java
public interface Callback {
    void onCompletion(RecordMetadata metadata, Exception exception);
}
```

当发送的消息被服务端确认成功后，消息的 metadata 将被返回；当发送出现异常时，可以通过 exception 拿到具体的异常，此时 metadata 对象中除 topicPartition 外的其他属性都将是-1。

来到 doSend 方法，消息发送的主逻辑得以展现：

```Java
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
    TopicPartition tp = null;
    try {
        throwIfProducerClosed();
        // 1. 确保该 topic 的元数据可用
        // first make sure the metadata for the topic is available
        long nowMs = time.milliseconds();
        ClusterAndWaitTime clusterAndWaitTime;
        try {
            clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
        } catch (KafkaException e) {
            if (metadata.isClosed())
                throw new KafkaException("Producer closed while send in progress", e);
            throw e;
        }
        // 计算剩余等待时间
        nowMs += clusterAndWaitTime.waitedOnMetadataMs;
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
        Cluster cluster = clusterAndWaitTime.cluster;
        // 2. 序列化 key 值和 value 值
        byte[] serializedKey;
        try {
            serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in key.serializer", cce);
        }
        byte[] serializedValue;
        try {
            serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
        } catch (ClassCastException cce) {
            throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                    " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                    " specified in value.serializer", cce);
        }
        // 3. 通过分区器得到发往的分区
        int partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);

        setReadOnly(record.headers());
        Header[] headers = record.headers().toArray();
		// 4. 预估消息的大小，如果超过消息大小限制或内存限制则报错
        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                compressionType, serializedKey, serializedValue, headers);
        ensureValidRecordSize(serializedSize);
        long timestamp = record.timestamp() == null ? nowMs : record.timestamp();
        if (log.isTraceEnabled()) {
            log.trace("Attempting to append record {} with callback {} to topic {} partition {}", record, callback, record.topic(), partition);
        }
        // producer callback will make sure to call both 'callback' and interceptor callback
        Callback interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

        if (transactionManager != null && transactionManager.isTransactional()) {
            transactionManager.failIfNotReadyForSend();
        }
        // 5. 将消息缓存到消息收集器 accumulator
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);

        if (result.abortForNewBatch) {
            int prevPartition = partition;
            partitioner.onNewBatch(record.topic(), cluster, prevPartition);
            partition = partition(record, serializedKey, serializedValue, cluster);
            tp = new TopicPartition(record.topic(), partition);
            if (log.isTraceEnabled()) {
                log.trace("Retrying append due to new batch creation for topic {} partition {}. The old partition was {}", record.topic(), partition, prevPartition);
            }
            // producer callback will make sure to call both 'callback' and interceptor callback
            interceptCallback = new InterceptorCallback<>(callback, this.interceptors, tp);

            result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs, false, nowMs);
        }

        if (transactionManager != null && transactionManager.isTransactional())
            transactionManager.maybeAddPartitionToTransaction(tp);
		// 6. batch 满了或创建了新的 batch 时，唤醒 sender 线程
        if (result.batchIsFull || result.newBatchCreated) {
            log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
            this.sender.wakeup();
        }
        return result.future;
        // handling exceptions and record the errors;
        // for API exceptions return them in the future,
        // for other exceptions throw directly
    } catch (ApiException e) {
        log.debug("Exception occurred during message send:", e);
        if (callback != null)
            callback.onCompletion(null, e);
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        return new FutureFailure(e);
    } catch (InterruptedException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw new InterruptException(e);
    } catch (BufferExhaustedException e) {
        this.errors.record();
        this.metrics.sensor("buffer-exhausted-records").record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (KafkaException e) {
        this.errors.record();
        this.interceptors.onSendError(record, tp, e);
        throw e;
    } catch (Exception e) {
        // we notify interceptor about all exceptions, since onSend is called before anything else in this method
        this.interceptors.onSendError(record, tp, e);
        throw e;
    }
}
```

可以看到，发送消息主要分为几个步骤：

1. 等待获取 Kafka 集群元数据信息，这里会把最大阻塞时间参数 maxBlockTimeMs 传进去，方法返回后重新计算剩余等待时间 remainingWaitMs ；
2. 使用 Serializer 器对消息的 key 和 value 序列化；
3. 获取消息发往的目标分区 partition；
4. 预估序列化后的消息大小，如果超过了限制，则抛出异常；
5. 调用 RecordAccumulator#append 方法，将消息放入消息收集器中的消息批次；
6. 如果消息批次满了，或者创建了新的批次，则唤醒 Sender 线程，后续由 Sender 线程从 RecordAccumulator 中批量发送消息到 Kafka。



### 生产者拦截器

在 KafkaProducer 里面，维护了一个生产者拦器集合 ProducerInterceptors ：

```Java
public interface ProducerInterceptor<K, V> extends Configurable {
    public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);

    public void onAcknowledgement(RecordMetadata metadata, Exception exception);

    public void close();
}
```

其 ProducerInterceptor 接口提供了三个方法，其中 onSend 方法可以用来对发送前的消息进行相应的定制化操作；onAcknowledgement 方法先于用户的 Callback 执行，可以用于对kafka集群响应进行预处理；close 方法则可以用于在关闭拦截器时执行一些资源的清理工作。



### 获取目标分区

在获取目标分区时：

```Java
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
    Integer partition = record.partition();
    return partition != null ?
            partition :
            partitioner.partition(
                    record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
}
```

如果指定了目标分区，则使用指定的；否则通过分区器 **Partitioner#partition** 计算得出，默认的分区器由 **DefaultPartitioner** 提供实现：

```Java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    // 如果没有消息 key ,则进一步通过 stickyPartitionCache 计算得到分区
    if (keyBytes == null) {
        return stickyPartitionCache.partition(topic, cluster);
    } 
    // 如果存在消息 key，则通过用 key 值对分区数取模得到目标分区数
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    // hash the keyBytes to choose a partition
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
}
```

获取目标分区时分两种情况 ：

1. 如果消息存在 key ，则使用 murmur2 算法取 key 的 hash 值，然后对分区总数取模，得到目标分区；
2. 如果消息没有 key ，则由 StickyPartitionCache  进一步获取。

那 StickyPartitionCache 的实现是什么呢？

```Java
public int partition(String topic, Cluster cluster) {
    Integer part = indexCache.get(topic);
    if (part == null) {
        return nextPartition(topic, cluster, -1);
    }
    return part;
}
```

indexCache 中维护了 topic 到 分区的映射，如果为空的话，则通过 nextPartition 方法重新获取：

```Java
public int nextPartition(String topic, Cluster cluster, int prevPartition) {
    // 获取 topic 的分区
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    Integer oldPart = indexCache.get(topic);
    Integer newPart = oldPart;
    // Check that the current sticky partition for the topic is either not set or that the partition that 
    // triggered the new batch matches the sticky partition that needs to be changed.
    // 当前 topic 没存有对应的分区，或分区需要更换时
    if (oldPart == null || oldPart == prevPartition) {
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        // 没有可用分区
        if (availablePartitions.size() < 1) {
            Integer random = Utils.toPositive(ThreadLocalRandom.current().nextInt());
            newPart = random % partitions.size();
        } else if (availablePartitions.size() == 1) {
            // 只有一个分区
            newPart = availablePartitions.get(0).partition();
        } else {
            // 获取新的发送分区，新的发送分区不能跟之前的相同
            while (newPart == null || newPart.equals(oldPart)) {
                Integer random = Utils.toPositive(ThreadLocalRandom.current().nextInt());
                newPart = availablePartitions.get(random % availablePartitions.size()).partition();
            }
        }
        // Only change the sticky partition if it is null or prevPartition matches the current sticky partition.
        // 写入 indexCache
        if (oldPart == null) {
            indexCache.putIfAbsent(topic, newPart);
        } else {
            indexCache.replace(topic, prevPartition, newPart);
        }
        return indexCache.get(topic);
    }
    return indexCache.get(topic);
}
```

初看起来 indexCache 中只要有值，那么向同一个 topic 发送消息时会一直使用同一个分区，其实不然，在 doSend 方法中，我们可以看到当需要创建一个新的消息批次时，也会触发 nextPartition 方法的执行。那么这样做的目的是什么呢？

前面我们提到发送消息时，会先流入消息收集器 RecordAccumulator 中将消息先缓存起来，后续由 Sender 线程发送，而触发发送的条件有两种：

1. 消息批次被填满；
2. 消息发送的等待时间超过了 linger.ms 的配置。

而 StickyPartitionCache 的作用其实是 “黏性选择”，它能尽可能地将消息发往同一个分区，使消息批次能尽快的填满被发送出去，这样就可以一定程度上降低消息发送的延迟，同时也降低了发送的频次。



### 总结

本文主要对 Kafka 客户端的消息发送方法作了主要逻辑的梳理，同时对 Kafka 的默认分区器作了简单的分析，以及使用 StickyPartitionCache 带来的好处。发送流程中涉及到的 RecordAccumulator 、Sender 线程将在后续文章中一起学习下。