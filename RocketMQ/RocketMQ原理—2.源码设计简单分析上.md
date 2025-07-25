# RocketMQ原理—2.源码设计简单分析上

**大纲**

**1.NameServer的启动脚本**

**2.NameServer启动时会解析哪些配置**

**3.NameServer如何初始化Netty网络服务器**

**4.NameServer如何启动Netty网络服务器**

**5.Broker启动时是如何初始化配置的**

**6.BrokerController的创建以及包含的组件**

**7.BrokerController的初始化**

**8.BrokerController的启动**

**9.Broker如何把自己注册到NameServer上**

**10.BrokerOuterAPI是如何发送注册请求的**

**11.NameServer如何处理Broker的注册请求**

**12.Broker如何发送定时心跳的以及故障感知**

<br>

**1.NameServer的启动脚本**

NameServer会通过rocketmq-master源码中distribution/bin目录下的mqnamesrv脚本来启动。在mqnamesrv脚本中，用于启动NameServer进程的命令如下。也就是使用sh命令执行runserver.sh脚本，然后通过这个脚本去启动NamesrvStartup这个Java类。

    sh ${ROCKETMQ_HOME}/bin/runserver.sh org.apache.rocketmq.namesrv.NamesrvStartup $@ 

在runserver.sh脚本中，启动NamesrvStartup这个Java类的命令如下：

    JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
    JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8  -XX:-UseParNewGC"
    JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:${GC_LOG_DIR}/rmq_srv_gc_%p_%t.log -XX:+PrintGCDetails"
    JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
    JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
    JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages"
    JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
    #JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
    JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
    JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

    $JAVA ${JAVA_OPT} $@

可以看到，上述命令大致简化一下就是类似如下这样的一行命令：

    java -server -Xms4g -Xmx4g -Xmn2g org.apache.rocketmq.namesrv.NamesrvStartup 

通过Java命令 + 一个有main()方法的NamesrvStartup类，就会启动一个JVM进程。这个JVM进程会执行NamesrvStartup类中的main()方法，从而完成启动NameServer需要的所有工作。

所以，启动NameServer的过程如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/518349f8-1ca7-478c-a540-0286458ec0ed" />

<br>

**2.NameServer启动时会解析哪些配置**

**(1)什么是NamesrvController**

**(2)NamesrvController是如何被创建出来的**

**(3)NameServer的两个配置类**

**(4)NameServer两个配置类的解析**

**(5)完成NamesrvController组件的创建**

<br>

**(1)什么是NamesrvController**

当执行NamesrvStartup的main()方法时，会执行NamesrvStartup的main0()方法：

    //下面这个NamesrvStartup类，就是最为关键的NameServer进程的启动类
    public class NamesrvStartup {
        ...
        //NameServer进程启动时，会执行NamesrvStartup类的main()方法
        public static void main(String[] args) {
            main0(args);
        }

        public static NamesrvController main0(String[] args) {
            //NamesrvController是NameServer的核心代码组件
            //NameServer一启动，就会去启动这个NamesrvController
            try {
                NamesrvController controller = createNamesrvController(args);
                start(controller);
                String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
                log.info(tip);
                System.out.printf("%s%n", tip);
                return controller;
            } catch (Throwable e) {
                e.printStackTrace();
                System.exit(-1);
            }

            return null;
        }
        ...
    }

在上述源码中，有这么一行代码：

    NamesrvController controller = createNamesrvController(args);

这行代码就是在创建一个NamesrvController类，这个类是NameServer中的一个核心组件。由于NameServer需要接收Broker发送过来的要把自己注册到NameServer上的请求(因为知道有哪些Broker和管理Broker)，以及需要接收客户端发送过来的从NameServer拉取元数据的请求(因为客户端需要知道一个Topic的MessageQueue都在哪些Broker上)，所以才需要在NameServer中创建NamesrvController这个类专门用来接收Broker和客户端的网络请求。如下图示，NamesrvController组件是NameServer中的核心组件，用来负责接受网络请求的。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/31355e30-18d6-4ced-84ec-00520995c0d9" />

<br>

**(2)NamesrvController是如何被创建出来的**

NamesrvStartup的createNamesrvController()方法会创建出NamesrvController这个关键组件。

    public class NamesrvStartup {
        ...
        public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
            System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
            //PackageConflictDetect.detectFastjson();

            Options options = ServerUtil.buildCommandlineOptions(new Options());
            commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());
            if (null == commandLine) {
                System.exit(-1);
                return null;
            }

            final NamesrvConfig namesrvConfig = new NamesrvConfig();
            final NettyServerConfig nettyServerConfig = new NettyServerConfig();
            nettyServerConfig.setListenPort(9876);
            if (commandLine.hasOption('c')) {
                String file = commandLine.getOptionValue('c');
                if (file != null) {
                    InputStream in = new BufferedInputStream(new FileInputStream(file));
                    properties = new Properties();
                    properties.load(in);
                    MixAll.properties2Object(properties, namesrvConfig);
                    MixAll.properties2Object(properties, nettyServerConfig);

                    namesrvConfig.setConfigStorePath(file);

                    System.out.printf("load config properties file OK, %s%n", file);
                    in.close();
                }
            }

            if (commandLine.hasOption('p')) {
                InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
                MixAll.printObjectProperties(console, namesrvConfig);
                MixAll.printObjectProperties(console, nettyServerConfig);
                System.exit(0);
            }

            MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);

            if (null == namesrvConfig.getRocketmqHome()) {
                System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
                System.exit(-2);
            }

            LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
            JoranConfigurator configurator = new JoranConfigurator();
            configurator.setContext(lc);
            lc.reset();
            configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

            log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

            MixAll.printObjectProperties(log, namesrvConfig);
            MixAll.printObjectProperties(log, nettyServerConfig);

            final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

            //remember all configs to prevent discard
            controller.getConfiguration().registerConfig(properties);

            return controller;
        }
        ...
    }

<br>

**(3)NameServer的两个配置类**

    public class NamesrvStartup {
        ...
        public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
            ...
            final NamesrvConfig namesrvConfig = new NamesrvConfig();
            final NettyServerConfig nettyServerConfig = new NettyServerConfig();
            nettyServerConfig.setListenPort(9876);
            ...
        }
        ...
    }

在createNamesrvController()方法中，创建了NamesrvConfig和NettyServerConfig两个配置类。其中，NamesrvConfig配置了NameServer自身运行的一些参数，NettyServerConfig配置了用于接收网络请求的Netty服务器的一些参数。

在这里也能知道NameServer对外接收Broker和客户端的网络请求时，是基于Netty来实现网络服务器的，而且可知NameServer默认的监听请求端口号就是9876。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/4feef18a-f2a4-4a2a-80a8-f16345adebdc" />

NameServer的两个配置类如下：

    public class NamesrvConfig {
        private static final InternalLogger log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);
        //RocketMQ的home主目录地址，它其实就是尝试去获取ROCKETMQ_HOME这个环境变量的值
        private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
        //NameServer存放kv配置属性的路径
        private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
        //NameServer自己的配置存储路径
        private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
        //生产环境的名称，默认为center
        private String productEnvName = "center";
        //是否启动了clusterTest测试集群，默认是false
        private boolean clusterTest = false;
        //是否支持有序消息，默认是false，不支持
        private boolean orderMessageEnable = false;
        ...
    }

<!---->

    public class NettyServerConfig implements Cloneable {
        //NettyServer默认的监听端口号是8888，但在NamesrvController中这个端口号会被设置为9876
        private int listenPort = 8888;
        //NettyServer的工作线程数量，默认是8
        private int serverWorkerThreads = 8;
        //Netty的public线程池的线程数量，默认是0
        private int serverCallbackExecutorThreads = 0;
        //Netty的IO线程池的线程数量，默认是3，这里的线程是负责解析网络请求的
        //这里的线程解析完网络请求后，就会把请求转发给work线程来处理
        private int serverSelectorThreads = 3;
        
        //下面两个是Broker端的参数：就是Broker端在基于Netty构建网络服务器时，会使用下面两个参数
        private int serverOnewaySemaphoreValue = 256;
        private int serverAsyncSemaphoreValue = 64;
        
        //如果一个网络连接空闲超时120s，就会被关闭
        private int serverChannelMaxIdleTimeSeconds = 120;
        
        //socket send buffer缓冲区以及receive buffer缓冲区的大小
        private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize;
        private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize;
        
        private int writeBufferHighWaterMark = NettySystemConfig.writeBufferHighWaterMark;
        private int writeBufferLowWaterMark = NettySystemConfig.writeBufferLowWaterMark;
        private int serverSocketBacklog = NettySystemConfig.socketBacklog;
        
        //ByteBuffer是否开启缓存，默认是开启的
        private boolean serverPooledByteBufAllocatorEnable = true;
        
        //是否启动epoll IO模型，默认是不开启的
        private boolean useEpollNativeSelector = false;
        ...
    }

<br>

**(4)NameServer两个配置类的解析**

     public class NamesrvStartup {
        ...
        public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
            ...
            final NamesrvConfig namesrvConfig = new NamesrvConfig();
            final NettyServerConfig nettyServerConfig = new NettyServerConfig();
            nettyServerConfig.setListenPort(9876);
            //下面这段代码的意思就是，如果用mqnamesrv启动时，带上了"-c"这个选项
            //那么"-c"这个选项的意思就是带上一个配置文件的地址，接着它就可以读取配置文件里的内容
            if (commandLine.hasOption('c')) {
                String file = commandLine.getOptionValue('c');
                if (file != null) {
                    //基于输入流从配置文件里读取配置，读取的配置会放入一个Properties里
                    InputStream in = new BufferedInputStream(new FileInputStream(file));
                    properties = new Properties();
                    properties.load(in);

                    //然后可以基于工具类，把读取到的配置都放入到两个核心配置类里
                    MixAll.properties2Object(properties, namesrvConfig);
                    MixAll.properties2Object(properties, nettyServerConfig);

                    namesrvConfig.setConfigStorePath(file);

                    System.out.printf("load config properties file OK, %s%n", file);
                    in.close();
                }
            }
            ...
        }
        ...
    }

上述代码意思是：在启动NameServer时，如果使用了-c选项带上一个配置文件的地址，那么就会把配置文件里的配置放入这两个配置类中。比如有一个配置文件是nameserver.properties，里面有一个配置是serverWorkerThreads=16，那么就会读取出这个配置，然后覆盖到NettyServerConfig中。

接着看配置相关的剩余代码；

    public class NamesrvStartup {
        ...
        public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
            ...
            //下面这段代码的意思是，如果用mqnamesrv启动时，带上了"-p"这个选项
            //那么它的意思就是print，会打印处出NameServer的所有配置信息
            if (commandLine.hasOption('p')) {
                InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_CONSOLE_NAME);
                MixAll.printObjectProperties(console, namesrvConfig);
                MixAll.printObjectProperties(console, nettyServerConfig);
                System.exit(0);
            }

            //下面一行代码会把mqnamesrv命令行中带上的配置选项，都读取出来，然后覆盖到NamesrvConfig里
            MixAll.properties2Object(ServerUtil.commandLine2Properties(commandLine), namesrvConfig);

            //如果发现ROCKETMQ_HOME是空的，那么就会输出一个异常日志，提示设置ROCKETMQ_HOME这个环境变量
            if (null == namesrvConfig.getRocketmqHome()) {
                System.out.printf("Please set the %s variable in your environment to match the location of the RocketMQ installation%n", MixAll.ROCKETMQ_HOME_ENV);
                System.exit(-2);
            }

            //下面5行代码也是日志、配置相关的，与Logger、Configurator相关
            LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
            JoranConfigurator configurator = new JoranConfigurator();
            configurator.setContext(lc);
            lc.reset();
            configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

            //这里会打印一下NameServer的所有配置信息
            log = InternalLoggerFactory.getLogger(LoggerName.NAMESRV_LOGGER_NAME);

            MixAll.printObjectProperties(log, namesrvConfig);
            MixAll.printObjectProperties(log, nettyServerConfig);
            ...
        }
        ...
    }

所以，NameServer启动时，刚开始就是在初始化和解析NameServerConfig、NettyServerConfig相关的配置信息。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/5ea6cef7-46de-4523-a9ad-278719363ef3" />

由于NameServer刚启动会初始化和解析一些核心配置信息，尤其是NettyServer的一些网络配置信息，所以初始化配置信息后，会打印如下启动日志：

    2020-02-05 15:10:05 INFO main - rocketmqHome=rocketmq-nameserver
    2020-02-05 15:10:05 INFO main - kvConfigPath=namesrv/kvConfig.json
    2020-02-05 15:10:05 INFO main - configStorePath=namesrv/namesrv.properties
    2020-02-05 15:10:05 INFO main - productEnvName=center
    2020-02-05 15:10:05 INFO main - clusterTest=false
    2020-02-05 15:10:05 INFO main - orderMessageEnable=false
    2020-02-05 15:10:05 INFO main - listenPort=9876
    2020-02-05 15:10:05 INFO main - serverWorkerThreads=8
    2020-02-05 15:10:05 INFO main - serverCallbackExecutorThreads=0
    2020-02-05 15:10:05 INFO main - serverSelectorThreads=3
    2020-02-05 15:10:05 INFO main - serverOnewaySemaphoreValue=256
    2020-02-05 15:10:05 INFO main - serverAsyncSemaphoreValue=64
    2020-02-05 15:10:05 INFO main - serverChannelMaxIdleTimeSeconds=120
    2020-02-05 15:10:05 INFO main - serverSocketSndBufSize=65535
    2020-02-05 15:10:05 INFO main - serverSocketRcvBufSize=65535
    2020-02-05 15:10:05 INFO main - serverPooledByteBufAllocatorEnable=true
    2020-02-05 15:10:05 INFO main - useEpollNativeSelector=false

<br>

**(5)完成NamesrvController组件的创建**

    public class NamesrvStartup {
        ...
        public static NamesrvController createNamesrvController(String[] args) throws IOException, JoranException {
            ...
            final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);
          
            //remember all configs to prevent discard
            controller.getConfiguration().registerConfig(properties);
            ...
        }
        ...
    }

这里直接创建了NamesrvController组件，同时传递NamesrvConfig和NettyServerConfig这两个配置类给它。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/ce4630a8-204e-423d-a9dc-dc238e600c77" />

<br>

**3.NameServer如何初始化Netty网络服务器**

**(1)NamesrvController的创建总结**

**(2)NamesrvController的构造函数**

**(3)NamesrvController启动时的工作**

**(4)创建NettyRemotingServer网络服务器组件**

**(5)NettyRemotingServer的初始化**

<br>

**(1)NamesrvController的创建总结**

NameServer架构图如下：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/cd5a9d32-bbcd-4465-8e44-231c68e405db" />

由前面分析可知：NameServer启动时首先会解析配置文件，然后初始化NamesrvConfig和NettyServerConfig两个配置类，接着基于这两个配置类构建出NamesrvController组件。

而且根据类名可以推测出，NamesrvController内部肯定会包含基于Netty实现的网络通信组件。所以上面NameServer架构图中的Netty网络服务器，会负责监听和处理Broker与客户端发来的网络请求。

<br>

**(2)NamesrvController的构造函数**

NamesrvController被创建出来后，需要启动Netty网络服务器，这样NameServer才能在默认的9876端口上接收Broker和客户端的网络请求，比如Broker注册自己、客户端拉取Broker路由数据等。

在NamesrvController的构造函数中，其实就是保存一些实例变量的值而已，并没做什么实质性事情。

    public class NamesrvController {
        ...
        public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
            this.namesrvConfig = namesrvConfig;
            this.nettyServerConfig = nettyServerConfig;
            this.kvConfigManager = new KVConfigManager(this);
            this.routeInfoManager = new RouteInfoManager();
            this.brokerHousekeepingService = new BrokerHousekeepingService(this);
            this.configuration = new Configuration(log, this.namesrvConfig, this.nettyServerConfig);
            this.configuration.setStorePathFromConfig(this.namesrvConfig, "configStorePath");
        }
        ...
    }

所以仅仅创建出一个NamesrvController实例还是不够，后续必须要有一些关键代码来启动里面的Netty服务器，才能接收网络请求。

<br>

**(3)NamesrvController启动时的工作**

回到NameServer启动时执行的main0()方法中：

    public class NamesrvStartup {
        ...
        public static NamesrvController main0(String[] args) {
            //NamesrvController是NameServer的核心代码组件
            //NameServer一启动，就会去启动这个NamesrvController
            try {
                NamesrvController controller = createNamesrvController(args);
                start(controller);
                String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
                log.info(tip);
                System.out.printf("%s%n", tip);
                return controller;
            } catch (Throwable e) {
                e.printStackTrace();
                System.exit(-1);
            }

            return null;
        }
        ...
    }

由上述代码可知，启动NameServer有两项工作：

工作一：创建NamesrvController，其中会创建和初始化两个配置类

工作二：执行start(controller)方法，也就是启动NamesrvController这个核心组件

NamesrvStartup的start()方法如下，首先会执行NamesrvController的initialize()方法初始化请求处理组件，然后会执行NamesrvController的start()方法启动NamesrvController请求处理组件。

    public class NamesrvStartup {
        ...
        public static NamesrvController start(final NamesrvController controller) throws Exception {
            if (null == controller) {
                throw new IllegalArgumentException("NamesrvController is null");
            }
            
            //执行NamesrvController的initialize()方法进行初始化操作
            boolean initResult = controller.initialize();
            if (!initResult) {
                controller.shutdown();
                System.exit(-3);
            }

            Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
                @Override
                public Void call() throws Exception {
                    controller.shutdown();
                    return null;
                }
            }));
            
            //执行NamesrvController的start()方法启动NamesrvController
            controller.start();
            return controller;
        }
        ...
    }

<br>

**(4)创建NettyRemotingServer网络服务器组件**

执行NamesrvController的initialize()方法进行初始化操作时，会创建一个NettyRemotingServer网络服务器组件。

    public class NamesrvController {
        ...
        public boolean initialize() {
            this.kvConfigManager.load();
            this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
            ...
        }
        ...
    }

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/03035a48-cb82-4b58-ad70-a578b3fdc0b7" />

<br>

**(5)NettyRemotingServer的初始化**

NettyRemotingServer构造方法的代码如下：

    public class NettyRemotingServer extends NettyRemotingAbstract implements RemotingServer {
        ...
        public NettyRemotingServer(final NettyServerConfig nettyServerConfig, final ChannelEventListener channelEventListener) {
            super(nettyServerConfig.getServerOnewaySemaphoreValue(), nettyServerConfig.getServerAsyncSemaphoreValue());
            this.serverBootstrap = new ServerBootstrap();
            ...
        }
        ...
    }

其中便通过Netty的ServerBootstrap类，创建一个Netty网络服务器。NettyRemotingServer是RocketMQ开发的一个网络服务器组件，它会基于Netty提供的ServerBootstrap类来实现一个Netty网络服务器。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/66aca29a-b36c-4a75-baca-99af789e2cc1" />

<br>

**4.NameServer如何启动Netty网络服务器**

**(1)NamesrvController初始化过程的完整代码**

**(2)NamesrvStartup.start()方法的工作**

**(3)Netty网络服务器是如何启动的**

<br>

**(1)NamesrvController初始化过程的完整代码**

在执行NamesrvController的initialize()方法进行请求处理组件的初始化时，会创建一个NettyRemotingServer网络服务器组件。在创建NettyRemotingServer时，会创建一个ServerBootstrap的Netty网络服务器。

接下来是NamesrvController.initialize()方法的完整源码：

    public class NamesrvController {
        ...
        public boolean initialize() {
            //加载kv之类的配置
            this.kvConfigManager.load();
            //初始化Netty网络服务器
            this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
            //Netty网络服务器的工作线程池
            this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
            //下面这行代码会把工作线程池交给Netty网络服务器
            this.registerProcessor();
            
            //下面这行代码就是启动一个后台线程，执行定时任务
            //从scanNotActiveBroker()可知这里会定时扫描哪些Broker没发送心跳而挂掉的
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    NamesrvController.this.routeInfoManager.scanNotActiveBroker();
                }
            }, 5, 10, TimeUnit.SECONDS);
            
            //下面这行代码就也是启动一个后台线程，执行定时任务
            //不过只是定时打印kv配置信息
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    NamesrvController.this.kvConfigManager.printAllPeriodically();
                }
            }, 1, 10, TimeUnit.MINUTES);


            //与FileWatchService相关
            if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
                //Register a listener to reload SslContext
                try {
                    fileWatchService = new FileWatchService(
                        new String[] {
                            TlsSystemConfig.tlsServerCertPath,
                            TlsSystemConfig.tlsServerKeyPath,
                            TlsSystemConfig.tlsServerTrustCertPath
                        },
                        new FileWatchService.Listener() {
                            boolean certChanged, keyChanged = false;
                            @Override
                            public void onChanged(String path) {
                                if (path.equals(TlsSystemConfig.tlsServerTrustCertPath)) {
                                    log.info("The trust certificate changed, reload the ssl context");
                                    reloadServerSslContext();
                                }
                                if (path.equals(TlsSystemConfig.tlsServerCertPath)) {
                                    certChanged = true;
                                }
                                if (path.equals(TlsSystemConfig.tlsServerKeyPath)) {
                                    keyChanged = true;
                                }
                                if (certChanged && keyChanged) {
                                    log.info("The certificate and private key changed, reload the ssl context");
                                    certChanged = keyChanged = false;
                                    reloadServerSslContext();
                                }
                            }
                            private void reloadServerSslContext() {
                                ((NettyRemotingServer) remotingServer).loadSslContext();
                            }
                        }
                    );
                } catch (Exception e) {
                    log.warn("FileWatchService created error, can't load the certificate dynamically");
                }
            }
            return true;
        }
        ...
    }

可见，NamesrvController.initialize()方法的主要工作还是初始化Netty网络服务器，其他的工作就是启动后台线程执行一些定时任务。

<br>

**(2)NamesrvStartup.start()方法的工作**

    public class NamesrvStartup {
        ...
        public static NamesrvController start(final NamesrvController controller) throws Exception {
            if (null == controller) {
                throw new IllegalArgumentException("NamesrvController is null");
            }
            //1.初始化NamesrvController请求处理组件
            boolean initResult = controller.initialize();
            if (!initResult) {
                controller.shutdown();
                System.exit(-3);
            }
            //2.通过Runtime类注册一个JVM关闭时的shutdown钩子
            Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
                @Override
                public Void call() throws Exception {
                    controller.shutdown();
                    return null;
                }
            }));
            //3.启动NamesrvController请求处理组件
            controller.start();
            return controller;
        }
        
        public void shutdown() {
            //1.首先关闭NettyRemotingServer释放网络资源
            this.remotingServer.shutdown();
            //2.然后关闭RemotingExecutor释放Netty网络服务器的工作线程池资源
            this.remotingExecutor.shutdown();
            //3.最后关闭ScheduledExecutorService释放执行定时任务的后台线程资源
            this.scheduledExecutorService.shutdown();


            if (this.fileWatchService != null) {
                this.fileWatchService.shutdown();
            }
        }
        ...
    }

在NamesrvStartup的start()方法中：首先会执行NamesrvController的initialize()方法初始化请求处理组件，在初始化的过程中，会创建出一个Netty网络服务器。

然后会通过Runtime类注册一个JVM关闭时的shutdown钩子，当JVM关闭时需要执行注册的回调函数，在回调函数中会执行NamesrvController的shutdown()方法关闭Netty服务器释放网络资源和关闭线程池释放线程资源。

最后便会执行NamesrvController的start()方法启动NamesrvController请求处理组件。这样，Netty网络服务器才会监听9876这个默认的端口号。

<br>

**(3)Netty网络服务器是如何启动的**

NamesrvController.start()方法的源码如下所示。可以看到，启动NamesrvContorller的核心就是启动NettyRemotingServer，即启动Netty网络服务器。

    public class NamesrvController {
        ...
        public void start() throws Exception {
            this.remotingServer.start();

            if (this.fileWatchService != null) {
                this.fileWatchService.start();
            }
        }
        ...
    }

NettyRemotingServer.start()方法的源码如下：

    public class NettyRemotingServer extends NettyRemotingAbstract implements RemotingServer {
        ...
        @Override
        public void start() {
            ...
            //下面是基于Netty的API去配置和启动一个Netty网络服务器
            ServerBootstrap childHandler =
                this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector)
                .channel(useEpoll() ? EpollServerSocketChannel.class : NioServerSocketChannel.class)
                .option(ChannelOption.SO_BACKLOG, nettyServerConfig.getServerSocketBacklog())
                .option(ChannelOption.SO_REUSEADDR, true)
                .option(ChannelOption.SO_KEEPALIVE, false)
                .childOption(ChannelOption.TCP_NODELAY, true)
                //设置Netty网络服务器要监听的端口号9876
                .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    //设置一些网络请求处理器，只要Netty网络服务器收到一个请求，就会依次使用下面的处理器来处理请求
                    //比如handshakeHandler就是负责连接握手、NettyDecoder是负责编码解码、IdleStateHandler是负责连接空闲管理
                    //connectionManageHandler是负责网络连接管理、serverHandler是负责最关键的网络请求的处理
                    @Override
                    public void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline()
                        .addLast(defaultEventExecutorGroup, HANDSHAKE_HANDLER_NAME, handshakeHandler)
                        .addLast(defaultEventExecutorGroup,
                            encoder,
                            new NettyDecoder(),
                            new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),
                            connectionManageHandler,
                            serverHandler
                        );
                    }
                }
            );
            ...
            try {
                //下面这行代码就是启动Netty网络服务器，其中的bind()方法就是绑定和监听一个端口号
                ChannelFuture sync = this.serverBootstrap.bind().sync();
                InetSocketAddress addr = (InetSocketAddress) sync.channel().localAddress();
                this.port = addr.getPort();
            } catch (InterruptedException e1) {
                throw new RuntimeException("this.serverBootstrap.bind().sync() InterruptedException", e1);
            }
            ...
        }
        ...
    }

至此Netty网络服务器便启动了，开始监听端口号9876，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/28ff9894-4008-4366-9509-3ff656faa212" />

<br>

**(4)总结**

NameServer启动的核心就是：基于Netty实现了一个网络服务器，然后监听默认的9876端口号来接收Broker和客户端发送的网络请求。

当NameServer启动完之后，就需要关注：Broker是如何启动的、如何向NameServer进行注册、如何进行心跳、NameServer又是如何管理Broker的。

<br>

**5.Broker启动时是如何初始化配置的**

**(1)NameServer的启动过程回顾**

**(2)BrokerStartup的入口源码分析**

**(3)创建Broker的几个配置组件**

**(4)为Broker的配置组件解析和填充信息**

<br>

**(1)NameServer的启动过程回顾**

前面从NameServer的启动脚本开始，首先介绍了它的配置的初始化，然后介绍核心的NamesrvController请求处理组件的初始化和启动，最后通过源码发现其底层会构建一个Netty网络服务器监听9876端口。

如下图示：当NameServer启动后，有一个Netty网络服务器监听9876端口，此时Broker和客户端就可以与NameServer建立长连接进行网络通信。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/8abdd33a-b6e8-4c23-a05b-014948b865b6" />

<br>

**(2)BrokerStartup的入口源码分析**

既然NameServer已经启动了，而且会有一个Netty网络服务器监听9876端口，等待接收Broker和客户端的连接和请求，接下来就看Broker是如何启动的了。

由于启动Broker是通过mqbroker脚本来实现的，所以脚本里一定会启动一个JVM进程来执行BrokerStartup的main()方法。这里就不再重复介绍Broker的启动脚本了，而直接分析BrokerStartup的main()方法。

这个BrokerStartup类，就在rocketmq源码中的broker模块里。其源码如下：

    public class BrokerStartup {
        ...
        public static void main(String[] args) {
            start(createBrokerController(args));
        }
        ...
    }

和NamesrvStratup的main()方法类似，同样会先创建一个Controller组件，再用start()方法启动这个Controller组件。

<br>

**(3)创建Broker的几个配置组件**

**一.解析传递进来命令行参数**

BrokerStartup的createBrokerContorller()方法源码如下：首先会通过ServerUtil的parseCmdLine()方法来解析传递进来命令行参数。

    public class BrokerStartup {
        ...
        public static BrokerController createBrokerController(String[] args) {
            System.setProperty(RemotingCommand.REMOTING_VERSION_KEY, Integer.toString(MQVersion.CURRENT_VERSION));
            try {
                Options options = ServerUtil.buildCommandlineOptions(new Options());
                commandLine = ServerUtil.parseCmdLine("mqbroker", args, buildCommandlineOptions(options), new PosixParser());
                if (null == commandLine) {
                    System.exit(-1);
                }
                ...
            }
            ...
        }
        ...
    }

<br>

**二.创建Broker的几个配置组件**

接着，createBrokerContorller()方法会创建Broker的几个配置组件。这些配置组件包括：Broker自己的配置BrokerConfig、Broker作为一个Netty服务器的配置NettyServerConfig、Broker作为一个Netty客户端的配置NettyClientConfig、Broker消息存储的配置MessageStoreConfig。

    public class BrokerStartup {
        ...
        public static BrokerController createBrokerController(String[] args) {
            ...
            //下面三个是Broker的核心配置类，分别是Broker的配置、Netty服务器的配置、Netty客户端的配置
            final BrokerConfig brokerConfig = new BrokerConfig();
            final NettyServerConfig nettyServerConfig = new NettyServerConfig();
            final NettyClientConfig nettyClientConfig = new NettyClientConfig();
            //这里设置了Netty客户端是否使用TLS的配置，TLS是一个加密机制
            nettyClientConfig.setUseTLS(Boolean.parseBoolean(System.getProperty(TLS_ENABLE, String.valueOf(TlsSystemConfig.tlsMode == TlsMode.ENFORCING))));
            //设置Netty服务器的监听端口号是10911
            nettyServerConfig.setListenPort(10911);
            //接着新建一个很关键的配置组件MessageStoreConfig，这些配置与在Broker中存储消息有关
            final MessageStoreConfig messageStoreConfig = new MessageStoreConfig();

            //如果当前这个Broker是Slave，那么就要设置一个特殊的参数accessMessageInMemoryMaxRatio
            if (BrokerRole.SLAVE == messageStoreConfig.getBrokerRole()) {
                int ratio = messageStoreConfig.getAccessMessageInMemoryMaxRatio() - 10;
                messageStoreConfig.setAccessMessageInMemoryMaxRatio(ratio);
            }
            ...
        }
        ...
    }

为什么Broker自己又是Netty服务器，又是Netty客户端？因为当客户端向Broker上发送请求时，Broker就是一个Netty服务器，负责监听客户端的连接请求。当Broker向NameServer发送请求时，Broker就是一个Netty客户端，要和NameServer的Netty服务器建立连接。

Broker的几个配置组件如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/e7c73e48-c069-4135-aea5-4fa50358a548" />

<br>

**(4)为Broker的配置组件解析和填充信息**

接下来看Broker启动时是如何为几个配置组件解析和填充信息的。如果在启动Broker时，用了-c选项带了一个配置文件的地址，那么就会读取配置文件里自定义的配置信息，然后读取出来覆盖到Broker的4个配置组件中。

    public class BrokerStartup {
        ...
        public static BrokerController createBrokerController(String[] args) {
            ...
            if (commandLine.hasOption('c')) {
                String file = commandLine.getOptionValue('c');
                if (file != null) {
                    configFile = file;
                    InputStream in = new BufferedInputStream(new FileInputStream(file));
                    properties = new Properties();
                    properties.load(in);

                    properties2SystemEnv(properties);
                    MixAll.properties2Object(properties, brokerConfig);
                    MixAll.properties2Object(properties, nettyServerConfig);
                    MixAll.properties2Object(properties, nettyClientConfig);
                    MixAll.properties2Object(properties, messageStoreConfig);

                    BrokerPathConfigHelper.setBrokerConfigPath(file);
                    in.close();
                }
            }
            ...
        }
        ...
    }

一般用mqbroker脚本启动Broker时，都需要提前创建好一个Broker配置文件，然后再使用-c选项带上这个配置文件的地址路径。这样，createBrokerController()方法便会读取到自定义的配置文件，填充到对应的Broker配置类里。

下面的源码，便介绍了Broker启动时是如何解析和填充配置的。

    public class BrokerStartup {
        ...
        public static BrokerController createBrokerController(String[] args) {
            ...
            //下面这段代码，就是判断一下Broker的角色，针对不同的角色做处理
            switch (messageStoreConfig.getBrokerRole()) {
                case ASYNC_MASTER:
                case SYNC_MASTER:
                    brokerConfig.setBrokerId(MixAll.MASTER_ID);
                    break;
                case SLAVE:
                    if (brokerConfig.getBrokerId() <= 0) {
                        System.out.printf("Slave's brokerId must be > 0");
                        System.exit(-3);
                    }
                    break;
                default:
                    break;
            }
            //这里会判断是否是基于dleger技术来管理主从同步和commitlog
            //如果是的话，就把Broker设置为-1
            if (messageStoreConfig.isEnableDLegerCommitLog()) {
                brokerConfig.setBrokerId(-1);
            }
            //下面这一行的配置，就是设置了HA监听端口号
            messageStoreConfig.setHaListenPort(nettyServerConfig.getListenPort() + 1);
            //下面10行代码与日志相关
            LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
            JoranConfigurator configurator = new JoranConfigurator();
            configurator.setContext(lc);
            lc.reset();
            System.setProperty("brokerLogDir", "");
            if (brokerConfig.isIsolateLogEnable()) {
                System.setProperty("brokerLogDir", brokerConfig.getBrokerName() + "_" + brokerConfig.getBrokerId());
            }
            if (brokerConfig.isIsolateLogEnable() && messageStoreConfig.isEnableDLegerCommitLog()) {
                System.setProperty("brokerLogDir", brokerConfig.getBrokerName() + "_" + messageStoreConfig.getdLegerSelfId());
            }
            configurator.doConfigure(brokerConfig.getRocketmqHome() + "/conf/logback_broker.xml");

            //如果命令行中包含了-p参数，就在启动Broker时打印一下所有配置类的启动参数
            if (commandLine.hasOption('p')) {
                InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
                MixAll.printObjectProperties(console, brokerConfig);
                MixAll.printObjectProperties(console, nettyServerConfig);
                MixAll.printObjectProperties(console, nettyClientConfig);
                MixAll.printObjectProperties(console, messageStoreConfig);
                System.exit(0);
            } else if (commandLine.hasOption('m')) {
                //如果命令行包含了-m参数，同样也是打印各种配置参数
                InternalLogger console = InternalLoggerFactory.getLogger(LoggerName.BROKER_CONSOLE_NAME);
                MixAll.printObjectProperties(console, brokerConfig, true);
                MixAll.printObjectProperties(console, nettyServerConfig, true);
                MixAll.printObjectProperties(console, nettyClientConfig, true);
                MixAll.printObjectProperties(console, messageStoreConfig, true);
                System.exit(0);
            }
            //下面5行代码会打印Broker的配置参数
            log = InternalLoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
            MixAll.printObjectProperties(log, brokerConfig);
            MixAll.printObjectProperties(log, nettyServerConfig);
            MixAll.printObjectProperties(log, nettyClientConfig);
            MixAll.printObjectProperties(log, messageStoreConfig);
            ...
        }
        ...
    }

其实其他的开源项目，可能也有类似上述的代码。就是构建配置类、读取配置文件的配置、解析命令行的配置参数，然后进行各种配置的校验和设置。

<br>

**6.BrokerController的创建以及包含的组件**

**(1)BrokerController是在哪里创建出来的**

**(2)为什么叫BrokerController**

**(3)BrokerController的构造函数会创建大量的组件和线程队列**

<br>

**(1)BrokerController是在哪里创建出来的**

上面介绍了Broker在启动时，首先会执行BrokerStartup的createBrokerController()方法。在这个方法里，会初始化以及解析Broker的4个配置组件，如下图示。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/4ab94f90-38c1-44b6-b1f1-e4f5ca66777f" />

这4个配置组件在初始化时，实际上就是用默认的配置参数值以及配置文件里的配置参数值、包括命令行传递的配置参数值，填充到配置组件中。然后在后续Broker运行的过程中，各种行为都会根据这些配置组件里的配置参数值来决定。

那么在准备好上述4个配置组件后，接下来就会创建最核心的Broker组件了。

    public class BrokerStartup {
        ...
        public static BrokerController createBrokerController(String[] args) {
            ...
            final BrokerController controller = new BrokerController(
                brokerConfig,
                nettyServerConfig,
                nettyClientConfig,
                messageStoreConfig
            );
            controller.getConfiguration().registerConfig(properties);
            ...
        }
        ...
    }

上述源码会创建一个BrokerController组件，这个BrokerController组件可以认为就是Broker自己。

<br>

**(2)为什么叫BrokerController**

下面介绍BrokerStartup、BrokerController和Broker之间的关系，以及为什么要这么设计。

首先BrokerStartup这个类是用来启动Broker的一个类，BrokerStartup里包含了对Broker进行初始化和完成Broker的启动工作的逻辑，所以BrokerStartup本身并不能代表一个Broker。

然后BrokerController类似于Java Web开发中使用Spring MVC框架开发的一系列Controller，专门用来负责接收和处理请求。但是毕竟中间件系统的架构设计思想和普通的Java Web业务系统还是不一样的，这两种Contorller还是存在区别。

BrokerController可以认为是Broker管理控制组件，也就是这个组件被创建出来以及完成初始化之后，是用来控制当前正在运行的Broker的。因此，使用mqbroker脚本启动的JVM进程，实际上也可以认为就是一个Broker。这里的Broker代表了一个JVM进程，而不是一个代码组件。而BrokerStartup作为一个拥有main()方法的类，则是一个代码组件。

因此，BrokerStartup的作用就是准备好4个配置组件，然后创建和启动BrokerController这个核心组件。也就是启动一个Broker管理控制组件，让BrokerController去控制和管理Broker这个JVM进程运行过程中的一切行为，包括接收网络请求、管理磁盘上的消息数据，以及管理后台线程的运行。

Broker、BrokerStartup、BrokerController之间的关系如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/47871470-63c5-41b8-a418-cceb9b3ce3f3" />

总结：首先Broker不是一个代码组件，而是用mqbroker脚本启动的JVM进程。然后BrokerStartup是用来启动JVM进程的拥有一个main()方法的类，它是一个启动组件，负责初始化4个配置组件，并基于这4个配置组件去启动BrokerControler这个管理控制组件。接着，在Broker这个JVM进程的运行期间，都是由BrokerController管理控制组件去管理Broker的请求处理、后台线程以及磁盘数据的。

<br>

**(3)BrokerController的构造函数会创建大量的组件和线程队列**

    public class BrokerController {
        ...
        public BrokerController(final BrokerConfig brokerConfig, final NettyServerConfig nettyServerConfig, final NettyClientConfig nettyClientConfig, final MessageStoreConfig messageStoreConfig) {
            //下面的4行代码就是把4个配置组件保存到本地缓存
            this.brokerConfig = brokerConfig;
            this.nettyServerConfig = nettyServerConfig;
            this.nettyClientConfig = nettyClientConfig;
            this.messageStoreConfig = messageStoreConfig;

            //接下来都是Broker的各种功能对应的组件
            //比如consumerOffsetManager是专门管理Consumer消费offset的
            //topicConfigManager是管理Topic配置的，pullMessageProcessor是处理Consumer发送请求过来拉取消息的
            //所以可以认为，Broker会有很多很多功能，每一个功能都是由一个组件来负责的
            //因此Broker在初始化时，内部就会有很多代码组件
            this.consumerOffsetManager = messageStoreConfig.isEnableLmq() ? new LmqConsumerOffsetManager(this) : new ConsumerOffsetManager(this);
            this.topicConfigManager = messageStoreConfig.isEnableLmq() ? new LmqTopicConfigManager(this) : new TopicConfigManager(this);
            this.pullMessageProcessor = new PullMessageProcessor(this);
            this.pullRequestHoldService = messageStoreConfig.isEnableLmq() ? new LmqPullRequestHoldService(this) : new PullRequestHoldService(this);
            this.messageArrivingListener = new NotifyMessageArrivingListener(this.pullRequestHoldService);
            this.consumerIdsChangeListener = new DefaultConsumerIdsChangeListener(this);
            this.consumerManager = new ConsumerManager(this.consumerIdsChangeListener);
            this.consumerFilterManager = new ConsumerFilterManager(this);
            this.producerManager = new ProducerManager();
            this.clientHousekeepingService = new ClientHousekeepingService(this);
            this.broker2Client = new Broker2Client(this);
            this.subscriptionGroupManager = messageStoreConfig.isEnableLmq() ? new LmqSubscriptionGroupManager(this) : new SubscriptionGroupManager(this);
            this.brokerOuterAPI = new BrokerOuterAPI(nettyClientConfig);
            this.filterServerManager = new FilterServerManager(this);

            this.slaveSynchronize = new SlaveSynchronize(this);

            //接下来是线程池的队列，它们是用来实现某些功能的后台线程池的队列
            //不同的后台线程和处理请求的线程放在不同的线程池里去执行
            //因为有些Broker的功能是接收请求进行处理时会用到上面的一些组件、有些Broker的功能就是由自己的后台线程去执行
            this.sendThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getSendThreadPoolQueueCapacity());
            this.putThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getPutThreadPoolQueueCapacity());
            this.pullThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getPullThreadPoolQueueCapacity());
            this.replyThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getReplyThreadPoolQueueCapacity());
            this.queryThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getQueryThreadPoolQueueCapacity());
            this.clientManagerThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getClientManagerThreadPoolQueueCapacity());
            this.consumerManagerThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getConsumerManagerThreadPoolQueueCapacity());
            this.heartbeatThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getHeartbeatThreadPoolQueueCapacity());
            this.endTransactionThreadPoolQueue = new LinkedBlockingQueue<Runnable>(this.brokerConfig.getEndTransactionPoolQueueCapacity());

            //下面这些同样是Broker的一些功能性组件
            //比如brokerStatsManager就是metric统计组件，用于对Broker内部进行统计的
            //比如brokerFastFailure是用来处理Broker故障的组件
            this.brokerStatsManager = messageStoreConfig.isEnableLmq() ? 
                new LmqBrokerStatsManager(this.brokerConfig.getBrokerClusterName(), this.brokerConfig.isEnableDetailStat()) : 
                new BrokerStatsManager(this.brokerConfig.getBrokerClusterName(), this.brokerConfig.isEnableDetailStat());
            this.setStoreHost(new InetSocketAddress(this.getBrokerConfig().getBrokerIP1(), this.getNettyServerConfig().getListenPort()));
            this.brokerFastFailure = new BrokerFastFailure(this);
            this.configuration = new Configuration(
                log,
                BrokerPathConfigHelper.getBrokerConfigPath(),
                this.brokerConfig, this.nettyServerConfig, this.nettyClientConfig, this.messageStoreConfig
            );
        }
        ...
    }

BrokerController内部有一系列的功能性组件，以及大量的后台线程池，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/fcd2ad0a-ccf7-4a98-81e3-7716c00f8cc6" />

<br>

**7.BrokerController的初始化**

**(1)创建完BrokerController之后需要初始化**

**(2)BrokerController的初始化过程**

<br>

**(1)创建完BrokerController之后需要初始化**

现已知Broker作为一个JVM进程启动后，会由BrokerStartup这个启动组件先初始化4个配置组件，然后再通过这4个配置组件创建BrokerController这个管理控制组件，在BrokerController管控组件中会包含大量的核心功能组件和后台线程池。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/d8919754-b7b5-4a9f-a3e8-c399ac38c157" />

当创建好BrokerController以及里面的核心功能组件和后台线程池之后，接着还要初始化BrokerController。

在BrokerStartup的createBrokerController()方法中，创建好BrokerController之后，就会调用BrokerController的initialize()方法来触发BrokerController的初始化，如下所示：

    public class BrokerStartup {
        ...
        public static BrokerController createBrokerController(String[] args) {
            ...
            //在这里触发BrokerController的初始化
            boolean initResult = controller.initialize();
            if (!initResult) {
                controller.shutdown();
                System.exit(-3);
            }
            //这里会注册一个JVM的关闭钩子，JVM退出时就会执行里面的回调函数，其本质也是在释放一些资源
            Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
                private volatile boolean hasShutdown = false;
                private AtomicInteger shutdownTimes = new AtomicInteger(0);
                @Override
                public void run() {
                    synchronized (this) {
                        log.info("Shutdown hook was invoked, {}", this.shutdownTimes.incrementAndGet());
                        if (!this.hasShutdown) {
                            this.hasShutdown = true;
                            long beginTime = System.currentTimeMillis();
                            controller.shutdown();
                            long consumingTimeTotal = System.currentTimeMillis() - beginTime;
                            log.info("Shutdown hook over, consuming total time(ms): {}", consumingTimeTotal);
                        }
                    }
                }
            }, "ShutdownHook"));

            //最后这个createBrokerController()方法就会返回创建和初始化好的BrokerController了
            return controller;
            ...
        }
        ...
    }

<br>

**(2)BrokerController的初始化过程**

BrokerController的initialize()方法的部分源码：

    public class BrokerController {
        ...
        public boolean initialize() throws CloneNotSupportedException {
            //首先开头4行代码，其实就是在加载一些磁盘上的数据到内存
            //比如加载Topic的配置、Consumer的消费offset、Consumer订阅组、过滤器，这些信息如果都加载成功了，那么result就是true
            boolean result = this.topicConfigManager.load();
            result = result && this.consumerOffsetManager.load();
            result = result && this.subscriptionGroupManager.load();
            result = result && this.consumerFilterManager.load();

            if (result) {
                try {
                    //下面一行代码创建了消息存储管理组件，用于管理磁盘上的消息
                    this.messageStore = new DefaultMessageStore(this.messageStoreConfig, this.brokerStatsManager, this.messageArrivingListener, this.brokerConfig);
                    //如果启用了DLeger技术进行主从同步以及管理CommitLog，就要初始化一些与DLeger相关的组件
                    if (messageStoreConfig.isEnableDLegerCommitLog()) {
                        DLedgerRoleChangeHandler roleChangeHandler = new DLedgerRoleChangeHandler(this, (DefaultMessageStore) messageStore);
                        ((DLedgerCommitLog)((DefaultMessageStore) messageStore).getCommitLog()).getdLedgerServer().getdLedgerLeaderElector().addRoleChangeHandler(roleChangeHandler);
                    }
                    //BrokerStats是Broker的统计组件
                    this.brokerStats = new BrokerStats((DefaultMessageStore) this.messageStore);
                    //load plugin
                    MessageStorePluginContext context = new MessageStorePluginContext(messageStoreConfig, brokerStatsManager, messageArrivingListener, brokerConfig);
                    this.messageStore = MessageStoreFactory.build(context, this.messageStore);
                    this.messageStore.getDispatcherList().addFirst(new CommitLogDispatcherCalcBitMap(this.brokerConfig, this.consumerFilterManager));
                } catch (IOException e) {
                    result = false;
                    log.error("Failed to initialize", e);
                }
            }

            result = result && this.messageStore.load();

            if (result) {
                //下面一行代码很关键
                //由于Broker需要接收客户端的请求，比如Producer发送请求过来发消息或Consumer发送请求过来拉取消息，所以Broker也是有Netty服务器的
                this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.clientHousekeepingService);
                NettyServerConfig fastConfig = (NettyServerConfig) this.nettyServerConfig.clone();
                fastConfig.setListenPort(nettyServerConfig.getListenPort() - 2);
                //创建Netty服务器组件
                this.fastRemotingServer = new NettyRemotingServer(fastConfig, this.clientHousekeepingService);

                //从下面开始的代码，都是在初始化一些线程池
                //这些线程池，有的是负责处理请求的线程池，有的是在后台运行的线程池，比如sendMessageExecutor就是处理发送过来的消息
                this.sendMessageExecutor = new BrokerFixedThreadPoolExecutor(this.brokerConfig.getSendMessageThreadPoolNums(), this.brokerConfig.getSendMessageThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.sendThreadPoolQueue, new ThreadFactoryImpl("SendMessageThread_"));
                //putMessageFutureExecutor是处理producer发送消息的线程池
                this.putMessageFutureExecutor = new BrokerFixedThreadPoolExecutor(this.brokerConfig.getPutMessageFutureThreadPoolNums(), this.brokerConfig.getPutMessageFutureThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.putThreadPoolQueue, new ThreadFactoryImpl("PutMessageThread_"));
                //pullMessageExecutor是处理consumer拉取消息的线程池
                this.pullMessageExecutor = new BrokerFixedThreadPoolExecutor(this.brokerConfig.getPullMessageThreadPoolNums(), this.brokerConfig.getPullMessageThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.pullThreadPoolQueue, new ThreadFactoryImpl("PullMessageThread_"));
                //replyMessageExecutor是回复消息的线程池
                this.replyMessageExecutor = new BrokerFixedThreadPoolExecutor(this.brokerConfig.getProcessReplyMessageThreadPoolNums(), this.brokerConfig.getProcessReplyMessageThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.replyThreadPoolQueue, new ThreadFactoryImpl("ProcessReplyMessageThread_"));
                //这是查询消息的线程池
                this.queryMessageExecutor = new BrokerFixedThreadPoolExecutor(this.brokerConfig.getQueryMessageThreadPoolNums(), this.brokerConfig.getQueryMessageThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.queryThreadPoolQueue, new ThreadFactoryImpl("QueryMessageThread_"));
                //adminBrokerExecutor是管理Broker的一些命令执行的线程池
                this.adminBrokerExecutor = Executors.newFixedThreadPool(this.brokerConfig.getAdminBrokerThreadPoolNums(), new ThreadFactoryImpl("AdminBrokerThread_"));
                //clientManageExecutor是管理客户端的线程池
                this.clientManageExecutor = new ThreadPoolExecutor(this.brokerConfig.getClientManageThreadPoolNums(), this.brokerConfig.getClientManageThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.clientManagerThreadPoolQueue, new ThreadFactoryImpl("ClientManageThread_"));
                //heartbeatExecutor是一个后台线程池，负责给NameServer发送心跳的
                this.heartbeatExecutor = new BrokerFixedThreadPoolExecutor(this.brokerConfig.getHeartbeatThreadPoolNums(), this.brokerConfig.getHeartbeatThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.heartbeatThreadPoolQueue, new ThreadFactoryImpl("HeartbeatThread_", true));
                //endTransactionExecutor是结束事务的线程池，与事务消息相关
                this.endTransactionExecutor = new BrokerFixedThreadPoolExecutor(this.brokerConfig.getEndTransactionThreadPoolNums(), this.brokerConfig.getEndTransactionThreadPoolNums(), 1000 * 60, TimeUnit.MILLISECONDS, this.endTransactionThreadPoolQueue, new ThreadFactoryImpl("EndTransactionThread_"));
                //consumerManageExecutor是管理consumer的线程池
                this.consumerManageExecutor = Executors.newFixedThreadPool(this.brokerConfig.getConsumerManageThreadPoolNums(), new ThreadFactoryImpl("ConsumerManageThread_"));

                this.registerProcessor();

                //下面这些代码，就是开始定时调度一些后台线程执行
                final long initialDelay = UtilAll.computeNextMorningTimeMillis() - System.currentTimeMillis();
                final long period = 1000 * 60 * 60 * 24;

                //下面就是定时进行Broker统计的任务
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            BrokerController.this.getBrokerStats().record();
                        } catch (Throwable e) {
                            log.error("schedule record error.", e);
                        }
                    }
                }, initialDelay, period, TimeUnit.MILLISECONDS);
                
                //下面就是定时进行Consumer消费offset持久化到磁盘的任务
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            BrokerController.this.consumerOffsetManager.persist();
                        } catch (Throwable e) {
                            log.error("schedule persist consumerOffset error.", e);
                        }
                    }
                }, 1000 * 10, this.brokerConfig.getFlushConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
                
                //下面就是定时对Consumer Filter过滤器进行持久化的任务
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            BrokerController.this.consumerFilterManager.persist();
                        } catch (Throwable e) {
                            log.error("schedule persist consumer filter error.", e);
                        }
                    }
                }, 1000 * 10, 1000 * 10, TimeUnit.MILLISECONDS);
                
                //下面是定时进行Broker保护的任务
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            BrokerController.this.protectBroker();
                        } catch (Throwable e) {
                            log.error("protectBroker error.", e);
                        }
                    }
                }, 3, 3, TimeUnit.MINUTES);
                
                //下面是定时打印waterMark，就是水位的任务
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            BrokerController.this.printWaterMark();
                        } catch (Throwable e) {
                            log.error("printWaterMark error.", e);
                        }
                    }
                }, 10, 1, TimeUnit.SECONDS);
                
                //下面是定时进行落后CommitLog分发的任务
                this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                    @Override
                    public void run() {
                        try {
                            log.info("dispatch behind commit log {} bytes", BrokerController.this.getMessageStore().dispatchBehindBytes());
                        } catch (Throwable e) {
                            log.error("schedule dispatchBehindBytes error.", e);
                        }
                    }
                }, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
                ...
            }
            ...
        }
        ...
    }

可见，BrokerController的初始化，会先创建Netty服务器组件，然后再初始化和启动大量用于处理请求的线程池和后台定时调度任务。毕竟Broker要处理各种请求，不同的请求需要用不同的线程池里的线程进行处理。然后Broker还要执行不同的后台定时调度任务，这些后台定时任务也要通过线程池来调度。

于是，在如下图中加入两类线程池：一类线程池是用来处理客户端发送过来的请求，另一类线程池是执行后台定时调度任务。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/f0b47703-a522-4f6c-a890-458fb9c72b51" />

BrokerController的initialize()方法的剩余源码：

    public class BrokerController {
        ...
        public boolean initialize() throws CloneNotSupportedException {
            ...
            if (result) {
                ...
                //下面这段代码是处理与DLeger相关的
                //如果开启了DLeger技术，下面会进行一些操作，可不用深究
                if (!messageStoreConfig.isEnableDLegerCommitLog()) {
                    if (BrokerRole.SLAVE == this.messageStoreConfig.getBrokerRole()) {
                        if (this.messageStoreConfig.getHaMasterAddress() != null && this.messageStoreConfig.getHaMasterAddress().length() >= 6) {
                            this.messageStore.updateHaMasterAddress(this.messageStoreConfig.getHaMasterAddress());
                            this.updateMasterHAServerAddrPeriodically = false;
                        } else {
                            this.updateMasterHAServerAddrPeriodically = true;
                        }
                    } else {
                        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    BrokerController.this.printMasterAndSlaveDiff();
                                } catch (Throwable e) {
                                    log.error("schedule printMasterAndSlaveDiff error.", e);
                                }
                            }
                        }, 1000 * 10, 1000 * 60, TimeUnit.MILLISECONDS);
                    }
                }
                //下面的代码是与文件相关的，不需要管
                if (TlsSystemConfig.tlsMode != TlsMode.DISABLED) {
                    ...
                }

                //和事务消息有关的，会进行事务相关的初始化工作
                initialTransaction();
                //和ACL权限控制有关
                initialAcl();
                //和RPC钩子有关
                initialRpcHooks();
            }
            return result;
        }
        ...
    }

<br>

**(3)总结**

当BrokerController完成初始化之后，其实就准备好了用来接收网络请求的Netty服务器、准备好了处理各种网络请求的线程池、准备好了各种用来执行后台定时调度任务的线程池，接下来就会启动BrokerController。

BrokerController的启动会完成Netty服务器的启动，这样Broker才可以接收请求。同时Broker会在BrokerController启动的过程中，向NameServer进行注册以及保持心跳。只有这样，Producer才能从NameServer上找到Broker来发送消息。

<br>

**8.BrokerController的启动**

**(1)初始化完BrokerController之后需要启动**

**(2)BrokerController的启动过程**

<br>

**(1)初始化完BrokerController之后需要启动**

假设现在BrokerController已经完成初始化了，也就是BrokerController中用于实现各种功能的核心组件都已初始化完毕，然后负责接收请求的Netty服务器也初始化完毕，同时负责处理请求的线程池以及执行定时调度任务的线程池也初始化完毕。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/e58b8b22-a571-46cd-85a0-6e5f61083796" />

这时就要对BrokerContorller执行启动的逻辑了，让它里面的一些功能组件完成启动时需要执行的一些工作。比如完成Netty服务器的启动，让Netty服务器去监听一个端口号来接收客户端发来的请求。

BrokerContorller的启动由BrokerStartup的main()方法调用BrokerStartup的start()方法触发：

    public class BrokerStartup {
        ...
        public static void main(String[] args) {
            start(createBrokerController(args));
        }
        
        public static BrokerController start(BrokerController controller) {
            try {
                controller.start();
                String tip = "The broker[" + controller.getBrokerConfig().getBrokerName() + ", " + controller.getBrokerAddr() + "] boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
                if (null != controller.getBrokerConfig().getNamesrvAddr()) {
                    tip += " and name server is " + controller.getBrokerConfig().getNamesrvAddr();
                }

                log.info(tip);
                System.out.printf("%s%n", tip);
                return controller;
            } catch (Throwable e) {
                e.printStackTrace();
                System.exit(-1);
            }
            return null;
        }
        ...
    }

在BrokerStartup的start()方法中，主要就是通过执行BrokerContorller的start()方法来启动初始化好的BrokerController。

<br>

**(2)BrokerController的启动过程**

BrokerContorller的start()方法源码，如下所示：

    public class BrokerController {
        ...
        public void start() throws Exception {
            //启动消息存储组件
            if (this.messageStore != null) {
                this.messageStore.start();
            }
            //启动Netty服务器，这样就可以接收请求了
            if (this.remotingServer != null) {
                this.remotingServer.start();
            }
            if (this.fastRemotingServer != null) {
                this.fastRemotingServer.start();
            }
            //启动和文件相关的服务组件fileWatchService
            if (this.fileWatchService != null) {
                this.fileWatchService.start();
            }
            //brokerOuterAPI是核心组件，该组件可以让Broker通过Netty客户端发送请求出去给别人
            //比如Broker发送请求到NameServer去注册以及进行心跳就是通过这个组件实现的
            if (this.brokerOuterAPI != null) {
                this.brokerOuterAPI.start();
            }
            if (this.pullRequestHoldService != null) {
                this.pullRequestHoldService.start();
            }
            if (this.clientHousekeepingService != null) {
                this.clientHousekeepingService.start();
            }
            if (this.filterServerManager != null) {
                this.filterServerManager.start();
            }
            if (!messageStoreConfig.isEnableDLegerCommitLog()) {
                startProcessorByHa(messageStoreConfig.getBrokerRole());
                handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
                this.registerBrokerAll(true, false, true);
            }
            //这里往线程池里提交了一个任务，让它去NameServer进行注册
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    try {
                        BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                    } catch (Throwable e) {
                        log.error("registerBrokerAll Exception", e);
                    }
                }
            }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
            //下面是一些功能组件的启动
            if (this.brokerStatsManager != null) {
                this.brokerStatsManager.start();
            }
            if (this.brokerFastFailure != null) {
                this.brokerFastFailure.start();
            }
        }
        ...
    }

上述源码的核心就是：启动Netty服务器来接收网络请求、启动BrokerOuterAPI组件让其基于Netty客户端将请求发送出去、同时启动一个线程向NameServer进行注册。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/9bb41c89-dfb2-4463-a933-3e124ca74f16" />

由于这里关注的是系统运行的主要流程和逻辑，所以不会去看BrokerOuterAPI、RemotingServer、FileWatchService、MessageStore这些组件的源码细节。

这里关注的事情总结如下：

一.Broker启动后，需要通过BrokerOuterAPI组件将自己注册到NameServer中。

二.Broker启动后，需要有一个网络服务器NettyServer组件去接收客户端请求。

三.NettyServer组件接收到网络请求后，需要有一个处理各种请求的线程池来进行处理。

四.处理请求的线程池在处理每个请求时，需要各种核心功能组件的协调。比如写入消息到CommitLog、写入索引到IndexFile和ConsumerQueue文件，此时需要MessageStore组件来配合。

五.最后还需要一些定时运行的后台线程来处理如定时发送心跳到NameServer。

接下来会通过一些运行场景来介绍RocketMQ的源码，包括Broker的注册和心跳、客户端Producer的启动和初始化、Producer从NameServer拉取路由信息、Producer根据负载均衡算法选择一个Broker机器、Producer跟Broker建立网络连接、Producer发送消息到Broker、Broker把消息存储到磁盘。从这些场景出发去理解RocketMQ源码的各个组件是如何配合起来运行的。

<br>

**9.Broker如何把自己注册到NameServer上**

**(1)Broker将自己注册到NameServer的入口**

**(2)BrokerController.registerBrokerAll()方法**

**(3)真正进行注册的doRegisterBrokerAll()方法**

**(4)BrokerOuterAPI中具体的Broker注册逻辑**

<br>

**(1)Broker将自己注册到NameServer的入口**

前面介绍了BrokerController启动的过程，其本质就是启动Netty服务器去接收网络请求，以及启动一些核心功能组件、启动一些处理请求的线程池、启动一些执行定时调度任务的后台线程。如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/c469fa71-041d-4cb3-bc89-38ebb875e321" />

当然最为关键的，就是BrokerController将自己注册到NameServer。这个将自己注册到NameServer的源码入口，就在BrokerController的start()方法中，如下所示：

    public class BrokerController {
        ...
        public void start() throws Exception {
            ...
            //这里往线程池里提交了一个任务，让它去NameServer进行注册
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    try {
                        BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                    } catch (Throwable e) {
                        log.error("registerBrokerAll Exception", e);
                    }
                }
            }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);
            ...
        }
        ...
    }

因此如果要继续了解RocketMQ源码，当然最好通过运行场景来介绍。前面已介绍完NameServer和Broker两个核心系统的启动场景，接下来介绍Broker往NameServer进行注册的场景。因为只有完成了注册，NameServer才能知道集群里有哪些Broker，然后Producer和Consumer才能向NameServer拉取路由数据、才能知道集群里有哪些Broker、才能和Broker进行通信。

<br>

**(2)BrokerController.registerBrokerAll()方法**

    public class BrokerController {
        ...
        public synchronized void registerBrokerAll(final boolean checkOrderConfig, boolean oneway, boolean forceRegister) {
            //下面一行代码与Topic配置信息相关，可先不管
            TopicConfigSerializeWrapper topicConfigWrapper = this.getTopicConfigManager().buildTopicConfigSerializeWrapper();
            //下面一段代码都是在处理TopicConfig，可先不管
            if (!PermName.isWriteable(this.getBrokerConfig().getBrokerPermission()) || !PermName.isReadable(this.getBrokerConfig().getBrokerPermission())) {
                ConcurrentHashMap<String, TopicConfig> topicConfigTable = new ConcurrentHashMap<String, TopicConfig>();
                for (TopicConfig topicConfig : topicConfigWrapper.getTopicConfigTable().values()) {
                    TopicConfig tmp = new TopicConfig(topicConfig.getTopicName(), topicConfig.getReadQueueNums(), topicConfig.getWriteQueueNums(), this.brokerConfig.getBrokerPermission());
                    topicConfigTable.put(topicConfig.getTopicName(), tmp);
                }
                topicConfigWrapper.setTopicConfigTable(topicConfigTable);
            }
            //接下来这段代码比较关键，判断是否要进行注册
            //如果要进行注册的话，就调用doRegisterBrokerAll()这个方法去真正注册
            if (forceRegister || needRegister(this.brokerConfig.getBrokerClusterName(),
                this.getBrokerAddr(),
                this.brokerConfig.getBrokerName(),
                this.brokerConfig.getBrokerId(),
                this.brokerConfig.getRegisterBrokerTimeoutMills())) {
                doRegisterBrokerAll(checkOrderConfig, oneway, topicConfigWrapper);
            }
        }
        ...
    }

<br>

**(3)真正进行注册的doRegisterBrokerAll()方法**

    public class BrokerController {
        ...
        private void doRegisterBrokerAll(boolean checkOrderConfig, boolean oneway, TopicConfigSerializeWrapper topicConfigWrapper) {
            //进行注册的核心代码就在这里
            //这里调用了BrokerOuterAPI去发送请求给NameServer
            //在这里就完成了Broker的注册，然后获取到了注册的结果
            //为什么注册结果是个List，因为Broker会把自己注册给所有的NameServer
            List<RegisterBrokerResult> registerBrokerResultList = this.brokerOuterAPI.registerBrokerAll(
                this.brokerConfig.getBrokerClusterName(),
                this.getBrokerAddr(),
                this.brokerConfig.getBrokerName(),
                this.brokerConfig.getBrokerId(),
                this.getHAServerAddr(),
                topicConfigWrapper,
                this.filterServerManager.buildNewFilterServerList(),
                oneway,
                this.brokerConfig.getRegisterBrokerTimeoutMills(),
                this.brokerConfig.isCompressedRegister()
            );

            //如果注册结果的数量大于0，那么就在这里对注册结果进行处理，处理的逻辑涉及到了MasterHAServer
            if (registerBrokerResultList.size() > 0) {
                RegisterBrokerResult registerBrokerResult = registerBrokerResultList.get(0);
                if (registerBrokerResult != null) {
                    if (this.updateMasterHAServerAddrPeriodically && registerBrokerResult.getHaServerAddr() != null) {
                        this.messageStore.updateHaMasterAddress(registerBrokerResult.getHaServerAddr());
                    }
                    this.slaveSynchronize.setMasterAddr(registerBrokerResult.getMasterAddr());
                    if (checkOrderConfig) {
                        this.getTopicConfigManager().updateOrderTopicConfig(registerBrokerResult.getKvTable());
                    }
                }
            }
        }
        ...
    }

上述代码实际上就是通过BrokerOuterAPI发送网络请求给所有NameServer，把这个Broker注册上去。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/c9a464bf-9437-4134-a0ee-0a9ba60cc94d" />

<br>

**(4)BrokerOuterAPI中具体的Broker注册逻辑**

BrokerOuterAPI的registerBrokerAll()方法，便会向所有的NameServer发起具体的Broker注册请求，如下所示：

    public class BrokerOuterAPI {
        ...
        public List<RegisterBrokerResult> registerBrokerAll(final String clusterName, final String brokerAddr, final String brokerName, final long brokerId, final String haServerAddr, final TopicConfigSerializeWrapper topicConfigWrapper, final List<String> filterServerList, final boolean oneway, final int timeoutMills, final boolean compressed) {
            //下面初始化了一个List，用来存放Broker向每个NameServer进行注册的结果
            final List<RegisterBrokerResult> registerBrokerResultList = new CopyOnWriteArrayList<>();
            //下面这个List就是NameServer的地址列表
            List<String> nameServerAddressList = this.remotingClient.getNameServerAddressList();
            if (nameServerAddressList != null && nameServerAddressList.size() > 0) {
                //下面代码在构建注册的网络请求
                //首先构造一个请求头，在请求头里加入了很多信息，比如Broker的ID和名称等
                final RegisterBrokerRequestHeader requestHeader = new RegisterBrokerRequestHeader();
                requestHeader.setBrokerAddr(brokerAddr);
                requestHeader.setBrokerId(brokerId);
                requestHeader.setBrokerName(brokerName);
                requestHeader.setClusterName(clusterName);
                requestHeader.setHaServerAddr(haServerAddr);
                requestHeader.setCompressed(compressed);

                //接着构造一个请求体，请求体里就会包含一些配置
                RegisterBrokerBody requestBody = new RegisterBrokerBody();
                requestBody.setTopicConfigSerializeWrapper(topicConfigWrapper);
                requestBody.setFilterServerList(filterServerList);
                final byte[] body = requestBody.encode(compressed);
                final int bodyCrc32 = UtilAll.crc32(body);
                requestHeader.setBodyCrc32(bodyCrc32);

                //通过CountDownLatch来限制要注册完全部的NameServer后才能往下继续执行
                final CountDownLatch countDownLatch = new CountDownLatch(nameServerAddressList.size());
                //这里会遍历NameServer地址列表，因为每个NameServer都要发送请求过去进行注册
                for (final String namesrvAddr : nameServerAddressList) {
                    brokerOuterExecutor.execute(new Runnable() {
                        @Override
                        public void run() {
                            try {
                                //真正执行注册的代码是下面这一行
                                RegisterBrokerResult result = registerBroker(namesrvAddr, oneway, timeoutMills, requestHeader, body);
                                //注册完了，注册结果就放到一个List里去
                                if (result != null) {
                                    registerBrokerResultList.add(result);
                                }
                                log.info("register broker[{}]to name server {} OK", brokerId, namesrvAddr);
                            } catch (Exception e) {
                                log.warn("registerBroker Exception, {}", namesrvAddr, e);
                            } finally {
                                //注册完了，就会执行CountDownLatch的countDown()方法
                                countDownLatch.countDown();
                            }
                        }
                    });
                }

                //通过CountDownLatch的await()方法等待所有的NameServer都注册完毕，才会继续执行后面的代码
                try {
                    countDownLatch.await(timeoutMills, TimeUnit.MILLISECONDS);
                } catch (InterruptedException e) {
                
                }
            }
            return registerBrokerResultList;
        }
        ...
    }

registerBrokerAll()方法，首先会封装好请求头RequestHeader和请求体RequestBody，从而构成一个请求。然后通过底层的NettyClient把这个注册请求发送到NameServer上。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/10936c96-291e-4cc0-9876-03b02f0dff1a" />

<br>

**10.BrokerOuterAPI是如何发送注册请求的**

**(1)发送Broker注册请求的方法入口**

**(2)NettyClient的网络请求方法invokeSync()**

**(3)Broker如何与NameServer建立网络连接**

**(4)如何通过Channel网络连接发送请求**

<br>

**(1)发送Broker注册请求的方法入口**

现在进入到向NameServer真正注册Broker的网络请求方法里去看，其入口就是：

    //真正执行注册的代码是下面这一行
    RegisterBrokerResult result = registerBroker(namesrvAddr, oneway, timeoutMills, requestHeader, body);
    //注册完了，注册结果就放到一个List里去
    if (result != null) {
        registerBrokerResultList.add(result);
    }

BrokerOuterAPI的registerBroker()方法如下：

    public class BrokerOuterAPI {
        ...
        private RegisterBrokerResult registerBroker(final String namesrvAddr, final boolean oneway, final int timeoutMills, final RegisterBrokerRequestHeader requestHeader, final byte[] body) throws RemotingCommandException, MQBrokerException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException, InterruptedException {
            //把请求头和请求体都封装到RemotingCommand中
            RemotingCommand request = RemotingCommand.createRequestCommand(RequestCode.REGISTER_BROKER, requestHeader);
            request.setBody(body);
            //这个oneway是指不用等待注册结果，属于特殊的请求
            if (oneway) {
                try {
                    this.remotingClient.invokeOneway(namesrvAddr, request, timeoutMills);
                } catch (RemotingTooMuchRequestException e) {
                    // Ignore
                }
                return null;
            }
            //一般情况，真正的发送网络请求的逻辑都在下面
            //这个remotingClient其实就是NettyClient，通过NettyClient将网络请求发送出去
            RemotingCommand response = this.remotingClient.invokeSync(namesrvAddr, request, timeoutMills);

            //接下来就是处理网络请求的返回结果
            //把注册请求的结果封装成Result保存起来，并且返回Result
            assert response != null;
            switch (response.getCode()) {
                case ResponseCode.SUCCESS: {
                    RegisterBrokerResponseHeader responseHeader = (RegisterBrokerResponseHeader) response.decodeCommandCustomHeader(RegisterBrokerResponseHeader.class);
                    RegisterBrokerResult result = new RegisterBrokerResult();
                    result.setMasterAddr(responseHeader.getMasterAddr());
                    result.setHaServerAddr(responseHeader.getHaServerAddr());
                    if (response.getBody() != null) {
                        result.setKvTable(KVTable.decode(response.getBody(), KVTable.class));
                    }
                    return result;
                }
                default:
                    break;
            }
            throw new MQBrokerException(response.getCode(), response.getRemark(), requestHeader == null ? null : requestHeader.getBrokerAddr());
        }
        ...
    }

由上述代码可知，注册请求最终是基于NettyClient组件发送给NameServer的。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/df7f60ee-c2fe-4201-a5f3-49ce8bd4c01c" />

<br>

**(2)NettyClient的网络请求方法invokeSync()**

接着进入NettyClient的网络请求方法invokeSync()去看，代码如下：

    public class NettyRemotingClient extends NettyRemotingAbstract implements RemotingClient {
        ...
        @Override
        public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis) throws InterruptedException, RemotingConnectException, RemotingSendRequestException, RemotingTimeoutException {
            //获取当前时间
            long beginStartTime = System.currentTimeMillis();
            //下面这行代码获取了一个Channel，这个Channel就是当前这台Broker机器和NameServer机器之间的一个连接
            //所以会将NameServer的地址作为参数传递进去，表示要和该NameServer机器建立一个网络连接
            //当连接建立后，就会用一个Channel来代表该连接
            final Channel channel = this.getAndCreateChannel(addr);
            //如果和NameServer之间的网络连接是OK的，那么就可以发送请求了
            if (channel != null && channel.isActive()) {
                try {
                    //发送请求前RPC钩子做的事情
                    doBeforeRpcHooks(addr, request);
                    long costTime = System.currentTimeMillis() - beginStartTime;
                    if (timeoutMillis < costTime) {
                        throw new RemotingTimeoutException("invokeSync call the addr[" + addr + "] timeout");
                    }
                    //下面这行代码会真正发送网络请求出去
                    RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);
                    //发送请求后RPC钩子做的事情
                    doAfterRpcHooks(RemotingHelper.parseChannelRemoteAddr(channel), request, response);
                    return response;
                } catch (RemotingSendRequestException e) {
                    log.warn("invokeSync: send request exception, so close the channel[{}]", addr);
                    this.closeChannel(addr, channel);
                    throw e;
                } catch (RemotingTimeoutException e) {
                    if (nettyClientConfig.isClientCloseSocketIfTimeout()) {
                        this.closeChannel(addr, channel);
                        log.warn("invokeSync: close socket because of timeout, {}ms, {}", timeoutMillis, addr);
                    }
                    log.warn("invokeSync: wait response timeout exception, the channel[{}]", addr);
                    throw e;
                }
            } else {
                this.closeChannel(addr, channel);
                throw new RemotingConnectException(addr);
            }
        }
        ...
    }

由上述代码可知，可以在下图中加入Channel来表示出Broker和NameServer之间的一个网络连接。然后通过这个Channel，Broker就可以发送实际的网络请求给NameServer了。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/e1f46a52-18d8-437a-bd70-590359a83f79" />

<br>

**(3)Broker如何与NameServer建立网络连接**

接下来进入this.getAndCreateChannel(addr)这行代码，看看Broker是如何与NameServer建立网络连接的。具体会先从缓存里尝试获取连接，如果没有缓存就创建一个连接。

    public class NettyRemotingClient extends NettyRemotingAbstract implements RemotingClient {
        ...
        private Channel getAndCreateChannel(final String addr) throws RemotingConnectException, InterruptedException {
            if (null == addr) {
                return getAndCreateNameserverChannel();
            }
            ChannelWrapper cw = this.channelTables.get(addr);
            if (cw != null && cw.isOK()) {
                return cw.getChannel();
            }
            return this.createChannel(addr);
        }
        ...
    }

接着看this.createChannel(addr)方法是如何通过一个NameServer的地址创建出一个网络连接的。

    public class NettyRemotingClient extends NettyRemotingAbstract implements RemotingClient {
        ...
        private Channel createChannel(final String addr) throws InterruptedException {
            //下面这一行就是在尝试获取缓存里的连接，如果有缓存就返回连接
            ChannelWrapper cw = this.channelTables.get(addr);
            if (cw != null && cw.isOK()) {
                return cw.getChannel();
            }

            if (this.lockChannelTables.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
                try {
                    //下面的十几行代码也是在获取缓存里的连接
                    boolean createNewConnection;
                    cw = this.channelTables.get(addr);
                    if (cw != null) {
                        if (cw.isOK()) {
                            return cw.getChannel();
                        } else if (!cw.getChannelFuture().isDone()) {
                            createNewConnection = false;
                        } else {
                            this.channelTables.remove(addr);
                            createNewConnection = true;
                        }
                    } else {
                        createNewConnection = true;
                    }

                    if (createNewConnection) {
                        //这里才是真正创建连接的地方，会基于Netty的Bootstrap这个类的connect()方法来构建出一个真正的Channel网络连接
                        ChannelFuture channelFuture = this.bootstrap.connect(RemotingHelper.string2SocketAddress(addr));
                        log.info("createChannel: begin to connect remote host[{}] asynchronously", addr);
                        cw = new ChannelWrapper(channelFuture);
                        this.channelTables.put(addr, cw);
                    }
                } catch (Exception e) {
                    log.error("createChannel: create channel exception", e);
                } finally {
                    this.lockChannelTables.unlock();
                }
            } else {
                log.warn("createChannel: try to lock channel table, but timeout, {}ms", LOCK_TIMEOUT_MILLIS);
            }

            //下面的代码就是把Channel连接返回回去
            if (cw != null) {
                ChannelFuture channelFuture = cw.getChannelFuture();
                if (channelFuture.awaitUninterruptibly(this.nettyClientConfig.getConnectTimeoutMillis())) {
                    if (cw.isOK()) {
                        log.info("createChannel: connect remote host[{}] success, {}", addr, channelFuture.toString());
                        return cw.getChannel();
                    } else {
                        log.warn("createChannel: connect remote host[" + addr + "] failed, " + channelFuture.toString(), channelFuture.cause());
                    }
                } else {
                    log.warn("createChannel: connect remote host[{}] timeout {}ms, {}", addr, this.nettyClientConfig.getConnectTimeoutMillis(), channelFuture.toString());
                }
            }
            return null;
        }
        ...
    }

<br>

**(4)如何通过Channel网络连接发送请求**

通过Channel网络连接发送各种请求的入口其实就是NettyRemotingClient类中的invokeSync()方法里面的代码：

    RemotingCommand response = this.invokeSyncImpl(channel, request, timeoutMillis - costTime);

进入这个invokeSyncImpl()方法，如下所示：

    public abstract class NettyRemotingAbstract {
        ...
        public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request, final long timeoutMillis) throws InterruptedException, RemotingSendRequestException, RemotingTimeoutException {
            final int opaque = request.getOpaque();
            try {
                final ResponseFuture responseFuture = new ResponseFuture(channel, opaque, timeoutMillis, null, null);
                this.responseTable.put(opaque, responseFuture);
                final SocketAddress addr = channel.remoteAddress();
                //基于Netty来开发，核心就是基于Channel把请求写出去
                channel.writeAndFlush(request).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture f) throws Exception {
                        if (f.isSuccess()) {
                            responseFuture.setSendRequestOK(true);
                            return;
                        } else {
                            responseFuture.setSendRequestOK(false);
                        }
                        responseTable.remove(opaque);
                        responseFuture.setCause(f.cause());
                        responseFuture.putResponse(null);
                        log.warn("send a request command to channel <" + addr + "> failed.");
                    }
                });
                //下面这行代码就在等待请求的响应结果回来
                RemotingCommand responseCommand = responseFuture.waitResponse(timeoutMillis);
                if (null == responseCommand) {
                    if (responseFuture.isSendRequestOK()) {
                        throw new RemotingTimeoutException(RemotingHelper.parseSocketAddressAddr(addr), timeoutMillis, responseFuture.getCause());
                    } else {
                        throw new RemotingSendRequestException(RemotingHelper.parseSocketAddressAddr(addr), responseFuture.getCause());
                    }
                }
                return responseCommand;
            } finally {
                this.responseTable.remove(opaque);
            }
        }
        ...
    }

上述代码的重点，就是最终会基于Netty的Channel API，把注册请求发送给NameServer。

<br>

**11.NameServer如何处理Broker的注册请求**

前面介绍完了Broker启动时是如何通过BrokerOuterAPI发送注册请求到NameServer去的，如下图示：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/c8d241fc-9b7e-4a5b-a6a4-bf22f58c2618" />

接下来介绍NameServer接收到这个注册请求后是如何进行处理的，这里会涉及Netty网络通信相关的内容。

现在回到NamesrvController这个类的初始化方法，也就是NamesrvController的initialize()方法，其中的一个源码片段如下所示：

    public class NamesrvController {
        ...
        public boolean initialize() {
            //加载kv之类的配置
            this.kvConfigManager.load();
            //初始化Netty服务器
            this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
            //Netty服务器的工作线程池
            this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
            //下面这行代码会把工作线程池交给Netty服务器
            this.registerProcessor();
            ...
        }
        ...
    }

继续看registerProcessor()方法的源码：

    public class NamesrvController {
        ...
        private void registerProcessor() {
            if (namesrvConfig.isClusterTest()) {
                //这里是用于处理测试集群的
                this.remotingServer.registerDefaultProcessor(new ClusterTestRequestProcessor(this, namesrvConfig.getProductEnvName()), this.remotingExecutor);
            } else {
                //这里会把NameServer的默认请求处理组件注册了进NettyServer中，所以NettyServer接收到的网络请求，都会交给这个组件来处理
                this.remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor);
            }
        }
        ...
    }

由上述源码可知，下图NameServer中的NettyServer会用于接收网络请求，然后交给DefaultRequestProcessor这个请求处理组件来进行处理。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/991b0e18-24ce-409d-9ac9-960541641313" />

所以如果想要知道Broker的注册请求是如何被NameServer进行处理的，直接看DefaultRequestProcessor中的代码即可。下面是DefaultRequestProcessor这个类的一些源码：

    public class DefaultRequestProcessor extends AsyncNettyRequestProcessor implements NettyRequestProcessor {
        ...
        public RemotingCommand processRequest(ChannelHandlerContext ctx,
            RemotingCommand request) throws RemotingCommandException {
            //打印调试日志
            if (ctx != null) {
                log.debug("receive request, {} {} {}", request.getCode(), RemotingHelper.parseChannelRemoteAddr(ctx.channel()), request);
            }

            //根据请求类型的不同进行不同的处理
            switch (request.getCode()) {
                case RequestCode.PUT_KV_CONFIG:
                    return this.putKVConfig(ctx, request);
                case RequestCode.GET_KV_CONFIG:
                    return this.getKVConfig(ctx, request);
                case RequestCode.DELETE_KV_CONFIG:
                    return this.deleteKVConfig(ctx, request);
                case RequestCode.QUERY_DATA_VERSION:
                    return queryBrokerTopicConfig(ctx, request);
                case RequestCode.REGISTER_BROKER:
                    //这里就是处理Broker注册的请求
                    Version brokerVersion = MQVersion.value2Version(request.getVersion());
                    if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                        return this.registerBrokerWithFilterServer(ctx, request);
                    } else {
                        //Broker注册的请求处理逻辑，就在下面的registerBroker()方法里
                        return this.registerBroker(ctx, request);
                    }
                 ...
            }
            return null;
        }
        ...
    }

接着进入DefaultRequestProcessor这个类的registerBroker()方法，去看如何完成Broker注册。

    public class DefaultRequestProcessor extends AsyncNettyRequestProcessor implements NettyRequestProcessor {
        ...
        public RemotingCommand registerBroker(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
            final RemotingCommand response = RemotingCommand.createResponseCommand(RegisterBrokerResponseHeader.class);
            //下面的代码就是解析注册请求，然后构造返回响应
            final RegisterBrokerResponseHeader responseHeader = (RegisterBrokerResponseHeader) response.readCustomHeader();
            final RegisterBrokerRequestHeader requestHeader = (RegisterBrokerRequestHeader) request.decodeCommandCustomHeader(RegisterBrokerRequestHeader.class);

            if (!checksum(ctx, request, requestHeader)) {
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark("crc32 not match");
                return response;
            }

            TopicConfigSerializeWrapper topicConfigWrapper;
            if (request.getBody() != null) {
                topicConfigWrapper = TopicConfigSerializeWrapper.decode(request.getBody(), TopicConfigSerializeWrapper.class);
            } else {
                topicConfigWrapper = new TopicConfigSerializeWrapper();
                topicConfigWrapper.getDataVersion().setCounter(new AtomicLong(0));
                topicConfigWrapper.getDataVersion().setTimestamp(0);
            }
        
            //核心在这里，这里会调用RouteInfoManager这个核心功能组件的注册Broker方法
            //RouteInfoManager就是路由信息管理组件，它是一个功能组件
            RegisterBrokerResult result = this.namesrvController.getRouteInfoManager().registerBroker(
                requestHeader.getClusterName(),
                requestHeader.getBrokerAddr(),
                requestHeader.getBrokerName(),
                requestHeader.getBrokerId(),
                requestHeader.getHaServerAddr(),
                topicConfigWrapper,
                null,
                ctx.channel()
            );
          
            //下面在构造返回的响应
            responseHeader.setHaServerAddr(result.getHaServerAddr());
            responseHeader.setMasterAddr(result.getMasterAddr());

            byte[] jsonValue = this.namesrvController.getKvConfigManager().getKVListByNamespace(NamesrvUtil.NAMESPACE_ORDER_TOPIC_CONFIG);
            response.setBody(jsonValue);
            response.setCode(ResponseCode.SUCCESS);
            response.setRemark(null);
            return response;
        }
        ...
    }

根据上述代码，在下图中添加RouteInfoManager这个路由数据管理组件，实际上Broker注册就是通过它来实现的。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/57d35884-45b9-4b8f-8bdd-d939e9f10971" />

而RouteInfoManager的注册Broker的方法，会把一个Broker机器的数据放入RouteInfoManager中维护的路由数据表里。其实现思路就是用一些Map类的数据结构，去存放Broker的核心路由数据：ClusterName、BrokerId、BrokerName等。而且在更新时，会基于Java并发包下的ReadWriteLock进行读写锁加锁，因为在这里更新那么多的内存Map数据结构，必须要加一个写锁，此时只能有一个线程来更新它们才行。

<br>

**16.Broker如何发送定时心跳的以及故障感知**

**(1)NameServer处理Broker注册请求的原理**

**(2)Broker如何定时发送心跳到NameServer**

<br>

**(1)NameServer处理Broker注册请求的原理**

NameServer核心就是基于Netty服务器来接收Broker注册请求，然后交给DefaultRequestProcessor这个请求处理组件来处理Broker注册请求。而真正的Broker注册逻辑是放在RouteInfoManager这个路由数据管理组件里来进行实现的，最终Broker路由数据都会存放在RouteInfoManager内部的一些Map数据结构组成的路由数据表中。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/fd4c39e4-3730-452c-a205-c9953354aa20" />

<br>

**(2)Broker如何定时发送心跳到NameServer**

接下来介绍Broker是如何定时发送心跳到NameServer让NameServer感知Broker一直都存活的，以及如果Broker一段时间没有发送心跳到NameServer，那么NameServer是如何感知到Broker已经挂掉了的。

首先看一下Broker中的发送注册请求给NameServer的一个源码入口，这其实就在BrokerController的start()方法中。BrokerController启动时并不仅仅会发送一次注册请求，还会启动一个定时任务：每隔一段时间就发送一次注册请求。

    public class BrokerController {
        ...
        public void start() throws Exception {
            //启动消息存储组件
            if (this.messageStore != null) { this.messageStore.start(); }
            //启动Netty服务器，这样就可以接收请求了
            if (this.remotingServer != null) { this.remotingServer.start(); }
            if (this.fastRemotingServer != null) { this.fastRemotingServer.start(); }
            //启动和文件相关的服务组件fileWatchService
            if (this.fileWatchService != null) { this.fileWatchService.start(); }
            //brokerOuterAPI是核心组件，该组件可以让Broker通过Netty客户端发送请求出去的
            //比如Broker发送请求到NameServer去注册以及进行心跳就是通过这个组件实现的
            if (this.brokerOuterAPI != null) { this.brokerOuterAPI.start(); }
            if (this.pullRequestHoldService != null) { this.pullRequestHoldService.start(); }
            if (this.clientHousekeepingService != null) { this.clientHousekeepingService.start(); }
            if (this.filterServerManager != null) { this.filterServerManager.start(); }
            if (!messageStoreConfig.isEnableDLegerCommitLog()) {
                startProcessorByHa(messageStoreConfig.getBrokerRole());
                handleSlaveSynchronize(messageStoreConfig.getBrokerRole());
                this.registerBrokerAll(true, false, true);
            }

            //这里往线程池里提交了一个任务，让它去NameServer进行注册
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    try {
                        BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                    } catch (Throwable e) {
                        log.error("registerBrokerAll Exception", e);
                    }
                }
            }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

            //下面是一些功能组件的启动
            if (this.brokerStatsManager != null) { this.brokerStatsManager.start(); }
            if (this.brokerFastFailure != null) { this.brokerFastFailure.start(); }
        }
        ...
    }

上面的代码会启动一个定时调度任务，默认是每隔30s执行一次Broker注册的过程。

所以默认情况下，第一次发送注册请求就是在进行注册，此时会把Broker路由数据放入到NameServer的RouteInfoManager的路由数据表里去。但是后续每隔30s都会发送一次注册请求，这些后续定时发送的注册请求，其实本质上就是Broker发送心跳给NameServer。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/b9ca225a-a79c-428d-bf1d-c1be57dc5a5c" />

那么后续每隔30s，Broker就会发送一次注册请求，作为心跳来发送给NameServer时，NameServer对后续重复发送过来的注册请求(也就是心跳)是如何进行处理的呢？接下来看RouteInfoManager的注册Broker的源码：

    public class RouteInfoManager {
        ...
        public RegisterBrokerResult registerBroker(final String clusterName, final String brokerAddr, final String brokerName, final long brokerId, final String haServerAddr, final TopicConfigSerializeWrapper topicConfigWrapper, final List<String> filterServerList, final Channel channel) {
            RegisterBrokerResult result = new RegisterBrokerResult();
            try {
                try {
                    //这里加写锁，同一时间只能一个线程来执行
                    this.lock.writeLock().lockInterruptibly();
                    //下面根据clusterAddrTable获取一个set集合：这就是在维护一个集群里有哪些Broker存在的一个set数据结构
                    Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
                    if (null == brokerNames) {
                        brokerNames = new HashSet<String>();
                        this.clusterAddrTable.put(clusterName, brokerNames);
                    }
                    //然后直接把brokerName放入这个set集合里
                    //如果Broker在注册后每隔30s发送注册请求作为心跳，这里是没影响的
                    //因为同样一个brokerName反复发送，这里的set集合会自动去重
                    brokerNames.add(brokerName);

                    boolean registerFirst = false;

                    //这里根据brokerName获取到BrokerData
                    //然后用一个brokerAddrTable作为核心路由数据表，存放了所有的Broker的详细的路由数据
                    BrokerData brokerData = this.brokerAddrTable.get(brokerName);

                    //以下几行是核心的处理逻辑：
                    //如果Broker是第一次发送注册请求过来，这里的brokerData就是null
                    //于是就会封装一个BrokerData，并放入到这个路由数据表里————这也就是Broker的注册过程
                    //如果Broker注册后每隔30s发送注册请求作为心跳，这里是没影响的
                    //因为当Broker重复发送注册请求时，这个BrokerData已经存在了，不会重复进行处理
                    if (null == brokerData) {
                        registerFirst = true;
                        brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
                        this.brokerAddrTable.put(brokerName, brokerData);
                    }
                
                    //下面会对路由数据做一些处理
                    Map<Long, String> brokerAddrsMap = brokerData.getBrokerAddrs();
                    //Switch slave to master: first remove <1, IP:PORT> in namesrv, then add <0, IP:PORT>
                    //The same IP:PORT must only have one record in brokerAddrTable
                    Iterator<Entry<Long, String>> it = brokerAddrsMap.entrySet().iterator();
                    while (it.hasNext()) {
                        Entry<Long, String> item = it.next();
                        if (null != brokerAddr && brokerAddr.equals(item.getValue()) && brokerId != item.getKey()) {
                            it.remove();
                        }
                    }

                    String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
                    registerFirst = registerFirst || (null == oldAddr);

                    if (null != topicConfigWrapper && MixAll.MASTER_ID == brokerId) {
                        if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion()) || registerFirst) {
                            ConcurrentMap<String, TopicConfig> tcTable = topicConfigWrapper.getTopicConfigTable();
                            if (tcTable != null) {
                                for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                                    this.createAndUpdateQueueData(brokerName, entry.getValue());
                                }
                            }
                        }
                    }

                    //以下几行是核心的处理逻辑
                    //当Broker每隔30s发送注册请求作为心跳到这里时，这里就会封装一个新的BrokerLiveInfo放入brokerLiveTable这个Map
                    //所以每隔30s，最新的BrokerLiveInfo都会覆盖上一次的BrokerLiveInfo
                    //而这个BrokerLiveInfo里就有一个当前时间戳，代表着Broker最近的一次心跳时间
                    //这也就是Broker每隔30s发送注册请求作为心跳的处理逻辑
                    BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr, new BrokerLiveInfo(System.currentTimeMillis(), topicConfigWrapper.getDataVersion(), channel, haServerAddr));
                    if (null == prevBrokerLiveInfo) {
                        log.info("new broker registered, {} HAServer: {}", brokerAddr, haServerAddr);
                    }

                    //下面的代码可先不管
                    if (filterServerList != null) {
                        if (filterServerList.isEmpty()) {
                            this.filterServerTable.remove(brokerAddr);
                        } else {
                            this.filterServerTable.put(brokerAddr, filterServerList);
                        }
                    }

                    if (MixAll.MASTER_ID != brokerId) {
                        String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                        if (masterAddr != null) {
                            BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                            if (brokerLiveInfo != null) {
                                result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                                result.setMasterAddr(masterAddr);
                            }
                        }
                    }
                } finally {
                    this.lock.writeLock().unlock();
                }
            } catch (Exception e) {
                log.error("registerBroker Exception", e);
            }
            return result;
        }
        ...
    }

如下图示，当Broker每隔30s发送一个注册请求作为心跳时，RouteInfoManager路由数据管理组件就会进行心跳时间的刷新处理。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/8e9459f5-77a3-43e7-af68-7abbaf1b7a99" />

假设Broker已经挂了或者故障了，隔了很久都没有发送每隔30s一次的注册请求，那么此时NameServer是如何感知Broker已经挂掉呢？

回到NamesrvController的initialize()方法里，里面有一处代码就是启动RouteInfoManager中的一个定时扫描不活跃Broker的任务的。

    public class NamesrvController {
        ...
        public boolean initialize() {
            ...
            //下面这行代码就是启动一个后台线程，执行定时任务
            //从scanNotActiveBroker()可知这里会定时扫描哪些Broker没发送心跳而挂掉的
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {
                @Override
                public void run() {
                    NamesrvController.this.routeInfoManager.scanNotActiveBroker();
                }
            }, 5, 10, TimeUnit.SECONDS);
            ...
        }
        ...
    }

上面这段代码会启动一个定时调度线程，每隔10s扫描一次目前不活跃的Broker。其中便调用了RouteInfoManager的scanNotActiveBroke()方法，该方法会感知一个Broker是否挂掉。

RouteInfoManager.scanNotActiveBroker()方法的源码如下：

    public class RouteInfoManager {
        ...
        public void scanNotActiveBroker() {
            //这里的代码会扫描存储着BrokerLiveInfo心跳数据的brokerLiveTable这个Map
            //通过遍历拿到每个Broker最近一次心跳刷新的BrokerLiveInfo，从而知道一个Broker最近一次发送心跳的时间
            Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
            while (it.hasNext()) {
                Entry<String, BrokerLiveInfo> next = it.next();
                long last = next.getValue().getLastUpdateTimestamp();
                //核心判断逻辑如下：
                //如果当前时间距离上一次心跳时间，超过了Broker心跳超时时间，默认是120s
                //也就是如果一个Broker两分钟没发送心跳，就认为它已经死掉了
                if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
                    RemotingUtil.closeChannel(next.getValue().getChannel());
                    it.remove();
                    log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
                    //此时就会把这个Broker从路由数据表里剔除出去
                    this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
                }
            }
        }
        ...
    }

