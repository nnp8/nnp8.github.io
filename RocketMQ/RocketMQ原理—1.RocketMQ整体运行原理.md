# RocketMQ原理—1.RocketMQ整体运行原理

**大纲**

**1.RocketMQ整体运行原理的介绍顺序**

**2.RocketMQ生产者是如何发送消息的**

**3.Broker是如何持久化接收到的消息到磁盘上**

**4.基于DLedger技术的Broker主从同步原理**

**5.消费者进行消息拉取和消费的过程**

**6.消费者从Master或Slave上拉取消息的策略**

**7.RocketMQ如何基于Netty进行高性能网络通信**

**8.基于mmap内存映射实现磁盘文件的高性能读写**

**9.RocketMQ的整体运行原理总结**

<br>

**1.RocketMQ整体运行原理的介绍顺序**

一.生产者往Broker集群发送消息的底层原理

二.Broker如何持久化接收到的消息到磁盘上

三.基于DLedger技术部署的Broker高可用集群如何进行数据同步

四.消费者选择Master或Slave拉取数据的策略

五.消费者如何从Broker拉取消息以及进行ACK

这个介绍顺序就基本涵盖了RocketMQ的整体运行流程，接下来首先分析RocketMQ生产者的工作原理。

<br>

**2.RocketMQ生产者是如何发送消息的**

**(1)创建Topic时为何要指定MessageQueue数量**

**(2)Topic、MessageQueue和Broker之间的关系**

**(3)生产者发送消息时写入哪个MessageQueue**

**(4)如果某个Broker出现故障了怎么办**

<br>

**(1)创建Topic时为何要指定MessageQueue数量**

介绍RocketMQ生产者的工作原理之前，需要先介绍MessageQueue的概念。

要使用RocketMQ，首先需要部署出一套RocketMQ集群。有了RocketMQ集群后，就可以根据业务需求创建出一些Topic，比如用于存放订单支付成功消息的Topic：OrderPaySuccess。

这些Topic可以通过RocketMQ可视化工作台来创建。在创建Topic时需要指定一个关键的参数——MessageQueue，表示指定这个Topic对应了多少个队列，也就是多少个MessageQueue数据分片。

<br>

**(2)Topic、MessageQueue和Broker之间的关系**

比如现在有一个Topic，已经为它指定创建了4个MessageQueue，那么这个Topic的数据在Broker集群中是如何分布的呢？

由于每个Topic的数据都是分布式存储在多个Broker中的(如下图示)，而为了决定这个Topic的哪些数据放在这个Broker上、哪些数据放在那个Broker上，所以RocketMQ引入了MessageQueue。

<img width="100%" height="100%" alt="image1" src="https://github.com/user-attachments/assets/3f0fa215-40bc-448d-83e5-fdceeb9cecb2" />

一个MessageQueue本质上就是一个数据分片。假设某个Topic有1万条数据，而且这个Topic有4个MessageQueue，那么可以认为会在每个MessageQueue中放入2500条数据。当然这不是绝对的，有可能有的MessageQueue数据多，有的数据少，这要根据消息写入MessageQueue的策略来决定。如果假设每个MessageQueue会平均分配这个Topic的数据，那么每个Broker就会有两个MessageQueue，如下图示：

<img width="100%" height="100%" alt="image2" src="https://github.com/user-attachments/assets/ae28a32e-a797-4549-9d92-ea1619aef2ca" />

通过将一个Topic的数据拆分为多个MessageQueue数据分片，然后在每个Broker上都会存储一部分MessageQueue数据分片，这样就可以实现Topic数据的分布式存储。

<br>

**(3)生产者发送消息时写入哪个MessageQueue**

由于生产者会和NameServer进行通信来获取Topic的路由数据，所以生产者可以从NameServer中得知：一个Topic有几个MessageQueue、每个MessageQueue在哪台Broker机器上。

假设消息写入MessageQueue的策略是：生产者会均匀把消息写入每个MessageQueue。也就是生产者发送20条数据出去，4个MessageQueue都会各自写入5条数据。

<img width="100%" height="100%" alt="image3" src="https://github.com/user-attachments/assets/cddd2864-9b70-449f-874c-b0217043bfe6" />

那么通过这个策略，就可以让生产者把写入请求分散给多个Broker，可以让每个Broker都均匀分摊写入请求压力。如果单个Broker可以抗每秒7万并发，那么两个Broker就可以抗每秒14万并发，这样就实现了RocketMQ集群抗下每秒10万+超高并发。

另外，通过该策略可让一个Topic的数据分散在多个MessageQueue中，进而分散在多个Broker机器中，从而实现RocketMQ集群分布式存储海量的消息数据。

<br>

**(4)如果某个Broker出现故障了怎么办**

如果某个Broker临时出现故障了，比如Master Broker挂了，那么需要等待其他Slave Broker自动热切换为Master Broker，此时对这一组Broker来说就没有Master Broker可以写入了。

<img width="100%" height="100%" alt="image4" src="https://github.com/user-attachments/assets/63497f9f-462b-4184-b930-70bbd84627ab" />

如果按照前面的策略来均匀把数据写入各个Broker上的MessageQueue，那么会导致在一段时间内，每次访问到这个挂掉的Master Broker都会访问失败。对于这个问题，通常来说可以在Producer中开启一个开关，就是sendLatencyFaultEnable。

一旦打开了这个开关，那么会有一个自动容错机制。如果某次访问一个Broker发现网络延迟有500ms，然后还无法访问，那么就会自动回避访问这个Broker一段时间。比如在接下来3000ms内，就不会访问这个Broker了。

这样就可以避免一个Broker故障后，短时间内生产者频繁发送消息到这个故障的Broker上去，出现较多次数的异常。通过sendLatencyFaultEnable开关，生产者会自动回避一段时间不去访问故障的Broker，过段时间再去进行访问。因为过一段时间后，这个故障的Master Broker就已经恢复好了，它的Slave Broker已切换为Master可以正常工作了。

<br>

**(5)总结**

一.创建Topic时需要指定关键的MessageQueue

二.Topic、MessageQueue和Broker之间的关系

三.生产者是如何将消息写入MessageQueue的

四.Broker故障时生产者如何进行自动容错处理

<br>

**3.Broker是如何持久化接收到的消息到磁盘上**

**(1)为什么Broker的数据存储机制是MQ的核心**

**(2)Broker收到的消息会顺序写入CommitLog**

**(3)MessageQueue在ConsumeQueue目录下的物理存储位置**

**(4)如何让消息写入CommitLog文件时的性能几乎等于往内存写入消息时的性能**

**(5)同步刷盘与异步刷盘**

<br>

**(1)为什么Broker的数据存储机制是MQ的核心**

实际上类似RocketMQ、Kafka、RabbitMQ的消息中间件系统，它们不只是简单提供写入消息和获取消息的功能，它们还提供强大的数据存储能力，能把亿万级的海量消息存储在自己的服务器磁盘上。

这样各种不同的系统从MQ中消费消息时，才可以从MQ服务器的磁盘中读取到自己需要的消息。如果MQ不在机器磁盘上存储大量消息，而是放在内存里，那么要么内存放不下、要么机器重启后内存里的消息数据丢失。

所以Broker数据存储是MQ的核心，它决定了生产者写入消息的吞吐量、决定了消息不能丢失、决定了消费者获取消息的吞吐量。

接下来介绍Broker的数据存储机制。

<br>

**(2)Broker收到的消息会顺序写入CommitLog**

当生产者的消息发送到一个Broker上时，Broker会对这个消息做什么事情？

首先Broker会把这个消息顺序写入磁盘上的一个日志文件CommitLog，也就是直接追加写入这个日志文件的末尾。一个CommitLog日志文件限定最多1GB，如果一个CommitLog日志文件写满了1GB，就会创建另一个新的CommitLog日志文件。所以，磁盘上会有很多个CommitLog日志文件。

<img width="100%" height="100%" alt="image5" src="https://github.com/user-attachments/assets/14c8c109-959d-496a-a08e-2589379263a4" />

<br>

**(3)MessageQueue在ConsumeQueue目录下的物理存储位置**

一个Topic的数据会分布式存储在多个Broker中，为了决定这个Topic的哪些数据应该放在哪个Broker上，RocketMQ引入了MessageQueue。通过将一个Topic的数据拆分为多个MessageQueue数据分片，然后在每个Broker上都会存储一部分MessageQueue数据分片，从而实现Topic数据的分布式存储。

如果这个Broker收到的消息都是写入到CommitLog日志文件中进行存储的，那么MessageQueue到底体现在哪里？

其实在一个Broker中，它存储的某个Topic的一部分MessageQueue会有一系列如下格式的ConsumeQueue文件：

    $HOME/store/consumequeue/{topic}/{queueId}/{fileName}

这个格式的含义是：由于每个Topic在一台Broker上都会有一些MessageQueue，所以{topic}指代的就是某个Topic，{queueId}指代的就是某个MessageQueue。然后对于存储在这台Broker机器上的Topic下的一个MessageQueue，它会有很多个ConsumeQueue文件。这个ConsumeQueue文件里存储的是一条消息对应在CommitLog文件中的offset偏移量。

假设有一个Topic，它有4个MessageQueue在两台Broker机器上，那么每台Broker机器会存储两个MessageQueue文件。此时如果生产者选择对其中一个MessageQueue发起写入消息的请求，那么消息会发送到其中一个Broker上，然后这个Broker首先会把消息写入到CommitLog文件中。

下图加入了两个ConsumeQueue，其中ConsumeQueue0和ConsumeQueue1分别对应着Topic里的MessageQueue0和MessageQueue1。也就是这个Topic的MessageQueue0和MessageQueue1就放在这个Broker机器上，而且每个MessageQueue此时在磁盘上就对应着一个ConsumeQueue，即MessageQueue0对应着该Broker磁盘上的ConsumeQueue0，MessageQueue1对应着该Broker磁盘上的ConsumeQueue1。

<img width="100%" height="100%" alt="image6" src="https://github.com/user-attachments/assets/ec5d549d-aef8-40b0-b384-3164aaca279a" />

接着假设这个Topic的名字叫：TopicOrderPaySuccess，那么此时在这个Broker的磁盘上应该有如下两个路径的文件：

    $HOME/store/consumequeue/TopicOrderPaySuccess/MessageQueue0/ConsumeQueue0磁盘文件；
    $HOME/store/consumequeue/TopicOrderPaySuccess/MessageQueue1/ConsumeQueue1磁盘文件；

当这个Broker收到一条消息并首先写入到一个CommitLog文件后，就会将这条消息在这个CommitLog文件中的物理位置，也就是文件偏移量offset，写入到这条消息所属的MessageQueue对应的ConsumeQueue文件中。

比如现在生产者发送一条消息给MessageQueue0，此时Broker就会将这条消息在CommitLog日志文件中的offset偏移量，写入到MessageQueue0对应的ConsumeQueue0中。所以ConsumeQueue0中存储的是：一条消息在CommitLog文件中的物理位置(即offset偏移量)，也可以理解ConsumeQueue中的一个物理位置其实是对CommitLog文件中一条消息的地址引用。

<img width="100%" height="100%" alt="image7" src="https://github.com/user-attachments/assets/d2c35f00-652e-44a3-811a-3117b05793c7" />

此外，在ConsumeQueue中存储的不只是消息在CommitLog中的offset偏移量，还会包含消息的长度、Tag、HashCode。ConsumeQueue中的一条数据是20字节，每个ConsumeQueue文件会保存30万条数据，所以每个文件大概是5.72MB。

所以，Topic的每个MessageQueue都对应了Broker机器上的多个ConsumeQueue文件，ConsumeQueue文件中保存了对应MessageQueue的所有消息在CommitLog文件中的物理位置(即offset偏移量)。

<br>

**(4)如何让消息写入CommitLog文件时的性能几乎等于往内存写入消息时的性能**

生产者把消息写入到Broker时，Broker会直接把消息写入磁盘上的CommitLog文件，那么Broker是如何提升整个过程的性能的呢？

这部分的性能提升会直接提升Broker处理消息写入的吞吐量。假设写入一条消息到CommitLog文件需要10ms，每个线程每秒可以处理100个写入消息的请求，那么100个线程每秒只能处理1万个写入消息的请求。但是如果把写入一条消息到CommitLog文件的性能优化为只需要1ms，那么每个线程每秒可以处理1000个写入消息的请求，100个线程每秒就可以处理10万个写入消息的请求。所以可以明显看到，Broker把接收到的消息写入CommitLog文件的性能，对RocketMQ的TPS有很大的影响。

RocketMQ中的Broker会基于OS操作系统的PageCache和顺序写来提升往CommitLog文件写入消息的性能。

首先，Broker会以顺序的方式将消息写入到CommitLog文件，也就是每次写入时就是在文件末尾追加一条数据即可，对文件进行顺序写的性能要比随机写的性能高得多。

然后，数据写入CommitLog文件时，不是直接写入底层的物理磁盘文件，而是先进入OS的PageCache内存缓存，然后再由OS的后台线程选一个时间，通过异步的方式将PageCache内存缓冲中的数据刷入到CommitLog磁盘文件。

如下图示，数据先写入OS的PageCache缓存，然后再由OS自己的线程将缓存里的数据刷入磁盘中。所以采用磁盘文件顺序写 + OS PageCache写入 + OS异步刷盘策略，可以让消息写入CommitLog的性能跟写入内存里是差不多的，从而让Broker能高吞吐地处理每秒大量的消息写入请求。

<img width="100%" height="100%" alt="image8" src="https://github.com/user-attachments/assets/ca35d7ad-5d4a-4ebf-81b3-e67689ebc50a" />

<br>

**(5)同步刷盘与异步刷盘**

如果采用上述模式，也就是异步刷盘的模式，生产者把消息发送给Broker后，Broker会将消息写入OS的PageCache中，然后就直接返回ACK给生产者了，此时生产者就会认为消息写入成功了。

虽然生产者认为消息写入成功，但实际上该消息此时是在Broker机器上的OS的PageCache中。如果此时Broker机器直接宕机，就会导致在PageCache中的这条消息丢失。所以，异步刷盘模式虽然可以让消息写入的吞吐量非常高，但会有数据丢失的风险。

如果使用同步刷盘的模式，那么生产者发送一条消息给Broker时，Broker会直接把消息刷入到磁盘文件中，然后才返回ACK给生产者，此时生产者才知道消息写入成功。只要消息进入了磁盘文件，除非磁盘坏了，否则数据就不会丢失。

如果Broker还没有来得及把数据同步刷入磁盘，自己就挂了。那么生产者就会感知到消息发送失败，然后会不停地进行重试发送，直到有Slave Broker切换成Master Broker可以重新写入消息，从而保证消息数据不丢失。

如果强制每次消息写入都要直接刷入磁盘，那么必然会导致每条消息的写入性能急剧下降，从而导致消息写入的吞吐量急剧下降。

<img width="100%" height="100%" alt="image9" src="https://github.com/user-attachments/assets/9d6c53df-b21d-4b08-b1fb-a260f4358ebb" />

<br>

**(6)总结**

Broker核心的数据存储机制包括如下内容：

一.为什么Broker的数据存储机制是MQ的核心

二.Broker收到的消息会顺序写入CommitLog

三.MessageQueue在ConsumeQueue目录下的物理存储位置

四.基于CommitLog顺序写 + OS PageCache + 异步刷盘的消息写入机制

五.同步刷盘和异步刷盘各自的优缺点

<br>

**4.基于DLedger技术的Broker主从同步原理**

**(1)Broker的高可用架构原理**

**(2)基于DLedger技术替换Broker的CommitLog**

**(3)DLedger如何基于Raft协议选举Leader**

**(4)DLedger如何基于Raft协议进行多副本同步**

**(5)如果Leader Broker崩溃了怎么办**

**(6)采用Raft协议同步数据是否会影响TPS**

<br>

**(1)Broker的高可用架构原理**

介绍完Broker的数据存储原理后，接下来说明：Broker接收到消息写入请求后，是如何将消息同步给其他Broker做多副本冗余的。

Producer发送消息到Broker后，Broker首先会将消息写入到CommitLog文件中，然后会将这条消息在这个CommitLog文件中的文件偏移量offset，写入到这条消息所属的MessageQueue对应的ConsumeQueue文件中。

如果要让Broker实现高可用，那么必须要有一组Broker：一个Leader Broker写入数据，两个Follower Broker备份数据。当Leader Broker接收到写入请求写入数据后，直接把数据同步给其他的Follower Broker。这样，一条数据就会在三个Broker上有三份副本。此时如果Leader Broker宕机，那么就让其他Follower Broker自动切换为新的Leader Broker，继续接收写入请求。

<img width="100%" height="100%" alt="image10" src="https://github.com/user-attachments/assets/55e4651c-59b7-49a6-b31c-d34049afd352" />

<br>

**(2)基于DLedger技术替换Broker的CommitLog**

其实，Broker的上述高可用架构就是基于DLedger技术来实现的。所以，接下来先介绍DLedger技术可以干什么。

DLedger技术也有一个CommitLog机制。把数据交给DLedger，DLedger就会将数据写入CommitLog文件里。所以，如果基于DLedger技术来实现Broker高可用架构，实际上就是由DLedger来管理CommitLog，替换掉原来由Broker自己来管理CommitLog。

同时，使用DLedger来管理CommitLog后，Broker还是可以基于DLedger管理的CommitLog去构建出各个ConsumeQueue文件的。

<img width="100%" height="100%" alt="image11" src="https://github.com/user-attachments/assets/d52391d5-3c73-4144-90a9-e0b310ca4ef8" />

<br>

**(3)DLedger如何基于Raft协议选举Leader**

既然会使用DLedger替换各个Broker上的CommitLog管理组件，那么每个Broker上都会有一个DLedger组件。如果我们配置了一组Broker，比如有3台机器，那么DLedger会如何从3台机器里选举出一个Leader呢？DLedger是基于Raft协议来进行Leader Broker选举的，那么Raft协议中是如何进行多台机器的Leader选举的呢？

这需要通过三台机器互相投票，然后选出一个Broker作为Leader。简单来说，三台Broker机器启动时，都会给自己投票选自己作为Leader，然后把这个投票发送给其他Broker。

比如Broker01、Broker02、Broker03各自先投票给自己，然后再把自己的投票发送给其他Broker。在第一轮选举中，每个Broker收到所有投票后发现，每个Broker都在投票给自己，所以第一轮选举是失败的。

接着每个Broker都会进入一个随机时间的休眠，比如Broker01休眠3毫秒，Broker02休眠5毫秒，Broker03休眠4毫秒。假设Broker01先苏醒过来，那么当它苏醒过来后，就会继续尝试投票给自己，并且将自己的投票发送给其他Broker。

接着Broker03休眠4毫秒后苏醒，它发现Broker01已经发来了一个投票是投给Broker01的。此时因为它自己还没有开始投票，所以会尊重别人的选择，直接把票投给Broker01了，同时把自己的投票发送给其他Broker。

接着Broker02休眠5毫秒后苏醒，它发现Broker01投票给Broker01，Broker03也投票给Broker01，而此时它自己还没有开始投票，于是也会把票投给Broker01，并且把自己的投票发送给给其他Broker。

最后，所有Broker都会收到三张投票，都是投给Broker01的，那么Broker01就会当选成为Leader。其实只要有(3 / 2) + 1个Broker投票给某个Broker，那么就会选举该Broker为Leader，这个半数加1就是大多数的意思。

以上就是Raft协议中选举Leader算法的简单描述。

Raft协议确保可以选出Broker成为Leader的核心设计就是：当一轮选举选不出Leader时，就让各Broker进行随机休眠，先苏醒过来的Broker会投票给自己，其他Broker苏醒后会收到发来的投票，然后根据这些投票也把票投给那个Broker。这样，依靠这个随机休眠的机制，基本上可以快速选举出一个Leader。

**总结：** 在三台Broker机器刚启动时，就是靠基于Raft协议的DLedger来实现Leader选举的。当选举出一个Broker成为Leader后，其他Broker就是Follower了。只有Leader可以处理数据写入请求，Follower只能处理Leader的同步数据请求或者Leader高负载下的消费者拉取数据请求。

<br>

**(4)DLedger如何基于Raft协议进行多副本同步**

Leader Broker收到数据写入请求后，会由DLedger把数据同步给其他Follower Broker。其中，数据同步会分为两个阶段：一是uncommitted阶段，二是commited阶段。

首先Leader Broker上的DLedger会将数据标记为uncommitted状态，然后通过自己的DLedgerServer把uncommitted状态的数据发送给Follower Broker的DLedgerServer。接着Follower Broker的DLedger的DLedgerServer收到uncommitted状态的数据后，会返回一个ACK给Leader Broker的DLedger的DLedgerServer。

如果Leader Broker收到超过半数的Follower Broker返回ACK，那么就将数据标记为committed状态。然后Leader Broker的DLedger的DLedgerServer就会发送commited状态的数据给Follower Broker的DLedger的DLedgerServer，让它们也把数据标记为comitted状态。

以上就是DLedger基于Raft协议实现的Broker多副本同步机制。

<img width="100%" height="100%" alt="image12" src="https://github.com/user-attachments/assets/d099f143-f843-44f6-bfe6-db912bcb62ca" />

<br>

**(5)如果Leader Broker崩溃了怎么办**

对于高可用的Broker架构而言，无论是写入CommitLog日志，还是多副本同步数据，都是由DLedger来实现的。

如果Leader Broker挂了，那么剩下的两个Follower Broker就会重新发起选举。它们会由DLedger基于Raft协议选举出一个新的Leader Broker继续对外提供服务，而且会对没有完成数据同步的Follower Broker进行恢复性操作，保证数据不丢失。

<br>

**(6)采用Raft协议同步数据是否会影响TPS**

使用DLedger技术管理CommitLog后，可以自动在一组Broker中选举出一个Leader。然后在Leader接收消息写入请求时，会基于DLedger技术将消息写入到本地CommitLog中，这个和Broker自己写入CommitLog没什么区别。

但有区别的是：Leader Broker上的DLedger收到消息写入请求，将uncommitted消息写入到本地存储后，还需要基于Raft协议，采用两阶段的方式把uncommitted消息同步给其他Follower Broker，而且必须要超半数的Follower Broker的DLedger对uncommitted消息返回ACK，此时Leader Broker才能返回ACK给生产者。

那么不需要等待Follower Broker它们执行完commit操作后，Leader Broker再返回ACK给生产者吗？

实际上只要有超过半数的Follower Broker都写入uncommitted消息后，就可以返回ACK给生产者了。哪怕此时Leader Broker宕机，超过半数的Follower Broker上也是有这个消息的，只不过是uncommitted状态。但新选举的Leader Broker可以根据剩余Follower Broker上该消息的状态去进行数据恢复，比如把消息状态调整为committed。

也就是说，这样的架构对每次写入都增加了一个成本：每次写入都必须有超过半数的Follower Broker都写入消息才可以算做一次写入成功。这样做确实会对Leader Broker的写入性能产生影响而降低TPS，但并不是必须要在所有场景都这么做。

<br>

**(6)总结**

基于DLedger技术的Broker集群的高可用原理：

一.高可用原理：Leader自动切换 + 多副本同步

二.基于DLedger技术来管理CommitLog文件

三.Broker集群启动时，会通过基于Raft协议的DLedger完成Leader选举

四.Broker集群写入数据时，会通过基于Raft协议的DLedger将数据从Leader同步到Follower

五.如果Leader Broker崩溃，会通过基于Raft协议的DLedger重新选举Leader

<br>

**5.消费者进行消息拉取和消费的过程**

**(1)什么是消费者组**

**(2)集群模式消费 vs 广播模式消费**

**(3)MessageQueue和ConsumeQueue以及CommitLog之间的关系**

**(4)MessageQueue与消费者的关系**

**(5)Push消费模式 vs Pull消费模式**

**(6)Broker如何读取消息返回给消费者**

**(7)消费者处理消息、进行ACK响应和提交消费进度**

**(8)消费者组出现宕机或扩容应如何处理**

<br>

**(1)什么是消费者组**

**一.消费者组举例**

消费者组的意思就是给一组消费者起一个名字。比如有一个Topic叫TopicOrderPaySuccess，库存系统、积分系统、营销系统、仓储系统都要去消费这个Topic中的消息，那么此时就应该给这四个系统分别起一个消费者组名字，如下所示：

    stock_consumer_group、marketing_consumer_group、
    credit_consumer_group、wms_consumer_group

设置消费者组的方式是在代码里进行的，如下所示：

    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("stock_consumer_group");

假设库存系统部署了4台机器，每台机器上的消费者组的名字都是stock\_consumer\_group，那么这4台机器就同属于一个消费者组。以此类推，每个系统的几台机器都是属于各自的消费者组。

下图展示了两个系统，每个系统都有2台机器，每个系统都有一个自己的消费者组。

<img width="100%" height="100%" alt="image13" src="https://github.com/user-attachments/assets/1fe5be2a-a561-4732-ab65-8a8ab1153b7a" />

<br>

**二.不同消费者组之间的关系**

假设库存系统和营销系统作为两个消费者组，都订阅了TopicOrderPaySuccess这个订单支付成功消息的Topic，此时如果订单系统作为生产者发送了一条消息到这个Topic，那么这条消息会被如何消费呢？

一般情况下，这条消息进入Broker后，库存系统和营销系统作为两个消费者组，每个组都会拉取到这条消息。也就是说，这个订单支付成功的消息，库存系统会获取到一条，营销系统也会获取到一条，它们俩都会获取到这条消息。

但库存系统这个消费者组里有两台机器，是两台机器都获取到这条消息、还是只有一台机器会获取到这条消息？

一般情况下，库存系统的两台机器中只有一台机器会获取到这条消息，营销系统也是同理。

下图展示了对于同一条订单支付成功的消息，库存系统的一台机器获取到了、营销系统的一台机器也获取到了。所以在消费时，不同的系统应该设置不同的消费者组。如果不同的消费者组订阅了同一个Topic，对Topic里的同一条消息，每个消费者组都会获取到这条消息。

<img width="100%" height="100%" alt="image14" src="https://github.com/user-attachments/assets/98ead976-5266-4426-85d6-93f591e2dc0b" />

<br>

**(2)集群模式消费 vs 广播模式消费**

对于一个消费者组而言，它获取到一条消息后，如果消费者组内部有多台机器，到底是只有一台机器可以获取到这个消息，还是每台机器都可以获取到这个消息？这就是集群模式和广播模式的区别。

默认情况下都是集群模式：即一个消费者组获取到一条消息，只会交给组内的一台机器去处理，不是每台机器都可以获取到这条消息的。

但是可以通过如下设置来改变为广播模式：

    consumer.setMessageModel(MessageModel.BROADCASTING);

如果修改为广播模式，那么对于消费者组获取到的一条消息，组内每台机器都可以获取到这条消息。但是相对而言，广播模式用的很少，基本上都是使用集群模式来进行消费的。

<br>

**(3)MessageQueue和ConsumeQueue以及CommitLog之间的关系**

在创建Topic时，需要设置Topic有多少个MessageQueue。Topic中的多个MessageQueue会分散在多个Broker上，一个Broker上的一个MessageQueue会有多个ConsumeQueue文件。但在一个Broker的运行过程中，一个MessageQueue只会对应一个ConsumeQueue文件。

对于Broker而言，存储在一个Broker上的所有Topic的所有MessageQueue数据都会写入一个统一的CommitLog文件，一个Broker收到的所有消息都会往CommitLog文件里面写。

对于Topic的各个MessageQueue而言，则是通过各个ConsumeQueue文件来存储属于MessageQueue的消息在CommitLog文件中的物理地址(即offset偏移量)。

<img width="100%" height="100%" alt="image15" src="https://github.com/user-attachments/assets/6e96c4d6-685e-4ea4-b45c-612f3fb99071" />

<br>

**(4)MessageQueue与消费者的关系**

一个Topic上的多个MessageQueue是如何让一个消费者组中的多台机器来进行消费的？可以简单理解为，它会均匀将MessageQueue分配给消费者组的多台机器来消费。

举个例子，假设TopicOrderPaySuccess有4个MessageQueue，这4个MessageQueue分布在两个Master Broker上，每个Master Broker上有2个MessageQueue。然后库存系统作为一个消费者组，库存系统里有两台机器。那么正常情况下，最好就是让这两台机器各自负责2个MessageQueue的消费。比如库存系统的机器01从Master Broker01上消费2个MessageQueue，库存系统的机器02从Master Broker02上消费2个MessageQueue。这样就可以把消费的负载均摊到两台Master Broker上。

所以大致可以认为一个Topic的多个MessageQueue会均匀分摊给消费者组内的多个机器去消费。

这里的一个原则是：一个MessageQueue只能被一个消费者机器去处理，但是一台消费者机器可以负责多个MessageQueue的消息处理。

<br>

**(5)Push消费模式 vs Pull消费模式**

**一.一般选择Push消费模式**

既然一个消费者组内的多台机器会分别负责一部分MessageQueue的消费的，那么每台机器都必须要连接到对应的Broker，尝试消费里面MessageQueue对应的消息。于是就涉及到两种消费模式了，一个是Push模式、一个是Pull模式。

这两个消费模式本质上是一样的，都是消费者机器主动发送请求到Broker机器去拉取一批消息来处理。

Push消费模式是基于Pull消费模式来实现的，只不过它的名字叫做Push而已。在Push模式下，Broker会尽可能实时把新消息交给消费者进行处理，它的消息时效性会更好。

一般我们使用RocketMQ时，消费模式通常都选择Push模式，因为Pull模式的代码写起来更加复杂和繁琐，而且Push模式底层本身就是基于Pull模式来实现的，只不过时效性更好而已。

<br>

**二.Push消费模式的实现思路**

当消费者发送请求到Broker去拉取消息时，如果有新的消息可以消费，那么就马上返回一批消息给消费者处理。消费者处理完之后，会接着发送请求到Broker机器去拉取下一批消息。

所以，消费者机器在Push模式下处理完一批消息，会马上发起请求拉取下一批消息，消息处理的时效性非常好，看起来就像Broker一直不停地推送消息到消费机器一样。

此外，Push模式下有一个请求挂起和长轮询的机制：当拉取消息的请求发送到Broker，Broker却发现没有新的消息可以处理时，就会让处理请求的线程挂起，默认是挂起15秒。然后在挂起期间，Broker会有一个后台线程，每隔一会就检查一下是否有新的消息。如果有新的消息，就主动唤醒被挂起的请求处理线程，然后把消息返回给消费者。

可见，常见的Push消费模式，本质也是消费者不断发送请求到Broker去拉取一批一批的消息。

<br>

**(6)Broker如何读取消息返回给消费者**

Broker在收到消费者的拉取请求后，是如何将消息读取出来，然后返回给消费者的？这涉及到ConsumeQueue和CommitLog。

假设一个消费者发送了拉取请求到Broker，表示它要拉取MessageQueue0中的消息，然后它之前都没拉取过消息，所以就从这个MessageQueue0中的第一条消息开始拉取。

于是，Broker就会找到MessageQueue0对应的ConsumeQueue0，从里面找到第一条消息的offset。接着Broker就需要根据ConsumeQueue0中找到的第一条消息的offset，去CommitLog中根据这个offset读取出这条消息的数据，然后把这条消息的数据返回给消费者。

所以消费者在消费消息时，本质就是：首先根据要消费的MessageQueue以及开始消费的位置，去找到对应的ConsumeQueue。然后在ConsumeQueue中读取要消费的消息在CommitLog中的offset偏移量。接着到CommitLog中根据offset读取出完整的消息数据，最后将完整的消息数据返回给消费者。

<br>

**(7)消费者处理消息、进行ACK响应和提交消费进度**

消费者拉取到一批消息后，就会将这批消息传入注册的回调函数，如下所示：

    consumer.registerMessageListener(new MessageListenerConcurrently() {
        @Override
        public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
            //处理消息
            //标记该消息已经被成功消费
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;    
        }
    });

当消费者处理完这批消息后，消费者就会提交目前的一个消费进度到Broker上，然后Broker就会存储消费者的消费进度。

比如现在对ConsumeQueue0的消费进度就是在offset=1的位置，那么Broker会记录下一个ConsumeOffset来标记该消费者的消费进度。这样下次这个消费者组只要再次拉取这个ConsumeQueue的消息，就可以从Broker记录的消费位置开始继续拉取，不用重头开始拉取了。

<img width="100%" height="100%" alt="image16" src="https://github.com/user-attachments/assets/bd021a9c-ff71-4d35-ab0d-6d266cad7d61" />

<br>

**(8)消费者组出现宕机或扩容应如何处理**

此时会进入一个Rebalance环节，也就是重新给各个消费者分配各自需要处理的MessageQueue。

比如现在机器01负责MessageQueue0和MessageQueue1，机器02负责MessageQueue2和MessageQueue3。如果现在机器02宕机了，那么机器01就会接管机器02之前负责的MessageQueue2和MessageQueue3。如果此时消费者组加入了一台机器03，那么就可以把机器02负责的MessageQueue3转移给机器03，然后机器01只负责一个MessageQueue2的消费，这就是负载重平衡。

<br>

**(9)总结**

消费者进行消息拉取和消费的过程要点如下：

一.消费者组和一条消息在多个消费者组中如何分配

二.消费者组内部的消费模式有集群模式和广播模式

三.MessageQueue和ConsumeQueue以及CommitLog之间的关系

四.消费者组内的机器会如何分配MessageQueue

五.消费者从Broker拉取消息的Push模式和Pull模式

六.Broker如何基于ConsumeQueue和CommitLog获取消息返回给消费者

七.消费者如何处理消息和提交消费进度

八.消费者组内出现机器宕机或者扩容时会对MessageQueue进行负载重平衡

<br>

**6.消费者从Master或Slave上拉取消息的策略**

**(1)消费者什么时候会从Slave Broker上拉取消息**

**(2)CommitLog会基于PageCache提升写性能**

**(3)ConsumeQueue会基于PageCache提升性能**

**(4)CommitLog基于PageCache + 磁盘来一起读**

**(5)何时从PageCache读以及何时从磁盘读**

**(6)Master Broker什么时候会让消费者从Slave Broker拉取消息**

<br>

**(1)消费者什么时候会从Slave Broker上拉取消息**

Broker在实现高可用架构时会有主从之分。消费者可以从Master Broker上拉取消息，也可以从Slave Broker上拉取消息，具体要看Master Broker的机器负载。

刚开始消费者都是连接到Master Broker机器去拉取消息的，然后如果Master Broker机器觉得自己负载比较高，就会告诉消费者下次可以去Slave Broker拉取消息。

<br>

**(2)CommitLog会基于PageCache提升写性能**

当Broker收到一个消息写入请求时，首先会把消息写入到OS的PageCache，然后OS会有后台线程过一段时间后异步把OS的PageCache中的消息刷入CommitLog磁盘文件中。

依靠这个将消息写入CommitLog时先进入OS的PageCache而不是直接写入磁盘的机制，才可以实现Broker写CommitLog文件的性能是内存写级别的，才可以实现Broker超高的消息写入吞吐量。

<img width="100%" height="100%" alt="image17" src="https://github.com/user-attachments/assets/704e2f4f-f49e-4f2e-8a48-0dfb181537ac" />

<br>

**(3)ConsumeQueue会基于PageCache提升性能**

当消费者发送大量请求给Broker高并发读取消息时，Broker的ConsumeQueue文件的读操作就会变得非常频繁，而且会极大影响消费者拉取消息的性能和吞吐量。因此，ConsumeQueue同样也会基于OS的PageCache来进行优化。即向Broker的ConsumeQueue文件写入消息时，会先写入OS的PageCache。而且OS自己也有一个优化机制，就是读取一个磁盘文件的某数据时会自动把整个磁盘文件的数据缓存到OS的PageCache中。

由于ConsumeQueue文件主要用来存放消息的offset，所以每个ConsumeQueue文件是很小的，30万条消息的offset也就5.72MB而已。因此ConsumeQueue文件不会占用多少空间，它们整体的数据量很小，完全可以被缓存在PageCache中。

这样，当消费者拉取消息时，Broker就可以直接到OS的PageCache里读取ConsumeQueue文件里的内容，其读取性能与读内存时的性能是一样的，从而保证了消费消息时的高性能以及高吞吐。

<br>

**(4)CommitLog基于PageCache + 磁盘来一起读**

当消费者拉取消息时，首先会读OS的PageCache里的少量ConsumeQueue数据，这时的性能是极高的，然后会根据读取到的offset去CommitLog文件里读取完整的消息数据。那么从CommitLog文件里读取完整的消息数据时，既会从OS的PageCache里读取，也会从磁盘里读取。

由于CommitLog文件是用来存放消息的完整数据的，所以它的数据量会很大。毕竟一个CommitLog文件就有1GB，所以整体可能多达几个TB。这么多的CommitLog数据，不可能都放在OS的PageCache里。因为OS的PageCache用的也是机器的内存，一般也就几十个GB而已。何况Broker自身的JVM也要用一些内存，那么留给OS的PageCache的内存就只有一部分罢了，比如10GB\~20GB。所以是无法把CommitLog的全部数据都放在OS的PageCache里来提升消息者拉取时的性能的。

也就是说，CommitLog主要是利用OS的PageCache来提升消息的写入性能。当Broker不停写入消息时，会先往OS的PageCache里写，这里可能会累积10GB\~20GB的数据。之后OS会自动把PageCache里比较旧的一些数据刷入到CommitLog文件，以腾出空间给新写入的消息。

因此有这样的结论：当消费者向Broker拉取消息时，可以轻松从OS的PageCache里读取到少量ConsumeQueue文件里的offset，这时候的性能是极高的。但当Broker去CommitLog文件里读取完整消息数据时，那么就会有两种可能。

第一种可能：如果读取的是刚刚写入CommitLog文件的消息，那么这些消息大概率还停留在OS的PageCache中。此时Broker可以直接从OS的PageCache里读取完整的消息数据，这时是内存读取，性能会很高。

第二种可能：如果读取的是较早之前写入CommitLog文件的消息，那么这些消息可能早就被刷入磁盘了，已经不在OS的PageCache里了。此时Broker只能从CommitLog文件里读取完整的消息数据了，这时的性能是比较差的。

<br>

**(5)何时从PageCache读以及何时从磁盘读**

如果消费者一直在快速地拉取和消费消息，紧紧的跟上生产者往Broker写入消息的速度，那么消费者每次拉取时几乎都是在拉取最近刚写入CommitLog的消息，这些消息的数据几乎都可以从OS的PageCache里读取到。

如果Broker的负载很高导致消费者拉取消息的速度很慢，或者消费者拉取到一批消息后处理的性能很低导致处理速度很慢，那么都会导致消费者拉取消息的速度跟不上生产者写入消息的速度。

比如生产者都已经写入10万条消息了，结果消费者才拉取2万条消息进行消费。此时可能有5万条最新的消息是在OS的PageCache里，有3万条还没拉取去消费的消息只在磁盘里的CommitLog文件了。那么当消费者再拉取消息时，必然大概率需要从磁盘里的CommitLog文件中读取消息。接着，之前在OS的PageCache里的5万条消息可能又被刷入磁盘了，取而代之的是更加新的几万条消息在OS的PageCache里。当消费者再次拉取时，又会从磁盘里的CommitLog文件中读取那5万条消息，从而形成恶性循环。

<br>

**(6)Master Broker什么时候会让消费者从Slave Broker拉取消息**

假设Broker已经写入了10万条消息，但是消费者仅仅拉取了2万条消息进行消费。那么下次消费者拉取消息时，会从第2万零1条数据开始继续往后拉取，此时Broker还有8万条消息是没有被拉取。

然后Broker知道最多还可以往OS的PageCache里放入多少条消息，比如最多也只能放5万条消息。这时候消费者过来拉取消息，Broker发现该消费者还有8万条消息没有拉取，而这8万是大于内存最多存放的5万。因此Broker便知肯定有3万条消息目前是在磁盘上的，而不在OS的PageCache内存里。于是，在这种情况下，Broker就会告诉消费者，这次会给它从磁盘里读取3万条消息，但下次消费者要去Slave Broker拉取消息了。

其实这个问题的本质就是：将消费者当前没有拉取的消息数量和Broker最多可以存放在OS的PageCache内存里的消息数量进行对比，如果消费者没拉取的消息总大小超过了最大能使用的PageCache内存大小，那么说明后续Broker会频繁从磁盘中加载数据，于是此时Broker就会通知消费者下次要从Slave Broker加载数据了。

<br>

**6.RocketMQ如何基于Netty进行高性能网络通信**

**(1)Reactor主线程与长短连接**

**(2)Producer和Broker建立一个长连接**

**(3)基于Reactor线程池监听连接中的请求**

**(4)基于Worker线程池完成一系列准备工作**

**(5)基于业务线程池完成请求的处理**

**(6)为什么这套网络通信框架是高性能以及高并发的**

<br>

**(1)Reactor主线程与长短连接**

首先，Broker有一个Reactor主线程，这个线程会负责监听一个网络端口，比如监听个2888，39150这样的端口。接着，假设有一个Producer现在想要跟Broker建立一个TCP长连接。

<img width="100%" height="100%" alt="image18" src="https://github.com/user-attachments/assets/602d0339-7f92-4844-9d9a-6778712c2b3f" />

<br>

什么是短连接：

如果要向对方发送一个请求，必须要建立连接 -> 发送请求 -> 接收响应 -> 断开连接。下一次要向对方发送请求时，这个过程得重新来一遍。每次建立一个连接后，使用这个连接发送请求的时间是很短的，很快就会断开连接，由于存在时间太短，便叫短连接。

<br>

什么是长连接：

如果要发送一个请求，必须要建立一个连接 -> 发送请求 -> 接收响应 -> 发送请求 -> 接收响应 -> 发送请求 -> 接收响应。可见，当建立好一个长连接后，可以不停的发送请求和接收响应，此时连接不会断开，等到不需要时再断开。这个连接会存在很长时间，所以叫长连接。

<br>

什么是TCP长连接：

TCP就是一个协议，所谓协议的意思就是，按照TCP这个协议规定好的步骤建立连接，按照它规定好的步骤发送请求。比如要建立一个TCP连接，必须先给对方发送它规定好的几个数据，然后对方按照规定返回几个数据，接着再给对方发送几个数据，一切都按TCP的规定来，这就双方就可以建立一个TCP连接。所以TCP长连接，就是按TCP协议建立的长连接。

<br>

**(2)Producer和Broker建立一个长连接**

假设有一个Producer要跟Broker建立一个TCP长连接，此时Broker上的Reactor主线程会在端口上监听到这个Producer建立连接的请求。

<img width="100%" height="100%" alt="image19" src="https://github.com/user-attachments/assets/4e5bf94b-2b61-4dbe-9cc4-c6efc2aba717" />

接着这个Reactor主线程就专门会负责跟这个Producer按照TCP协议规定的一系列步骤和规范，建立好一个长连接。而在Broker中，会使用一个叫SocketChannel的对象来代表跟Producer之间建立的这个长连接。

Producer里会有一个SocketChannel，Broker里也会有一个SocketChannel，这两个SocketChannel就代表了它们俩建立好的这个长连接。

既然Producer和Broker之间已经通过SocketChannel维持了一个长连接了，接着Producer就会通过这个SocketChannel去发送消息给Broker。

<img width="100%" height="100%" alt="image20" src="https://github.com/user-attachments/assets/87e868e4-be19-48f7-bb7c-434a4e0a77b5" />

<br>

**(3)基于Reactor线程池监听连接中的请求**

此时还不能让Producer发送消息给Broker，因为虽然有一个SocketChannel组成的长连接，但它仅仅是一个长连接而已。假设Producer此时通过SocketChannel发送消息给到Broker那边的SocketChannel了，但是Broker中应该用哪个线程来负责从SocketChannel里获取这个消息呢？

从SocketChannel里获取消息的工作，会通过一个叫Reactor线程池(默认有3个线程)来负责。Reactor主线程建立好的每个连接SocketChannel，都会交给这个Reactor线程池里的其中一个线程去监听请求。有了Reactor线程池后，就可以让Producer发送请求过来了。Producer发送一个消息到达Broker里的SocketChannel，此时Reactor线程池里的一个线程就会监听到该SocketChannel中有请求到达。

<img width="100%" height="100%" alt="image21" src="https://github.com/user-attachments/assets/92c0fb0b-0a8d-4fb6-a9cb-af22e3bb39b4" />

<br>

**(4)基于Worker线程池完成一系列准备工作**

接着Reactor线程从SocketChannel中读取出一个请求，这个请求在正式进行处理之前，必须先进行一些准备工作和预处理，比如SSL加密验证、编码解码、连接空闲检查、网络连接管理等事情。那么这些事情又会由哪个线程来负责处理呢？

这些准备工作和预处理会由一个叫Worker线程池(默认有8个线程)来负责。也就是Reactor线程从SocketChannel中读取出一个请求后，就会交给Worker线程池中的一个线程进行处理，来完成上述一系列的准备工作和预处理。

<img width="100%" height="100%" alt="image22" src="https://github.com/user-attachments/assets/c3b9a9e8-f07d-4382-a3f3-313d9e7483d4" />

<br>

**(5)基于业务线程池完成请求的处理**

当Worker线程完成了一系列的预处理后，比如SSL加密验证、编码解码、连接空闲检查、网络连接管理等，接着就要对这个请求进行正式的业务处理了。

正式的业务处理逻辑，就包括了Broker数据存储过程。也就是Broker接收到消息后，要写入CommitLog文件，以及ConsumeQueue文件等。

所以，此时就需要继续把经过一系列预处理过后的请求转交给业务线程池。比如把发送消息的请求转交给SendMessage线程池，这个SendMessage线程数是可以配置的，配置得越多，处理发送消息请求的吞吐量就越高。

<img width="100%" height="100%" alt="image23" src="https://github.com/user-attachments/assets/991c13c7-e468-4cdb-8674-17ef8f655cc5" />

<br>

**(6)为什么这套网络通信框架是高性能以及高并发的**

假设只有一个线程来处理所有的网络连接的请求，包括读写磁盘文件之类的业务操作，那么必然会导致并发能力很低。

所以必须专门分配一个Reactor主线程出来，专门负责和Producer + Consumer建立长连接。一旦连接建立好之后，大量的长连接会均匀分配给Reactor线程池里的多个线程。

每个Reactor线程负责监听一部分的连接请求，这也是一个优化点。通过多线程并发监听不同连接的请求，可以有效提升大量并发请求过来时的处理能力，也就是提升网络框架的并发能力。

接着后续对大量并发过来的请求都基于Worker线程池进行预处理。当Worker线程池预处理多个请求时，Reactor线程还是可以有条不紊的继续监听和接收大量连接的请求是否到达。

最后读写磁盘文件之类的操作都是交给业务线程池来处理，当它并发执行多个请求的磁盘读写操作时，不会影响其他线程池同时接收请求、预处理请求。

所以最终的效果就是：

一.Reactor主线程在端口上监听Producer建立连接的请求来建立长连接

二.Reactor线程池里的线程可以并发监听多个连接的请求是否到达

三.Worker线程池里的线程可以并发对多个到达的请求进行预处理

四.业务线程池可以并发对多个请求进行磁盘读写等业务操作

上述这些事情全部是利用不同的线程池并发执行的，任何一个环节在执行时，都不会影响其他线程池在其他环节处理请求。

这样的一套网络通信架构，最终实现的效果就是可以高并发、高吞吐的处理大量请求。这套网络通信架构也是保证Broker实现高吞吐的一个非常关键因素。

因此对于这类中间件，如果将它部署在高配置的物理机上，有几十个CPU核，那么就可以增加它的各种线程池的线程数量，让各个环节可以同时高并发的处理大量请求。

<br>

**8.基于mmap内存映射实现磁盘文件的高性能读写**

**(1)mmap是Broker读写磁盘文件的核心技术**

**(2)传统文件IO操作的多次数据拷贝问题**

**(3)RocketMQ如何基于mmap技术 + PageCache技术进行文件读写优化**

**(4)基于mmap技术 + PageCache技术来实现高性能的文件读写**

**(5)内存预映射机制 + 文件预热机制**

<br>

**(1)mmap是Broker读写磁盘文件的核心技术**

Broker中大量使用了mmap技术去实现CommitLog这种大磁盘文件的高性能读写优化。Broker对磁盘文件的写入主要是通过直接写入OS的PageCache来实现性能优化的。因为直接写入OS的PageCache的性能与写入内存一样，之后OS内核中的线程会异步把PageCache中的数据刷入到磁盘文件，这个过程中就涉及到了mmap技术。

<br>

**(2)传统文件IO操作的多次数据拷贝问题**

如果RocketMQ没有使用mmap技术，而是使用普通的文件IO操作去进行磁盘文件的读写，那么会存在多次数据拷贝的性能问题。

假设有个程序需要对磁盘文件发起IO操作，需要读取文件里的数据到程序，那么会经过以下一个顺序：

首先从磁盘上把数据读取到内核IO缓冲区，然后从内核IO缓存区里读取到用户进程私有空间，程序才能拿到该文件里的数据。

<img width="100%" height="100%" alt="image24" src="https://github.com/user-attachments/assets/f3f3c50a-0cc4-425b-bcee-a7bc631d6cdd" />

为了读取磁盘文件里的数据，发生了两次数据拷贝。这就是普通IO操作的一个弊端，必然涉及到两次数据拷贝操作，这对磁盘读写性能是有影响的。

如果要将一些数据写入到磁盘文件里去，也是一样的过程。必须先把数据写入到用户进程私有空间，然后从用户进程私有空间再进入内核IO缓冲区，最后进入磁盘文件里。在数据进入磁盘文件的过程中，同样发生了两次数据拷贝。

这就是普通IO的问题：有两次数据拷贝。

<img width="100%" height="100%" alt="image25" src="https://github.com/user-attachments/assets/7e545214-b34b-4361-aaaa-41947e9ba77a" />

<br>

**(3)RocketMQ如何基于mmap技术 + PageCache技术进行文件读写优化**

RocketMQ底层对CommitLog、ConsumeQueue之类的磁盘文件的读写操作，基本上都会采用mmap技术来实现。具体到代码层面，第一步就是基于JDK NIO包下的MappedByteBuffer的map()方法：将一个磁盘文件(比如一个CommitLog文件，或者是一个ConsumeQueue文件)映射到内存里来。

<br>

**关于内存映射：** 可能有人会误以为是直接把那些磁盘文件里的数据读取到内存中，但这并不完全正确。因为刚开始建立映射时，并没有任何的数据拷贝操作，其实磁盘文件还是停留在那里，只不过map()方法把物理上的磁盘文件的一些地址和用户进程私有空间的一些虚拟内存地址进行了一个映射。

<img width="100%" height="100%" alt="image26" src="https://github.com/user-attachments/assets/67502e36-b550-4fd3-b2c2-0eac3491df75" />

这个地址映射的过程，就是JDK NIO包下的MappedByteBuffer.map()方法做的事情，其底层就是基于mmap技术实现的。另外这个mmap技术在进行文件映射时，一般有大小限制，在1.5GB\~2GB之间。所以RocketMQ才让CommitLog单个文件在1GB、ConsumeQueue文件在5.72MB，不会太大。这样限制了RocketMQ底层文件的大小后，就可以在进行文件读写时，很方便的进行内存映射了。

<br>

**关于PageCache：** 实际上PageCache在这里就是对应于虚拟内存。

<img width="100%" height="100%" alt="image27" src="https://github.com/user-attachments/assets/904e2e42-3b47-4405-a54d-19c621dea043" />

<br>

**(4)基于mmap技术 + PageCache技术来实现高性能的文件读写**

第二步就可以对这个已经映射到内存里的磁盘文件进行读写操作了。比如程序要写入消息数据到CommitLog文件：

首先程序把一个CommitLog文件通过MappedByteBuffer的map()方法映射其地址到程序的虚拟内存地址。

接着程序就可以对这个MappedByteBuffer执行写入操作了，写入时消息数据会直接进入PageCache。

然后过一段时间后，由OS的线程异步刷入磁盘中。

<img width="100%" height="100%" alt="image28" src="https://github.com/user-attachments/assets/87eb05d4-2cb2-4d65-8906-13452d496b76" />

从上图可以看出，只有一次数据拷贝的过程，也就是从PageCache里拷贝到磁盘文件。这个就是使用mmap技术后，相比于普通IO的一个性能优化。

接着如果要从磁盘文件里读取数据：那么就会判断一下，当前要读取的数据是否在PageCache里，如果在则可以直接从PageCache里读取。

比如刚写入CommitLog的数据还在PageCache里，此时消费者来消费肯定是从PageCache里读取数据的。但如果PageCache里没有要的数据，那么此时就会从磁盘文件里加载数据到PageCache中。

而且PageCache技术在加载数据时 **，** 还会将需要加载的数据块的临近的其他数据块也一起加载到PageCache里。

可见在读取数据时，其实也只发生了一次拷贝，而不是两次拷贝，所以这个性能相比于普通IO又提高了。

<img width="100%" height="100%" alt="image29" src="https://github.com/user-attachments/assets/74415f18-0433-40b9-b7ef-7b3097322995" />

<br>

**(5)内存预映射机制 + 文件预热机制**

下面是Broker针对上述磁盘文件高性能读写机制做的一些优化：

一.内存预映射机制

Broker会针对磁盘上的各种CommitLog、ConsumeQueue文件预先分配好MappedFile，也就是提前对一些可能接下来要读写的磁盘文件，提前使用MappedByteBuffer执行map()方法完成映射，这样后续读写文件时，就可以直接执行了。

二.文件预热

在提前对一些文件完成映射之后，因为映射不会直接将数据加载到内存里来，那么后续在读取CommitLog、ConsumeQueue时，其实有可能会频繁的从磁盘里加载数据到内存中去。所以在执行完map()方法之后，会进行madvise系统调用，就是提前尽可能多的把磁盘文件加载到内存里去。

通过上述优化，才能真正实现这么一个效果：就是写磁盘文件时都是进入PageCache的，保证写入高性能。同时尽可能多的通过map() + madvise的映射后预热机制，把磁盘文件里的数据尽可能多的加载到PageCache里来，后续对ConsumeQueue、CommitLog进行读取时，才能尽量从内存读取数据。

<br>

**(6)总结**

Broker在读写磁盘时，会大量把mmap技术和PageCache技术结合起来使用。通过mmap技术减少数据拷贝次数，然后利用PageCache技术实现尽可能优先读写内存，而不是读写物理磁盘。

<br>

**9.RocketMQ的整体运行原理总结**

RocketMQ的一些底层原理：MessageQueue的概念、在Broker上的分布式存储、Producer写入消息的底层原理、Broker的数据存储机制、Broker高可用架构的实现原理、Consumer的底层原理、基于Broker读写分离架构读取消息的原理。

<img width="100%" height="100%" alt="image30" src="https://github.com/user-attachments/assets/e506e9a2-982c-4a47-aad1-433bc2a862bb" />
