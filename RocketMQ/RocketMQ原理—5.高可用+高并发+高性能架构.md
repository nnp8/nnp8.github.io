# RocketMQ原理—5.高可用+高并发+高性能架构

**大纲**

**1.RocketMQ的整体架构与运行流程**

**2.基于NameServer管理Broker集群的架构**

**3.Broker集群的主从复制架构**

**4.基于Topic和Queue实现的数据分片架构**

**5.Broker基于Pull模式的主从复制原理**

**6.Broker层面到底如何做到数据0丢失**

**7.数据0丢失与写入高并发的取舍**

**8.RocketMQ读写分离主从漂移设计**

**9.RocketMQ为什么采取惰性读写分离模式**

**10.Broker数据与服务是否都实现高可用了**

**11.Broker数据与服务的数据一致性设计**

**12.Broker基于Raft协议的主从架构设计**

**13.Raft协议的Leader选举算法介绍**

**14.Broker基于状态机实现的Leader选举**

**15.Broker基于DLedger的数据写入流程**

**16.Broker引入DLedger后的存储兼容设计**

**17.Broker主从节点之间的元数据同步**

**18.Broker基于Raft协议的主从切换机制**

**19.Consumer端队列负载均衡分配机制**

**20.Consumer消息拉取的挂起机制分析**

**21.Consumer的处理队列与并发消费**

**22.Consumer处理成功后的消费进度管理**

**23.Consumer消息重复消费原理剖析**

**24.Consumer处理失败时的延迟消费机制**

**25.ConsumerGroup变动时的重平衡机制**

<br>

**1.RocketMQ的整体架构与运行流程**

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/341e2d60-dba2-4cde-a7a4-1d84603ffe93" />

<br>

**2.基于NameServer管理Broker集群的架构**

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/c40332f3-ea8c-41a6-a158-7b0fc005c8e6" />

<br>

**3.Broker集群的主从复制架构**

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/337cc955-838c-4d2c-b94c-581071320c32" />

<br>

**4.基于Topic和Queue实现的数据分片架构**

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/d3806e13-c441-49ce-90ae-760bcfc27fe8" />

<br>

**5.Broker基于Pull模式的主从复制原理**

**(1)Broker主从复制的Push模式和Pull模式**

**(2)Broker基于Pull模式的主从复制原理**

**(3)Push消费模式 vs Pull消费模式**

<br>

**(1)Broker主从复制的Push模式和Pull模式**

Push同步模式：Producer往Broker主节点写入数据后，Broker主节点会主动把数据推送Push到Broker从节点里。

Pull同步模式：Producer往Broker主节点写入数据后，Broker主节点会等待从节点发送拉取数据的Pull请求。Broker主节点收到从节点的拉取数据请求后，才会把数据发送给从节点。

其实，Broker发送消息给Consumer进行消费，同样也是有两种模式：Push模式和Pull模式。

Push消费模式：就是Broker会主动把消息发送给消费者，消费者是被动接收Broker推送过来的消息，然后进行处理。

Pull消费模式：就是Broker不会主动推送消息给消费者，而是消费者主动发送请求到Broker去拉取消息，然后进行处理。

<br>

**(2)Broker基于Pull模式的主从复制原理**

Broker主节点在启动后，会监听来自从节点的连接请求。Broker从节点在启动后，会主动向主节点发起连接请求。

首先，当Broker主节点和从节点建立好网络连接后，各自会初始化一些组件。Broker主节点会创建并初始化一个HAConnection组件，专门用于处理从节点的同步请求。Broker从节点会创建并初始化两个组件，一个是HAClient主从同步请求线程、一个是HAClient主从同步响应线程。

然后，Broker从节点的HAClient主从同步请求线程，会不断发送主从同步请求，到主节点的HAConnection组件，期间会带上从节点向主节点已拉取的最大的物理偏移量max offset。

接着，主节点的HAConnection组件便会到磁盘文件取出最大物理偏移量max offset之后的数据，然后返回给从节点。

之后，从节点的HAClient主从同步响应线程便会对收到的这些max offset之后的数据进行处理，写入到其磁盘文件中。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/9be649ea-e126-405f-afa5-22b5ee5b3a24" />

<br>

**(3)Push消费模式 vs Pull消费模式**

一个消费者组内的多台机器会分别负责一部分MessageQueue的消费。那么每台机器就必须要连接到对应的Broker，尝试消费里面的MessageQueue对应的消息。此时就涉及到两种消费模式了：Push消费模式和Pull消费模式。

实际上，这两个消费模式本质是一样的，都是消费者机器主动发送请求到Broker机器去拉取一批消息来处理。Push消费模式底层是基于Pull消费模式来实现的，只不过它的名字叫做Push而已。意思是Broker会尽可能实时的把新消息交给消费者机器来进行处理，它的消息时效性会更好。

一般使用RocketMQ时，消费模式通常选择Push模式，因为Pull模式的代码写起来更加复杂和繁琐，而且Push模式底层本身就是基于消息拉取的方式来实现的，只不过时效性更好而已。

Push模式的实现思路：当消费者发送请求到Broker去拉取消息时，如果有新的消息可以消费就马上返回一批消息到消费机器去处理，处理完之后会接着立刻发送请求到Broker机器去拉取下一批消息。

所以消费者在Push模式下处理完一批消息，会马上发起请求拉取下一批消息。消息处理的时效性非常好，看起来就像Broker一直不停地推送消息给消费者一样。

此外，Push模式下有一个请求挂起和长轮询机制。当拉取消息的请求发送到Broker，结果发现没有新的消息给处理时，就会让请求线程挂起，默认是挂起15秒。然后这个期间Broker会有后台线程每隔一会儿就去检查一下是否有新的消息。另外在这个挂起过程中，如果有新的消息到达了会主动唤醒挂起的线程，然后把消息返回给消费者。

当然消费者进行消息拉取的底层源码是比较复杂的，涉及大量细节，但核心思路大致如此。需要注意：哪怕是常用的Push消费模式，本质也是消费者不停地发送请求到Broker去拉取一批一批的消息。

<br>

**6.Broker层面到底如何做到数据0丢失**

第一种场景：Broker主节点JVM进程崩溃 ==> 数据在PageCache，即便异步刷盘也不影响。

第二种场景：Broker主节点服务器崩溃 ==> 数据在PageCache，异步刷盘会丢失，同步刷盘不丢失。

第三种场景：Broker主节点服务器硬盘坏掉 ==> 如果Broker从节点数据还没同步最新的，就会丢失。

Broker要做到数据0丢失，需要同步刷盘 + 同步复制。也就是Producer向Broker主节点写数据时，Broker主节点返回写入成功响应的前提是：主节点落盘成功 + 从节点同步成功。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/34020801-04ec-4450-833b-a8baefc1e6f3" />

<br>

**7.数据0丢失与写入高并发的取舍**

Producer将一条消息发送给Broker：

若Broker采用异步刷盘 + 异步复制，那么基本几ms \~ 几十ms即可返回写入成功的响应。

若Broker采用同步刷盘 + 异步复制，那么需要几十ms \~ 几百ms才可返回写入成功的响应。

若Broker采用同步刷盘 + 同步复制，那么需要几百ms \~ 几s才可返回写入成功的响应。

同步复制需要等待：从节点发送Pull请求 + 读磁盘数据 + 网络发送数据到从节点 + 从节点写数据到磁盘 + 等下一次Pull请求。

Broker默认采用的是异步刷盘 + 异步复制。

<br>

**8.RocketMQ读写分离主从漂移设计**

**(1)优先从Broker主节点消费消息**

**(2)读写分离主从漂移的规则**

<br>

**(1)优先从Broker主节点消费消息**

RocketMQ是不倾向让Producer和Consumer进行读写分离的，而是倾向让写和读都由主节点来负责。从节点则用于进行数据复制和同步来实现热备份，如果主节点挂了才会选择从节点进行数据读取。所以Consumer默认下会消费Broker主节点的ConsumeQueue里的消息。

<br>

**(2)读写分离主从漂移的规则**

如果Broker主节点过于繁忙，比如积压了大量写和读的消息超过本地内存的40%，那么当Consumer向Broker主节点发起一次拉取消息的请求后，Broker主节点会通知Consumer下一次去该Broker的某个从节点拉取消息。

而当Consumer向Broker从节点拉取消息一段时间后，从节点发现自己本地消息积压小于本地内存30%，拉取消息很顺利，那么Broker从节点会通知Consumer下一次回到Broker主节点去拉取消息。

<br>

**9.RocketMQ为什么采取惰性读写分离模式**

**(1)什么是惰性读写分离模式**

**(2)MQ要实现真正的读写分离比较麻烦**

**(3)Broker从节点每隔10秒同步消费进度**

<br>

**(1)什么是惰性读写分离模式**

惰性读写分离其实就是上面说的读写分离主从漂移。这种漂移指的是：主从机器对外提供一个完整的服务，客户端有时候访问主、有时候访问从。

惰性读写分离不属于彻底的读写分离，从节点的数据更多时候是用于备用。在以下两种情况下，Consumer才会选择到从节点去读取消费数据。

情况一：如果主节点过于繁忙，积压没有消费的消息太多，已经超过本地内存40%。此时主节点可能出现大量读写线程并发运行，机器运行效率可能已经降低，来不及处理这么多的请求，那么主节点就会让消费请求漂移到从节点去读取消费数据。如果在从节点消费得非常好，消息的积压数量很快下降到从节点本地内存的30%以内，就又会让Consumer漂移回主节点消费。

情况二：如果主节点崩溃了，那么Consumer也只能到从节点去读取数据进行消费。

<br>

**(2)MQ要实现真正的读写分离比较麻烦**

RocketMQ作为一个MQ，一个Topic对应多个Queue，可以认为支持去从节点读取数据进行消费。

Kafka作为一个MQ，一个Topic对应多个Partition，不同节点组成Leader和Follower主从结构进行数据复制，不支持去从节点读取数据进行进行。

MQ作为一个特殊的中间件系统，它要维护每个Consumer对一个Queue/Partition的消费进度。如果要实现真正的读写分离，那么维护这个消费进度就会非常麻烦。

比如在从节点上进行读取消费时，一旦这个从节点宕机，此时主节点和其他从节点是不知道该从节点的消费进度的，消费进度要进行集中式保存就比较麻烦。

所以考虑到消费进度的维护和保存，通常各个MQ都会让消费者在主节点进行读和写，这样就可以简单地对消费进度进行集中式维护和存储。

<br>

**(3)Broker从节点每隔10秒同步消费进度**

由于Consumer向RocketMQ的Broker主节点进行消费时，有时候会漂移到Broker从节点进行消费。所以Broker从节点会每隔10s去Broker主节点进行元数据同步，比如同步给主节点最新的消费进度。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/a171d61a-646c-4964-af15-2b1b17e5bed3" />

由此可见，RocketMQ采用惰性读写分离，主要是为了避免维护不同消费者组去不同从节点消费时产生的复杂的消费进度。

<br>

**10.Broker数据与服务是否都实现高可用了**

**(1)RocketMQ4.5.0之前**

**(2)RocketMQ4.5.0之后**

<br>

**(1)RocketMQ4.5.0之前**

Broker主节点崩溃后，是没有高可用主从切换机制的，从节点只用于热备份，保证大部分的数据不会丢失而已。

由于Broker提供的服务就两个：一个是写数据、一个是读数据。所以此时主节点崩溃后，只能靠从节点提供有限的数据和服务了，即只能提供读数据服务而不能提供写数据服务。对应的Producer都会出现写数据失败，但是Consumer可以继续去从节点读取数据进行有限的消费，数据消费完就没了。

此外主节点崩溃后，从节点可能存在有些最新的数据没来得及同步过来，出现数据丢失的问题，所以数据和服务没有实现高可用。

<br>

**(2)RocketMQ4.5.0之后**

实现了主从同步 + 主从切换的高可用机制，保证数据和服务都是高可用的。

注意：在RocketMQ4.5.0以前的老版本，只是实现了单纯的主从复制，只能做到大部分数据不丢失，效果不是特别好。某个Broker分组内的主节点挂掉后，从节点是没法接管主节点的工作的。

<br>

**11.Broker数据与服务的数据一致性设计**

要实现主从数据强一致同步：

**情况一：如果主从同步采用Pull模式**

那么Broker主节点就要等待从节点过来Pull数据，从而增加Producer的写请求耗时，此时整个写请求的性能损耗比较大。

<br>

**情况二：如果主从同步采用Push模式**

Broker主节点将消息写入PageCache后，就Push给从节点进行同步，那么写请求只要等待从节点的Push成功即可返回。Broker从节点继续采取异步刷盘的策略，它收到主节点Push的消息后，直接写入PageCache就返回给主节点。此时整个写请求的性能损耗比较小。

<br>

**12.Broker基于Raft协议的主从架构设计**

如果基于Raft协议，那么一组Broker最少需要使用三台机器。

**一.当这3台Broker启动后**

会基于Raft协议进行Leader选举，选举出的Leader便会成为Broker主节点。

<br>

**二.当Producer往Broker主节点发起写请求时**

Broker主节点首先会将新消息先写入到OS的PageCache中，接着将新消息同步Push到其余两台从节点。由于基于Raft协议，所以只要Broker主节点发现过半数Broker节点(包括它自己)写入新消息成功，那么Broker主节点就可以返回写入成功。而Broker从节点收到主节点的Push新消息请求后，也是首先写入OS的PageCache，然后就直接返回写入成功给Broker主节点。

<br>

**(3)当Broker主节点宕机后**

剩余的两台Broker从节点便会根据Raft协议进行Leader选举，选举其中一台Broker作为新的主节点。这样一个Broker主节点 + 一个Broker从节点，依然可以满足Raft协议，继续提供写服务和保证数据及服务的高可用。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/6edf9664-60ef-4663-b795-58f38571d518" />

<br>

**13.Raft协议的Leader选举算法介绍**

说明一：各个节点在启动时都是Follower。

说明二：每个Follower都会给自己设置一个150ms\~300ms之间的一个随机时间，可以理解为一个随机的倒计时时间。也就是说，有的Follower可能倒计时是150ms、有的Follower可能倒计时是200ms，每个Follower的倒计时一般不一样。这时必然会存在一个Follower，它的倒计时是最小的，它会最先到达倒计时的时间。

说明三：第一个完成倒计时的Follower会把自己的身份转变为Candidate，变成一个Leader候选者，会开始竞选Leader。于是它会给自己投票想成为Leader，此外它还会发送请求给其他节点表示它完成了一轮投票，希望对方也投票给自己。

说明四：其他还处于倒计时中的Follower节点收到这个请求后，如果发现自己还没给其他Candidate投过票，那么就把它自己的票投给这个Candidate，并发送请求给其他节点进行通知。如果发现自己已经给其他Candidate投过票，那么就忽略这个Candidate发送过来的请求。

说明五：当某个Candidate发现自己的得票数已超半数quorum，那么它就成为Leader了，这时它会向其他节点发送Leader心跳。那些节点收到这个Leader的心跳后，就会重置自己的倒计时，比如原来的倒计时还剩10ms，收到Leader心跳时就重置为200ms。Follower节点通过Leader的心跳去不断重置自己的倒计时，不让倒计时到达，以此来维持Leader的地位，否则倒计时一到达，它就会从Follower转变为Candidate发起新一轮的Leader选举。

<br>

**14.Broker基于状态机实现的Leader选举**

**(1)什么是状态设计模式**

**(2)什么是状态机**

**(3)使用状态机机制实现Leader选举**

<br>

**(1)什么是状态设计模式**

就是系统可以维护多个State状态，多个State状态之间可以进行切换。每次切换到一个新的State状态后，执行的行为是不同的，行为是跟State状态是绑定在一起的。状态机就是状态设计模式的一个运用。

<br>

**(2)什么是状态机**

状态设计模式可以演变成一个状态机，即StateMachine。状态机和状态设计模式一样，可以维护多个State状态，不同的State状态可以对应不同的行为。RocketMQ的Broker就是采取状态机机制来实现Leader选举的。

<br>

**(3)使用状态机机制实现Leader选举**

说明一：同一个组的每个Broker节点在启动时都会有一个状态机StateMachine，这个状态机会对节点状态进行判断。而这些节点在启动时的初始化状态都是Follower，所以状态机就会根据Follower状态让节点执行maintainAsFollower行为。

说明二：maintainAsFollower行为会判断是否收到Leader的心跳包。如果没收到心跳包就等待倒计时结束，节点切换成Candidate状态。如果收到心跳包就重置倒计时，节点切换成Follower状态。

说明三：当状态机发现节点的状态由Follower变成了Candidate，那么就会让节点执行maintainAsCandidate行为。

说明四：刚开始启动时各个节点都没有收到心跳包，都在等待各自的随机倒计时的结束。假设节点A的倒计时先结束，其节点状态由Follower切换为Candidate。

说明五：maintainAsCandidate行为会发起一轮新的投票，比如节点A会先投票给自己，然后发送该投票结果给组内的其他节点。组内其他节点收到该投票结果后会进行投票响应，如果发现其状态为Follower且没投过票，响应就是投票给节点A。

说明六：节点A收到其他节点的投票响应后，其状态机就会判断是否有超过半数的节点都对自己投票了。如果没有收到半数投票，就重置倒计时，等待下轮选举投票。如果收到半数投票，就开始把自己切换为Leader状态。当状态机发现节点的状态由Candidate变成了Leader，就会让节点执行maintainAsLeader行为。

说明七：maintainAsLeader行为会定时发送HeartBeat心跳请求给其他Follower节点。当其他Follower节点收到心跳请求包后，就会重置倒计时，并且返回心跳结果响应给Leader节点，然后等待倒计时结束。

说明八：当Leader节点收到这些心跳结果响应后，会判断是否超过半数节点进行了心跳响应。如果是则继续定时发送HeartBeat心跳请求给其他Follower节点，如果不是则把状态切换为Candidate状态。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/85a5734c-9d38-4b06-a6da-a783a200277f" />

注意关键点：是否超半数投票 + 是否超半数心跳响应。

<br>

**15.Broker基于DLedger的数据写入流程**

在Raft协议下，RocketMQ的Leader可以对外提供读和写服务，Follower则一般不对外提供服务，仅仅进行数据复制和同步，以及在Leader故障时完成Leader重新选举继续对外服务。

DLedger是一个实现了Raft协议的框架，它实现了Leader如何选举、数据如何复制、主从如何切换等功能。当Broker拿到一条消息准备写入时，就会切换为基于DLedger来进行写入，不过DLedger里写的不叫消息，而叫日志。

Broker节点的DLedger也是先往PageCache里写日志，然后会有后台线程进行异步刷盘将日志写入磁盘。而Leader节点的DLedger在往PageCache写完日志后，会异步复制日志到其他Follower节点，然后Leader节点会同步阻塞等待这些Follower节点写入日志的结果。当Leader节点发现过半Follower节点写入消息成功后，才会向Producer返回写入成功的响应，代表这条消息写入成功。

如下是Broker基于DLedger的数据写入流程：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/6f56b100-381e-4442-9da0-ed9204ba7599" />

<br>

**16.Broker引入DLedger后的存储兼容设计**

**(1)过半节点写成功才返回写入成功**

**(2)DLedger的日志格式**

<br>

**(1)过半节点写成功才返回写入成功**

在基于DLedger的数据写入过程中，消息仅仅写入Leader上DLedger的PageCache时，还不能代表这条消息的写入已经成功。还需要等待超过半数节点写入成功后，Leader向Producer返回这条消息已经写入成功了，才能让消费者去消费该条消息。

<br>

**(2)DLedger的日志格式**

消息被写入DLedger的PageCache时，由于数据会被调整为DLedger的日志格式，不再是没有使用DLedger时写入PageCache的CommitLog消息格式，那么该如何兼容这个DLedger的日志格式？

DLedger的日志会分成Body和Header两部分：Header部分中会包含很多的Header头字段，Body部分会包含长度不固定的Body体。

所以DLedger的一条日志中，会把CommitLog原始的一条数据放入到其Body部分，也就是：一条DLedger日志 = Header(多个头字段) + Body(CommitLog原始数据)。

因此使用DLedger写入消息到PageCache后，后台线程异步刷盘到CommitLog文件的每一条数据都会有"Header + Body"。此时如果从ConsumeQueue获取到偏移量后继续从Header开始去计算就找不到原始的CommitLog数据了。

所以需要对ConsumeQueue的数据也进行设计兼容：ConsumeQueue的一条数据里的offset物理偏移量，需要更改为CommitLog里一条数据的Body的起始物理偏移量。

<br>

**17.Broker主从节点之间的元数据同步**

元数据包括：Topic路由信息(比如Topic在当前的Broker组里有几个Queue)、消费进度数据。

因为这些元数据都是存储在Broker的Leader节点上的，也需要同步到Broker的Follower节点，所以Follower节点会启动一个定时任务每10s去Leader节点同步元数据。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/b1473744-f8e9-4966-a057-2c36f442d02a" />

<br>

**18.Broker基于Raft协议的主从切换机制**

Broker基于Raft协议的主从切换机制如下：

说明一：Broker组内的各个节点一开始启动时都是Follower状态，都会判断自己是否有收到Leader心跳包。由于刚开始启动时没有Leader，所以各个节点不会收到心跳包，于是都会等待随机倒计时结束，准备切换成Candidate状态。

说明二：其中的一个节点必然会优先结束随机倒计时并切换成Candidate状态。该节点切换成Candidate状态后，就会发起一轮新的选举，也就是给自己进行投票，并且把该投票也发送给其他节点。

说明三：其他节点收到该投票后，就会判断自己是否已给某节点投票。此时这些节点并没有给某节点投过票，并且都还处于Follower状态，其倒计时还没有结束，于是这些节点便会把票投给第一个结束随机倒计时的节点。否则，就忽略该投票。

说明四：第一个结束随机倒计时的节点收到其他节点的投票信息后，会判断投自己的票是否已超半数。如果是，则把Candidate状态切换成Leader状态。

说明五：第一个结束随机倒计时的节点把状态切换成Leader后，就会定时给其他节点发送HeartBeat心跳。其他节点收到心跳后，就会重置倒计时。

所以只要Leader正常运行，定时发送心跳过来重置倒计时，那么这些节点的倒计时永远不会结束，从而这些节点会一直维持着Follower状态。

说明六：这样，处于Leader状态的节点，和一直维持Follower状态的那些节点，就会正常工作。Producer的消息会往Leader节点进行写入，然后会被复制到Follower节点。Consumer消费消息也会从Leader节点进行读取，然后其Leader节点的元数据也会定时同步到Follower节点上。

之所以Leader节点能一直维持其Leader地位，是因为Leader节点会一直定时给Follower节点发送HeartBeat心跳，然后让这些Follower节点一直在重置自己的随机倒计时，让倒计时永远无法结束。否则，一旦Follower节点的随机倒计时结束，它就会将自己的状态切换成Candidate，并发起一轮新的Leader选举。

说明七：假设此时Leader节点崩溃了，比如Broker JVM进程进行了正常的重启。那么该Leader节点就无法给Follower节点定时发送HeartBeat心跳了。于是那些Follower节点便会判断出没有收到心跳包，从而会等待其倒计时结束，切换成Candidate状态。

其中必定会有一个Follower节点先结束倒计时切换成Candidate状态，然后发起新的Leader选举。当它发现有过半数节点给自己投票了之后，便会切换成Leader状态，完成主从切换，恢复工作。

说明八：由于新的Leader之前是有完整的消息数据和元数据，所以新Leader只要切换成功，Consumer和Producer继续往新Leader读写即可。新的Leader会继续给其他Follower节点同步数据、定时发送Leader心跳包让Follower节点无法切换成Candidate状态。

注意：基于Raft协议的Broker集群，每一组Broker至少需要部署3个节点。

<br>

**19.Consumer端如何负载均衡分配Queue**

**(1)关于Topic的Queue分配问题**

**(2)Consumer的RebalanceService组件和算法**

<br>

一个Topic会有多个Queue，而且会分布在不同的Broker上。而一个ConsumerGroup会有多个Consumer，所以需要把多个Queue分配给多个Consumer，这样每个Consumer都会分配到一部分的Queue。

<br>

**(1)关于Topic的Queue分配问题**

**问题1：谁来负责把一个Topic的多个Queue分配到多个Consumer**

Consumer自己可以负责进行分配。每个Consumer都可以获取到一个Topic有多少个Queue，以及自己所处的ConsumerGroup里有多少个Consumer，每个Consumer都可以按照相同的算法去做一次分配。

<br>

**问题2：一个Topic里的Queue信息应该从哪里获取**

从NameServer获取Topic里的Queue信息。Broker往NameServer进行注册和发送心跳时，都会带上该Broker上的Topic路由信息，可以理解为通过Broker也能获取完整的Topic路由信息(只需向所有Broker查询即可)。

<br>

**问题3：如何知道一个ConsumerGroup里到底有多少个Consumer**

每个Broker都可以知道一个ConsumerGroup的所有Consumer。Consumer启动时，会向所有的Broker进行注册。所以每个Broker都可以知道一个ConsumerGroup的所有Consumer都有哪些，可以通过随便一个Broker来获取Topic的路由信息 + ConsumerGroup信息。

<br>

**(2)Consumer的RebalanceService组件和算法**

Consumer中会有一个RebalanceService组件，负责每隔20秒去拉取Topic的Queue信息、ConsumerGroup信息，然后根据算法分配Queue，最后确认自己要拉取哪些Queue上的信息。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/386d9899-4182-4e6c-985d-6b8b172ec7a8" />

这些算法有：平均分配算法(热门)、轮询分配算法(热门)、一致性Hash(冷门)、机房分配(冷门)、配置分配(冷门)。

假设有两个Broker组、某个Topic有8个Queue：q1、q2、q3、q4、q5、q6、q7、q8，还有2个Consumer：Consumer1、Consumer2。

如果按照平均分配算法进行分配，那么Consumer1可能会分配到这4个Queue：q1、q2、q3、q4，Consumer2可能会分配到这4个Queue：q5、q6、q7、q8。

如果按照轮询分配算法进行分配，那么Consumer1可能会分配到这4个Queue：q1、q3、q5、q7，Consumer2可能会分配到这4个Queue：q2、q4、q6、q8。

<br>

**20.Consumer消息拉取的挂起机制分析**

**(1)短轮询机制(Short Polling)**

**(2)长轮询机制(Long Polling)**

**(3)总结Consumer的Push模式和Pull模式**

<br>

Consumer拉取消息时会有两种机制：长轮询机制(Long Polling)和短轮询机制(Short Polling)。Consumer如果没有开启长轮询机制(Long Polling)，那么就会使用短轮询机制(Short Polling)去拉取消息。

<br>

**(1)短轮询机制(Short Polling)**

短轮询指的是短时间(默认1秒)挂起去进行消息拉取，这个1秒可以由shortPollingMillis参数进行控制。当Consumer发起请求去Leader节点拉取消息时，默认会采用短轮询机制。

如果Leader节点上处理该请求的线程时，发现没有消息就会挂起1秒，挂起过程中并不会有响应返回给Consumer。1秒后该线程会苏醒，然后再去检查Leader节点是否有消息了，如果还是没有消息，就返回Not Found Message给Consumer。

<br>

**(2)长轮询机制(Long Polling)**

长轮询其实指的就是长时间挂起去进行消息拉取。在开启了长轮询机制的情况下，当Consumer发起请求去Leader节点拉取消息时，如果Leader节点上处理该请求的线程发现没有消息，那么就会直接挂起，挂起过程中并不会有响应返回给Consumer。

同时，Leader节点中会有一个长轮询后台线程，每隔5秒去检查Leader节点是否有新的消息进来。如果检查到有新消息则唤醒挂起的线程，并判断该消息是否是Consumer所感兴趣的。如果不是Consumer感兴趣的，则再判断是否长轮询超时，如果超时则返回Not Found Message给Consumer。

当Consumer采用Push模式去拉取消息时，那么会：挂起 + 每隔5秒检查 + 超时时间为15秒，15秒都没拉到消息就超时返回。

当Consumer采用Pull模式去拉取消息时，那么会：挂起 + 每隔5秒检查 + 超时时间为20秒，20秒都没拉到消息就超时返回。

<br>

**(3)总结Consumer的Push模式和Pull模式**

实际上，这两个消费模式本质是一样的，都是消费者机器主动发送请求到Broker机器去拉取一批消息来进行处理。

Push消费模式底层也是基于消费者Pull模式来实现的，只不过它的名字叫做Push而已。意思是Broker会尽可能实时的把新消息交给消费者机器来进行处理，它的消息时效性会更好。

一般使用RocketMQ时，消费模式通常都是选择Push模式，因为Pull模式的代码写起来更加的复杂和繁琐，而且Push模式底层本身就是基于消息拉取的方式来实现的，只不过时效性更好而已。

Push模式的实现思路：当消费者发送请求到Broker去拉取消息时，如果有新的消息可以消费就马上返回一批消息到消费机器去处理，处理完之后会接着立刻发送请求到Broker机器去拉取下一批消息。

所以，消费机器在Push模式下，会处理完一批消息，马上发起请求拉取下一批消息，消息处理的时效性非常好，看起来就像Broker一直不停的推送消息到消费机器一样。

此外，Push模式下有一个请求挂起和长轮询的机制：当拉取消息的请求发送到Broker，结果发现没有新的消息给处理时，就会让请求线程挂起，默认是挂起15秒。然后在这个期间，Broker会有一个后台线程，每隔5秒就去检查一下是否有新的消息。另外在这个挂起过程中，如果有新的消息到达了会主动唤醒挂起的线程，然后把消息返回给消费者。

<br>

**21.Consumer的处理队列与并发消费**

**(1)PullMessageService线程和ProcessQueue**

**(2)ConsumeMessageThread消息消费线程**

**(3)消费者并发消费总结**

<br>

**(1)PullMessageService线程和ProcessQueue**

Consumer中负责拉取消息的线程只有一个，就是PullMessageService线程。Consumer从Broker拉取到消息后，会有一个ProcessQueue处理队列，用于进行消息中转。

Consumer的PullMessageService线程拉取到消息后，会将消息写入一个叫ProcessQueue的内存数据结构中，这个ProcessQueue数据结构的作用其实是用来对消息进行中转用的。

由于Consumer负责消费的会是Broker中的某几个ConsumeQueue里的消息，所以Consumer拉取到的ConsumeQueue数据都会写到其内存的某几个ProcessQueue里面。也就是Consumer从Broker中拉取了几个ConsumeQueue的数据，就会对应有几个ProcessQueue，可以理解ProcessQueue和ConsumeQueue之间存在一一对应的映射关系。

<br>

**(2)ConsumeMessageThread消息消费线程**

Consumer把拉取到的消息写入ProcessQueue完成中转后，就会提交消费任务到一个线程池里。通过这个线程池，就可以开辟多个ConsumeMessageThread线程(即消息消费线程)，来对消息进行并发消费。线程池里的每个线程处理消费消息完毕后，就会回调用户自己写代码实现的回调监听处理函数，处理具体业务。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/0d003eba-fe27-4462-a672-5a47a431bdd6" />

<br>

**(3)消费者并发消费总结**

说明一：Consumer在启动时会往Broker注册，会通过RebalanceService组件获取Topic路由信息和ConsumerGroup信息。然后RebalanceService组件会通过负载均衡算法实现Queue到Consumer的分配，确定自己要拉取哪些Queue。

说明二：Consumer在拉取Queue的消息时会有长轮询和短轮询两种模式，默认采用短轮询拉取消息。

说明三：当Consumer拉取到消息后，就会写入在内存中和ConsumeQueue一一对应的ProcessQueue队列，并提交任务到线程池，由线程池里的线程并发地从ProcessQueue获取消息进行处理。

说明四：这些线程从ProcessQueue获取到消息后，就会回调用户实现的回调监听处理函数listener.consumeMessage()。当回调监听处理函数执行完毕后，便会返回SUCCESS给线程，线程便会删除ProcessQueue里的该消息，这样线程又可以继续从ProcessQueue里获取下一条消息进行处理。

<br>

**22.Consumer处理成功后的消费进度管理**

**(1)消息从ProcessQueue中删除后要提交消费进度**

**(2)消费进度先存本地内存再异步提交**

<br>

**(1)消息被处理完后要提交消费进度**

当线程池里的线程从ProcessQueue获取到某消息，并回调用户实现的回调监听处理函数listener.consumeMessage()，然后执行成功返回线程SUCCESS后，就可以将该消息从ProcessQueue中删掉了。

当消息从ProcessQueue中删掉后，Consumer需要向Broker的Leader节点提交消息对应的ConsumeQueue的消费进度。因为Broker的Leader节点需要维护和管理：每个ConsumeQueue被各个ConsumeGroup消费的进度。

<br>

**(2)消费进度先存本地内存再异步提交**

当回调监听处理函数返回SUCCESS后，Consumer本地的内存里会存储该Consumer对ConsumeQueue的消费进度。然后Consumer端会有一个后台线程，异步提交这个消费进度到Broker的Leader节点。即Consumer会先将消费进度提交到自己的本地内存里，接着有一个后台线程异步提交消费进度到Leader节点。

Broker的Leader节点收到Consumer提交的消费进度后，也会先存放到自己的内存中。然后Broker也会有一个后台线程将消费进度异步刷入磁盘文件里。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/32e97804-f849-42a4-9e4e-7a15309324bc" />

<br>

**23.Consumer消息重复消费原理剖析**

**(1)消息被重复消费的消费端原因**

**(2)造成消费端重复消费消息的场景**

<br>

**(1)消息被重复消费的消费端原因**

由于一条消息被消费后，消费进度不管在Consumer端还是在Broker端，都会先进入内存。所以当消费进度还在内存时机器崩溃了或者系统重启，那么就会导致消息重复消费。

<br>

**(2)造成消费端重复消费消息的场景**

主要就是如下两个情景造成消费被重复消息：

情景一：Consumer端消费完消息后，消费进度还没进入内存或已经写入内存但还没提交给Broker，机器宕机或系统重启。

情景二：Broker端收到Consumer提交的消费进度还没写入内存或刚写入内存，还没刷入磁盘，机器宕机或系统重启。

对于Consumer消息重复消费的问题，在Consumer端需要实现一套严格的分布式锁和幂等性保障机制来进行处理。

<br>

**24.Consumer处理失败时的延迟消费机制**

**(1)处理失败返回RECONSUME\_LATER**

**(2)消息进入RETRY\_Topic并检查延迟时间**

**(3)消息到达延迟时间再次进入原Topic重新消费**

<br>

**(1)处理失败返回RECONSUME\_LATER**

Consumer端线程池里的线程从ProcessQueue获取到某消息后：如果在回调用户实现的回调监听处理函数listener.consumeMessage()时，消费失败返回了RECONSUME\_LATER。那么Consumer也会把ProcessQueue里的这条消息进行删除，然后返回一个处理消息失败的ACK给Broker。

<br>

**(2)消息进入RETRY\_Topic并检查延迟时间**

Broker收到这个处理消息失败的ACK后，会对该消息的Topic进行改写，改写成RETRY\_Topic\_%。该消息也就成为了延迟消息，接着将消息写入到RETRY\_Topic\_%对应的CommitLog和ConsumeQueue中。然后Broker端会有一个延迟消息的后台线程对改写Topic的ConsumeQueue进行检查，检查里面的消息是否达到延迟时间。其中延迟时间会有多个，并且可以进行配置。

<br>

**(3)消息到达延迟时间再次进入原Topic重新消费**

如果达到延迟时间，就会把该消息取出来再次进行改写Topic，改写为原来的Topic。这样该消息会被写入到原Topic对应的CommitLog对应的CommitLog和ConsumeQueue中，从而让该消息被Consumer在后续的消费中拉取到，进行重新消费。

<br>

**25.ConsumerGroup变动时的重平衡机制**

每当一个ConsumerGroup中少了一个Consumer(机器宕机或重启)、或者多了一个Consumer(新增机器)时，就需要重新分配Topic的那些Queue给Consumer，而这部分工作会由Consumer端的RebalanceService组件完成。

RebalanceService组件会每隔20秒去Broker拉取最新的Topic路由信息 + ConsumerGroup信息。

当某个Consumer宕机后，Broker是知道该宕机的Consumer对其负责的ConsumeQueue的消费进度的。所以在最多20秒后，其他Consumer就会进行重新的负载均衡，将宕机Consumer负责的ConsumeQueue分配好。

当ConsumerGroup新增一个Consumer时，由于新增的Consumer会往Broker进行注册，所以Broker能知道新增Consumer。新老Consumer都会每隔20秒拉取最新的Topic路由信息 + ConsumerGroup信息。这样新老Consumer都可以通过RebalanceService重平衡组件重新分配ConsumeQueue。
