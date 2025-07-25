# RocketMQ原理—4.消息读写的性能优化

**大纲**

**1.Producer基于队列的消息分发机制**

**2.Producer基于Hash的有序消息分发**

**3.Broker如何实现高并发消息数据写入**

**4.RocketMQ读写队列的运作原理分析**

**5.Consumer拉取消息的流程原理分析**

**6.ConsumeQueue的随机位置读取需求分析**

**7.ConsumeQueue的物理存储结构设计**

**8.ConsumeQueue如何实现高性能消息读取**

**9.CommitLog基于内存的高并发写入优化**

**10.Broker数据丢失场景以及解决方案**

**11.PageCache内存高并发读写问题分析**

**12.基于JVM OffHeap的内存读写分离机制**

**13.JVM OffHeap + PageCache的数据丢失问题**

**14.ConsumeQueue异步写入失败的恢复机制**

**15.Broker写入与读取流程性能优化总结**

<br>

**1.Producer基于队列的消息分发机制**

**(1)消息是如何发送到Broker去的**

**(2)消息发送失败会如何处理**

**(3)发送消息时有哪些高阶特性可以使用**

<br>

**(1)消息是如何发送到Broker去的**

Producer发送消息时需要指定一个Topic，需要知道Topic里有哪些Queue，以及这些Queue分别分布在哪些Broker上。因此，Producer发送消息到Broker的流程如下：

首先，Producer会从NameServer中拉取Topic的路由信息，然后缓存本地的Topic缓存中，并每隔30s进行刷新。

然后，Producer会对获取到的Topic的Queues进行负载均衡，选择其中的一个Queue并找出对应的Broker主节点。

最后，Producer才会与这个Broker的主节点进行通信并发送消息过去。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/528220a8-8518-405a-a533-afaebe1f81c3" />

<br>

**(2)消息发送失败会如何处理**

如果Producer往Broker的主节点写入消息失败，那么就会执行重试——重新选择一个Queue和Broker主节点。此外，故障的Broker在一段时间内也不会再次被选中。这就是重试机制 + 故障退避机制。

Broker故障的延迟感知机制：Broker故障后，虽然NameServer会通过心跳 + 定时任务可以感知并摘除掉它，但该故障的Broker信息无需实时通知Producer。通过Producer端的重试机制 + 故障退避机制，就可以让Broker故障时也能做到高可用。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/2389ea77-66b1-4069-a2ce-1aed0b4594d4" />

<br>

**(3)发送消息时有哪些高阶特性可以使用**

按照Key进行Hash，比如让orderId相同的消息都进入同一个Queue里，以保证它们的顺序性。

<br>

**2.Producer基于Hash的有序消息分发**

一个Topic会有很多个Queue，这些Queue会落到各个Broker上。默认情况下，Producer往一个Topic写入的数据，会均匀地分散到各个Broker的各个Queue里。所以各个Broker的各个Queue里都会有一些Producer的数据。

那么发送到Topic里的数据，是否是有顺序的？其实一个Queue就代表了一个队列，当消息进入到同一个Queue时，这些同一个Queue里的消息便是有顺序的，但是不同的Queue之间的消息则是没有顺序的。

如果需要让某一类数据有一定的特殊顺序性，比如同样orderId对应的多条消息(下单->支付->发券)是有顺序的，那么唯一的选择就是让同样orderId对应的所有消息都进入同一个Queue，并保证它们在同一个Queue里是有顺序的。

具体的做法就是：根据消息的某个字段值对应的Hash值，对Queue的数量进行取模，按取模结果来选择Queue。从而实现同样的字段值对应同样的Queue，由此来实现顺序性。

<br>

**3.Broker如何实现高并发消息数据写入**

**(1)数据持久化**

**(2)磁盘的随机写和顺序写**

**(3)内存的随机写和顺序写**

**(4)RocketMQ的消息写入**

写消息的方式有两种：一是随机写，二是顺序写。写消息的落地有两种：一是内存，二是磁盘。

**(1)数据持久化**

一般来说，如果要持久化保存写入的消息数据，消息必须要落地到磁盘。如果消息只落地到内存，为了避免内存里的数据丢失，此时就需要设计一套避免内存里数据不丢失的机制，而且这套机制一般都会基于Write Ahead Log(预写日志)，也是需要写磁盘的。所以写入RocketMQ的消息数据，都会写入到磁盘里来保证持久化存储。

<br>

**(2)磁盘的随机写和顺序写**

磁盘随机写：往磁盘文件里写的数据，其格式都是用户自定义的。用户每次写入数据都需要找到磁盘文件里的某个位置(在磁盘文件里随机寻址)，才能在那个位置里插入最新的数据。

磁盘顺序写：每次写入数据都是在一个文件末尾去进行追加，绝对不会随机在文件里寻址来找到一个中间的位置进行插入。

磁盘文件随机写的性能：几十ms到几百ms。

磁盘文件顺序写的性能：约等同于在内存里随机写数据，毫秒级，低于1ms或几ms。

<br>

**(3)内存的随机写和顺序写**

如果把数据往内存里写，也分顺序写和随机写。

内存随机写：内存也是有地址空间的，在内存里随机写需要在内存地址里随机寻址，然后再去插入数据。

内存顺序写：在一块连续的内存空间里，顺序地追加数据，避免了内存地址的随机寻址。

<br>

**(4)RocketMQ的消息写入**

首先会将所有消息数据都顺序写入到CommitLog磁盘文件。1个CommitLog的大小为1GB，写满一个CommitLog就切换下一个CommitLog。这时CommitLog便会包含很多种数据，所以需要区分一条消息到底是属于哪个Topic哪个Queue的。

然后RocketMQ会有一个后台线程负责对CommitLog里的数据进行转发。该后台线程会监听CommitLog里的最新数据写入，负责把消息写入到所属的Topic对应的Queue里面。这个Queue本身也是属于一个磁盘文件，而Queue里面的内容则是每条数据在CommitLog磁盘文件里的offset偏移量。

RocketMQ便是通过以上的方式来实现消息数据的高并发写入和存储。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/54648c1c-f780-48bb-a67a-afeb579aa798" />

<br>

**4.RocketMQ读写队列的运作原理分析**

多个Consumer会组成一个消费者组来消费同一个Topic，该Topic里的ReadQueue会均匀分配给同一个消费者组里的各个Consumer。

创建一个Topic时会默认4个WriteQueue和4个ReadQueue。其中WriteQueue会和磁盘里的文件对应起来，属于物理概念，而ReadQueue则属于虚拟概念。通常来说WriteQueue和ReadQueue会一一对应。

WriteQueue用于Producer写消息时进行路由，ReadQueue用于Consumer读消息时进行路由。

WriteQueue其实只是对于Producer而言的，实际上只是逻辑上的一个名字而已。在物理上，WriteQueue对应的是ConsumeQueue。

RocketMQ设计出WriteQueue和ReadQueue的原因是为了保持灵活性，方便扩容和缩容。

如果创建Topic时配置了4个WriteQueue、8个ReadQueue：由于WriteQueue和ReadQueue是一一对应的，当这8个ReadQueue均匀下发到4个Consumer时，可能会导致某Consumer分到的ReadQueue是完全没有数据的。

如果创建Topic时配置了8个WriteQueue、4个ReadQueue：由于WriteQueue和ReadQueue是一一对应的，可能会导致只有4个WriteQueue有对应的ReadQueue会进行下发到Consumer，剩余4个WriteQueue的数据没法给到Consumer进行消费。

<br>

**5.Consumer拉取消息的流程原理分析**

基于RocketMQ编写Consumer代码时，用户一般会定义一个ConsumerListener回调监听函数，并在函数里添加处理消息的具体逻辑。这样，当Consumer拉取到消息后，会调用用户定义的ConsumerListener回调监听函数，从而实现对消息的处理。

当Consumer分配到Topic的某些ReadQueue之后，会有一个专门的线程处理ReadQueue里的消息。这个线程叫做PullMessageService，它会和对应的Broker建立好网络连接，然后不停地循环拉取ReadQueue里的消息。Consumer和某Broker建立好连接后，便可以感知分配给该Consumer的ReadQueue所映射的WriteQueue里最新的数据。Broker会根据最新数据的offset偏移量，从CommitLog中获取完整的消息数据，最后通过网络发送给Consumer。

Consumer的PullMessageService线程拉取到消息之后，会将消息写入一个叫ProcessQueue的内存数据结构中，这个ProcessQueue数据结构的作用其实是用来对消息进行中转用的。完成中转后，Consumer便能够开辟多个ConsumeMessageThread线程(即消息消费线程)，来对消息进行消费。ConsumeMessageThread线程从ProcessQueue读取到消息后，便会调用用户实现的consumeMessage()方法。这个consumeMessage()方法就是用户自定义ConsumerListener回调监听函数里的方法，里面便是用户对消息的业务逻辑处理。

如果执行用户的业务逻辑处理成功了，便会返回SUCCESS标记给ConsumeMessageThread线程。如果执行用户的业务逻辑处理失败了，便会返回RECONSUME\_LATER标记给ConsumeMessageThread线程。

当ConsumeMessageThread线程收到执行用户的业务逻辑处理的SUCCESS标记后，Consumer便会上报某消息处理成功到Broker的WriteQueue，这样下次该消息就不会重复分发给Consumer去消费了。

当ConsumeMessageThread线程收到执行用户的业务逻辑处理的RECONSUME\_LATER标记后，Consumer便会上报某消息处理失败到Broker的WriteQueue，这样下次该消息就会继续分发给Consumer去消费。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/31c6f409-b7cc-4532-bea3-87afcba99d87" />

<br>

**6.ConsumeQueue的随机位置读取需求分析**

ConsumeQueue应该如何设计才能让Consumer高性能地读取一条一条的消息？

首先，一个Topic是可以给多个ConsumerGroup去进行消费的。比如对于一个有8个Queue的TestTopic，业务系统A和业务系统B都可以订阅TestTopic分别进行各自消费。其中业务系统A部署了3台机器，每台机器就是一个Consumer，那么这8个Queue会分配给这3台机器去消费。而业务系统B部署了5台机器，每台机器就是一个Consumer，那么这8个Queue会分配给这5台机器去消费。

然后，不同的ConsumerGroup对一个Queue的消费进度是不一样的。有的ConsumerGroup可能已经消费这个Queue的500条消息，有的ConsumerGroup可能只消费这个Queue的100条消息。

所以Queue需要能够随时根据要消费的消息序号，定位到某条消息在磁盘文件里的不同位置，然后从该位置去进行读取。

针对这样的读取需求，ConsumeQueue磁盘文件应该如何设计才能支持高效的磁盘位置定位以及读取？

<br>

**7.ConsumeQueue的物理存储结构设计**

首先会有一个目录来存放ConsumeQueue磁盘文件，比如\~/topicName/queueId/多个磁盘文件，可见每个Topic都会有属于自己的目录。

然后每个ConsumeQueue磁盘文件里都会存储一条一条的消息数据，每条消息数据在磁盘文件里存储的内容如下：

    8个字节的offset + 4个字节的消息大小 + 8个字节的Tag的Hash值

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/50f659a5-5596-4149-be61-82858d7daa03" />

按上述方式设计ConsumeQueue消息内容的好处是定长，可以让ConsumeQueue的每条消息在ConsumeQueue磁盘文件里存储的大小是固定的20字节。这样每个ConsumeQueue磁盘文件最多30万条数据，只占5.72MB大小。当消息定长和磁盘文件固定大小后，就可以方便快速地根据逻辑偏移量在ConsumeQueue里定位和读取某条消息，从而快速地获取物理偏移量，然后再从CommitLog里定位和读取到具体的消息。

<br>

**8.ConsumeQueue如何实现高性能消息读取**

**(1)从ConsumeQueue高效读取消息**

**(2)从CommitLog高效读取消息**

**(3)两次随机磁盘读**

<br>

**(1)从ConsumeQueue高效读取消息**

ConsumerGroup读取的消息，都会有一个自己的逻辑offset。逻辑offset可以认为是逻辑上的偏移量，指的是Queue里的第几条消息。ConsumerGroup里的一个Consumer会负责读取某个Queue里的消息。当Consumer从Queue里读取消息时，它会知道需要读取的是这个Queue在逻辑上的某个offset，也就是第几条消息。

因为消息是定长的，而且ConsumeQueue磁盘文件也是固定大小的，以及会最多存放30万条消息。所以当ConsumerGroup的一个Consumer需要读取ConsumeQueue里的某逻辑偏移量offset位置的消息时，就会根据该消息的逻辑偏移量offset，去定位到ConsumeQueue磁盘文件中的位置((offset - 1) \* 20字节)作为起始位置，然后再连续读取20字节即可，这样就可以快速地将消息读取出来。

这就是ConsumeQueue的高效率、高性能的读取方式，无需遍历读取磁盘文件里一条一条的消息来进行查找，类似于根据索引去数组定位元素。

<br>

**(2)从CommitLog高效读取消息**

一个CommitLog的大小就是1G。CommitLog的文件名就是这个文件里的第一条消息在整个CommitLog所有文件组成的文件组里的一个总的物理偏移量。也就是Broker会将CommitLog文件里第一条消息的这个总物理偏移量作为该CommitLog的文件名。比如：

    00000000000000000000、00000000000004545345

当Broker从ConsumeQueue里将消息在CommitLog的偏移量offset + 大小size读取出来后，就可以在所有的CommitLog文件里根据偏移量offset进行二分查找，便能确定该消息是存在于哪个CommitLog文件里。然后用偏移量offset对CommitLog文件名进行减法运算，便能知道消息在该CommitLog文件里的起始位置。最后从该CommitLog文件的对应起始位置开始读取size大小的内容，便能获取到该消息。

<br>

**(3)两次随机磁盘读**

Broker读取一条消息时只需要两次随机磁盘读。

第一次的随机磁盘读是针对ConsumeQueue文件，根据逻辑偏移量计算出具体位置后，再进行随机定位的读取。

第二次的随机磁盘读是针对CommitLog文件，根据读出的物理偏移量利用二分查找定位具体位置，再进行随机定位的读取。

<br>

**9.CommitLog基于内存的高并发写入优化**

**(1)Broker写入性能优化**

**(2)Broker读取性能优化**

**(3)基于内存来提升读取和写入的性能**

**(4)Broker写入性能优化的总结**

<br>

**(1)Broker写入性能优化**

一.往CommitLog写入消息时会基于磁盘顺序写来提升写入的性能

二.往ConsumeQueue写入消息时会基于异步转发写入机制来提升写入的性能

<br>

**(2)Broker读取性能优化**

一.从ConsumeQueue中读取消息时会基于定长消息 + 定长文件实现消息的一次定位和读取

二.从CommitLog中读取消息时会基于文件名(第一条消息的总物理偏移量)+消息偏移量实现消息的一次定位和读取

<br>

**(3)基于内存来提升读取和写入的性能**

无论是写入还是读取，此时最大的问题就是需要对物理磁盘文件进行顺序写或随机读。所以优化的方向就是：基于内存来进一步提升Broker的整体写入性能和读取性能。

实际上，RocketMQ会基于内存来提升CommitLog的写入性能。虽然磁盘顺序写已经比磁盘随机写好很多，但也比内存写差。

具体上，Broker会基于MappedFile这种mapping文件内存映射机制，实现把消息数据先写入到内存、再从内存刷到磁盘。MappedFile可以把磁盘文件映射成一个内存空间，这里的内存是操作系统的PageCache。

所以当Broker往CommitLog写入消息时，会先写入到CommitLog磁盘文件映射的操作系统的PageCache中，而PageCache里的消息会在后续通过异步刷盘写入到CommitLog磁盘文件里。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/aa468777-7c63-47c4-a7ca-f30039ff997b" />

<br>

**(4)Broker写入性能优化的总结**

一.往CommitLog写入消息时，会通过顺序写内存 + 异步顺序刷磁盘来提升性能

二.往ConsumeQueue写入消息时，会基于异步转发的写入机制来提升性能

<br>

**10.Broker数据丢失场景以及解决方案**

**(1)Broker数据丢失场景分析**

**(2)解决方案**

<br>

**(1)Broker数据丢失场景分析**

第一种情况：由于RocketMQ是用Java开发的中间件系统，Broker启动后就是一个JVM进程。所以如果Broker这个JVM进程突然崩溃了，那么此时仅仅是JVM进程没了，其写到PageCache里的数据由于是OS管理的，因此数据不会丢失。这种情况发生的概率还是比较高的。

第二种情况：Broker这个JVM进程所在的服务器故障宕机了，此时就可能导致Broker写入到PageCache里的数据丢失。这种情况发生的概率还是非常低的。

<br>

**(2)解决方案**

将异步刷盘改成同步刷盘：Broker将消息顺序写入PageCache后，就等操作系统将PageCache的数据同步到磁盘文件后再返回响应给Producer。

<br>

**11.PageCache内存高并发读写问题分析**

如果可以忽略在小概率下的一点点数据丢失问题，将顺序写磁盘文件换成顺序写内存，那么就可以明显提升性能和吞吐量。而且即便是丢失数据，也是默认丢失500ms内的数据。

当Producer和Consumer都在高并发地往Broker写消息和读消息时：PageCache的内存数据可能会出现一个经典的问题，就是RocketMQ里的Broker Busy异常，也就是Broker过于繁忙，这会导致一些操作阻塞甚至失败。

Broker Busy就是在高并发的读写情况下，出现的竞争同一块PageCache数据太频繁太激烈的问题。为了解决这个问题，RocketMQ提供了一个叫TransientStorePoolEnabled的机制。

<br>

**12.基于JVM OffHeap的内存读写分离机制**

TransientStorePoolEnabled机制，可以理解为瞬时存储池启用机制。如果对Broker的读写压力真的大到出现Broker Busy异常，那么通过开启瞬时存储池，就可以实现内存级别的读写分离模式。

一般而言，在一个服务器上部署一个Java系统后，这个系统会作为一个JVM进程运行在操作系统上。其使用的内存会分为三种：第一种是JVM Heap内存(即JVM管理的堆内存)，第二种是OffHeap内存(即JVM堆外的内存)，第三种是PageCache内存(即由操作系统管理的页缓存)。

所以如果开启了TransientStorePoolEnabled，那么Broker在写消息时，就会先把消息写到JVM OffHeap堆外内存里。然后会有一个额外的后台线程每隔一段时间定时把JVM OffHeap堆外内存里的数据写入到PageCache中。这样高并发写消息便往JVM OffHeap堆外内存里写，高并发读消息便从操作系统的PageCache中读，从而实现内存级别的读写分离。

所以，RocketMQ为了解决高并发场景下对PageCache竞争读写导致的Broker Busy问题，引入了JVM OffHeap做了缓存分离，实现了内存级别的读写分离，解决了对一块内存空间的写和读出现频繁竞争的问题。

<br>

**13.JVM OffHeap + PageCache的数据丢失问题**

系统设计里，凡事皆有利弊，没有什么方案是十全十美的。为了解决一个问题，往往会引入新的问题。

RocketMQ为了解决高并发场景下对PageCache竞争读写导致的Broker Busy问题，引入了JVM OffHeap做了缓存分离，实现了内存级别的读写分离，解决了对一块内存空间的写和读出现频繁竞争的问题。但这会大大提高数据丢失的风险，数据丢失的情况主要有两种。

情况一：Broker JVM进程关闭。比如Broker崩溃宕机、JVM进程意外退出，或者正常关闭Broker JVM进程进行重启等。由于OffHeap堆外内存是由JVM管理的，所以OffHeap堆外内存的数据此时会丢失，而OS管理的PageCache则不会丢失。

情况二：Broker所在的服务器宕机。此时OffHeap堆外内存和PageCache里的数据都会丢失。

因此没有一个技术方案是完美的，只能抓住当前场景里的主要矛盾。如果是金融级的数据绝对不能丢失，可能就要牺牲性能和吞吐量，让数据的每一次写入都直接刷盘到磁盘文件。如果是大部分的普通情况，数据允许丢一点点，也就在服务器宕机的极端场景下才会丢几百毫秒的数据，保持默认即可。

RocketMQ默认就是只写PageCache + 异步刷盘，如果出现高并发竞争PageCache的问题，那么可以开启写JVM OffHeap。容忍一定的JVM崩溃也丢失一点数据，但实现了利用缓存分离抗高并发读写。

<br>

**14.ConsumeQueue异步写入失败的恢复机制**

**(1)ConsumeQueue异步写入消息的两个线程**

**(2)ConsumeQueue异步写入失败有两种情况**

<br>

**(1)ConsumeQueue异步写入消息的两个线程**

后台线程一：将写入到PageCache的数据异步刷盘到CommitLog磁盘文件里

后台线程二：监听CommitLog磁盘文件的最新数据写入到ConsumeQueue磁盘文件里

<br>

**(2)ConsumeQueue异步写入失败有两种情况**

情况一：消息进入到PageCache时，Broker对应的JVM进程就宕机了，对应上述的两个后台线程便停止工作。此时只要Broker对应的JVM进程重启后，便会继续让上述两个线程处理PageCache里的消息。

情况二：消息进入到CommitLog磁盘文件时，Broker对应的JVM进程就宕机了，对应上述的两个后台线程便停止工作。此时只要Broker对应的JVM进程重启后，负责监听CommitLog磁盘文件的后台线程也会继续处理里面的新增消息。而且Broker重启时也会对比CommitLog的最新数据和ConsumeQueue的最新数据，保证被中断的消息写入能正常恢复。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/aa321ab3-f581-4d81-8797-63400a78412d" />

<br>

**15.Broker写入与读取流程性能优化总结**

**(1)写入流程的优化**

**(2)存储结构的优化**

**(3)读取流程的优化**

<br>

**(1)写入流程的优化**

**一.默认先写入OS的PageCache后就直接返回成功了，优化成内存级的顺序写**

这里会基于MappedFile机制来实现，将磁盘文件映射为一块OS的PageCache内存，让写文件等同于写内存。其中的亮点是基于OS的PageCache来写入数据，Broker的JVM进程崩溃(高概率)是不会导致PageCache的数据丢失的。只有服务器崩溃的小概率极端场景才会导致几百毫秒内写入的数据会丢失，所以丢数据的概率是很低的。

<br>

**二.对ConsumeQueue文件和IndexFile文件的写入，是通过异步来进行写入的**

虽然将消息写入这两种文件时是异步写的，但只要数据还在CommitLog中没有丢失，那么即便异步写入失败也没影响。

<br>

**(2)存储结构的优化**

**一.ConsumeQueue文件的存储结构是为了能够实现高性能的读取而设计的**

在ConsumeQueue文件里存储的每条消息都是定长的20字节，每个ConsumeQueue文件满了是30万条消息，约5.72MB。此外，一个Topic目录会有多个MessageQueue目录，一个MessageQueue目录会有多个ConsumeQueue磁盘文件。

<br>

**二.CommitLog文件默认满了就是1GB**

CommitLog的物理存储结构核心就是其文件名，每一条消息都会有一个在所有CommitLog里的总的物理偏移量，每个文件的名称就是文件里第一条消息在所有CommitLog里的总物理偏移量。

<br>

**(3)读取流程的优化**

**一.根据消息的逻辑偏移量offset来定位哪个磁盘文件的哪个物理位置**

通过第一次定位，就能找到这条消息在CommitLog里的物理偏移量offset。然后再通过第二次定位，也就是根据物理偏移量利用二分查找去对应的CommitLog中便能读取出该消息。

<br>

**二.高并发对PageCache进行读写竞争时可能会出现Broker Busy问题**

此时可以通过开启TransientStorePoolEnabled，也就是启用JVM OffHeap堆外内存，来实现内存级的读写分离。
