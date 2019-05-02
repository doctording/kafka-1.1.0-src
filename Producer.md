## 基于`Future`实现消息发送的同步/异步

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements. See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License. You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package kafka.examples;

import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Properties;
import java.util.concurrent.ExecutionException;

public class Producer extends Thread {
    private final KafkaProducer<Integer, String> producer;
    private final String topic;
    private final Boolean isAsync;

    public Producer(String topic, Boolean isAsync) {
        Properties props = new Properties();
        props.put("bootstrap.servers", KafkaProperties.KAFKA_SERVER_URL + ":" + KafkaProperties.KAFKA_SERVER_PORT);
        props.put("client.id", "DemoProducer");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        producer = new KafkaProducer<>(props);
        this.topic = topic;
        this.isAsync = isAsync;
    }

    public void run() {
        int messageNo = 1;
        while (true) {
            String messageStr = "Message_" + messageNo;
            long startTime = System.currentTimeMillis();
            if (isAsync) { // Send asynchronously
                producer.send(new ProducerRecord<>(topic,
                    messageNo,
                    messageStr), new DemoCallBack(startTime, messageNo, messageStr));
            } else { // Send synchronously
                try {
                    producer.send(new ProducerRecord<>(topic,
                        messageNo,
                        messageStr)).get();
                    System.out.println("Sent message: (" + messageNo + ", " + messageStr + ")");
                } catch (InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            }
            ++messageNo;
        }
    }
}

class DemoCallBack implements Callback {

    private final long startTime;
    private final int key;
    private final String message;

    public DemoCallBack(long startTime, int key, String message) {
        this.startTime = startTime;
        this.key = key;
        this.message = message;
    }

    /**
     * A callback method the user can implement to provide asynchronous handling of request completion. This method will
     * be called when the record sent to the server has been acknowledged. Exactly one of the arguments will be
     * non-null.
     *
     * @param metadata  The metadata for the record that was sent (i.e. the partition and offset). Null if an error
     *                  occurred.
     * @param exception The exception thrown during processing of this record. Null if no error occurred.
     */
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        if (metadata != null) {
            System.out.println(
                "message(" + key + ", " + message + ") sent to partition(" + metadata.partition() +
                    "), " +
                    "offset(" + metadata.offset() + ") in " + elapsedTime + " ms");
        } else {
            exception.printStackTrace();
        }
    }
}
```

## 为消息选择分区

```java
/**
 * This is used to describe per-partition state in the MetadataResponse.
 */
public class PartitionInfo {

    private final String topic;
    private final int partition;    // 分区编号
    private final Node leader;      // 分区的主副本节点
    private final Node[] replicas;  // 分区的所有副本
    private final Node[] inSyncReplicas;    // 分区中处于 ISR 的 副本
    private final Node[] offlineReplicas;
```

```java
/**
 * Partitioner Interface
 */

public interface Partitioner extends Configurable, Closeable {

    /**
     * Compute the partition for the given record.
     *
     * @param topic The topic name
     * @param key The key to partition on (or null if no key)
     * @param keyBytes The serialized key to partition on( or null if no key)
     * @param value The value to partition on or null
     * @param valueBytes The serialized value to partition on or null
     * @param cluster The current cluster metadata
     */
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);

    /**
     * This is called when partitioner is closed.
     */
    public void close();

}
```

```java
 public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
    // 获取 topic 的所有分区
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();
    if (keyBytes == null) { // 消息没有key, 就按照分区均衡分布
        int nextValue = nextValue(topic);
        List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
        if (availablePartitions.size() > 0) {
            int part = Utils.toPositive(nextValue) % availablePartitions.size();
            return availablePartitions.get(part).partition();
        } else {
            // no partitions are available, give a non-available partition
            return Utils.toPositive(nextValue) % numPartitions;
        }
    } else { // 消息有 key， 对消息散列化后取模
        // hash the keyBytes to choose a partition
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
}
```

## RecordAccumulator

生产这发送的消息现在客户端缓存到记录收集器`RecordAccumulator`中，等到一定时机再有发送线程的Sender批量的写入到Kafka集群

```java
RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs);
    if (result.batchIsFull || result.newBatchCreated) {
        log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
        this.sender.wakeup();
    }
    return result.future;
```

```java
public RecordAppendResult append(TopicPartition tp,
                                    long timestamp,
                                    byte[] key,
                                    byte[] value,
                                    Header[] headers,
                                    Callback callback,
                                    long maxTimeToBlock) throws InterruptedException {
    // We keep track of the number of appending thread to make sure we do not miss batches in
    // abortIncompleteBatches().
    appendsInProgress.incrementAndGet();
    ByteBuffer buffer = null;
    if (headers == null) headers = Record.EMPTY_HEADERS;
    try {
        // check if we have an in-progress batch
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);
        synchronized (dq) {
            if (closed)
                throw new IllegalStateException("Cannot send after the producer is closed.");
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null)
                return appendResult;
        }

        // we don't have an in-progress record batch try to allocate a new batch
        byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
        int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
        log.trace("Allocating a new {} byte message buffer for topic {} partition {}", size, tp.topic(), tp.partition());
        buffer = free.allocate(size, maxTimeToBlock);
        synchronized (dq) {
            // Need to check if producer is closed again after grabbing the dequeue lock.
            if (closed)
                throw new IllegalStateException("Cannot send after the producer is closed.");

            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null) {
                // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                return appendResult;
            }

            MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
            FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

            dq.addLast(batch);
            incomplete.add(batch);

            // Don't deallocate this buffer in the finally block as it's being used in the record batch
            buffer = null;

            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
        }
    } finally {
        if (buffer != null)
            free.deallocate(buffer);
        appendsInProgress.decrementAndGet();
    }
}
```

1. 队列中不存在批记录，进入步骤（5）
2. 如果存在旧的批记录，尝试追加到当前一条记录，并判断能否成功
3. 如果追加成功，说明已有的批记录可以容纳当前消息，返回结果
4. 如果追加时表，说明已有的批记录不可以容纳当前消息，进入下一步
5. 创建一个新的批记录，并往其中添加当前消息

##  数据发送

```java
/**
 * The background thread that handles the sending of produce requests to the Kafka cluster. This thread makes metadata
 * requests to renew its view of the cluster and then sends produce requests to the appropriate nodes.
 */
public class Sender implements Runnable {


void run(long now) {
    if (transactionManager != null) {
        try {
            if (transactionManager.shouldResetProducerStateAfterResolvingSequences())
                // Check if the previous run expired batches which requires a reset of the producer state.
                transactionManager.resetProducerId();

            if (!transactionManager.isTransactional()) {
                // this is an idempotent producer, so make sure we have a producer id
                maybeWaitForProducerId();
            } else if (transactionManager.hasUnresolvedSequences() && !transactionManager.hasFatalError()) {
                transactionManager.transitionToFatalError(new KafkaException("The client hasn't received acknowledgment for " +
                        "some previously sent messages and can no longer retry them. It isn't safe to continue."));
            } else if (transactionManager.hasInFlightTransactionalRequest() || maybeSendTransactionalRequest(now)) {
                // as long as there are outstanding transactional requests, we simply wait for them to return
                client.poll(retryBackoffMs, now);
                return;
            }

            // do not continue sending if the transaction manager is in a failed state or if there
            // is no producer id (for the idempotent case).
            if (transactionManager.hasFatalError() || !transactionManager.hasProducerId()) {
                RuntimeException lastError = transactionManager.lastError();
                if (lastError != null)
                    maybeAbortBatches(lastError);
                client.poll(retryBackoffMs, now);
                return;
            } else if (transactionManager.hasAbortableError()) {
                accumulator.abortUndrainedBatches(transactionManager.lastError());
            }
        } catch (AuthenticationException e) {
            // This is already logged as error, but propagated here to perform any clean ups.
            log.trace("Authentication exception while processing transactional request: {}", e);
            transactionManager.authenticationFailed(e);
        }
    }

    long pollTimeout = sendProducerData(now);
    client.poll(pollTimeout, now);
}
```

1. 消息被记录收集器收集，并按照分区追加到队列的最后一个批记录中
2. 发送线程通过ready()从记录收集器中找出已经准备好的服务端节点
3. 节点已经准备好，如果客户端还没有和他们建立连接，通过connect()建立到服务端的连接
4. 发送线程通过drain()从记录收集器获取按照节点整理好的每个分区的批记录
5. 发送线程得到每个节点的批记录后，为每个节点创建客户端请求，并将请求发送到服务端

## 客户端网络连接对象 NetworkClient

NetworkClient管理客户端与服务端之间的网络通信，包括连接的建立，发送客户端请求，读取客户端响应

1. ready() 方法，从记录收集器获取准备完毕的节点，并连接所有准备好的节点
2. send() 方法，为每个节点创建一个客户端请求，将请求暂存到节点对应的通道中
3. poll() 方法，轮询动作会真正的执行网络请求，比如发送请求给节点，并读取响应

