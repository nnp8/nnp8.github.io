# RocketMQ原理—3.源码设计简单分析下

**大纲**

**1.Producer作为生产者是如何创建出来的**

**2.Producer启动时是如何准备好相关资源的**

**3.Producer是如何从拉取Topic元数据的**

**4.Producer是如何选择MessageQueue的**

**5.Producer与Broker是如何进行网络通信的**

**6.Broker收到一条消息后是如何存储的**

**7.Broker是如何实时更新索引文件的**

**8.Broker是如何实现同步刷盘以及异步刷盘的**

**9.Broker是如何清理存储较久的磁盘数据的**

**10.Consumer作为消费者是如何创建和启动的**

**11.消费者组的多个Consumer会如何分配消息**

**12.Consumer会如何从Broker拉取一批消息**

<br>

**1.Producer作为生产者是如何创建出来的**

**(1)NameServer的启动**

**(2)Broker的启动**

**(3)Broker的注册和心跳**

**(4)通过Producer发送消息**

<br>

**(1)NameServer的启动**

NameServer启动后的核心架构，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/403de01d-588f-46f1-b996-bde8d518ce6a" />

NameServer启动后，会有一个NamesrvController组件管理控制NameServer的所有行为，包括内部会启动一个Netty服务器去监听一个9876端口号，然后接收处理Broker和客户端发送过来的请求。

<br>

**(2)Broker的启动**

Broker启动后的核心架构，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/4bd4d3f3-8b1b-455a-8bec-16181198bcad" />

Broker启动后，也会有一个BrokerController组件管理控制Broker的整体行为，包括初始化Netty服务器用于接收客户端的网络请求、启动处理请求的线程池、执行定时任务的线程池、初始化核心功能组件，同时还会发送注册请求到NameServer去注册自己。

<br>

**(3)Broker的注册和心跳**

Broker启动后，会向NameServer进行注册和定时发送注册请求作为心跳。NameServer会有一个后台进程定时检查每个Broker的最近一次心跳时间，如果长时间没心跳就认为Broker已经故障。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/cd8d0696-4941-4943-bc19-f1ea70e9c7ca" />

<br>

**(4)通过Producer发送消息**

假设RocketMQ集群已经启动好了NameServer，而且还启动了一批Broker，同时Broker都已经把自己注册到NameServer里去了，NameServer也会定时检查这批Broker是否存活。那么就可以让开发好的业务系统去发送消息到RocketMQ集群里，于是需要创建一个Producer实例。

实际上我们开发好的系统，最终都需要创建一个Producer实例，然后通过Producer实例发送消息到RocketMQ的Broker上去。

下面是使用Producer实例发送消息到RocketMQ的代码，可以看到Producer是如何构造出来的。

    DefaultMQProducer producer = new DefaultMQProducer("order_producer_group");
    producer.setNamesrvAddr("localhost:9876");
    producer.start();

构造Producer的过程很简单：也就是创建一个DefaultMQProducer对象实例。在构造方法中，首先会传入所属的Producer分组，然后设置一下NameServer的地址，最后调用它的start()方法启动这个Producer即可。

创建DefaultMQProducer对象实例是一个非常简单的过程：就是创建出一个对象，然后保存它的Producer分组。设置NameServer地址也是一个很简单的过程，就是保存一下NameServer地址。

所以，最关键的还是调用DefaultMQProducer的start()方法去启动Producer这个消息生产者。

<br>

**2.Producer启动时是如何准备好相关资源的**

**(1)DefaultMQProducer的start()方法**

**(2)Producer在第一次向Topic发送消息时才拉取Topic的路由数据**

**(3)Producer在第一次向Broker发送消息时才与Broker建立网络连接**

<br>

**(1)DefaultMQProducer的start()方法**

接下来分析Producer在启动时是如何准备好相关资源的。Producer内部必须要有独立的线程资源，以及需要和Broker已经建立好网络连接，这样才能把消息发送出去。

在构造Producer时，它内部便会构造一个真正用于执行消息发送逻辑的DefaultMQProducerImpl组件。所以，真正的Producer生产者其实是这个DefaultMQProducerImpl组件。那么这个组件在启动的时都干了什么呢？

    public class DefaultMQProducer extends ClientConfig implements MQProducer {
        protected final transient DefaultMQProducerImpl defaultMQProducerImpl;
        ...
        @Override
        public void start() throws MQClientException {
            this.setProducerGroup(withNamespace(this.producerGroup));
            this.defaultMQProducerImpl.start();
            if (null != traceDispatcher) {
                try {
                    traceDispatcher.start(this.getNamesrvAddr(), this.getAccessChannel());
                } catch (MQClientException e) {
                    log.warn("trace dispatcher start failed ", e);
                }
            }
        }
        ...
    }

    public class DefaultMQProducerImpl implements MQProducerInner {
        ...
        public void start() throws MQClientException {
            this.start(true);
        }
        
        public void start(final boolean startFactory) throws MQClientException {
            switch (this.serviceState) {
                case CREATE_JUST:
                    this.serviceState = ServiceState.START_FAILED;
                    this.checkConfig();
                    if (!this.defaultMQProducer.getProducerGroup().equals(MixAll.CLIENT_INNER_PRODUCER_GROUP)) {
                        this.defaultMQProducer.changeInstanceNameToPID();
                    }
                    this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQProducer, rpcHook);

                    boolean registerOK = mQClientFactory.registerProducer(this.defaultMQProducer.getProducerGroup(), this);
                    if (!registerOK) {
                        this.serviceState = ServiceState.CREATE_JUST;
                        throw new MQClientException("The producer group[" + this.defaultMQProducer.getProducerGroup() + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL), null);
                    }
                    this.topicPublishInfoTable.put(this.defaultMQProducer.getCreateTopicKey(), new TopicPublishInfo());
                    if (startFactory) {
                        mQClientFactory.start();
                    }

                    log.info("the producer [{}] start OK. sendMessageWithVIPChannel={}", this.defaultMQProducer.getProducerGroup(), this.defaultMQProducer.isSendMessageWithVIPChannel());
                    this.serviceState = ServiceState.RUNNING;
                    break;
                case RUNNING:
                case START_FAILED:
                case SHUTDOWN_ALREADY:
                    throw new MQClientException("The producer service state not OK, maybe started once, " + this.serviceState + FAQUrl.suggestTodo(FAQUrl.CLIENT_SERVICE_NOT_OK), null);
                default:
                    break;
            }
            this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
            this.startScheduledTask();
        }
        ...
    }

其实，上述start()方法的具体逻辑暂时不需要深入分析，因为其中的逻辑并没有直接与Producer发送消息相关联。比如拉取Topic的路由数据、选择MessageQueue、跟Broker建立长连接、发送消息到Broker等这些核心逻辑，其实都封装在发送消息的方法中。

<br>

**(2)Producer在第一次向Topic发送消息时才拉取Topic的路由数据**

假设后续Producer要发送消息，那么就要指定往哪个Topic发送消息。因此Producer需要知道Topic的路由数据，比如Topic有哪些MessageQueue，每个MessageQueue在哪些Broker上。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/6ecb7707-0a49-41e0-825b-dbc229b6a58b" />

从start()方法源码可知，在Producer启动时，并不会去拉取Topic的路由数据。实际上，Producer在第一次向Topic发送消息时，才会去拉取Topic的路由数据。包括这个Topic有几个MessageQueue、每个MessageQueue在哪个Broker上。然后从中选择一个MessageQueue，接着与对应的Broker建立网络连接，最后才把消息发送过去。

<br>

**(3)Producer在第一次向Broker发送消息时才与Broker建立网络连接**

从start()方法源码可知，在Producer启动时，并不会和所有Broker建立网络连接。很多核心的逻辑，包括拉取Topic路由数据、选择MessageQueue、和Broker建立网络连接等，都是在Producer第一次发送消息时才进行处理的。

<br>

**3.Producer是如何从拉取Topic元数据的**

**(1)Producer发送消息的方法**

**(2)Producer拉取Topic路由数据的过程**

<br>

**(1)Producer发送消息的方法**

当调用Producer的send()方法发送消息时，最终会调用到DefaultMQProducerImpl的sendDefaultImpl()方法。

在sendDefaultImpl()方法里，开始会有一行非常关键的代码，如下所示：

    public class DefaultMQProducerImpl implements MQProducerInner {
        ...
        private SendResult sendDefaultImpl(Message msg, final CommunicationMode communicationMode, final SendCallback sendCallback, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
            ...
            TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
            ...
        }
        ...
    }

该行代码的意思是，每次Producer发送消息时，都会先检查一下要发送消息的那个Topic的路由数据是否在本地。如果不在，才会发送请求到NameServer去拉取Topic的路由数据，然后缓存在本地。

<br>

**(2)Producer拉取Topic路由数据的过程**

进入tryToFindTopicPublishInfo()方法，会发现其逻辑非常简单：就是会先检查一下自己本地是否有这个Topic的路由数据的缓存，如果没有就发送网络请求到NameServer去拉取，如果有就直接返回本地Topic路由数据缓存，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/cbf7adeb-f5c3-481b-8c27-91c6d6f0926e" />

那么Producer是如何发送网络请求到NameServer去拉取Topic路由数据的呢？这其实就对应了tryToFindTopicPublishInfo()方法内的一行代码，如下所示：

    public class DefaultMQProducerImpl implements MQProducerInner {
        ...
        private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
            TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
            if (null == topicPublishInfo || !topicPublishInfo.ok()) {
                this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
                this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
                topicPublishInfo = this.topicPublishInfoTable.get(topic);
            }

            if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
                return topicPublishInfo;
            } else {
                this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
                topicPublishInfo = this.topicPublishInfoTable.get(topic);
                return topicPublishInfo;
            }
        }
        ...
    }

通过以下这行代码，Producer就可以从NameServer拉取某个Topic的路由数据，然后更新到自己本地的缓存里去。

    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);

Producer发送请求到NameServer的拉取Topic路由数据的过程：首先封装一个Request请求对象，然后通过Netty客户端发送请求到NameServer，接着会接收到NameServer返回的一个Response响应对象，于是就可以从Response响应对象里取出所需的Topic路由数据并更新到自己本地缓存里。更新时会做一些判断，比如Topic路由数据是否有改变过等，然后把Topic路由数据放入本地缓存。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/6f2b1898-2702-4831-ad93-d0c3f4398983" />

<br>

**4.Producer是如何选择MessageQueue的**

**(1)Topic是由多个MessageQueue组成的**

**(2)选择MessageQueue的源码和算法**

<br>

**(1)Topic是由多个MessageQueue组成的**

Producer发送消息时，会先检查一下要发送消息的Topic的路由数据是否在本地缓存。如果不在，就会通过底层的Netty网络通信模块发送一个请求到NameServer拉取Topic路由数据，然后缓存在Producer本地。当Producer拿到一个Topic的路由数据后，就应该选择要发送消息到这个Topic的哪一个MessageQueue上了。

因为Topic是一个逻辑上的概念，一个Topic的数据往往会分布式存储在多台Broker机器上，所以Topic本质是由多个MessageQueue组成的。

每个MessageQueue都可以在不同的Broker机器上，当然也可能一个Topic的多个MessageQueue在一个Broker机器上。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/d15d237c-3c76-4ee5-a2a4-c73a7e469793" />

只要Producer知道了要发送消息到哪个MessageQueue上去，其实就已经知道了这个MessageQueue在哪台Broker机器上，接着和该Broker机器建立连接，发送消息过去即可。

<br>

**(2)选择MessageQueue的源码和算法**

发送消息的源码在DefaultMQProducerImpl的sendDefaultImpl()方法中。该方法里只要Producer获取到Topic的路由数据，不管从本地缓存获取还是从NameServer拉取，就会执行下面的代码：

    public class DefaultMQProducerImpl implements MQProducerInner {
        ...
        private SendResult sendDefaultImpl(Message msg, final CommunicationMode communicationMode, final SendCallback sendCallback, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
            ...
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            ...
        }
        ...
    }

selectOneMessageQueue()方法其实就是在选择Topic中的一个MessageQueue，然后发送消息到这个MessageQueue。下面是选择MessageQueue的算法：

    public class MQFaultStrategy {
        ...
        public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
            ...
            int index = tpInfo.getSendWhichQueue().incrementAndGet();
            for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                if (pos < 0) {
                    pos = 0;
                }
                MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                    return mq;
                }
            }
            ...
        }
        ...
    }

这是一种简单的负载均衡算法：首先获取一个自增长的Index，接着就用这个Index对Topic的MessageQueue列表进行取模运算，从而获取到一个MessageQueue列表的位置，最后返回这个位置的MessageQueue。

但是如果某个Broker故障了，那么就不能把消息发送到故障Broker的MessageQueue了。所以selectOneMessageQueue()方法里还有其他代码，用来实现Broker故障时的自动回避机制。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/5f05da00-139e-49eb-801c-4ef8d52002f2" />

<br>

**5.Producer与Broker是如何进行网络通信的**

**(1)Producer是如何把消息发送给Broker的**

**(2)Producer和Broker基于长连接进行通信**

<br>

**(1)Producer是如何把消息发送给Broker的**

在DefaultMQProducerImpl.sendDefaultImpl()方法中，会先获取到MessageQueue所在的Broker名称，如下所示：

    public class DefaultMQProducerImpl implements MQProducerInner {
        ...
        private SendResult sendDefaultImpl(Message msg, final CommunicationMode communicationMode, final SendCallback sendCallback, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
            ...
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            if (mqSelected != null) {
                mq = mqSelected;
                brokersSent[times] = mq.getBrokerName();
                ...
            }
            ...
        }
        ...
    }

获取到这个BrokerName后，就会调用sendKernelImpl()方法把消息发送到Broker上。

    public class DefaultMQProducerImpl implements MQProducerInner {
        ...
        private SendResult sendDefaultImpl(Message msg, final CommunicationMode communicationMode, final SendCallback sendCallback, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
            ...
            sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
            ...
        }
        
        private SendResult sendKernelImpl(final Message msg, final MessageQueue mq, final CommunicationMode communicationMode, final SendCallback sendCallback, final TopicPublishInfo topicPublishInfo, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
            long beginStartTime = System.currentTimeMillis();
            String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
            if (null == brokerAddr) {
                tryToFindTopicPublishInfo(mq.getTopic());
                brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
            }
            ...
        }
        ...
    }

在sendKernelImpl()方法中：

首先会通过BrokerName去本地缓存查找它的实际地址。如果找不到，就到NameServer中拉取Topic的路由数据，再次在本地缓存获取Broker的实际地址，有了这个地址才能进行网络通信。

然后会封装一个Request请求，包括请求头、发送的消息等，并且会给消息分配全局唯一ID，以及对超过4KB的消息体进行压缩。

在Request请求中，会包含生产者组、Topic名称、Topic的MessageQueue数量、MessageQueue的ID、消息发送时间、消息的flag、消息扩展属性、消息重试次数、是否批量发送等信息，如果是事务消息则带上prepared标记等。

把这些数据都封装到一个Request请求后，就会通过Netty把Request请求发送到指定的Broker上。

<br>

**(2)Producer和Broker基于长连接进行通信**

其中，Producer和Broker会通过Netty建立长连接，然后基于长连接进行持续通信。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/f4d255f3-73a4-47a4-ac69-04e277bcd2f1" />

那么Broker上的Netty服务器接收到消息后，会如何进行处理？这个过程比较复杂，涉及到CommitLog、ConsumeQueue、IndexFile、Checkpoint等一系列机制，这也是RocketMQ中核心机制。

<br>

**6.Broker收到一条消息后是如何存储的**

**(1)Broker收到消息后的处理流程**

**(2)Broker如何将消息写入CommitLog文件**

<br>

**(1)Broker收到消息后的处理流程**

Broker中的Netty网络服务器获取到一条消息后：

首先，会把这条消息写入到一个CommitLog文件里。一个Broker机器上就只有一个CommitLog文件，所有Topic的消息都会写入到这个文件里。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/2f754e2d-294d-47dd-ba79-96d27befd56d" />

然后，Broker会以异步的方式把消息写入到一个ConsumeQueue文件里，因为一个Topic会有多个MessageQueue。任何一条消息都需要写入到一个MessageQueue的，一个MessageQueue其实就是对应了一个ConsumeQueue文件。所以一条写入MessageQueue的消息，必然会异步进入对应的ConsumeQueue文件，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/e9ec254d-fa93-4847-a1f6-1573d038da52" />

接着，Broker还会以异步的方式把消息写入到一个IndexFile文件里。在IndexFile文件里，会把每条消息的key和消息在CommitLog中的offset偏移量做一个索引，这样后续如果要根据消息key从CommitLog文件里查询消息，就可以根据IndexFile文件的索引来查询，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/363f9c68-0e85-4da7-811e-3898a9fc1477" />

<br>

**(2)Broker如何将消息写入CommitLog文件**

Broker收到一个消息后，首先会顺序写入CommitLog文件。CommitLog文件的存储目录是\${ROCKETMQ\_HOME}/store/commitlog，目录里会有很多CommitLog文件。每个文件默认是1GB大小，一个CommitLog文件写满了就创建一个新的CommitLog文件，文件名就是文件中的第一个偏移量。文件名如果不足20位，就用0来补齐。

    00000000000000000000
    000000000003052631924

Broker在把消息写入CommitLog文件时，会申请一个putMessageLock锁。也就是说，Broker写入消息到CommitLog文件时都是串行的，不会并发写入，因为并发写入文件必然会有数据错乱的问题。

下面是相关的源码片段：

    public class CommitLog {
        ...
        protected final PutMessageLock putMessageLock;
        ...
        public CompletableFuture<PutMessageResult> asyncPutMessage(final MessageExtBrokerInner msg) {
            ...
            putMessageLock.lock();
            ...
            result = mappedFile.appendMessage(msg, this.appendMessageCallback, putMessageContext);
            ...
        }
        ...
    }

在asyncPutMessage()方法中，获取到锁之后，会对消息做出一系列处理，包括设置消息的存储时间、创建全局唯一的消息ID、计算消息的总长度等。然后会执行MappedFile的appendMessage()方法，把消息写入到MappedFile里。

    public class MappedFile extends ReferenceResource {
        ...
        public AppendMessageResult appendMessage(final MessageExtBrokerInner msg, final AppendMessageCallback cb, PutMessageContext putMessageContext) {
            return appendMessagesInner(msg, cb, putMessageContext);
        }
        
        public AppendMessageResult appendMessagesInner(final MessageExt messageExt, final AppendMessageCallback cb, PutMessageContext putMessageContext) {
            ...
            ByteBuffer byteBuffer = writeBuffer != null ? writeBuffer.slice() : this.mappedByteBuffer.slice();
            byteBuffer.position(currentPos);
            AppendMessageResult result;
            if (messageExt instanceof MessageExtBrokerInner) {
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBrokerInner) messageExt, putMessageContext);
            } else if (messageExt instanceof MessageExtBatch) {
                result = cb.doAppend(this.getFileFromOffset(), byteBuffer, this.fileSize - currentPos, (MessageExtBatch) messageExt, putMessageContext);
            } else {
                return new AppendMessageResult(AppendMessageStatus.UNKNOWN_ERROR);
            }
            this.wrotePosition.addAndGet(result.getWroteBytes());
            this.storeTimestamp = result.getStoreTimestamp();
            return result;
        }
        ...
    }

上述源码中，其实最关键的是cb.doAppend()这行代码。cb.doAppend()会把消息追加到MappedFile映射的一块内存里去，并没有直接刷入到磁盘上的CommitLog文件，如下图示。至于具体什么时候才会把内存里的数据刷入磁盘上的CommitLog文件，这就要看配置的刷盘策略了。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/8455af12-a92c-49d0-9637-1c087f81aab9" />

另外，不管是同步刷盘还是异步刷盘，如果配置了主从同步，一旦将消息写入到CommitLog文件之后，接下来都会进行主从同步复制。

<br>

**7.Broker是如何实时更新索引文件的**

**(1)消息如何进入CommitLog**

**(2)消息如何进入ConsumeQueue和IndexFile**

<br>

**(1)消息如何进入CommitLog**

Broker收到一条消息后，会先把消息写入到CommitLog里。但是刚开始写入也仅仅是写入到MappedFile映射的一块内存，后续才会根据刷盘策略决定是否立即把数据从内存刷入磁盘。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/7470a48f-c0cc-4235-aa5e-661d874edc74" />

<br>

**(2)消息如何进入ConsumeQueue和IndexFile**

Broker启动时会启动一个叫ReputMessageService的线程，这个线程会把写入CommitLog的消息转发出去，也就是将消息写入(转发)到ConsumeQueue和IndexFile。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/4608d09b-065a-47f8-acf0-f17c140d3143" />

在DefaultMessageStore的start()方法里，会启动这个ReputMessageService线程。而DefaultMessageStore的start()方法是在Broker启动时被调用的，所以相当于Broker启动时就会启动这个ReputMessageService线程。

    public class BrokerController {
        ...
        public void start() throws Exception {
            //启动消息存储组件
            if (this.messageStore != null) {
                this.messageStore.start();
            }
            ...
        }
        ...
    }

    public class DefaultMessageStore implements MessageStore {
        ...
        public void start() throws Exception {
            ...
            this.reputMessageService.setReputFromOffset(maxPhysicalPosInLogicQueue);
            this.reputMessageService.start();
            ...
        }
        ...
    }

下面是ReputMessageService线程的源码：

    public class DefaultMessageStore implements MessageStore {
        ...
        class ReputMessageService extends ServiceThread {
            ...
            @Override
            public void run() {
                DefaultMessageStore.log.info(this.getServiceName() + " service started");
                while (!this.isStopped()) {
                    try {
                        Thread.sleep(1);
                        this.doReput();
                    } catch (Exception e) {
                        DefaultMessageStore.log.warn(this.getServiceName() + " service has exception. ", e);
                    }
                }
                DefaultMessageStore.log.info(this.getServiceName() + " service end");
            }
            
            private void doReput() {
                ...
                DispatchRequest dispatchRequest = DefaultMessageStore.this.commitLog.checkMessageAndReturnSize(result.getByteBuffer(), false, false);
                ...
                DefaultMessageStore.this.doDispatch(dispatchRequest);
                ...
            }
            ...
        }
        
        public void doDispatch(DispatchRequest req) {
            for (CommitLogDispatcher dispatcher : this.dispatcherList) {
                dispatcher.dispatch(req);
            }
        }
        ...
    }

由上述代码可知：在ReputMessageService线程里，每隔1毫秒就会把最近写入CommitLog的消息进行一次转发。其中会通过doReput()方法将消息转发到ConsumeQueue和IndexFile中。

在doReput()方法里，会从CommitLog中去获取到一个DispatchRequest，也就是从CommitLog中获取一份需要进行转发的消息。

接着，就会通过调用doDispatch()方法将消息转发到ConsumeQueue和IndexFile里，其中会通过遍历CommitLogDispatcher来实现。因为这个CommitLogDispatcher的实现类有两个，分别负责把消息转发到ConsumeQueue和IndexFile。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/3c233d60-2dfe-4dbe-9dc7-1e95592fc50d" />

ConsumeQueueDispatcher的写入逻辑，就是找到当前Topic的messageQueueId对应的一个ConsumeQueue文件。一个MessageQueue会对应多个ConsumeQueue文件，只要找到一个即可，然后就可以把消息写入其中。

    public class DefaultMessageStore implements MessageStore {
        ...
        class CommitLogDispatcherBuildConsumeQueue implements CommitLogDispatcher {
            @Override
            public void dispatch(DispatchRequest request) {
                final int tranType = MessageSysFlag.getTransactionValue(request.getSysFlag());
                switch (tranType) {
                    case MessageSysFlag.TRANSACTION_NOT_TYPE:
                    case MessageSysFlag.TRANSACTION_COMMIT_TYPE:
                        DefaultMessageStore.this.putMessagePositionInfo(request);
                        break;
                    case MessageSysFlag.TRANSACTION_PREPARED_TYPE:
                    case MessageSysFlag.TRANSACTION_ROLLBACK_TYPE:
                        break;
                }
            }
        }
        
        public void putMessagePositionInfo(DispatchRequest dispatchRequest) {
            ConsumeQueue cq = this.findConsumeQueue(dispatchRequest.getTopic(), dispatchRequest.getQueueId());
            cq.putMessagePositionInfoWrapper(dispatchRequest, checkMultiDispatchQueue(dispatchRequest));
        }
        ...
    }

IndexDispatcher的写入逻辑，就是在IndexFile里构建对应的索引。

    public class DefaultMessageStore implements MessageStore {
        ...
        class CommitLogDispatcherBuildIndex implements CommitLogDispatcher {
            @Override
            public void dispatch(DispatchRequest request) {
                if (DefaultMessageStore.this.messageStoreConfig.isMessageIndexEnable()) {
                    DefaultMessageStore.this.indexService.buildIndex(request);
                }
            }
        }
        ...
    }

<br>

**(3)总结**

当Broker把消息写入CommitLog后，会有一个后台线程每隔1毫秒拉取CommitLog中最新的一批消息，然后分别转发到ConsumeQueue和IndexFile中。

<br>

**8.Broker是如何实现同步刷盘以及异步刷盘的**

**(1)Broker收到消息后的存储流程**

**(2)消息的刷盘时机和策略**

**(3)Broker是如何处理刷盘的**

<br>

**(1)Broker收到消息后的存储流程**

Broker首先会将消息写入CommitLog，并且是先写入MappedFile映射的一块内存，而不是先写入磁盘。然后会有一个后台线程把CommitLog里的消息写入到ConsumeQueue和IndexFile里，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/950ab843-3de1-4ba3-90cf-6678c24b7240" />

<br>

**(2)消息的刷盘时机和策略**

当需要写入CommitLog的数据进入到MappedFile映射的一块内存后，就会开始执行刷盘策略。如果是同步刷盘，那么就会直接把内存里的数据写入磁盘文件。如果是异步刷盘，那么就会过一段时间后再把数据刷入磁盘文件。

在往CommitLog写数据时，会调用CommitLog的asyncPutMessage()方法，在这个方法的末尾有两行很关键的代码。一个是调用submitFlushRequest()方法，用于决定如何进行刷盘。一个是调用submitReplicaRequest()方法，用于决定如何把消息同步给Slave Broker。如下所示：

    public class CommitLog {
        ...
        public CompletableFuture<PutMessageResult> asyncPutMessages(final MessageExtBatch messageExtBatch) {
            ...
            //用于决定如何进行刷盘
            CompletableFuture<PutMessageStatus> flushOKFuture = submitFlushRequest(result, messageExtBatch);
            //用于决定如何把消息同步给Slave Broker
            CompletableFuture<PutMessageStatus> replicaOKFuture = submitReplicaRequest(result, messageExtBatch);
            return flushOKFuture.thenCombine(replicaOKFuture, (flushStatus, replicaStatus) -> {
                if (flushStatus != PutMessageStatus.PUT_OK) {
                    putMessageResult.setPutMessageStatus(flushStatus);
                }
                if (replicaStatus != PutMessageStatus.PUT_OK) {
                    putMessageResult.setPutMessageStatus(replicaStatus);
                }
                return putMessageResult;
            });
        }
        ...
    }

<br>

**(3)Broker是如何处理刷盘的**

接下来进入submitFlushRequest()方法看看Broker是如何处理刷盘的。

    public class CommitLog {
        ...
        public CompletableFuture<PutMessageStatus> submitFlushRequest(AppendMessageResult result, MessageExt messageExt) {
            //Synchronization flush——同步刷盘
            if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
                final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
                if (messageExt.isWaitStoreMsgOK()) {
                    GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes(), this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                    flushDiskWatcher.add(request);
                    service.putRequest(request);
                    return request.future();
                } else {
                    service.wakeup();
                    return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
                }
            }
            //Asynchronous flush——异步刷盘
            else {
                if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                    flushCommitLogService.wakeup();
                } else  {
                    commitLogService.wakeup();
                }
                return CompletableFuture.completedFuture(PutMessageStatus.PUT_OK);
            }
        }
        ...
    }

上述代码，就会根据配置的两种不同的刷盘策略，来分别进行处理的。

<br>

**一.同步刷盘的策略是如何处理的**

首先会构建一个GroupCommitRequest，然后提交给GroupCommitService去进行处理，接着调用request.future()等待同步刷盘成功。

具体的刷盘是由GroupCommitService执行的，它的doCommit()方法会执行同步刷盘的逻辑，代码如下：

    public class CommitLog {
        ...
        class GroupCommitService extends FlushCommitLogService {
            ...
            private void doCommit() {
                ...
                CommitLog.this.mappedFileQueue.flush(0);
                ...
            }
            ...
        }
        ...
    }

上述代码一层一层调用下去，可发现最终刷盘其实是靠MappedByteBuffer的force()方法，如下所示：

    public class MappedFile extends ReferenceResource {
        ...
        public int flush(final int flushLeastPages) {
            ...
            this.mappedByteBuffer.force();
            ...
        }
        ...
    }

这个MappedByteBuffer就是JDK NIO包下的API，MappedByteBuffer的force()方法会强迫将写入内存的数据刷入到磁盘文件里，执行完force()方法就代表同步刷盘成功了。

<br>

**二.异步刷盘的策略是如何处理的**

此时会唤醒一个flushCommitLogService组件。由于FlushCommitLogService是一个线程，它是一个抽象父类，它的子类是CommitRealTimeService。所以真正唤醒的是FlushCommitLogService的子类CommitRealTimeService线程。

在该线程里，会每隔一定时间执行一次刷盘，最大间隔是10s。所以一旦执行异步刷盘，那么最多10秒就会执行一次刷盘。

<br>

**9.Broker是如何清理存储较久的磁盘数据的**

**(1)定时检查是否要删除磁盘上的文件**

**(2)触发删除文件的条件**

**(3)删除文件的具体操作**

<br>

**(1)定时检查是否要删除磁盘上的文件**

默认情况下，Broker会启动一个后台线程，这个后台线程会自动检查CommitLog文件、ConsumeQueue文件，因为这些文件都会存在多个。如果发现比较旧的、超过72小时的文件，那么就会清理这些文件。

所以，默认情况下，Broker只会将消息保留3天，当然我们也可以通过fileReservedTime来自定义配置这个时间。

这个定时检查过期数据文件的线程，在DefaultMessageStore这个类里。在DefaultMessageStore的start()方法中，会调用addScheduleTask()方法每隔10s定时执行一个后台检查任务。如下所示：

    public class DefaultMessageStore implements MessageStore {
        ...
        public void start() throws Exception {
            ...
            this.addScheduleTask();
            ...
        }
        
        private void addScheduleTask() {
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    DefaultMessageStore.this.cleanFilesPeriodically();
                }
            }, 1000 * 60, this.messageStoreConfig.getCleanResourceInterval(), TimeUnit.MILLISECONDS);
            ...
        }
        ...
    }

在这个任务中，就会执行DefaultMessageStore的cleanFilesPeriodically()方法。其实也就是会周期性地清理掉磁盘上超过72小时的CommitLog、ConsumeQueue文件。

cleanFilesPeriodically()方法中包含了清理CommitLog和ConsumeQueue文件的逻辑：

    public class DefaultMessageStore implements MessageStore {
        ...
        private void cleanFilesPeriodically() {
            this.cleanCommitLogService.run();
            this.cleanConsumeQueueService.run();
        }
        ...
    }

<br>

**(2)触发删除文件的条件**

条件一：如果当前时间是预先设置的凌晨4点，就会触发执行一次删除文件的逻辑，这个时间是默认的

条件二：如果磁盘空间不足了也就是超过了85%的使用率，就会马上触发执行一次删除文件的逻辑

条件一指的是：如果磁盘没有满 ，那么每天会进行一次删除磁盘文件的操作，默认在凌晨4点执行，因为那个时候基本是业务低峰期。

条件二指的是：如果磁盘使用率超过85%了，那么此时可以允许继续在磁盘里写入数据，但会马上触发一次删除文件的操作。

注意：如果磁盘使用率超过90%了，那么此时是不允许再往磁盘里写入新数据的，同时会马上删除文件。因为一旦磁盘满了，那么写入磁盘就会失败，此时MQ就会出现故障。

<br>

**(3)删除文件的具体操作**

在删除文件时，无非就是对文件进行遍历。如果一个文件超过72小时都没修改过了，此时就可以删除了，哪怕有的消息可能还没有被消费，此时也不会再让消费者去消费了，直接删除掉。

<br>

**10.Consumer作为消费者是如何创建和启动的**

**(1)Cosumer是如何创建和启动的**

**(2)Consumer启动时的三个核心组件总结**

<br>

**(1)Cosumer是如何创建和启动的**

一般会通过DefaultMQPushConsumerImpl来创建Consumer，然后调用它的start()方法进行启动。

在执行start()方法启动Consumer的过程中，就会执行如下代码让Consumer和Broker建立长连接。只有建立了长连接，Consumer才能不断地从Broker中拉取消息。其中，MQClientFactory也是基于Netty来实现的。

    public class DefaultMQPushConsumerImpl implements MQConsumerInner {
        ...
        public synchronized void start() throws MQClientException {
            ...
            this.mQClientFactory = MQClientManager.getInstance().getOrCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);
            ...
        }
        ...
    }

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/86d915fe-04e2-4dc6-8766-5c67951eecb9" />

接着看start()方法的如下代码：

    public class DefaultMQPushConsumerImpl implements MQConsumerInner {
        ...
        public synchronized void start() throws MQClientException {
            ...
            this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
            this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
            this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
            this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);
            ...
        }
        ...
    }

上述代码的RebalanceImpl就是专门负责Consumer重平衡的。如果ConsumerGroup中加入了一个新的Consumer，那么就会重新分配每个Consumer消费的MessageQueue。如果ConsumerGroup里某个Consumer宕机了，那么也会重新分配MessageQueue，这就是所谓的重平衡。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/e39a9baa-b28a-49ed-9249-5a4ed8877e74" />

接着看start()方法的如下代码：

    public class DefaultMQPushConsumerImpl implements MQConsumerInner {
        ...
        public synchronized void start() throws MQClientException {
            ...
            this.pullAPIWrapper = new PullAPIWrapper(mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
            this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);
            ...
        }
        ...
    }

这个PullAPIWrapper就是消费者专门用来拉取消息的API组件。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/73795251-79b9-4cdf-aab5-49a3f03770d3" />

接着看start()方法的如下代码：

    public class DefaultMQPushConsumerImpl implements MQConsumerInner {
        ...
        public synchronized void start() throws MQClientException {
            ...
            if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
            } else {
                switch (this.defaultMQPushConsumer.getMessageModel()) {
                    case BROADCASTING:
                        this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    case CLUSTERING:
                        this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                        break;
                    default:
                        break;
                }
                this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
            }
            this.offsetStore.load();       
            ...
        }
        ...
    }

上面代码中的OffsetStore其实就是用来存储和管理Consumer消费进度offset的一个组件。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/00feed56-7783-435c-b887-ab8ddb057837" />

<br>

**(2)Consumer启动时的三个核心组件总结**

DefaultMQPushConsumerImpl的start()方法最核心的就是这三个组件。

首先Consumer刚启动，需要根据Rebalancer组件进行重平衡，确定自己要分配哪些MessageQueue之后才去拉取消息。

然后在拉取消息时，需要根据PullAPI组件通过底层网络通信发送请求进行拉取。

接着在拉取消息的过程中，需要根据OffsetStore组件来维护offset消费进度。

如果ConsumerGroup中多了Consumer或者少了Consumer，那么就需要根据Rebalancer组件来进行重平衡。

<br>

**11.消费者组的多个Consumer会如何分配消息**

**(1)Consumer的负载均衡问题**

**(2)重平衡组件如何分配MessageQueue**

<br>

**(1)Consumer的负载均衡问题**

当一个业务系统部署多台机器时，每个系统里都启动了一个Consumer。多个Consumer会组成一个ConsumerGroup，也就是消费者组。此时就会有一个消费者组内的多个Consumer同时消费一个Topic，而且这个Topic是有多个MessageQueue分布在多个Broker上的。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/31686a1a-5681-4532-93be-e004ccf493fd" />

那么问题来了：如果一个业务系统部署在两台机器上，对应一个消费者组里就有两个Consumer。而业务系统需要消费的一个Topic有三个MessageQueue，那么应该怎么分配呢？这就涉及到Consumer的负载均衡问题了。

前面介绍Consumer启动时，就介绍了几个关键的组件，分别是：重平衡组件、消息拉取组件、消费进度组件。其中的重平衡组件，就是专门负责处理多个Consumer的负载均衡问题的。

<br>

**(2)重平衡组件如何分配MessageQueue**

那么这个RebalancerImpl重平衡组件是如何将多个MessageQueue均匀的分配给一个消费者组内的多个Consumer的？

实际上，每个Consumer在启动后，都会向所有的Broker进行注册，并且持续保持自己的心跳，让每个Broker都能感知到一个消费者组内有哪些Consumer。下图中没有画出Consumer向每个Broker进行注册以及心跳，只能大致示意一下。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/0cdcc4cf-edf3-47ff-ba11-bc065542b7ef" />

每个Consumer在启动后，重平衡组件都会随机挑选一个Broker，从里面获取该消费者组里有哪些Consumer存在。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/0a08550d-c45a-4546-b804-6e5ae809b48c" />

当重平衡组件知道了消费者组内有哪些Consumer后，接下来就好办了。无非就是把Topic下的MessageQueue均匀地分配给这些Consumer。这时候其实有几种算法可以进行分配，但比较常用的一种算法就是平均分配。

假设现在一共有3个MessageQueue，有2个Consumer。那么就会给1个Consumer分配2个MessageQueue，给另外1个Consumer分配剩余的1个MessageQueue。

假设现在一共有4个MessageQueue，有2个Consumer。那么就可以2个Consumer各自分配2个MessageQueue。

总之一切都是平均分配，尽量保证每个Consumer的负载差不多。这样，一旦MessageQueue负载确定后，Consumer就知道自己要消费哪几个MessageQueue的消息，于是就可以连接到那个Broker上，从里面不停拉取消息过来进行消费。

<br>

**12.Consumer会如何从Broker拉取一批消息**

**(1)什么是消费者组**

**(2)集群模式消费 vs 广播模式消费**

**(3)MessageQueue和ConsumeQueue以及CommitLog之间的关系**

**(4)MessageQueue与消费者的关系**

**(5)Push消费模式 vs Pull消费模式**

**(6)Broker如何读取消息返回给消费者**

**(7)消费者如何处理消息、进行ACK响应以及提交消费进度**

**(8)消费者组出现宕机或扩容应如何处理**

**(9)消费源码的流程**

<br>

**(1)什么是消费者组**

**一.消费者组举例**

消费者组的意思就是给一组消费者起一个名字。比如有一个Topic叫TopicOrderPaySuccess，库存系统、积分系统、营销系统、仓储系统都要去消费这个Topic中的数据，那么此时应该给这四个系统分别起一个消费者组名字，如下所示：

    stock_consumer_group、marketing_consumer_group、
    credit_consumer_group、wms_consumer_group

设置消费者组的方式如下所示：

    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("stock_consumer_group");

假设库存系统部署了4台机器，每台机器上的消费者组的名字都是stock\_consumer\_group，那么这4台机器就同属于一个消费者组。以此类推，每个系统的几台机器都是属于各自的消费者组。

下图展示了两个系统，每个系统都有2台机器，每个系统都有一个自己的消费者组。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/3f6554fd-ee23-4fc3-a847-7e732ef6f859" />

<br>

**二.不同消费者组之间的关系**

假设库存系统和营销系统作为两个消费者组，都订阅了TopicOrderPaySuccess这个订单支付成功消息的Topic，此时如果订单系统作为生产者发送了一条消息到这个Topic，那么这条消息会被如何消费呢？

一般情况下，这条消息进入Broker后，库存系统和营销系统作为两个消费者组，每个组都会拉取到这条消息。也就是说，这个订单支付成功的消息，库存系统会获取到一条，营销系统也会获取到一条，它们俩都会获取到这条消息。

但库存系统这个消费者组里有两台机器，是两台机器都获取到这条消息、还是只有一台机器会获取到这条消息？

一般情况下，库存系统的两台机器中只有一台机器会获取到这条消息，营销系统也是同理。

下图展示了对于同一条订单支付成功的消息，库存系统的一台机器获取到了、营销系统的一台机器也获取到了。所以在消费时，不同的系统应该设置不同的消费者组。如果不同的消费者组订阅了同一个Topic，对Topic里的同一条消息，每个消费者组都会获取到这条消息。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/f997cbc5-2c95-46de-ad5f-e862435c4b12" />

<br>

**(2)集群模式消费 vs 广播模式消费**

对于一个消费者组而言，它获取到一条消息后，如果消费者组内部有多台机器，到底是只有一台机器可以获取到这个消息，还是每台机器都可以获取到这个消息？这就是集群模式和广播模式的区别。

默认情况下都是集群模式：即一个消费者组获取到一条消息，只会交给组内的一台机器去处理，不是每台机器都可以获取到这条消息的。

但是可以通过如下设置来改变为广播模式：

    consumer.setMessageModel(MessageModel.BROADCASTING);

如果修改为广播模式，那么对于消费者组获取到的一条消息，组内每台机器都可以获取到这条消息。但是相对而言广播模式用的很少，基本上都是使用集群模式来进行消费的。

<br>

**(3)MessageQueue和ConsumeQueue以及CommitLog之间的关系**

在创建Topic时，需要设置Topic有多少个MessageQueue。Topic中的多个MessageQueue会分散在多个Broker上，一个Broker上的一个MessageQueue会有多个ConsumeQueue文件。但在一个Broker运行过程中，一个MessageQueue只会对应一个ConsumeQueue文件。

对于Broker而言，存储在一个Broker上的所有Topic及MessageQueue数据都会写入一个统一的CommitLog文件，一个Broker收到的所有消息都会往CommitLog文件里面写。

对于Topic的各个MessageQueue而言，则是通过各个ConsumeQueue文件来存储属于MessageQueue的消息在CommitLog文件中的物理地址(即offset偏移量)。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/dcd9170c-86a4-4654-b889-aebdc1ece939" />

<br>

**(4)MessageQueue与消费者的关系**

一个Topic上的多个MessageQueue是如何让一个消费者组中的多台机器来进行消费的？可以简单理解为，它会均匀将MessageQueue分配给消费者组的多台机器来消费。

举个例子，假设TopicOrderPaySuccess有4个MessageQueue，这4个MessageQueue分布在两个Master Broker上，每个Master Broker上有2个MessageQueue。然后库存系统作为一个消费者组，库存系统里有两台机器。那么正常情况下，最好就是让这两台机器各自负责2个MessageQueue的消费。比如库存系统的机器01从Master Broker01上消费2个MessageQueue，库存系统的机器02从Master Broker02上消费2个MessageQueue。这样就能把消费的负载均摊到两台Master Broker上。

所以大致可以认为一个Topic的多个MessageQueue会均匀分摊给消费者组内的多个机器去消费。

这里的一个原则是：一个MessageQueue只能被一个消费者机器去处理，但是一台消费者机器可以负责多个MessageQueue的消息处理。

<br>

**(5)Push消费模式 vs Pull消费模式**

**一.一般选择Push消费模式**

既然一个消费者组内的多台机器会分别负责一部分MessageQueue的消费的，那么每台机器都必须要连接到对应的Broker，尝试消费里面MessageQueue对应的消息。于是就涉及到两种消费模式了，一个是Push模式、一个是Pull模式。

这两个消费模式本质上是一样的，都是消费者主动发送请求到Broker去拉取一批消息进行处理。

Push消费模式是基于Pull消费模式来实现的，只不过它的名字叫做Push而已。在Push模式下，Broker会尽可能实时把新消息交给消费者进行处理，它的消息时效性会更好。

一般我们使用RocketMQ时，消费模式通常都选择Push模式来，因为Pull模式的代码写起来更加复杂和繁琐，而且Push模式底层本身就是基于Pull模式来实现的，只不过时效性更好而已。

<br>

**二.Push消费模式的实现思路**

当消费者发送请求到Broker去拉取消息时，如果有新的消息可以消费，那么就马上返回一批消息到消费机器去处理。消费者处理完之后，会接着发送请求到Broker机器去拉取下一批消息。

所以，消费者机器在Push模式下处理完一批消息，会马上发起请求拉取下一批消息，消息处理的时效性非常好，看起来就像Broker一直不停的推送消息到消费机器一样。

此外，Push模式下有一个请求挂起和长轮询的机制：当拉取消息的请求发送到Broker，Broker却发现没有新的消息可以处理时，就会让处理请求的线程挂起，默认是挂起15秒。然后在挂起期间，Broker会有一个后台线程，每隔一会就检查一下是否有新的消息。如果有新的消息，就主动唤醒被挂起的请求处理线程，然后把消息返回给消费者。

可见，常见的Push消费模式，本质也是消费者不断发送请求到Broker去拉取一批一批的消息。

<br>

**(6)Broker如何读取消息返回给消费者**

Broker在收到消费者的拉取请求后，是如何将消息读取出来，然后返回给消费者的？这涉及到ConsumeQueue和CommitLog。

假设一个消费者发送了拉取请求到Broker，表示它要拉取MessageQueue0中的消息，然后它之前都没拉取过消息，所以就从这个MessageQueue0中的第一条消息开始拉取。

于是，Broker就会找到MessageQueue0对应的ConsumeQueue0，从里面找到第一条消息的offset。接着Broker就需要根据ConsumeQueue0中找到的第一条消息的地址，去CommitLog中根据这个offset地址读取出这条消息的数据，然后把这条消息的数据返回给消费者。

所以消费者在消费消息时，本质就是：首先根据要消费的MessageQueue以及开始消费的位置，去找到对应的ConsumeQueue。然后在ConsumeQueue中读取要消费的消息在CommitLog中的offset偏移量。接着到CommitLog中根据offset读取出完整的消息数据，最后将完整的消息数据返回给消费者。

<br>

**(7)消费者如何处理消息、进行ACK响应以及提交消费进度**

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

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/23bc2c38-eab8-4543-a40d-3996f425789b" />

<br>

**(8)消费者组出现宕机或扩容应如何处理**

此时会进入一个Rebalance环节，也就是重新给各个消费者分配各自需要处理的MessageQueue。

比如现在机器01负责MessageQueue0和MessageQueue1，机器02负责MessageQueue2和MessageQueue3。如果现在机器02宕机了，那么机器01就会接管机器02之前负责的MessageQueue2和MessageQueue3。如果此时消费者组加入了一台机器03，那么就可以把机器02负责的MessageQueue3转移给机器03，然后机器01只负责一个MessageQueue2的消费，这就是负载重平衡。

<br>

**(9)消费源码的流程**

拉取消息的源码入口在DefaultMQPushConsumerImpl类的pullMessage()方法，里面涉及了：拉取请求、消息流量控制、通过PullAPIWrapper与服务端进行网络交互、服务端根据ConsumeQueue文件拉取消息等事情。
