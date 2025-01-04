---
title: Kafka源码学习：Sender 线程
date: 2025-01-04 16:22:35
tags: Kafka
---



### 前言

在前面我们学习到 Kafka 生产者在发送时，消息会先流入到消息收集器 RecordAccumulator，随后再由另外的线程—— Sender 线程将累积的消息发往 Kafka 服务端。本篇文章一起学习下 Sender 线程的工作流程。

Sender 线程的启动可以在 KafkaProducer 的构造函数中找到：

```Java
//...
this.sender = newSender(logContext, kafkaClient, this.metadata);
String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;
this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
this.ioThread.start();
//...
```

<!--more-->

> 以下内容基于Kafka 2.5.1版本

它实现 Runnable 接口，而其主要逻辑也包含在 run 方法中：

```Java
//...
while (running) {
    try {
        runOnce();
    } catch (Exception e) {
        log.error("Uncaught error in kafka producer I/O thread: ", e);
    }
}
//... 
```

run 方法中会循环调用 runOnce 方法，我们直接来看 runOnce 方法的实现。



### runOnce 方法

```Java
void runOnce() {
    // 省略事务消息处理逻辑

    long currentTimeMs = time.milliseconds();
    // 创建发送到 kafka 的请求
    long pollTimeout = sendProducerData(currentTimeMs);
    // 将请求发送出去，同时处理收到的响应
    client.poll(pollTimeout, currentTimeMs);
}
```



### sendProducerData 请求发起

```Java
private long sendProducerData(long now) {
    // 1. 获取缓存的 kafka 集群元数据
    Cluster cluster = metadata.fetch();
    // get the list of partitions with data ready to send
    // 2. 获取可发送请求的节点信息
    RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

    // if there are any partitions whose leaders are not known yet, force metadata update
    // 3. 如果待发送的主题分区中，存在 leader 不存在的分区，则更新集群元数据
    if (!result.unknownLeaderTopics.isEmpty()) {
        // The set of topics with unknown leader contains topics with leader election pending as well as
        // topics which may have expired. Add the topic again to metadata to ensure it is included
        // and request metadata update, since there are messages to send to the topic.
        for (String topic : result.unknownLeaderTopics)
            this.metadata.add(topic, now);

        log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}",
            result.unknownLeaderTopics);
        this.metadata.requestUpdate();
    }

    // remove any nodes we aren't ready to send to
    // 4. 检查节点是否可以发送请求
    Iterator<Node> iter = result.readyNodes.iterator();
    long notReadyTimeout = Long.MAX_VALUE;
    while (iter.hasNext()) {
        Node node = iter.next();
        if (!this.client.ready(node, now)) {
            iter.remove();
            notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
        }
    }

    // create produce requests
    // 5. 获取待发送的消息集合
    Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
    // 6. 将消息批次发送记录到 inFlightBatches 集合
    addToInflightBatches(batches);
    // 7. 是否保证消息发送顺序（max.in.flight.requests.per.connection 为 1），是的话则记录到 muted 集合
    if (guaranteeMessageOrder) {
        // Mute all the partitions drained
        for (List<ProducerBatch> batchList : batches.values()) {
            for (ProducerBatch batch : batchList)
                this.accumulator.mutePartition(batch.topicPartition);
        }
    }
	// 8. 获取并处理过期的消息批次
    accumulator.resetNextBatchExpiryTime();
    List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);
    List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);
    expiredBatches.addAll(expiredInflightBatches);

    // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
    // for expired batches. see the documentation of @TransactionState.resetIdempotentProducerId to understand why
    // we need to reset the producer id here.
    if (!expiredBatches.isEmpty())
        log.trace("Expired {} batches in accumulator", expiredBatches.size());
    for (ProducerBatch expiredBatch : expiredBatches) {
        String errorMessage = "Expiring " + expiredBatch.recordCount + " record(s) for " + expiredBatch.topicPartition
            + ":" + (now - expiredBatch.createdMs) + " ms has passed since batch creation";
        failBatch(expiredBatch, -1, NO_TIMESTAMP, new TimeoutException(errorMessage), false);
        if (transactionManager != null && expiredBatch.inRetry()) {
            // This ensures that no new batches are drained until the current in flight batches are fully resolved.
            transactionManager.markSequenceUnresolved(expiredBatch);
        }
    }
    sensors.updateProduceRequestMetrics(batches);

    // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
    // loop and try sending more data. Otherwise, the timeout will be the smaller value between next batch expiry
    // time, and the delay time for checking data availability. Note that the nodes may have data that isn't yet
    // sendable due to lingering, backing off, etc. This specifically does not include nodes with sendable data
    // that aren't ready to send since they would cause busy looping.
    // 9. 计算 pollTimeout 
    long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
    pollTimeout = Math.min(pollTimeout, this.accumulator.nextExpiryTimeMs() - now);
    pollTimeout = Math.max(pollTimeout, 0);
    if (!result.readyNodes.isEmpty()) {
        log.trace("Nodes with data ready to send: {}", result.readyNodes);
        // if some partitions are already ready to be sent, the select time would be 0;
        // otherwise if some partition already has some data accumulated but not ready yet,
        // the select time will be the time difference between now and its linger expiry time;
        // otherwise the select time will be the time difference between now and the metadata expiry time;
        pollTimeout = 0;
    }
    10. 预发送消息
    sendProduceRequests(batches, now);
    return pollTimeout;
}
```

其大致流程如下：

1. 获取缓存中的集群元数据信息；

2. 获取可发送请求的节点信息，这里调用的是消息收集器的 RecordAccumulator#ready 方法：

   ```Java
   public ReadyCheckResult ready(Cluster cluster, long nowMs) {
       Set<Node> readyNodes = new HashSet<>();
       // 记录下一次需要调用 ready 方法的时间间隔
       long nextReadyCheckDelayMs = Long.MAX_VALUE;
       Set<String> unknownLeaderTopics = new HashSet<>();
   	// 是否有线程正等待释放空间
       boolean exhausted = this.free.queued() > 0;
       // 遍历所有的消息批次
       for (Map.Entry<TopicPartition, Deque<ProducerBatch>> entry : this.batches.entrySet()) {
           Deque<ProducerBatch> deque = entry.getValue();
           synchronized (deque) {
               // When producing to a large number of partitions, this path is hot and deques are often empty.
               // We check whether a batch exists first to avoid the more expensive checks whenever possible.
               ProducerBatch batch = deque.peekFirst();
               if (batch != null) {
                   TopicPartition part = entry.getKey();
                   Node leader = cluster.leaderFor(part);
                   // 判断 leader 是否存在，不存在则无法发送消息，同时需要更新集群元数据信息
                   if (leader == null) {
                       // This is a partition for which leader is not known, but messages are available to send.
                       // Note that entries are currently not removed from batches when deque is empty.
                       unknownLeaderTopics.add(part.topic());
                   } else if (!readyNodes.contains(leader) && !isMuted(part, nowMs)) {
                       // 消息批次等待发送（自上一次尝试发送的时间至今）的时间
                       long waitedTimeMs = batch.waitedTimeMs(nowMs);
                       // 是否重试退避时间内
                       boolean backingOff = batch.attempts() > 0 && waitedTimeMs < retryBackoffMs;
                       // 消息发送前的最大留存时间
                       long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                       // 是否存在消息已满的批次
                       boolean full = deque.size() > 1 || batch.isFull();
                       // 消息批次在队列中是否超时
                       boolean expired = waitedTimeMs >= timeToWaitMs;
                       // 是否应该发送该消息批次
                       boolean sendable = full || expired || exhausted || closed || flushInProgress();
                       if (sendable && !backingOff) {
                           // 如果可以的话，就放入 readyNodes
                           readyNodes.add(leader);
                       } else {
                           long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                           // Note that this results in a conservative estimate since an un-sendable partition may have
                           // a leader that will later be found to have sendable data. However, this is good enough
                           // since we'll just wake up and then sleep again for the remaining time.
                           nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                       }
                   }
               }
           }
       }
       return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);
   }
   ```

   这里通过遍历所有的消息批次，为了能得到能够发送消息的节点集合，其中能满足发送条件的有以下几种：

   - 消息批次队列满了；
   - 消息批次在队列的时间超时了；
   - 有其他线程下等待释放空间；
   - 生产者关闭了；
   - 是不触发了 flush （立即发送）操作；

3.  更新集群元数据信息，这里只是作了标记；

4. 检查 readyNodes 集合里的节点是否能发送请求，通过 NetworkClient#ready 判断：

   ```Java
   public boolean ready(Node node, long now) {
       if (node.isEmpty())
           throw new IllegalArgumentException("Cannot connect to empty node " + node);
   	// 是否已经建立了连接，并且当前已发送但未响应的请求未达到上限
       if (isReady(node, now))
           return true;
   	// 初始化连接
       if (connectionStates.canConnect(node.idString(), now))
           // if we are interested in sending to a node and we don't have a connection to it, initiate one
           initiateConnect(node, now);
   
       return false;
   }
   ```

5. 获取待发送的消息集合。这里调用消息收集器 RecordAccumulator#drain 方法获取：

   ```Java
   public Map<Integer, List<ProducerBatch>> drain(Cluster cluster, Set<Node> nodes, int maxSize, long now) {
       if (nodes.isEmpty())
           return Collections.emptyMap();
   	// 收集的结果为Map＜Node，List＜ ProducerBatch＞
       Map<Integer, List<ProducerBatch>> batches = new HashMap<>();
       for (Node node : nodes) {
           List<ProducerBatch> ready = drainBatchesForOneNode(cluster, node, maxSize, now);
           batches.put(node.id(), ready);
       }
       return batches;
   }
   ```

   ```Java
   private List<ProducerBatch> drainBatchesForOneNode(Cluster cluster, Node node, int maxSize, long now) {
       int size = 0;
       // 获取当前节点的分区集合
       List<PartitionInfo> parts = cluster.partitionsForNode(node.id());
       List<ProducerBatch> ready = new ArrayList<>();
       /* to make starvation less likely this loop doesn't start at 0 */
       // drainIndex 记录了上次停止的位置，这样每次不会从0开始
       // 避免造成总是发送前几个分区的情况，造成靠后的分区的饥饿
       int start = drainIndex = drainIndex % parts.size();
       do {
           // 分区信息
           PartitionInfo part = parts.get(drainIndex);
           TopicPartition tp = new TopicPartition(part.topic(), part.partition());
           this.drainIndex = (this.drainIndex + 1) % parts.size();
   
           // Only proceed if the partition has no in-flight batches.
           // 是否需要保证发送顺序
           if (isMuted(tp, now))
               continue;
   
           Deque<ProducerBatch> deque = getDeque(tp);
           if (deque == null)
               continue;
   
           synchronized (deque) {
               // invariant: !isMuted(tp,now) && deque != null
               // 只取第一个消息批次
               ProducerBatch first = deque.peekFirst();
               if (first == null)
                   continue;
   
               // first != null
               // 是否重试避时间内
               boolean backoff = first.attempts() > 0 && first.waitedTimeMs(now) < retryBackoffMs;
               // Only drain the batch if it is not during backoff period.
               if (backoff)
                   continue;
               // 是否超过请求大小限制，是的话单次请求只发送该消息批次
               if (size + first.estimatedSizeInBytes() > maxSize && !ready.isEmpty()) {
                   // there is a rare case that a single batch size is larger than the request size due to
                   // compression; in this case we will still eventually send this batch in a single request
                   break;
               } else {
                   if (shouldStopDrainBatchesForPartition(first, tp))
                       break;
   
                   // 省略事务相关...
                   batch.close();
                   size += batch.records().sizeInBytes();
                   ready.add(batch);
   
                   batch.drained(now);
               }
           }
       } while (start != drainIndex);
       return ready;
   }
   ```

   最终实际上会将原本＜分区，Deque＜ProducerBatch＞＞的保存形式转变成＜Node，List＜ ProducerBatch＞的形式返回，因为对于网络连接来说，生产者只需要与具体的 broker 节点建立的连接，也就是向具体的 broker 节点发送消息。

6. 将待发送的消息批次记录到 inFlightBatches 中；

7. 如果需要保证消息发送顺序（max.in.flight.requests.per.connection 为 1）的话，记录对应的分区到 muted 集合；

8. 处理过期的消息批次。其中 getExpiredInflightBatches 方法用于获取 inFlightBatches 集合中已经过期的消息批次，而 expiredBatches 方法用于获取消息收集器 RecordAccumulator 中过期的消息批次；另外，这里的过期指的是消息在流入消息收集器后的过期时间（由 delivery.timeout.ms 配置指定）；获取到过期的消息批次集合后，接着逐个通过 failBatch 处理：

   ```Java
   private void failBatch(ProducerBatch batch,
                          long baseOffset,
                          long logAppendTime,
                          RuntimeException exception,
                          boolean adjustSequenceNumbers) {
       if (transactionManager != null) {
           transactionManager.handleFailedBatch(batch, exception, adjustSequenceNumbers);
       }
   
       this.sensors.recordErrors(batch.topicPartition.topic(), batch.recordCount);
   	// 超时调用回调
       if (batch.done(baseOffset, logAppendTime, exception)) {
           maybeRemoveAndDeallocateBatch(batch);
       }
   }
   ```

    这里的处理直接将该消息批次标记为完成，而后调用其中每条消息的回调方法。

9. 计算 pollTimeout，该时间也是最近一个消息批次的过期时长，也是一次 runOnce 循环等待的最长时间 ；

10. 调用 sendProduceRequests 方法：

    ```Java
    private void sendProduceRequests(Map<Integer, List<ProducerBatch>> collated, long now) {
        for (Map.Entry<Integer, List<ProducerBatch>> entry : collated.entrySet())
            sendProduceRequest(now, entry.getKey(), acks, requestTimeoutMs, entry.getValue());
    }
    ```

    逐个调用 sendProduceRequest：

    ```Java
    private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {
        if (batches.isEmpty())
            return;
    
        Map<TopicPartition, MemoryRecords> produceRecordsByPartition = new HashMap<>(batches.size());
        final Map<TopicPartition, ProducerBatch> recordsByPartition = new HashMap<>(batches.size());
    
        // find the minimum magic version used when creating the record sets
        byte minUsedMagic = apiVersions.maxUsableProduceMagic();
        for (ProducerBatch batch : batches) {
            if (batch.magic() < minUsedMagic)
                minUsedMagic = batch.magic();
        }
    	// 按分区填充 produceRecordsByPartition 和 recordsByPartition 
        for (ProducerBatch batch : batches) {
            TopicPartition tp = batch.topicPartition;
            MemoryRecords records = batch.records();
    
    		//...
            if (!records.hasMatchingMagic(minUsedMagic))
                records = batch.records().downConvert(minUsedMagic, 0, time).records();
            produceRecordsByPartition.put(tp, records);
            recordsByPartition.put(tp, batch);
        }
    
        String transactionalId = null;
        if (transactionManager != null && transactionManager.isTransactional()) {
            transactionalId = transactionManager.transactionalId();
        }
        ProduceRequest.Builder requestBuilder = ProduceRequest.Builder.forMagic(minUsedMagic, acks, timeout,
                produceRecordsByPartition, transactionalId);
        // 创建回调
        RequestCompletionHandler callback = new RequestCompletionHandler() {
            public void onComplete(ClientResponse response) {
                handleProduceResponse(response, recordsByPartition, time.milliseconds());
            }
        };
    
        String nodeId = Integer.toString(destination);
        // 创建客户端请求对象 clientRequest
        ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
                requestTimeoutMs, callback);
        // 把 clientRequest 发送给 NetworkClient ，完成消息的预发送
        client.send(clientRequest, now);
        log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);
    }
    ```

最终生成相应的客户端请求对象，并写入到 NetworkClient ，后续交由 NetworkClient 完成网络I/O。

那当收到响应后，Sender 又是如何处理的呢

​	

### handleProduceResponse 处理响应

```Java
private void handleProduceResponse(ClientResponse response, Map<TopicPartition, ProducerBatch> batches, long now) {
    RequestHeader requestHeader = response.requestHeader();
    long receivedTimeMs = response.receivedTimeMs();
    int correlationId = requestHeader.correlationId();
    // 连接失败
    if (response.wasDisconnected()) {
        log.trace("Cancelled request with header {} due to node {} being disconnected",
            requestHeader, response.destination());
        for (ProducerBatch batch : batches.values())
            completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.NETWORK_EXCEPTION), correlationId, now, 0L);
    } else if (response.versionMismatch() != null) {
        // 版本不匹配
        log.warn("Cancelled request {} due to a version mismatch with node {}",
                response, response.destination(), response.versionMismatch());
        for (ProducerBatch batch : batches.values())
            completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.UNSUPPORTED_VERSION), correlationId, now, 0L);
    } else {
        log.trace("Received produce response from node {} with correlation id {}", response.destination(), correlationId);
        // if we have a response, parse it
        // 处理正常响应
        if (response.hasResponse()) {
            ProduceResponse produceResponse = (ProduceResponse) response.responseBody();
            for (Map.Entry<TopicPartition, ProduceResponse.PartitionResponse> entry : produceResponse.responses().entrySet()) {
                TopicPartition tp = entry.getKey();
                ProduceResponse.PartitionResponse partResp = entry.getValue();
                ProducerBatch batch = batches.get(tp);
                completeBatch(batch, partResp, correlationId, now, receivedTimeMs + produceResponse.throttleTimeMs());
            }
            this.sensors.recordLatency(response.destination(), response.requestLatencyMs());
        } else {
            // this is the acks = 0 case, just complete all requests
            for (ProducerBatch batch : batches.values()) {
                completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.NONE), correlationId, now, 0L);
            }
        }
    }
}
```

对于有正常的响应，会通过 completeBatch 方法处理：

```Java
private void completeBatch(ProducerBatch batch, ProduceResponse.PartitionResponse response, long correlationId,
                           long now, long throttleUntilTimeMs) {
    Errors error = response.error;
	// 单条消息过大，将消息切割为多个批次重新入队
    if (error == Errors.MESSAGE_TOO_LARGE && batch.recordCount > 1 && !batch.isDone() &&
            (batch.magic() >= RecordBatch.MAGIC_VALUE_V2 || batch.isCompressed())) {
        // If the batch is too large, we split the batch and send the split batches again. We do not decrement
        // the retry attempts in this case.
        log.warn(
            "Got error produce response in correlation id {} on topic-partition {}, splitting and retrying ({} attempts left). Error: {}",
            correlationId,
            batch.topicPartition,
            this.retries - batch.attempts(),
            error);
        if (transactionManager != null)
            transactionManager.removeInFlightBatch(batch);
        this.accumulator.splitAndReenqueue(batch);
        maybeRemoveAndDeallocateBatch(batch);
        this.sensors.recordBatchSplit();
    } else if (error != Errors.NONE) {
        // 能否进行重试，可以的话重新入队
        if (canRetry(batch, response, now)) {
            log.warn(
                "Got error produce response with correlation id {} on topic-partition {}, retrying ({} attempts left). Error: {}",
                correlationId,
                batch.topicPartition,
                this.retries - batch.attempts() - 1,
                error);
            reenqueueBatch(batch, now);
        } else if (error == Errors.DUPLICATE_SEQUENCE_NUMBER) {
            // 顺序号重复了，直接标记成功
            // If we have received a duplicate sequence error, it means that the sequence number has advanced beyond
            // the sequence of the current batch, and we haven't retained batch metadata on the broker to return
            // the correct offset and timestamp.
            //
            // The only thing we can do is to return success to the user and not return a valid offset and timestamp.
            completeBatch(batch, response);
        } else {
            final RuntimeException exception;
            // 未授权异常，直接失败处理
            if (error == Errors.TOPIC_AUTHORIZATION_FAILED)
                exception = new TopicAuthorizationException(Collections.singleton(batch.topicPartition.topic()));
            else if (error == Errors.CLUSTER_AUTHORIZATION_FAILED)
                exception = new ClusterAuthorizationException("The producer is not authorized to do idempotent sends");
            else
                exception = error.exception();
            // tell the user the result of their request. We only adjust sequence numbers if the batch didn't exhaust
            // its retries -- if it did, we don't know whether the sequence number was accepted or not, and
            // thus it is not safe to reassign the sequence.
            failBatch(batch, response, exception, batch.attempts() < this.retries);
        }
        // 元数据异常，则更新元数据
        if (error.exception() instanceof InvalidMetadataException) {
            if (error.exception() instanceof UnknownTopicOrPartitionException) {
                log.warn("Received unknown topic or partition error in produce request on partition {}. The " +
                        "topic-partition may not exist or the user may not have Describe access to it",
                    batch.topicPartition);
            } else {
                log.warn("Received invalid metadata error in produce request on partition {} due to {}. Going " +
                        "to request metadata update now", batch.topicPartition, error.exception().toString());
            }
            metadata.requestUpdate();
        }
    } else {
        // 将消息批次标记为已完成
        completeBatch(batch, response);
    }

    // Unmute the completed partition.
    if (guaranteeMessageOrder)
        this.accumulator.unmutePartition(batch.topicPartition, throttleUntilTimeMs);
}
```

可以看到，handleProduceResponse 主要是针对各种预期异常进行区分并处理，对于正常响应的情况则调用 completeBatch 处理：

```Java
private void completeBatch(ProducerBatch batch, ProduceResponse.PartitionResponse response) {
    if (transactionManager != null) {
        transactionManager.handleCompletedBatch(batch, response);
    }

    if (batch.done(response.baseOffset, response.logAppendTime, null)) {
        maybeRemoveAndDeallocateBatch(batch);
    }
}
```

其实现也是调用batch的done()方法，然后从集合中删除batch并释放batch占用的内存。

至此，关于 Sender 线程中，消息的发送和消息的处理就先学习到这里了。

