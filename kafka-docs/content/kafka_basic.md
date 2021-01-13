# kafka基础

## kafka架构

* Kafka中消息是以topic进行分类的，生产者生产消息，消费者消费消息，都是面向topic的
* Producer将每一条消息顺序IO追加到partition对应到log文件末尾
    * 每条消息都有一个offset
    * Partition是用来存储数据的，但并不是最小的数据存储单元。Partition下还可以细分成Segment，每个Partition是由一个或多个Segment组成。每个Segment分别对应两个文件：一个是以.index结尾的索引文件，另一个是以.log结尾的数据文件，且两个文件的文件名完全相同。所有的Segment均存在于所属Partition的目录下。
    * 消息按规则路由到不同的partition中，每个partition内部消息是有序的，但是不同partition之间不是有序的
* Producer生产消息push到Kafka集群；Consumer通过pull的方式从Kafka集群拉取消息

## kafka基础概念

### Replica（副本）

Kafka 定义了两类副本：领导者副本（Leader Replica）和追随者副本（Follower Replica）

* 前者对外提供服务，这里的对外指的是与客户端程序进行交互；
* 后者只是被动地追随领导者副本而已，不能与外界进行交互。当然，你可能知道在很多其他系统中追随者副本是可以对外提供服务的，比如 MySQL 的从库是可以处理读操作的，但是在 Kafka 中追随者副本不会对外提供服务。

### AR && ISR

* 分区中的所有副本统称为`AR`(Assigned Replicas)。

* 所有与leader副本保持一定程度同步的副本（包括leader副本在内）组成`ISR`(In Sync Replicas)。 

* ISR 集合是 AR 集合的一个子集。消息会先发送到leader副本，然后follower副本才能从leader中拉取消息进行同步。同步期间，follower副本相对于leader副本而言会有一定程度的滞后。前面所说的 ”一定程度同步“ 是指可忍受的滞后范围，这个范围可以通过参数进行配置

* leader副本同步滞后过多的副本（不包括leader副本）将组成`OSR` （Out-of-Sync Replied）

* AR = ISR + OSR。正常情况下，所有的follower副本都应该与leader 副本保持 一定程度的同步，即AR=ISR，OSR集合为空


0.9.0.0 版本之前判断副本之间是否同步，主要是靠参数`replica.lag.max.messages`决定的，即允许 follower 副本落后 leader 副本的消息数量，超过这个数量后，follower 会被踢出 ISR。`replica.lag.max.messages`也很难在生产上给出一个合理值，如果给的小，会导致 follower 频繁被踢出 ISR，如果给的大，broker 发生宕机导致 leader 变更时，可能会发生日志截断，导致消息严重丢失的问题。

在 0.9.0.0 版本之后，Kafka 给出了一个更好的解决方案，去除了`replica.lag.max.messages`，用`replica.lag.time.max.ms`参数来代替，该参数的意思指的是允许 follower 副本不同步消息的最大时间值，即只要在`replica.lag.time.max.ms`时间内 follower 有同步消息，即认为该 follower 处于 ISR 中，这就很好地避免了在某个瞬间生产者一下子发送大量消息到 leader 副本导致该分区 ISR 频繁收缩与扩张的问题了。

### 伸缩性(broker可扩展性，分布式)

虽然有了副本机制可以保证数据的持久化或消息不丢失，但没有解决伸缩性的问题。伸缩性即所谓的 Scalability，，是分布式系统中非常重要且必须要谨慎对待的问题。什么是伸缩性呢？我们拿副本来说，虽然现在有了领导者副本和追随者副本，但倘若领导者副本积累了太多的数据以至于单台 Broker 机器都无法容纳了，此时应该怎么办呢？一个很自然的想法就是，能否把数据分割成多份保存在不同的 Broker 上？如果你就是这么想的，那么恭喜你，Kafka 就是这么设计的。这种机制就是所谓的分区（Partitioning）

### Kafka的持久化

Kafka直接将数据写入到日志文件中，以追加的形式写入，避免随机访问磁盘，而是`顺序读写`方式

1. 一个Topic可以认为是一类消息，每个topic将被分成多partition(区)，每个partition在存储层面是append log文件。任何发布到此partition的消息都会被直接追加到log文件的尾部，每条消息在文件中的位置称为offset（偏移量），partition是以文件的形式存储在文件系统中
2. Logs文件根据broker中的配置要求，保留一定时间后删除来释放磁盘空间
