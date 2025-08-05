# Dubbo源码—2.服务引用时创建代理的流程

**大纲**

**1.Dubbo服务引用的主流程**

**2.服务引用时创建动态代理的流程**

**3.DynamicDirectory与Invoker的关系**

**4.基于Invoker来创建动态代理对象**

**5.动态代理生成后检查Invoker是否有效**

<br>

**1.Dubbo服务引用的主流程**

ReferenceConfig的get()方法在进行服务引用时，首先也会初始化相关的组件，然后再创建并初始化服务的动态代理对象，最后返回服务的动态代理对象，其中会通过ReferenceConfig的createProxy()方法创建服务的动态代理对象。

    public class Application {
        public static void main(String[] args) throws Exception {
            //Reference和ReferenceConfig是什么
            //Reference是一个引用，是对Provider端的一个服务实例的引用
            //ReferenceConfig这个服务实例的引用的一些配置
            //通过泛型传递了这个服务实例对外暴露的接口
            ReferenceConfig<DemoService> reference = new ReferenceConfig<>();

            //设置应用名称
            reference.setApplication(new ApplicationConfig("dubbo-demo-api-consumer"));

            //设置注册中心的地址，默认是ZooKeeper
            reference.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));

            //设置元数据上报地址
            reference.setMetadataReportConfig(new MetadataReportConfig("zookeeper://127.0.0.1:2181"));

            //设置要调用的服务的接口
            reference.setInterface(DemoService.class);

            //直接通过ReferenceConfig的get()方法来拿到一个DemoService接口
            //它是一个动态代理接口，只要被调用，便会通过底层调用Provider服务实例的对应接口
            DemoService service = reference.get();

            //下面调用动态代理的接口
            //会进入InvokerInvocationHandler的invoke()方法中
            String message = service.sayHello("dubbo");

            System.out.println(message);
            Thread.sleep(10000000L);
        }
    }

    public class ReferenceConfig<T> extends ReferenceConfigBase<T> {
        //ref指向了服务的动态代理对象
        private transient volatile T ref;
        ...

        @Override
        public T get() {
            //检测当前ReferenceConfig的状态
            if (destroyed) {
                throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
            }

            //ref指向了服务的动态代理对象
            if (ref == null) {
                //1.执行相关组件的初始化
                //通过获取到的ModuleDeployer来对相关组件进行初始化
                //比如会对MetadataReport元数据上报组件进行构建和初始化，以及启动(建立跟zk的连接)
                getScopeModel().getDeployer().start();
                synchronized (this) {
                    if (ref == null) {
                        //2.初始化服务的动态代理对象ref
                        init();
                    }
                }
            }

            //3.返回服务的动态代理对象
            return ref;
        }

        protected synchronized void init() {
            ...
            //执行服务实例的刷新操作
            //也就是刷新ProviderConfig -> MethodConfig -> ArgumentConfig
            if (!this.isRefreshed()) {
                this.refresh();
            }

            //init serviceMetadata
            //对Metadata元数据进行初始化以及存储注册等一系列的操作
            //封装一个ConsumerModel，即把Consumer的数据封装起来放到对应的repository里
            initServiceMetadata(consumer);
            ...
            
            //调用ReferenceConfig的createProxy()方法创建服务的动态代理对象
            //这是构建整个代理的入口
            ref = createProxy(referenceParameters);

            //创建完动态代理后，把这个代理设置给serviceMetadata以及consumerModel
            serviceMetadata.setTarget(ref);
            serviceMetadata.addAttribute(PROXY_CLASS_REF, ref);

            consumerModel.setDestroyRunner(getDestroyRunner());
            consumerModel.setProxyObject(ref);
            consumerModel.initMethodModels();

            //检查一下Invoker是否可用
            checkInvokerAvailable();
        }
        ...
    }

<br>

**2.服务引用时创建动态代理的流程**

ReferenceConfig的createProxy()方法创建动态代理创建时，首先会通过parseUrl()方法等方法进行地址处理，然后才通过createInvokerForRemote()方法创建动态代理。

而createInvokerForRemote()方法首先会通过RegistryProtocol的refer()方法构建出一个Invoker，然后再通过ProxyFactory适配器选择合适的ProxyFactory扩展实现，最后通过该ProxyFactory扩展实现 + 该Invoker创建出动态代理对象。

    //-> ReferenceConfig.createProxy()
    //-> ReferenceConfig.parseUrl() + ReferenceConfig.aggregateUrlFromRegistry()
    //-> ReferenceConfig.createInvokerForRemote()
    //-> Protocol$Adaptive.refer()
    //-> ProtocolSerializationWrapper.refer()
    //-> ProtocolFilterWrapper.refer()
    //-> ProtocolListenerWrapper.refer()
    //-> RegistryProtocol.refer() => invoker
    //-> proxyFactory.getProxy(invoker, ProtocolUtils.isGeneric(generic))
    //-> ProxyFactory$Adaptive.getProxy()
    //-> StubProxyFactoryWrapper.getProxy()
    //-> AbstractProxyFactory.getProxy()
    //-> JavassistProxyFactory.getProxy()
    //-> Proxy.getProxy()

    public class ReferenceConfig<T> extends ReferenceConfigBase<T> {
        //The invoker of the reference service
        private transient volatile Invoker<?> invoker;
        ...

        private T createProxy(Map<String, String> referenceParameters) {
            //Provider端有一个本地发布的动作，本地发布即把目标实现类封装成一个ProxyInvoker
            //然后再通过InjvmProtocol发布出去，也就是把ProxyInvoker封装成InjvmExporter
            //所以在同一个JVM里进行本地服务调用时，调用的便是本地发布出去的ProxyInvoker

            //根据url的协议、scope以及injvm等参数检测是否需要本地引用
            if (shouldJvmRefer(referenceParameters)) {
                createInvokerForLocal(referenceParameters);
            } else {
                urls.clear();
                meshModeHandleUrl(referenceParameters);
                if (StringUtils.isNotEmpty(url)) {
                    //user specified URL, could be peer-to-peer address, or register center's address.
                    //解析用户指定的URL，可能是一个点对点的调用地址，或者是一个注册中心的地址
                    parseUrl(referenceParameters);
                } else {
                    //if protocols not in jvm checkRegistry
                    if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                        //加载注册中心的地址RegistryURL
                        aggregateUrlFromRegistry(referenceParameters);
                    }
                }
                //接下来是核心源码的第一步：
                //通过RegistryProtocol创建invoker
                createInvokerForRemote();
            }
            ...

            //这些都是url的处理
            URL consumerUrl = new ServiceConfigURL(CONSUMER_PROTOCOL, referenceParameters.get(REGISTER_IP_KEY), 0, referenceParameters.get(INTERFACE_KEY), referenceParameters);
            consumerUrl = consumerUrl.setScopeModel(getScopeModel());
            consumerUrl = consumerUrl.setServiceModel(consumerModel);
            //对于Consumer的元数据，会通过如下代码在这里进行发布
            MetadataUtils.publishServiceDefinition(consumerUrl, consumerModel.getServiceModel(), getApplicationModel());

            //create service proxy
            //通过ProxyFactory适配器选择合适的ProxyFactory扩展实现，基于invoker对象创建动态代理对象
            return (T) proxyFactory.getProxy(invoker, ProtocolUtils.isGeneric(generic));
        }
        ...

        private void createInvokerForRemote() {
            //createInvokerForRemote()方法的核心就两个步骤：
            //第一是执行RegistryProtocol.refer()构建出一个invoker；
            //第二是执行Cluster.getCluster().join() + new StaticDirectory()对invokers进行加工处理，最后覆盖到invoker上；
            //protocolSPI.refer -> invokers -> new StaticDirectory() -> cluster.join -> invoker
            if (urls.size() == 1) {
                //注册中心单Provider或直连单个服务提供方时，通过Protocol的适配器选择对应的Protocol实现创建Invoker对象
                //也就是会基于RegistryProtocol.refer构建出一个Invoker
                //比如dubbo-demo-api下的示例会构建出一个MigrationInvoker，且curUrl不是注册中心的
                //而且该MigrationInvoker的构成是：
                //其registry属性为装饰ZookeeperRegistry的ListenerRegistryWrapper
                //其cluster属性为装饰FailoverCluster的MockCLusterWrapper
                URL curUrl = urls.get(0);

                //调用Protocol$Adaptive的refer()方法
                invoker = protocolSPI.refer(interfaceClass, curUrl);

                //registry url, mesh-enable and unloadClusterRelated is true, not need Cluster.
                if (!UrlUtils.isRegistry(curUrl) && !curUrl.getParameter(UNLOAD_CLUSTER_RELATED, false)) {
                    List<Invoker<?>> invokers = new ArrayList<>();
                    invokers.add(invoker);
                    //对Invoker进行深加工和处理后又拿到了一个Invoker，覆盖原invoker
                    invoker = Cluster.getCluster(scopeModel, Cluster.DEFAULT).join(new StaticDirectory(curUrl, invokers), true);
                }
            } else {
                //注册中心多Provider或直连多个服务提供方时，根据每个URL创建Invoker对象
                List<Invoker<?>> invokers = new ArrayList<>();
                URL registryUrl = null;
                for (URL url : urls) {
                    //For multi-registry scenarios, it is not checked whether each referInvoker is available.
                    //Because this invoker may become available later.
                    invokers.add(protocolSPI.refer(interfaceClass, url));

                    if (UrlUtils.isRegistry(url)) {
                        //use last registry url
                        //确定是多注册中心，还是直连多个Provider
                        registryUrl = url;
                    }
                }

                if (registryUrl != null) {
                    //在依赖注册中心的场景中，则使用ZoneAwareCluster作为Cluster的默认实现，生成对应的Invoker对象
                    //registry url is available
                    //for multi-subscription scenario, use 'zone-aware' policy by default
                    String cluster = registryUrl.getParameter(CLUSTER_KEY, ZoneAwareCluster.NAME);
                    //The invoker wrap sequence would be: ZoneAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker
                    //(RegistryDirectory, routing happens here) -> Invoker
                    invoker = Cluster.getCluster(registryUrl.getScopeModel(), cluster, false).join(new StaticDirectory(registryUrl, invokers), false);
                } else {
                    //在直连Provider的场景中，则使用Cluster适配器选择合适的扩展实现
                    //not a registry url, must be direct invoke.
                    if (CollectionUtils.isEmpty(invokers)) {
                        throw new IllegalArgumentException("invokers == null");
                    }
                    URL curUrl = invokers.get(0).getUrl();
                    String cluster = curUrl.getParameter(CLUSTER_KEY, Cluster.DEFAULT);
                    invoker = Cluster.getCluster(scopeModel, cluster).join(new StaticDirectory(curUrl, invokers), true);
                }
            }
        }
        ...
    }

    @SPI(value = "dubbo", scope = ExtensionScope.FRAMEWORK)
    public interface Protocol {
        //默认端口
        int getDefaultPort();

        //Protocol接收到一个请求之后，必须要记录请求的源地址
        //对同一个服务实例(url)发布一次和发布多次，是没有任何区别的
        //export()方法会将一个Invoker发布出去
        //export()方法的实现需要是幂等的，即同一个服务暴露多次和暴露一次的效果是相同的
        @Adaptive
        <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

        //Protocol必须根据一个url和接口类型，获取到对应的Invoker
        //refer()方法会引用一个Invoker
        //refer()方法会根据参数返回一个Invoker对象，Consumer端可以通过这个Invoker请求到Provider端的服务
        @Adaptive
        <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

        //销毁export()方法以及refer()方法使用到的Invoker对象，释放当前Protocol对象底层占用的资源
        void destroy();

        //返回当前Protocol底层的全部ProtocolServer
        default List<ProtocolServer> getServers() {
            return Collections.emptyList();
        }
    }

    @SPI(value = "javassist", scope = FRAMEWORK)
    public interface ProxyFactory {
        //为传入的Invoker对象创建代理对象
        @Adaptive({PROXY_KEY})
        <T> T getProxy(Invoker<T> invoker) throws RpcException;

        //为传入的Invoker对象创建代理对象
        @Adaptive({PROXY_KEY})
        <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;

        //将传入的代理对象封装成Invoker对象
        @Adaptive({PROXY_KEY})
        <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;
    }

<!---->

    //-> Protocol$Adaptive.refer()
    //-> ProtocolSerializationWrapper.refer()
    //-> ProtocolFilterWrapper.refer()
    //-> ProtocolListenerWrapper.refer()
    //-> RegistryProtocol.refer() => invoker

    public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
        public int getDefaultPort()  {
            throw new UnsupportedOperationException("The method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
        }

        public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
            if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
            if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
            org.apache.dubbo.common.URL url = arg0.getUrl();
            String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
            if (extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
            ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), org.apache.dubbo.rpc.Protocol.class);
            org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)scopeModel.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
            return extension.export(arg0);
        }

        public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
            if (arg1 == null) throw new IllegalArgumentException("url == null");
            org.apache.dubbo.common.URL url = arg1;
            String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
            if (extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (" + url.toString() + ") use keys([protocol])");
            ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), org.apache.dubbo.rpc.Protocol.class);
            org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)scopeModel.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
            return extension.refer(arg0, arg1);
        }

        public void destroy()  {
            throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
        }

        public java.util.List getServers()  {
            throw new UnsupportedOperationException("The method public default java.util.List org.apache.dubbo.rpc.Protocol.getServers() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
        }
    }

    @Activate
    public class ProtocolSerializationWrapper implements Protocol {
        private Protocol protocol;

        public ProtocolSerializationWrapper(Protocol protocol) {
            this.protocol = protocol;
        }

        @Override
        public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
            getFrameworkModel(invoker.getUrl().getScopeModel()).getServiceRepository().registerProviderUrl(invoker.getUrl());
            //下面会调用ProtocolFilterWrapper.export()方法
            return protocol.export(invoker);
        }

        @Override
        public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            //下面会调用ProtocolFilterWrapper.refer()方法
            return protocol.refer(type, url);
        }
        ...
    }

    @Activate(order = 100)
    public class ProtocolFilterWrapper implements Protocol {
        private final Protocol protocol;

        public ProtocolFilterWrapper(Protocol protocol) {
            if (protocol == null) {
                throw new IllegalArgumentException("protocol == null");
            }
            this.protocol = protocol;
        }

        @Override
        public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
            if (UrlUtils.isRegistry(invoker.getUrl())) {
                return protocol.export(invoker);
            }
            FilterChainBuilder builder = getFilterChainBuilder(invoker.getUrl());
            //下面会调用ProtocolListenerWrapper.export()方法
            return protocol.export(builder.buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
        }

        private <T> FilterChainBuilder getFilterChainBuilder(URL url) {
            return ScopeModelUtil.getExtensionLoader(FilterChainBuilder.class, url.getScopeModel()).getDefaultExtension();
        }

        @Override
        public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            if (UrlUtils.isRegistry(url)) {
                //下面会调用ProtocolListenerWrapper.refer()方法
                return protocol.refer(type, url);
            }
            FilterChainBuilder builder = getFilterChainBuilder(url);
            return builder.buildInvokerChain(protocol.refer(type, url), REFERENCE_FILTER_KEY, CommonConstants.CONSUMER);
        }
        ...
    }

    @Activate(order = 200)
    public class ProtocolListenerWrapper implements Protocol {
        private final Protocol protocol;

        public ProtocolListenerWrapper(Protocol protocol) {
            if (protocol == null) {
                throw new IllegalArgumentException("protocol == null");
            }
            this.protocol = protocol;
        }

        @Override
        public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
            if (UrlUtils.isRegistry(invoker.getUrl())) {
                return protocol.export(invoker);
            }
            //下面会调用DubboProtocol.export()方法
            return new ListenerExporterWrapper<T>(protocol.export(invoker), Collections.unmodifiableList(
                ScopeModelUtil.getExtensionLoader(ExporterListener.class, invoker.getUrl().getScopeModel()).getActivateExtension(invoker.getUrl(), EXPORTER_LISTENER_KEY))
            );
        }

        @Override
        public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            if (UrlUtils.isRegistry(url)) {
                //下面会调用RegistryProtocol.refer()方法
                return protocol.refer(type, url);
            }
            Invoker<T> invoker = protocol.refer(type, url);
            if (StringUtils.isEmpty(url.getParameter(REGISTRY_CLUSTER_TYPE_KEY))) {
                invoker = new ListenerInvokerWrapper<>(invoker, Collections.unmodifiableList(
                    ScopeModelUtil.getExtensionLoader(InvokerListener.class, invoker.getUrl().getScopeModel()).getActivateExtension(url, INVOKER_LISTENER_KEY)
                ));
            }
            return invoker;
        }
        ...
    }

<!---->

    //-> ProxyFactory$Adaptive.getProxy()
    //-> StubProxyFactoryWrapper.getProxy()
    //-> AbstractProxyFactory.getProxy()
    //-> JavassistProxyFactory.getProxy()
    //-> Proxy.getProxy

    public class ProxyFactory$Adaptive implements org.apache.dubbo.rpc.ProxyFactory {
        ...
        public java.lang.Object getProxy(org.apache.dubbo.rpc.Invoker arg0, boolean arg1) throws org.apache.dubbo.rpc.RpcException {
            if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
            if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
            org.apache.dubbo.common.URL url = arg0.getUrl();
            String extName = url.getParameter("proxy", "javassist");
            if (extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.rpc.ProxyFactory) name from url (" + url.toString() + ") use keys([proxy])");
            ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), org.apache.dubbo.rpc.ProxyFactory.class);
            org.apache.dubbo.rpc.ProxyFactory extension = (org.apache.dubbo.rpc.ProxyFactory)scopeModel.getExtensionLoader(org.apache.dubbo.rpc.ProxyFactory.class).getExtension(extName);
            return extension.getProxy(arg0, arg1);
        }
        ...
    }

    public class StubProxyFactoryWrapper implements ProxyFactory {
        //包含一个JavassistProxyFactory
        private final ProxyFactory proxyFactory;
        private Protocol protocol;

        public StubProxyFactoryWrapper(ProxyFactory proxyFactory) {
            this.proxyFactory = proxyFactory;
        }

        public void setProtocol(Protocol protocol) {
            this.protocol = protocol;
        }

        @Override
        public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
            //为什么它的名字叫stub打桩？
            //stub这个名字一般代表的是远程网络访问的动态代理
            //通过这个stub可以对远程的机器进行网络访问
            //stub打桩思想，是在网络远程访问里经常会使用的一种方法，类似于电线杆打桩一样
            //比如机器A和机器B之间进行的远程网络访问，必然要通过网络来进行连接和访问
            //机器A在进行远程RPC访问时，其调用代码，最好不要直接写网络通信的逻辑，而是写针对某个接口类型对象的方法调用
            //这样在机器A上面进行RPC调用时就像是在本地调用一样，都是直接调用某个对象的方法
            //然后在这个对象的方法里，再去写网络通信的逻辑和目标机器B进行网络连接，并发送调用请求
            //所以在机器A上会针对调用的那个接口，来动态去生成一个实现了该接口的类，也就是动态代理的代理类或者是stub打桩
            //类似于在机器A上打下的一个伪装成目标类的桩，然后针对打的这个stub桩去进行调用，stub内部会编写网络通信代码实现跨机器的访问
            //这就是远程网络连接和通信的stub打桩思想
            //下面使用了抽象代理工厂来获取真正的动态代理
            //下面会调用JavassistProxyFactory的getProxy()方法
            //也就是调用AbstractProxyFactory的getProxy()方法
            T proxy = proxyFactory.getProxy(invoker, generic);
            ...

            return proxy;
        }
        ...
    }

    public abstract class AbstractProxyFactory implements ProxyFactory {
        ...
        @Override
        public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
            //记录要代理的接口
            LinkedHashSet<Class<?>> interfaces = new LinkedHashSet<>();
            ClassLoader classLoader = getClassLoader(invoker);

            //获取URL中interfaces参数指定的接口
            String config = invoker.getUrl().getParameter(INTERFACES);
            if (StringUtils.isNotEmpty(config)) {
                //按照逗号切分interfaces参数，得到接口集合，然后进行遍历
                String[] types = COMMA_SPLIT_PATTERN.split(config);
                //遍历接口集合，并记录这些接口信息
                for (String type : types) {
                    //对每个接口都拿出对应的class对象，放到interfaces集合里
                    interfaces.add(ReflectUtils.forName(classLoader, type));
                }
            }
            ...

            //获取Invoker中type字段指定的接口
            interfaces.add(invoker.getInterface());

            //添加EchoService、Destroyable两个默认接口
            interfaces.addAll(Arrays.asList(INTERNAL_INTERFACES));

            //调用抽象的getProxy()重载方法
            //Dubbo的动态代理技术有如下两种：
            //一.javassist(动态拼接类的代码字符串，通过动态编译来动态生成一个类)
            //二.jdk(通过jdk提供的API利用反射来生成动态代理)
            //默认下面会调用子类JavassistProxyFactory的getProxy()方法
            return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
        }

        public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);
        ...
    }

    public class JavassistProxyFactory extends AbstractProxyFactory {
        private final static Logger logger = LoggerFactory.getLogger(JavassistProxyFactory.class);
        private final JdkProxyFactory jdkProxyFactory = new JdkProxyFactory();

        @Override
        @SuppressWarnings("unchecked")
        public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
            try {
                //下面的Proxy是Dubbo自己实现的Proxy
                return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
            } catch (Throwable fromJavassist) {
                //try fall back to JDK proxy factory
                try {
                    T proxy = jdkProxyFactory.getProxy(invoker, interfaces);
                    logger.error("Failed to generate proxy by Javassist failed. Fallback to use JDK proxy success. " + "Interfaces: " + Arrays.toString(interfaces), fromJavassist);
                    return proxy;
                } catch (Throwable fromJdk) {
                    logger.error("Failed to generate proxy by Javassist failed. Fallback to use JDK proxy is also failed. " + "Interfaces: " + Arrays.toString(interfaces) + " Javassist Error.", fromJavassist);
                    logger.error("Failed to generate proxy by Javassist failed. Fallback to use JDK proxy is also failed. " + "Interfaces: " + Arrays.toString(interfaces) + " JDK Error.", fromJdk);
                    throw fromJavassist;
                }
            }
        }
        ...
    }

    public class Proxy {
        private static final Map<ClassLoader, Map<String, Proxy>> PROXY_CACHE_MAP = new WeakHashMap<>();
        ...

        public static Proxy getProxy(Class<?>... ics) {
            //ClassLoader from App Interface should support load some class from Dubbo
            ClassLoader cl = ics[0].getClassLoader();
            ProtectionDomain domain = ics[0].getProtectionDomain();

            //use interface class name list as key.
            //生成的动态代理类主要实现了如下三个接口：
            //org.apache.dubbo.demo.DemoService
            //org.apache.dubbo.rpc.service.EchoService
            //org.apache.dubbo.rpc.service.Destoryable
            String key = buildInterfacesKey(cl, ics);

            //get cache by class loader.
            final Map<String, Proxy> cache;
            synchronized (PROXY_CACHE_MAP) {
                cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new ConcurrentHashMap<>());
            }

            //接口列表将会作为第二层集合的Key
            Proxy proxy = cache.get(key);
            if (proxy == null) {
                synchronized (ics[0]) {
                    proxy = cache.get(key);
                    if (proxy == null) {
                        //create Proxy class.
                        proxy = new Proxy(buildProxyClass(cl, ics, domain));
                        cache.put(key, proxy);
                    }
                }
            }
            return proxy;
        }
        ...
    }

在RegistryProtocol的refer()的方法中，首先会获取一个包含zk注册中心ZooKeeperRegistry的ListenerRegistryWrapper，然后再获取一个包含故障转移集群FailoverCluster的MockClusterWrapper。

    //-> RegistryProtocol.refer()
    //-> getRegistry() => ZooKeeperRegistry
    //-> Cluster.getCluster() => MockClusterWrapper、FailoverCluster
    //-> RegistryProtocol.doRefer()
    //-> RegistryProtocol.getMigrationInvoker()
    //-> RegistryProtocol.interceptInvoker()
    //-> MigrationRuleListener.onRefer()
    //-> MigrationRuleHandler.doMigrate()
    //-> MigrationRuleHandler.refreshInvoker()
    //-> MigrationInvoker.refreshInterfaceInvoker() + MigrationInvoker.refreshServiceDiscoveryInvoker()
    //-> InterfaceCompatibleRegistryProtocol.getInvoker() + RegistryProtocol.getServiceDiscoveryInvoker()
    //-> RegistryProtocol.doCreateInvoker()

    public class RegistryProtocol implements Protocol, ScopeModelAware {
        ...
        //下面会返回一个MigrationInvoker，其构成是：
        //registry属性为装饰ZookeeperRegistry的ListenerRegistryWrapper
        //cluster属性为装饰FailoverCluster的MockCLusterWrapper
        //invoker和serviceDiscoveryInvoker属性都为MockClusterInvoker
        //其中前者的directory属性=注入了ZookeeperRegistry的RegistryDirectory
        //后者的directory属性=注入了ServiceDiscoveryRegistry的ServiceDiscoveryRegistryDirectory
        //但两者的invoker属性都是AbstractCluster的内部类ClusterFilterInvoker
        @Override
        public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            //从URL中获取注册中心的URL
            url = getRegistryUrl(url);

            //获取Registry实例，这里的RegistryFactory对象是通过Dubbo SPI的自动装载机制注入的
            //1.先拿到一个ListenerRegistryWrapper，里面包含一个zk注册中心ZooKeeperRegistry
            Registry registry = getRegistry(url);
            if (RegistryService.class.equals(type)) {
                return proxyFactory.getInvoker((T) registry, type, url);
            }

            //group="a,b" or group="*"
            //从注册中心URL的refer参数中获取此次服务引用的一些参数，其中就包括group
            Map<String, String> qs = (Map<String, String>) url.getAttribute(REFER_KEY);
            String group = qs.get(GROUP_KEY);
            if (StringUtils.isNotEmpty(group)) {
                if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                    //如果此次可以引用多个group的服务，则Cluser实现使用MergeableCluster实现
                    //这里的getMergeableCluster()方法就会通过Dubbo SPI方式找到MergeableCluster实例
                    return doRefer(Cluster.getCluster(url.getScopeModel(), MergeableCluster.NAME), registry, type, url, qs);
                }
            }

            //2.接着会拿到一个MockClusterWrapper，里面会包含一个故障转移集群FailoverCluster
            //Cluster是和集群容错强相关的
            //不同的Cluster会对应不同的Cluster Invoker
            //不同的Cluster Invoker就有调用失败时不同的集群容错策略和算法
            Cluster cluster = Cluster.getCluster(url.getScopeModel(), qs.get(CLUSTER_KEY));

            //doRefer()这种代码的编写技巧在很多框架系统都会使用
            //refer()方法会先做一些提前的准备工作，而正式的工作可以在方法名称的前面加一个do变成doRefer()方法来执行真正的工作
            //下面的doRefer()方法，执行时发现如果没有group参数或是group参数，则通过Cluster适配器选择Cluster实现
            return doRefer(cluster, registry, type, url, qs);
        }

        protected Registry getRegistry(final URL registryUrl) {
            //通过SPI自适应机制，去拿到对应的extension实例
            //这里的registryFactory为RegistryFactory$Adaptive
            RegistryFactory registryFactory = ScopeModelUtil.getExtensionLoader(RegistryFactory.class, registryUrl.getScopeModel()).getAdaptiveExtension();

            //调用RegistryFactory$Adaptive的getRegistry()方法
            //返回一个装饰ZookeeperRegistry的ListenerRegistryWrapper
            return registryFactory.getRegistry(registryUrl);
        }
        ...
    }

    @SPI(Cluster.DEFAULT)
    public interface Cluster {
        String DEFAULT = "failover";

        //Merge the directory invokers to a virtual invoker.
        @Adaptive
        <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) throws RpcException;
        static Cluster getCluster(ScopeModel scopeModel, String name) {
            return getCluster(scopeModel, name, true);
        }

        static Cluster getCluster(ScopeModel scopeModel, String name, boolean wrap) {
            if (StringUtils.isEmpty(name)) {
                //name的默认值是failover
                name = Cluster.DEFAULT;
            }

            //比如RegistryProtocol.refer()方法的"Cluster.getCluster(url.getScopeModel(), qs.get(CLUSTER_KEY))"代码
            //就会获取到一个装饰着FailoverCluster的MockClusterWrapper
            return ScopeModelUtil.getApplicationModel(scopeModel).getExtensionLoader(Cluster.class).getExtension(name, wrap);
        }
    }

    public class MockClusterWrapper implements Cluster {
        //包含一个故障转移集群FailoverCluster
        private final Cluster cluster;

        public MockClusterWrapper(Cluster cluster) {
            //Wrapper类都会有一个拷贝构造函数，装饰器模式
            this.cluster = cluster;
        }
        ...
    }

    public class RegistryProtocol implements Protocol, ScopeModelAware {
        ...
        //doRefer()方法传入的参数cluster就是装饰着FailoverCluster的MockClusterWrapper
        //传入的参数registry为装饰ZookeeperRegistry的ListenerRegistryWrapper
        //这里会把组装好的Invoker体系里的最源头的一个Invoker进行返回，下一步就会去创建接口的动态代理并把该Invoker放进去
        //这样动态代理的方法被调用时，就可以通过该Invoker及其关联的Invoker来完成一个完整的RPC调用过程
        protected <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url, Map<String, String> parameters) {
            Map<String, Object> consumerAttribute = new HashMap<>(url.getAttributes());
            consumerAttribute.remove(REFER_KEY);
            String p = isEmpty(parameters.get(PROTOCOL_KEY)) ? CONSUMER : parameters.get(PROTOCOL_KEY);
            URL consumerUrl = new ServiceConfigURL(p, null, null, parameters.get(REGISTER_IP_KEY), 0, getPath(parameters, type), parameters, consumerAttribute);
            url = url.putAttribute(CONSUMER_URL_KEY, consumerUrl);

            //下面会以MigrationInvoker作为一个起点去构建一个Invoker链条；
            //这使用了经典的责任链模式，一环扣一环，先执行一个步骤再下一个步骤，把RPC调用的过程，一步步执行完毕；
            //但是在代码层面，却并非是严格的责任链模式来编码的，只是用到了这种责任链思想而已；
            //所以在这里会把不同的步骤和环节，拆分成不同的Invoker，把每个环节的代码逻辑内聚在单个Invoker内部；
            //再通过一系列的代码逻辑，把不同的环节的Invoker串联在一起，一个个按照顺序去串联；
            //这样真正执行时，就可以以MigrationInvoker作为一个起点，它会一个个的执行下一个Invoker；

            //下面这行代码可以理解为开始构建Invoker体系链条，也就是把源头Invoker给先准备好
            //传入的参数cluster就是装饰着FailoverCluster的MockClusterWrapper
            //getMigrationInvoker()所返回的MigrationInvoker中，它的invoker和serviceDiscoveryInvoker属性此时还是null
            ClusterInvoker<T> migrationInvoker = getMigrationInvoker(this, cluster, registry, type, url, consumerUrl);

            //接着下一步，要去继续组装后续的Invoker，然后把后续的Invoker构建好时，便设置给其上层Invoker；
            //所以在interceptInvoker()方法中，RegistryProtocol有责任在此把Invoker链条组装好；
            //但interceptInvoker字面上的意思就是拦截Invoker，字面上并不像是把后续的Invoker构建好设置给上层，有语义不清晰的嫌疑；

            //Invoker链条代表了RPC调用链路的全过程，每个步骤代表了一个环节，比如RPC调用过程里包含了如下环节：
            //migration(会有两个源头Invoker可以进行互相切换) -> mock降级 -> filter链条 -> 集群容错cluster -> 负载均衡 -> DubboInvoker
            //Dubbo应该针对每个环节进行对应的Invoker构建，并在每个环节的Invoker被构建好后，注入到上层的Invoker，从而完成Invoker链条的构建
            return interceptInvoker(migrationInvoker, url, consumerUrl);
        }

        protected <T> ClusterInvoker<T> getMigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            //构建基于服务发现机制迁移的Invoker
            //传入的cluster就是装饰着FailoverCluster的MockClusterWrapper
            //传入的registry为装饰ZookeeperRegistry的ListenerRegistryWrapper
            return new ServiceDiscoveryMigrationInvoker<T>(registryProtocol, cluster, registry, type, url, consumerUrl);
        }
        ...
    }

    public class ServiceDiscoveryMigrationInvoker<T> extends MigrationInvoker<T> {
        public ServiceDiscoveryMigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            super(registryProtocol, cluster, registry, type, url, consumerUrl);
        }
        ...
    }

    public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
        //服务引用时，它是一个MockClusterInvoker
        private volatile ClusterInvoker<T> invoker;

        //这是一个名为serviceDiscoveryInvoker的ClusterInvoker实现类的实例，也是一个MockClusterInvoker
        private volatile ClusterInvoker<T> serviceDiscoveryInvoker;
        private RegistryProtocol registryProtocol;

        //服务引用时，它是一个故障转移的集群FailoverCluster
        private Cluster cluster;

        //服务引用时，它是一个服务注册中心ZooKeeperRegistry
        ...
        
        public MigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            this(null, null, registryProtocol, cluster, registry, type, url, consumerUrl);
        }

        public MigrationInvoker(ClusterInvoker<T> invoker, ClusterInvoker<T> serviceDiscoveryInvoker, RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            //一开始创建MigrationInvoker时invoker属性是null
            //经过MigrationRuleListener.onRefer()方法的刷新后
            //也就是执行RegistryProtocol.doCreateInvoker()方法后
            //就会变为封装了DynamicDirectory和FailoverClusterInvoker的MockClusterInvoker
            this.invoker = invoker;

            //一开始创建MigrationInvoker时serviceDiscoveryInvoker属性是null
            //经过MigrationRuleListener.onRefer()方法的刷新后
            //也就是执行RegistryProtocol.doCreateInvoker()方法后
            //就会变为封装了DynamicDirectory和FailoverClusterInvoker的MockClusterInvoker
            this.serviceDiscoveryInvoker = serviceDiscoveryInvoker;
            this.registryProtocol = registryProtocol;

            //cluster就是装饰着FailoverCluster的MockClusterWrapper
            this.cluster = cluster;

            //registry是装饰着ZookeeperRegistry的ListenerRegistryWrapper
            this.registry = registry;
            ...
        }
        ...
    }

    public class MockClusterInvoker<T> implements ClusterInvoker<T> {
        private final Directory<T> directory;
        private final Invoker<T> invoker;

        public MockClusterInvoker(Directory<T> directory, Invoker<T> invoker) {
            //传入的directory为DynamicDirectory
            //当初始化MigrationInvoker.invoker时，传入的directory为RegistryDirectory
            //当初始化MigrationInvoker.serviceDiscoveryInvoker时，传入的directory为ServiceDiscoveryRegistryDirectory
            //不管是RegistryDirectory，还是ServiceDiscoveryRegistryDirectory，其实都是DynamicDirectory
            this.directory = directory;

            //传入的invoker为AbstractCluster的内部类ClusterFilterInvoker
            //其filterInvoker属性的originalInvoker属性便为封装了DynamicDirectory的FailoverClusterInvoker
            this.invoker = invoker;
        }
        ...
    }

RegistryProtocol的interceptInvoker()方法会加载相应的监听器实现，其中就包括了MigrationRuleListener。

MigrationRuleListener的onRefer()方法又会调用MigrationRuleHandler的doMigrate()方法。

MigrationRuleHandler的doMigrate()方法会调用MigrationRuleHandler的refreshInvoker()方法。

MigrationRuleHandler的refreshInvoker()方法会调用MigrationInvoker的refreshInterfaceInvoker()和refreshServiceDiscoveryInvoker()两个方法。MigrationInvoker的这两个方法又会调用RegistryProtocol的getInvoker()和getServiceDiscoveryInvoker()两个方法，也就是通过RegistryProtocol的doCreateInvoker()方法来刷新MigrationInvoker的invoker属性和serviceDiscoveryInvoker属性。

    public class RegistryProtocol implements Protocol, ScopeModelAware {
        ...
        //这里首先会取出RegistryProtocol里的监听回调器
        //然后遍历监听回调器进行回调处理
        protected <T> Invoker<T> interceptInvoker(ClusterInvoker<T> invoker, URL url, URL consumerUrl) {
            //根据URL中的registry.protocol.listener参数加载相应的监听器实现
            //也就是RegistryProtocol里的监听回调器
            List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
            if (CollectionUtils.isEmpty(listeners)) {
                return invoker;
            }

            //此时传入的invoker也就是MigrationInvoker中
            //它的invoker和serviceDiscoveryInvoker属性此时还是null
            for (RegistryProtocolListener listener : listeners) {
                //传递给监听器，比如下面会调用MigrationRuleListener.onRefer()方法
                //事实上默认下listeners只有MigrationRuleListener一个元素
                listener.onRefer(this, invoker, consumerUrl, url);
            }

            //此时传入的invoker也就是MigrationInvoker中
            //它的invoker和serviceDiscoveryInvoker属性就都为MockClusterInvoker
            //其中前者的directory属性=RegistryDirectory
            //后者的directory属性=ServiceDiscoveryRegistryDirectory
            //但两者的invoker属性都是AbstractCluster的内部类ClusterFilterInvoker
            return invoker;
        }

        protected List<RegistryProtocolListener> findRegistryProtocolListeners(URL url) {
            //通过SPI的Activate自动激活机制，针对指定的接口去获取符合的所有实现类
            return ScopeModelUtil.getExtensionLoader(RegistryProtocolListener.class, url.getScopeModel())
                .getActivateExtension(url, REGISTRY_PROTOCOL_LISTENER_KEY);
        }
        ...
    }

    @Activate
    public class MigrationRuleListener implements RegistryProtocolListener, ConfigurationListener {
        //默认的迁移规则
        private volatile MigrationRule rule;
        ...
        
        @Override
        public void onRefer(RegistryProtocol registryProtocol, ClusterInvoker<?> invoker, URL consumerUrl, URL registryURL) {
            MigrationRuleHandler<?> migrationRuleHandler = handlers.computeIfAbsent((MigrationInvoker<?>) invoker, _key -> {
                //给这个MigrationInvoker设置一个MigrationRuleListener，也就是当前的这个listener
                //此外还构建一个MigrationRuleHandler
                ((MigrationInvoker<?>) invoker).setMigrationRuleListener(this);
                return new MigrationRuleHandler<>((MigrationInvoker<?>) invoker, consumerUrl);
            });

            //然后拿默认构建好的迁移规则处理器，来执行对应的迁移逻辑：
            //rule=INIT规则，比如下面会调用MigrationRuleHandler.doMigrate()
            migrationRuleHandler.doMigrate(rule);
        }
        ...
    }

    public class MigrationRuleHandler<T> {
        ...
        public synchronized void doMigrate(MigrationRule rule) {
            ...
            if (refreshInvoker(step, threshold, rule)) {
                setMigrationRule(rule);
            }
        }

        private boolean refreshInvoker(MigrationStep step, Float threshold, MigrationRule newRule) {
            ...
            switch (step) {
                case APPLICATION_FIRST:
                    //比如会调用MigrationInvoker.migrateToApplicationFirstInvoker()
                    migrationInvoker.migrateToApplicationFirstInvoker(newRule);
                    break;
                case FORCE_APPLICATION:
                    //比如会调用MigrationInvoker.refreshServiceDiscoveryInvoker()
                    success = migrationInvoker.migrateToForceApplicationInvoker(newRule);
                    break;
                case FORCE_INTERFACE:
                default:
                    //比如会调用MigrationInvoker.refreshInterfaceInvoker()
                    success = migrationInvoker.migrateToForceInterfaceInvoker(newRule);
            }
            ...
        }
        ...
    }

    public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
        //服务引用时，它是一个MockClusterInvoker
        private volatile ClusterInvoker<T> invoker;

        //这是一个名为serviceDiscoveryInvoker的ClusterInvoker实现类的实例，也是一个MockClusterInvoker
        private volatile ClusterInvoker<T> serviceDiscoveryInvoker;
        ...

        @Override
        public void migrateToApplicationFirstInvoker(MigrationRule newRule) {
            CountDownLatch latch = new CountDownLatch(0);

            //这里是比较关键的：
            //migration迁移有两个核心的Invoker: InterfaceInvoker，ServiceDiscoveryInvoker；
            //下面会进行Invoker的刷新处理，就是在给我们的MigrationInvoker去注入下一环节的Invoker
            refreshInterfaceInvoker(latch);//第一个InterfaceInvoker
            refreshServiceDiscoveryInvoker(latch);//第二个ServiceDiscoveryInvoker

            //directly calculate preferred invoker, will not wait until address notify
            //calculation will re-occurred when address notify later
            //下面会把MigrationRule传递进去；
            //也就是当两个Invoker刷新好后，会根据最新的迁移规则来决定当前应该用哪个Invoker来作为MigrationInvoker的下一级Invoker；
            calcPreferredInvoker(newRule);
        }

        protected void refreshInterfaceInvoker(CountDownLatch latch) {
            ...
            //这里是核心逻辑
            //入参cluster是装饰着FailoverCluster的MockClusterWrapper
            //入参registry是装饰着ZookeeperRegistry的ListenerRegistryWrapper
            //下面会调用InterfaceCompatibleRegistryProtocol的getInvoker()方法
            //这个invoker其实最后发现就是FailoverClusterInvoker
            invoker = registryProtocol.getInvoker(cluster, registry, type, url);
            ...
        }

        protected void refreshServiceDiscoveryInvoker(CountDownLatch latch) {
            ...
            //比如下面会调用RegistryProtocol子类InterfaceCompatibleRegistryProtocol的getServiceDiscoveryInvoker()方法
            serviceDiscoveryInvoker = registryProtocol.getServiceDiscoveryInvoker(cluster, registry, type, url);
            ...
        }
        ...
    }

    public class InterfaceCompatibleRegistryProtocol extends RegistryProtocol {
        ...
        @Override
        public <T> ClusterInvoker<T> getInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
            //这里传入的registry是ZookeeperRegistry
            DynamicDirectory<T> directory = new RegistryDirectory<>(type, url);

            //下面会调用RegistryProtocol的doCreateInvoker()方法
            return doCreateInvoker(directory, cluster, registry, type);
        }

        @Override
        public <T> ClusterInvoker<T> getServiceDiscoveryInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
            //这个getRegistry()会获取到装饰着ServiceDiscoveryRegistry的ListenerRegistryWrapper，对传入的registry进行覆盖
            registry = getRegistry(super.getRegistryUrl(url));
            DynamicDirectory<T> directory = new ServiceDiscoveryRegistryDirectory<>(type, url);

            //下面会调用RegistryProtocol的doCreateInvoker()方法
            return doCreateInvoker(directory, cluster, registry, type);
        }
        ...
    }

    public class RegistryProtocol implements Protocol, ScopeModelAware {
        ...
        //入参directory是继承了DynamicDirectory的RegistryDirectory或者是继承了DynamicDirectory的ServiceDiscoveryRegistryDirectory
        //入参cluster是装饰着FailoverCluster的MockClusterWrapper
        protected <T> ClusterInvoker<T> doCreateInvoker(DynamicDirectory<T> directory, Cluster cluster, Registry registry, Class<T> type) {
            //设置注册中心
            directory.setRegistry(registry);
            //设置protocol协议
            directory.setProtocol(protocol);
            //all attributes of REFER_KEY
            //生成urlToRegistry，协议为consumer，具体的参数是RegistryURL中refer参数指定的参数
            Map<String, String> parameters = new HashMap<>(directory.getConsumerUrl().getParameters());
            URL urlToRegistry = new ServiceConfigURL(parameters.get(PROTOCOL_KEY) == null ? CONSUMER : parameters.get(PROTOCOL_KEY), parameters.remove(REGISTER_IP_KEY), 0, getPath(parameters, type), parameters);
            urlToRegistry = urlToRegistry.setScopeModel(directory.getConsumerUrl().getScopeModel());
            urlToRegistry = urlToRegistry.setServiceModel(directory.getConsumerUrl().getServiceModel());
            if (directory.isShouldRegister()) {
                //在urlToRegistry中添加category=consumers和check=false参数
                directory.setRegisteredConsumerUrl(urlToRegistry);
                //服务注册，在Zookeeper的consumers节点下，添加该Consumer对应的节点
                registry.register(directory.getRegisteredConsumerUrl());
            }

            //根据urlToRegistry创建服务路由
            directory.buildRouterChain(urlToRegistry);

            //订阅服务，toSubscribeUrl()方法会将urlToRegistry中category参数修改为"providers,configurators,routers"
            //DynamicDirectory子类RegistryDirectory的subscribe()会通过Registry订阅服务，同时还会添加相应的监听器
            directory.subscribe(toSubscribeUrl(urlToRegistry));

            //注册中心中可能包含多个Provider，相应的也就有多个Invoker
            //这里通过前面选择的cluster(即装饰着FailoverCluster的MockClusterWrapper)将多个Invoker对象封装成一个Invoker对象
            //所以下面首先会调用MockClusterWrapper的join()方法
            //最后返回的MockClusterInvoker会封装好DynamicDirectory和FailoverClusterInvoker
            return (ClusterInvoker<T>) cluster.join(directory, true);
        }
        ...
    }

    public class MockClusterWrapper implements Cluster {
        //包含一个故障转移集群FailoverCluster
        private final Cluster cluster;

        public MockClusterWrapper(Cluster cluster) {
            //Wrapper类都会有一个拷贝构造函数，装饰器模式
            this.cluster = cluster;
        }

        @Override
        public <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) throws RpcException {
            //先调用AbstractCluster.join()方法处理传入的RegistryDirectory
            //然后用MockClusterInvoker进行包装，将DynamicDirectory和FailoverClusterInvoker包装在里面
            return new MockClusterInvoker<T>(directory, this.cluster.join(directory, buildFilterChain));
        }

        public Cluster getCluster() {
            return cluster;
        }
    }

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/bce84de6-768a-4ae4-adb2-7432ab5d05c4" />

<br>

**3.DynamicDirectory与Invoker的关系**

DynamicDirectory的使用入口是RegistryProtocol的refer()方法：该方法首先会获取一个包含zk注册中心ZooKeeperRegistry的ListenerRegistryWrapper，然后会获取一个包含故障转移集群FailoverCluster的MockClusterWrapper，接着会调用interceptInvoker()方法加载相应的监听器实现，其中就包括了MigrationRuleListener监听器。

    //-> ReferenceConfig.createInvokerForRemote()
    //-> RegistryProtocol.refer()
    //-> getRegistry() => ZooKeeperRegistry
    //-> Cluster.getCluster() => MockClusterWrapper、FailoverCluster
    //-> RegistryProtocol.doRefer()
    //-> RegistryProtocol.getMigrationInvoker()
    //-> RegistryProtocol.interceptInvoker()
    //-> MigrationRuleListener.onRefer()

    public class RegistryProtocol implements Protocol, ScopeModelAware {
        ...
        //下面会返回一个MigrationInvoker，其构成是：
        //registry属性为装饰ZookeeperRegistry的ListenerRegistryWrapper
        //cluster属性为装饰FailoverCluster的MockCLusterWrapper
        //invoker和serviceDiscoveryInvoker属性都为MockClusterInvoker
        //其中前者的directory属性=注入了ZookeeperRegistry的RegistryDirectory
        //后者的directory属性=注入了ServiceDiscoveryRegistry的ServiceDiscoveryRegistryDirectory
        //但两者的invoker属性都是AbstractCluster的内部类ClusterFilterInvoker
        @Override
        public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
            //从URL中获取注册中心的URL
            url = getRegistryUrl(url);

            //获取Registry实例，这里的RegistryFactory对象是通过Dubbo SPI的自动装载机制注入的
            //1.先拿到一个ListenerRegistryWrapper，里面包含一个zk注册中心ZooKeeperRegistry
            Registry registry = getRegistry(url);
            if (RegistryService.class.equals(type)) {
                return proxyFactory.getInvoker((T) registry, type, url);
            }

            //group="a,b" or group="*"
            //从注册中心URL的refer参数中获取此次服务引用的一些参数，其中就包括group
            Map<String, String> qs = (Map<String, String>) url.getAttribute(REFER_KEY);
            String group = qs.get(GROUP_KEY);
            if (StringUtils.isNotEmpty(group)) {
                if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                    //如果此次可以引用多个group的服务，则Cluser实现使用MergeableCluster实现
                    //这里的getMergeableCluster()方法就会通过Dubbo SPI方式找到MergeableCluster实例
                    return doRefer(Cluster.getCluster(url.getScopeModel(), MergeableCluster.NAME), registry, type, url, qs);
                }
            }

            //2.接着会拿到一个MockClusterWrapper，里面会封装一个故障转移集群FailoverCluster
            //Cluster是和集群容错强相关的
            //不同的Cluster会对应不同的Cluster Invoker
            //不同的Cluster Invoker就有调用失败时不同的集群容错策略和算法
            Cluster cluster = Cluster.getCluster(url.getScopeModel(), qs.get(CLUSTER_KEY));

            //doRefer()这种代码的编写技巧在很多框架系统都会使用
            //refer()方法会先做一些提前的准备工作，而正式的工作会在方法名称的前面加一个do变成doRefer()方法来执行真正的工作
            //下面的doRefer()方法，执行时发现如果没有group参数或是group参数，则通过Cluster适配器选择Cluster实现
            return doRefer(cluster, registry, type, url, qs);
        }

        //doRefer()方法传入的参数cluster就是装饰着FailoverCluster的MockClusterWrapper
        //传入的参数registry为装饰ZookeeperRegistry的ListenerRegistryWrapper
        //这里会把组装好的Invoker体系里的最源头的一个Invoker进行返回，下一步就会去创建接口的动态代理并把该Invoker放进去
        //这样动态代理的方法被调用时，就可以通过该Invoker及其关联的Invoker来完成一个完整的RPC调用过程
        protected <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url, Map<String, String> parameters) {
            Map<String, Object> consumerAttribute = new HashMap<>(url.getAttributes());
            consumerAttribute.remove(REFER_KEY);
            String p = isEmpty(parameters.get(PROTOCOL_KEY)) ? CONSUMER : parameters.get(PROTOCOL_KEY);
            URL consumerUrl = new ServiceConfigURL(p, null, null, parameters.get(REGISTER_IP_KEY), 0, getPath(parameters, type), parameters, consumerAttribute);
            url = url.putAttribute(CONSUMER_URL_KEY, consumerUrl);

            //下面会以MigrationInvoker作为一个起点去构建一个Invoker链条；
            //这使用了经典的责任链模式，一环扣一环，先执行一个步骤再下一个步骤，把RPC调用的过程，一步步执行完毕；
            //但是在代码层面，却并非是严格的责任链模式来编码的，只是用到了这种责任链思想而已；
            //所以在这里会把不同的步骤和环节，拆分成不同的Invoker，把每个环节的代码逻辑内聚在单个Invoker内部；
            //再通过一系列的代码逻辑，把不同的环节的Invoker串联在一起，一个个按照顺序去串联；
            //这样真正执行时，就可以以MigrationInvoker作为一个起点，它会一个个的执行下一个Invoker；

            //下面这行代码可以理解为开始构建Invoker体系链条，也就是把源头Invoker给先准备好
            //传入的参数cluster就是装饰着FailoverCluster的MockClusterWrapper
            //getMigrationInvoker()所返回的MigrationInvoker中，它的invoker和serviceDiscoveryInvoker属性此时还是null
            ClusterInvoker<T> migrationInvoker = getMigrationInvoker(this, cluster, registry, type, url, consumerUrl);

            //接着下一步，要去继续组装后续的Invoker，然后把后续的Invoker构建好时，便设置给其上层Invoker；
            //所以在interceptInvoker()方法中，RegistryProtocol有责任在此把Invoker链条组装好；
            //但interceptInvoker字面上的意思就是拦截Invoker，字面上并不像是把后续的Invoker构建好设置给上层，有语义不清晰的嫌疑；

            //Invoker链条代表了RPC调用链路的全过程，每个步骤代表了一个环节，比如RPC调用过程里包含了如下环节：
            //migration(会有两个源头Invoker可以进行互相切换) -> mock降级 -> filter链条 -> 集群容错cluster -> 负载均衡 -> DubboInvoker
            //Dubbo应该针对每个环节进行对应的Invoker构建，并在每个环节的Invoker被构建好后，注入到上层的Invoker，从而完成Invoker链条的构建
            return interceptInvoker(migrationInvoker, url, consumerUrl);
        }

        protected <T> ClusterInvoker<T> getMigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            //构建基于服务发现机制迁移的Invoker
            //传入的cluster就是装饰着FailoverCluster的MockClusterWrapper
            //传入的registry为装饰ZookeeperRegistry的ListenerRegistryWrapper
            return new ServiceDiscoveryMigrationInvoker<T>(registryProtocol, cluster, registry, type, url, consumerUrl);
        }

        //这里首先会取出RegistryProtocol里的监听回调器
        //然后遍历监听回调器进行回调处理
        protected <T> Invoker<T> interceptInvoker(ClusterInvoker<T> invoker, URL url, URL consumerUrl) {
            //根据URL中的registry.protocol.listener参数加载相应的监听器实现
            //也就是RegistryProtocol里的监听回调器
            List<RegistryProtocolListener> listeners = findRegistryProtocolListeners(url);
            if (CollectionUtils.isEmpty(listeners)) {
                return invoker;
            }

            //此时传入的invoker也就是MigrationInvoker中
            //它的invoker和serviceDiscoveryInvoker属性此时还是null
            for (RegistryProtocolListener listener : listeners) {
                //传递给监听器，比如下面会调用MigrationRuleListener.onRefer()方法
                //事实上默认下listeners只有MigrationRuleListener一个元素
                listener.onRefer(this, invoker, consumerUrl, url);
            }

            //此时传入的invoker也就是MigrationInvoker中
            //它的invoker和serviceDiscoveryInvoker属性就都为MockClusterInvoker
            //其中前者的directory属性=RegistryDirectory
            //后者的directory属性=ServiceDiscoveryRegistryDirectory
            //但两者的invoker属性都是AbstractCluster的内部类ClusterFilterInvoker
            return invoker;
        }
        ...
    }

    public class MockClusterWrapper implements Cluster {
        //包含一个故障转移集群FailoverCluster
        private final Cluster cluster;
            
        public <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) throws RpcException {
            //先调用AbstractCluster.join()方法处理传入的RegistryDirectory
            //然后用MockClusterInvoker进行包装，将DynamicDirectory和FailoverClusterInvoker包装在里面
            return new MockClusterInvoker<T>(directory, this.cluster.join(directory, buildFilterChain));
        }
        ...
    }

    public class ServiceDiscoveryMigrationInvoker<T> extends MigrationInvoker<T> {
        ...
        public ServiceDiscoveryMigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            super(registryProtocol, cluster, registry, type, url, consumerUrl);
        }
        ...
    }

    public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
        ...
        //服务引用时，它是一个故障转移的集群FailoverCluster
        private Cluster cluster;
        //服务引用时，它是一个服务注册中心ZooKeeperRegistry
        private Registry registry;
        //服务引用时，它是一个MockClusterInvoker
        private volatile ClusterInvoker<T> invoker;
        //这是一个名为serviceDiscoveryInvoker的ClusterInvoker实现类的实例，也是一个MockClusterInvoker
        private volatile ClusterInvoker<T> serviceDiscoveryInvoker;
        ...

        public MigrationInvoker(RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            this(null, null, registryProtocol, cluster, registry, type, url, consumerUrl);
        }

        public MigrationInvoker(ClusterInvoker<T> invoker, ClusterInvoker<T> serviceDiscoveryInvoker, RegistryProtocol registryProtocol, Cluster cluster, Registry registry, Class<T> type, URL url, URL consumerUrl) {
            //一开始创建MigrationInvoker时invoker属性是null
            //经过MigrationRuleListener.onRefer()方法的刷新后
            //也就是执行RegistryProtocol.doCreateInvoker()方法后
            //就会变为封装了DynamicDirectory和FailoverClusterInvoker的MockClusterInvoker
            this.invoker = invoker;
            //一开始创建MigrationInvoker时serviceDiscoveryInvoker属性是null
            //经过MigrationRuleListener.onRefer()方法的刷新后
            //也就是执行RegistryProtocol.doCreateInvoker()方法后
            //就会变为封装了DynamicDirectory和FailoverClusterInvoker的MockClusterInvoker
            this.serviceDiscoveryInvoker = serviceDiscoveryInvoker;
            this.registryProtocol = registryProtocol;
            //cluster就是装饰着FailoverCluster的MockClusterWrapper
            this.cluster = cluster;
            //registry是装饰着ZookeeperRegistry的ListenerRegistryWrapper
            this.registry = registry;
            ...
        }
    }

MigrationRuleListener的onRefer()方法又会调用MigrationRuleHandler的doMigrate()方法。

MigrationRuleHandler的doMigrate()方法会调用MigrationRuleHandler的refreshInvoker()方法。

MigrationRuleHandler的refreshInvoker()方法会调用MigrationInvoker的refreshInterfaceInvoker()和refreshServiceDiscoveryInvoker()两个方法。MigrationInvoker的这两个方法又会调用RegistryProtocol的getInvoker()和getServiceDiscoveryInvoker()两个方法，也就是通过RegistryProtocol的doCreateInvoker()方法来刷新MigrationInvoker的invoker属性和serviceDiscoveryInvoker属性。

RegistryProtocol的doCreateInvoker()方法最终返回的是封装了DynamicDirectory和FailoverClusterInvoker的MockClusterInvoker，而MockClusterInvoker又会包含在MigrationInvoker里面。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/1be4236d-59c9-4fa0-a34c-1a60e0304a03" />

    //-> MigrationRuleListener.onRefer()
    //-> MigrationRuleHandler.doMigrate()
    //-> MigrationRuleListener.refreshInvoker()
    //-> MigrationInvoker.refreshInterfaceInvoker() + MigrationInvoker.refreshServiceDiscoveryInvoker()
    //-> RegistryProtocol.getInvoker() + RegistryProtocol.getServiceDiscoveryInvoker()
    //-> RegistryProtocol.doCreateInvoker()

    //该类用于监听配置中心里的MigrationRule迁移规则的变化
    //由于MigrationRule迁移规则在consumer应用级别的范围都有效的
    //所以MigrationRuleListener在所有的Invoker里都是有效的
    //MigrationRuleListener是用来维系接口和handler之间的关系

    //MigrationRuleListener会在如下两种情形下执行：
    //1.当RegistryProtocol在refer()方法中针对MigrationInvoker进行回调处理时，会基于默认的迁移规则来决定Invoker后续的行为
    //这说明了MigrationInvoker后续的一些行为都是由默认的迁移规则来指定的
    //2.如果配置中心的迁移规则发生变化，此时Invoker后续的行为会由新规则来决定

    //为什么要出现MigrationRuleListener这么一个监听器，通过该监听器来回调处理源头的MigrationInvoker?
    //因为希望通过该监听器里的规则来决定MigrationInvoker后续的一些行为，也就是决定后续的一些Invoker的组装逻辑和行为
    //从而对RPC调用过程产生影响，比如可以通过对这个规则进行监听，一旦规则变化则改变Invoker链条的行为

    @Activate
    public class MigrationRuleListener implements RegistryProtocolListener, ConfigurationListener {
        //默认的迁移规则
        private volatile MigrationRule rule;
        ...

        @Override
        public void onRefer(RegistryProtocol registryProtocol, ClusterInvoker<?> invoker, URL consumerUrl, URL registryURL) {
            MigrationRuleHandler<?> migrationRuleHandler = handlers.computeIfAbsent((MigrationInvoker<?>) invoker, _key -> {
                //给这个MigrationInvoker设置一个MigrationRuleListener，也就是当前的这个listener
                //此外还构建一个MigrationRuleHandler
                ((MigrationInvoker<?>) invoker).setMigrationRuleListener(this);
                return new MigrationRuleHandler<>((MigrationInvoker<?>) invoker, consumerUrl);
            });

            //然后拿默认构建好的迁移规则处理器，来执行对应的迁移逻辑：
            //rule=INIT规则，比如下面会调用MigrationRuleHandler.doMigrate()
            migrationRuleHandler.doMigrate(rule);
        }
        ...
    }

    public class MigrationRuleHandler<T> {
        ...
        public synchronized void doMigrate(MigrationRule rule) {
            ...
            if (refreshInvoker(step, threshold, rule)) {
                setMigrationRule(rule);
            }
        }

        private boolean refreshInvoker(MigrationStep step, Float threshold, MigrationRule newRule) {
            ...
            switch (step) {
                case APPLICATION_FIRST:
                    //比如会调用MigrationInvoker.migrateToApplicationFirstInvoker()
                    migrationInvoker.migrateToApplicationFirstInvoker(newRule);
                    break;
                case FORCE_APPLICATION:
                    //比如会调用MigrationInvoker.refreshServiceDiscoveryInvoker()
                    success = migrationInvoker.migrateToForceApplicationInvoker(newRule);
                    break;
                case FORCE_INTERFACE:
                default:
                    //比如会调用MigrationInvoker.refreshInterfaceInvoker()
                    success = migrationInvoker.migrateToForceInterfaceInvoker(newRule);
            }
            ...
        }
        ...
    }

    public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
        //服务引用时，它是一个MockClusterInvoker
        private volatile ClusterInvoker<T> invoker;

        //这是一个名为serviceDiscoveryInvoker的ClusterInvoker实现类的实例，也是一个MockClusterInvoker
        private volatile ClusterInvoker<T> serviceDiscoveryInvoker;
        ...

        @Override
        public void migrateToApplicationFirstInvoker(MigrationRule newRule) {
            CountDownLatch latch = new CountDownLatch(0);

            //这里是比较关键的：
            //migration迁移有两个核心的Invoker: InterfaceInvoker，ServiceDiscoveryInvoker；
            //下面会进行Invoker的刷新处理，就是在给我们的MigrationInvoker去注入下一环节的Invoker
            refreshInterfaceInvoker(latch);//第一个InterfaceInvoker
            refreshServiceDiscoveryInvoker(latch);//第二个ServiceDiscoveryInvoker

            //directly calculate preferred invoker, will not wait until address notify
            //calculation will re-occurred when address notify later
            //下面会把MigrationRule传递进去；
            //也就是当两个Invoker刷新好后，会根据最新的迁移规则来决定当前应该用哪个Invoker来作为MigrationInvoker的下一级Invoker；
            calcPreferredInvoker(newRule);
        }

        protected void refreshInterfaceInvoker(CountDownLatch latch) {
            ...
            //这里是核心逻辑
            //入参cluster是装饰着FailoverCluster的MockClusterWrapper
            //入参registry是装饰着ZookeeperRegistry的ListenerRegistryWrapper
            //下面会调用InterfaceCompatibleRegistryProtocol的getInvoker()方法
            //这个invoker其实最后发现就是FailoverClusterInvoker
            invoker = registryProtocol.getInvoker(cluster, registry, type, url);
            ...
        }

        protected void refreshServiceDiscoveryInvoker(CountDownLatch latch) {
            ...
            //比如下面会调用RegistryProtocol子类InterfaceCompatibleRegistryProtocol的getServiceDiscoveryInvoker()方法
            serviceDiscoveryInvoker = registryProtocol.getServiceDiscoveryInvoker(cluster, registry, type, url);
            ...
        }
        ...
    }

    public class InterfaceCompatibleRegistryProtocol extends RegistryProtocol {
        ...
        @Override
        public <T> ClusterInvoker<T> getInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
            //这里传入的registry是ZookeeperRegistry
            DynamicDirectory<T> directory = new RegistryDirectory<>(type, url);

            //下面会调用RegistryProtocol的doCreateInvoker()方法
            return doCreateInvoker(directory, cluster, registry, type);
        }

        @Override
        public <T> ClusterInvoker<T> getServiceDiscoveryInvoker(Cluster cluster, Registry registry, Class<T> type, URL url) {
            //这个getRegistry()会获取到装饰着ServiceDiscoveryRegistry的ListenerRegistryWrapper，对传入的registry进行覆盖
            registry = getRegistry(super.getRegistryUrl(url));
            DynamicDirectory<T> directory = new ServiceDiscoveryRegistryDirectory<>(type, url);

            //下面会调用RegistryProtocol的doCreateInvoker()方法
            return doCreateInvoker(directory, cluster, registry, type);
        }
        ...
    }

    public class RegistryDirectory<T> extends DynamicDirectory<T> {
        ...
        ...
    }

    public class ServiceDiscoveryRegistryDirectory<T> extends DynamicDirectory<T> {
        ...
        ...
    }

    //DynamicDirectory相当于2.7.x里的RegistryDirectory
    //它是一个用来进行服务发现的组件，是一个可以动态管理可变的服务实例集群地址的组件
    //当Consumer调用Provider时，便会用到该动态目录DynamicDirectory进行服务发现

    //动态目录可以理解为：
    //动态可变的一个目标服务实例的集群地址(invokers)
    //有多少个目标服务实例就会有多少Invoker

    //动态的意思其实就是：
    //不仅可以通过主动查询来获取目标服务实例集群地址
    //还可以在目标服务实例集群地址发生变化时，接收zk的推送并更新内存里的地址
    public abstract class DynamicDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
        ...
        ...
    }

    public class RegistryProtocol implements Protocol, ScopeModelAware {
        ...
        //入参directory是继承了DynamicDirectory的RegistryDirectory或者是继承了DynamicDirectory的ServiceDiscoveryRegistryDirectory
        //入参cluster是装饰着FailoverCluster的MockClusterWrapper
        protected <T> ClusterInvoker<T> doCreateInvoker(DynamicDirectory<T> directory, Cluster cluster, Registry registry, Class<T> type) {
            //设置注册中心
            directory.setRegistry(registry);

            //设置protocol协议
            directory.setProtocol(protocol);

            //all attributes of REFER_KEY
            //生成urlToRegistry，协议为consumer，具体的参数是RegistryURL中refer参数指定的参数
            Map<String, String> parameters = new HashMap<>(directory.getConsumerUrl().getParameters());
            URL urlToRegistry = new ServiceConfigURL(parameters.get(PROTOCOL_KEY) == null ? CONSUMER : parameters.get(PROTOCOL_KEY), parameters.remove(REGISTER_IP_KEY), 0, getPath(parameters, type), parameters);
            urlToRegistry = urlToRegistry.setScopeModel(directory.getConsumerUrl().getScopeModel());
            urlToRegistry = urlToRegistry.setServiceModel(directory.getConsumerUrl().getServiceModel());

            if (directory.isShouldRegister()) {
                //在urlToRegistry中添加category=consumers和check=false参数
                directory.setRegisteredConsumerUrl(urlToRegistry);
                //服务注册，在Zookeeper的consumers节点下，添加该Consumer对应的节点
                registry.register(directory.getRegisteredConsumerUrl());
            }

            //根据urlToRegistry创建服务路由
            directory.buildRouterChain(urlToRegistry);

            //订阅服务，toSubscribeUrl()方法会将urlToRegistry中category参数修改为"providers,configurators,routers"
            //DynamicDirectory子类RegistryDirectory的subscribe()会通过Registry订阅服务，同时还会添加相应的监听器
            directory.subscribe(toSubscribeUrl(urlToRegistry));

            //注册中心中可能包含多个Provider，相应的也就有多个Invoker
            //这里通过前面选择的cluster(即装饰着FailoverCluster的MockClusterWrapper)将多个Invoker对象封装成一个Invoker对象
            //所以下面首先会调用MockClusterWrapper的join()方法
            //最后返回的MockClusterInvoker会封装好DynamicDirectory和FailoverClusterInvoker
            return (ClusterInvoker<T>) cluster.join(directory, true);
        }
        ...
    }

    public class MockClusterWrapper implements Cluster {
        //包含一个故障转移集群FailoverCluster
        private final Cluster cluster;

        public MockClusterWrapper(Cluster cluster) {
            //Wrapper类都会有一个拷贝构造函数，装饰器模式
            this.cluster = cluster;
        }

        @Override
        public <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) throws RpcException {
            //先调用AbstractCluster.join()方法处理传入的RegistryDirectory
            //然后用MockClusterInvoker进行包装，将DynamicDirectory和FailoverClusterInvoker包装在里面
            return new MockClusterInvoker<T>(directory, this.cluster.join(directory, buildFilterChain));
        }

        public Cluster getCluster() {
            return cluster;
        }
    }

    public abstract class AbstractCluster implements Cluster {
        ...
        @Override
        public <T> Invoker<T> join(Directory<T> directory, boolean buildFilterChain) throws RpcException {
            //比如下面会执行FailoverCluster的doJoin()方法
            if (buildFilterChain) {
                return buildClusterInterceptors(doJoin(directory));
            } else {
                return doJoin(directory);
            }
        }

        private <T> Invoker<T> buildClusterInterceptors(AbstractClusterInvoker<T> clusterInvoker) {
            //服务引用时这里传入的clusterInvoker会为FailoverClusterInvoker
            //然后构建出一个AbstractCluster的内部类ClusterFilterInvoker
            AbstractClusterInvoker<T> last = buildInterceptorInvoker(new ClusterFilterInvoker<>(clusterInvoker));
            return last;
        }

        private <T> AbstractClusterInvoker<T> buildInterceptorInvoker(AbstractClusterInvoker<T> invoker) {
            //通过SPI方式加载InvocationInterceptorBuilder扩展实现
            //在服务引用时默认情况下，下面获取到的builders为空
            //框架没提供InvocationInterceptorBuilder的默认实现
            List<InvocationInterceptorBuilder> builders = ScopeModelUtil.getApplicationModel(invoker.getUrl().getScopeModel()).getExtensionLoader(InvocationInterceptorBuilder.class).getActivateExtensions();
            if (CollectionUtils.isEmpty(builders)) {
                return invoker;
            }
            return new InvocationInterceptorInvoker<>(invoker, builders);
        }

        protected abstract <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException;
        ...
    }

    public class FailoverCluster extends AbstractCluster {
        public final static String NAME = "failover";

        @Override
        public <T> AbstractClusterInvoker<T> doJoin(Directory<T> directory) throws RpcException {
            //执行RegistryProtocol.refer()时，这里传入的directory会是RegistryDirectory
            //这里会创建出一个FailoverClusterInvoker，同时把directory传入FailoverClusterInvoker
            return new FailoverClusterInvoker<>(directory);
        }
    }

    public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {
        ...
        public FailoverClusterInvoker(Directory<T> directory) {
            super(directory);
        }
        ...
    }

    public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
        protected Directory<T> directory;
        ...

        public AbstractClusterInvoker(Directory<T> directory) {
            this(directory, directory.getUrl());
        }

        public AbstractClusterInvoker(Directory<T> directory, URL url) {
            ...
            this.directory = directory;
            ...
        }
        ...
    }

    public class MockClusterInvoker<T> implements ClusterInvoker<T> {
        private final Directory<T> directory;
        private final Invoker<T> invoker;

        public MockClusterInvoker(Directory<T> directory, Invoker<T> invoker) {
            //传入的directory为DynamicDirectory
            //当初始化MigrationInvoker.invoker时，传入的directory为RegistryDirectory
            //当初始化MigrationInvoker.serviceDiscoveryInvoker时，传入的directory为ServiceDiscoveryRegistryDirectory
            //不管是RegistryDirectory，还是ServiceDiscoveryRegistryDirectory，其实都是DynamicDirectory
            this.directory = directory;

            //传入的invoker为AbstractCluster的内部类ClusterFilterInvoker
            //其filterInvoker属性的originalInvoker属性便为封装了DynamicDirectory的FailoverClusterInvoker
            this.invoker = invoker;
        }
        ...
    }

<br>

**4.基于Invoker来创建动态代理对象**

ReferenceConfig的createProxy()方法的最后一行代码，会基于Invoker创建一个动态代理对象。

    public class ReferenceConfig<T> extends ReferenceConfigBase<T> {
        //The invoker of the reference service
        private transient volatile Invoker<?> invoker;
        ...

        private T createProxy(Map<String, String> referenceParameters) {
            //provider端有一个本地发布的动作；
            //本地发布即把目标实现类封装成一个ProxyInvoker，然后再通过InjvmProtocol发布出去，也就是把ProxyInvoker封装成InjvmExporter；
            //所以在同一个JVM里进行本地服务调用时，调用的便是本地发布出去的provider服务实例

            //根据url的协议、scope以及injvm等参数检测是否需要本地引用
            if (shouldJvmRefer(referenceParameters)) {
                createInvokerForLocal(referenceParameters);
            } else {
                urls.clear();
                meshModeHandleUrl(referenceParameters);
                if (StringUtils.isNotEmpty(url)) {
                    //user specified URL, could be peer-to-peer address, or register center's address.
                    //解析用户指定的URL，可能是一个点对点的调用地址，或者是一个注册中心的地址
                    parseUrl(referenceParameters);
                } else {
                    //if protocols not in jvm checkRegistry
                    if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())) {
                        //加载注册中心的地址RegistryURL
                        aggregateUrlFromRegistry(referenceParameters);
                    }
                }
                //接下来是核心源码的第一步，通过RegistryProtocol创建invoker
                createInvokerForRemote();
            }
            ...

            //这些都是url的处理
            URL consumerUrl = new ServiceConfigURL(CONSUMER_PROTOCOL, referenceParameters.get(REGISTER_IP_KEY), 0, referenceParameters.get(INTERFACE_KEY), referenceParameters);
            consumerUrl = consumerUrl.setScopeModel(getScopeModel());
            consumerUrl = consumerUrl.setServiceModel(consumerModel);
            //对于Consumer的元数据，会通过如下代码在这里进行发布
            MetadataUtils.publishServiceDefinition(consumerUrl, consumerModel.getServiceModel(), getApplicationModel());

            //create service proxy
            //通过ProxyFactory适配器选择合适的ProxyFactory扩展实现，基于Invoker创建动态代理对象
            //接下来会调用ProxyFactory$Adaptive.getProxy() -> StubProxyFactoryWrapper.getProxy()
            return (T) proxyFactory.getProxy(invoker, ProtocolUtils.isGeneric(generic));
        }

        private void createInvokerForRemote() {
            //createInvokerForRemote核心就两个步骤：
            //第一是执行RegistryProtocol.refer()构建出一个invoker；
            //第二是执行Cluster.getCluster().join() + new StaticDirectory()对invokers进行加工处理，最后覆盖到invoker上；
            //protocolSPI.refer -> invokers -> new StaticDirectory() -> cluster.join -> invoker
            if (urls.size() == 1) {
                //在单注册中心或是直连单个服务提供方的时候，通过Protocol的适配器选择对应的Protocol实现创建Invoker对象
                //也就是会基于RegistryProtocol.refer构建出一个Invoker(比如dubbo-demo-api下的示例会构建出一个MigrationInvoker，且curUrl不是注册中心的)
                //而且该MigrationInvoker构成：registry属性为装饰ZookeeperRegistry的ListenerRegistryWrapper + cluster属性为装饰FailoverCluster的MockCLusterWrapper
                ...
                invoker = protocolSPI.refer(interfaceClass, curUrl);
                invokers.add(invoker);
                //对Invoker进行了一个深加工和处理，又拿到了一个Invoker，覆盖赋值处理
                invoker = Cluster.getCluster(scopeModel, Cluster.DEFAULT).join(new StaticDirectory(curUrl, invokers), true);
                ...
            } else {
                //多注册中心或是直连多个服务提供方的时候，会根据每个URL创建Invoker对象
                for (URL url : urls) {
                    invokers.add(protocolSPI.refer(interfaceClass, url));
                    ...
                }
                if (registryUrl != null) {
                    ...
                    invoker = Cluster.getCluster(registryUrl.getScopeModel(), cluster, false).join(new StaticDirectory(registryUrl, invokers), false);
                } else {
                    ...
                    invoker = Cluster.getCluster(scopeModel, cluster).join(new StaticDirectory(curUrl, invokers), true);          
                }
            }
            ...
        }
        ...
    }

最后一行代码"proxyFactory.getProxy()"最终会调用StubProxyFactoryWrapper的getProxy()方法，该方法生成的动态代理类主要实现了如下三个接口：

    1.org.apache.dubbo.demo.DemoService
    2.org.apache.dubbo.rpc.service.EchoService
    3.org.apache.dubbo.rpc.service.Destoryable

<!---->

    //-> ReferenceConfig.createProxy()
    //-> ReferenceConfig.createInvokerForRemote()
    //-> protocolSPI.refer()
    //-> proxyFactory.getProxy()
    //-> ProxyFactory$Adaptive.getProxy()
    //-> StubProxyFactoryWrapper.getProxy()
    //-> AbstractProxyFactory.getProxy()
    //-> JavassistProxyFactory.getProxy()
    //-> Proxy.getProxy()
    //-> new InvokerInvocationHandler()

    public class StubProxyFactoryWrapper implements ProxyFactory {
        //包含一个JavassistProxyFactory
        private final ProxyFactory proxyFactory;
        private Protocol protocol;

        public StubProxyFactoryWrapper(ProxyFactory proxyFactory) {
            this.proxyFactory = proxyFactory;
        }

        public void setProtocol(Protocol protocol) {
            this.protocol = protocol;
        }

        @Override
        public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
            //为什么它的名字叫stub打桩？
            //stub这个名字一般代表的是远程网络访问的动态代理
            //通过这个stub可以对远程的机器进行网络访问
            //stub打桩思想，是在网络远程访问里经常会使用的一种方法，类似于电线杆打桩一样
            //比如机器A和机器B之间进行的远程网络访问，必然要通过网络来进行连接和访问
            //机器A在进行远程RPC访问时，其调用代码，最好不要直接写网络通信的逻辑，而是写针对某个接口类型对象的方法调用
            //这样在机器A上面进行RPC调用时就像是在本地调用一样，都是直接调用某个对象的方法
            //然后在这个对象的方法里，再去写网络通信的逻辑和目标机器B进行网络连接，并发送调用请求
            //所以在机器A上会针对调用的那个接口，来动态去生成一个实现了该接口的类，也就是动态代理的代理类或者是stub打桩
            //类似于在机器A上打下的一个伪装成目标类的桩，然后针对打的这个stub桩去进行调用，stub内部会编写网络通信代码实现跨机器的访问
            //这就是远程网络连接和通信的stub打桩思想
            //下面使用了抽象代理工厂来获取真正的动态代理
            //下面会调用JavassistProxyFactory的getProxy()方法
            //也就是调用AbstractProxyFactory的getProxy()方法
            T proxy = proxyFactory.getProxy(invoker, generic);
            ...

            return proxy;
        }
        ...
    }

    public abstract class AbstractProxyFactory implements ProxyFactory {
        ...
        @Override
        public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
            //记录要代理的接口
            LinkedHashSet<Class<?>> interfaces = new LinkedHashSet<>();
            ClassLoader classLoader = getClassLoader(invoker);

            //获取URL中interfaces参数指定的接口
            String config = invoker.getUrl().getParameter(INTERFACES);
            if (StringUtils.isNotEmpty(config)) {
                //按照逗号切分interfaces参数，得到接口集合，然后进行遍历
                String[] types = COMMA_SPLIT_PATTERN.split(config);

                //遍历接口集合，并记录这些接口信息
                for (String type : types) {
                    //对每个接口都拿出对应的class对象，放到interfaces集合里
                    interfaces.add(ReflectUtils.forName(classLoader, type));
                }
            }
            ...

            //获取Invoker中type字段指定的接口
            interfaces.add(invoker.getInterface());

            //添加EchoService、Destroyable两个默认接口
            interfaces.addAll(Arrays.asList(INTERNAL_INTERFACES));

            //调用抽象的getProxy()重载方法
            //Dubbo的动态代理技术有如下两种：
            //一.javassist(动态拼接类的代码字符串，通过动态编译来动态生成一个类)
            //二.jdk(通过jdk提供的API利用反射来生成动态代理)
            //默认下面会调用子类JavassistProxyFactory的getProxy()方法
            return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
        }

        public abstract <T> T getProxy(Invoker<T> invoker, Class<?>[] types);
        ...
    }

    public class JavassistProxyFactory extends AbstractProxyFactory {
        private final static Logger logger = LoggerFactory.getLogger(JavassistProxyFactory.class);
        private final JdkProxyFactory jdkProxyFactory = new JdkProxyFactory();

        @Override
        @SuppressWarnings("unchecked")
        public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
            try {
                //下面的Proxy是Dubbo自己实现的Proxy
                return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
            } catch (Throwable fromJavassist) {
                //try fall back to JDK proxy factory
                try {
                    T proxy = jdkProxyFactory.getProxy(invoker, interfaces);
                    logger.error("Failed to generate proxy by Javassist failed. Fallback to use JDK proxy success. " + "Interfaces: " + Arrays.toString(interfaces), fromJavassist);
                    return proxy;
                } catch (Throwable fromJdk) {
                    logger.error("Failed to generate proxy by Javassist failed. Fallback to use JDK proxy is also failed. " + "Interfaces: " + Arrays.toString(interfaces) + " Javassist Error.", fromJavassist);
                    logger.error("Failed to generate proxy by Javassist failed. Fallback to use JDK proxy is also failed. " + "Interfaces: " + Arrays.toString(interfaces) + " JDK Error.", fromJdk);
                    throw fromJavassist;
                }
            }
        }
        ...
    }

    public class Proxy {
        private static final Map<ClassLoader, Map<String, Proxy>> PROXY_CACHE_MAP = new WeakHashMap<>();
        ...

        public static Proxy getProxy(Class<?>... ics) {
            ...
            //ClassLoader from App Interface should support load some class from Dubbo
            ClassLoader cl = ics[0].getClassLoader();
            ProtectionDomain domain = ics[0].getProtectionDomain();

            //use interface class name list as key.
            //生成的动态代理类主要实现了如下三个接口：
            //1.org.apache.dubbo.demo.DemoService
            //2.org.apache.dubbo.rpc.service.EchoService
            //3.org.apache.dubbo.rpc.service.Destoryable
            String key = buildInterfacesKey(cl, ics);

            //get cache by class loader.
            final Map<String, Proxy> cache;
            synchronized (PROXY_CACHE_MAP) {
                cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new ConcurrentHashMap<>());
            }

            //接口列表将会作为第二层集合的Key
            Proxy proxy = cache.get(key);
            if (proxy == null) {
                synchronized (ics[0]) {
                    proxy = cache.get(key);
                    if (proxy == null) {
                        //create Proxy class.
                        proxy = new Proxy(buildProxyClass(cl, ics, domain));
                        cache.put(key, proxy);
                    }
                }
            }
            return proxy;
        }

        private static Class<?> buildProxyClass(ClassLoader cl, Class<?>[] ics, ProtectionDomain domain) {
            ClassGenerator ccp = null;
            ccp = ClassGenerator.newInstance(cl);
            Set<String> worked = new HashSet<>();
            List<Method> methods = new ArrayList<>();
            String pkg = ics[0].getPackage().getName();
            Class<?> neighbor = ics[0];

            for (Class<?> ic : ics) {
                String npkg = ic.getPackage().getName();
                //向ClassGenerator中添加接口
                ccp.addInterface(ic);

                //遍历接口中的每个方法
                for (Method method : ic.getMethods()) {
                    String desc = ReflectUtils.getDesc(method);
                    //跳过已经重复方法以及static方法
                    if (worked.contains(desc) || Modifier.isStatic(method.getModifiers())) {
                        continue;
                    }
                    //将方法描述添加到worked这个Set集合中，进行去重
                    worked.add(desc);

                    int ix = methods.size();
                    //获取方法的返回值
                    Class<?> rt = method.getReturnType();
                    //获取方法的参数列表
                    Class<?>[] pts = method.getParameterTypes();
                    //创建方法体
                    StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                    for (int j = 0; j < pts.length; j++) {
                        code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(';');
                    }
                    code.append(" Object ret = handler.invoke(this, methods[").append(ix).append("], args);");
                    //生成return语句
                    if (!Void.TYPE.equals(rt)) {
                        code.append(" return ").append(asArgument(rt, "ret")).append(';');
                    }
                    //将生成好的方法添加到ClassGenerator中缓存
                    methods.add(method);
                    ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
                }
            }

            //create ProxyInstance class.
            //生成并设置代理类类名
            String pcn = neighbor.getName() + "DubboProxy" + PROXY_CLASS_COUNTER.getAndIncrement();
            ccp.setClassName(pcn);

            //添加字段
            //一个是前面生成的methods集合
            //另一个是InvocationHandler对象
            ccp.addField("public static java.lang.reflect.Method[] methods;");
            ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
            //添加构造方法
            ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
            //默认构造方法
            ccp.addDefaultConstructor();
            Class<?> clazz = ccp.toClass(neighbor, cl, domain);
            clazz.getField("methods").set(null, methods.toArray(new Method[0]));
            return clazz;
        }
        ...
    }

    //实现了JDK反射机制里的InvocationHandler接口
    public class InvokerInvocationHandler implements InvocationHandler {
        ...

        public InvokerInvocationHandler(Invoker<?> handler) {
            this.invoker = handler;
            URL url = invoker.getUrl();
            this.protocolServiceKey = url.getProtocolServiceKey();
            this.serviceModel = url.getServiceModel();
        }

        //通过这个代理对象可以拿到这个代理对象的接口
        //通过接口就可以定位到目标服务实例发布的服务接口，从而针对目标服务实例进行调用
        //拿到了调用目标服务实例的方法、传递进去的参数等信息，就可以进行完整的RPC调用
        //动态代理就是针对接口动态去生成该接口的实现类并进行调用

        //比如针对DemoService这个接口去生成一个实现类
        //由于这个实现类是动态生成的，那么它如何知道会有调用方去调用它的方法，以及又如何去执行？
        
        //为此，动态代理底层都会封装InvocationHandler
        //封装完后对动态代理所有方法的调用，都会跑到InvocationHandler这里来
        
        //这样，InvocationHandler的invoke方法就可以拿到：
        //Proxy动态代理对象、需要调用对象的哪个方法Method、以及被调用方法会传入的参数args
        
        //此时，对动态代理不同方法的调用以及具体方法被调用后会如何处理
        //都可以由如下invoke()方法发起远程调用由服务提供者来决定
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            //对于Object中定义的方法，直接调用Invoker对象的相应方法即可
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(invoker, args);
            }
            String methodName = method.getName();
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 0) {
                if ("toString".equals(methodName)) {
                    //对toString()方法进行特殊处理
                    return invoker.toString();
                } else if ("$destroy".equals(methodName)) {
                    //对$destroy等方法的特殊处理
                    invoker.destroy();
                    return null;
                } else if ("hashCode".equals(methodName)) {
                    //对hashCode()方法进行特殊处理
                    return invoker.hashCode();
                }
            } else if (parameterTypes.length == 1 && "equals".equals(methodName)) {
                return invoker.equals(args[0]);
            }

            //创建RpcInvocation对象，后面会作为远程RPC调用的参数
            //Consumer端进行RPC调用时，必须要封装一个RpcInvocation
            //这个RpcInvocation会传递到Provider端
            //Provider端拿到请求数据后，也会封装一个RpcInvocation
            RpcInvocation rpcInvocation = new RpcInvocation(serviceModel, method.getName(), invoker.getInterface().getName(), protocolServiceKey, method.getParameterTypes(), args);

            if (serviceModel instanceof ConsumerModel) {
                rpcInvocation.put(Constants.CONSUMER_MODEL, serviceModel);
                rpcInvocation.put(Constants.METHOD_MODEL, ((ConsumerModel) serviceModel).getMethodModel(method));
            }

            //调用invoke()方法发起远程调用
            //拿到AsyncRpcResult之后，调用recreate()方法获取响应结果(或是Future)
            //下面的invoker其实就是装饰了MockClusterInvoker的MigrationInvoker
            return InvocationUtil.invoke(invoker, rpcInvocation);
        }
    }

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/dba21c9c-5c91-4989-9871-1530119b7a92" />

<br>

**5.动态代理生成后检查Invoker是否有效**

    public class ReferenceConfig<T> extends ReferenceConfigBase<T> {
        ...
        protected synchronized void init() {
            ...
            //执行服务实例的刷新操作
            //也就是刷新ProviderConfig -> MethodConfig -> ArgumentConfig
            if (!this.isRefreshed()) {
                this.refresh();
            }

            //init serviceMetadata
            //对Metadata元数据进行初始化以及存储注册等一系列的操作
            //封装一个ConsumerModel，即把Consumer的数据封装起来放到对应的repository里
            initServiceMetadata(consumer);
            ...

            //调用ReferenceConfig的createProxy()方法创建服务的动态代理对象
            //这是构建整个代理的入口
            ref = createProxy(referenceParameters);

            //创建完动态代理后，把这个代理设置给serviceMetadata以及consumerModel
            serviceMetadata.setTarget(ref);
            serviceMetadata.addAttribute(PROXY_CLASS_REF, ref);

            consumerModel.setDestroyRunner(getDestroyRunner());
            consumerModel.setProxyObject(ref);
            consumerModel.initMethodModels();

            //检查一下Invoker是否可用
            checkInvokerAvailable();
        }

        private void checkInvokerAvailable() throws IllegalStateException {
            //根据check配置决定是否检测Provider是否可用
            //下面的invoker.isAvailable()调用的是MigrationInvoker的isAvailable()方法
            if (shouldCheck() && !invoker.isAvailable()) {
                //2-2 - No provider available.
                ...
                throw illegalStateException;
            }
        }
        ...
    }

    public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
        //服务引用时，它是一个MockClusterInvoker
        private volatile ClusterInvoker<T> invoker;
        //这是一个名为serviceDiscoveryInvoker的ClusterInvoker实现类的实例，也是一个MockClusterInvoker
        private volatile ClusterInvoker<T> serviceDiscoveryInvoker;
        //表示当前使用的Invoker，服务引用时，它是一个MockClusterInvoker
        private volatile ClusterInvoker<T> currentAvailableInvoker;
        ...

        @Override
        public boolean isAvailable() {
            //currentAvailableInvoker.isAvailable()会调用到MockClusterInvoker.isAvailable()的方法
            //最后又会调用RegistryDirectory.isAvailable()的方法
            return currentAvailableInvoker != null
                ? currentAvailableInvoker.isAvailable()
                : (invoker != null && invoker.isAvailable()) 
                || (serviceDiscoveryInvoker != null && serviceDiscoveryInvoker.isAvailable());
        }
        ...
    }

当Invoker的检查执行完毕后，就表示如下代码中"reference.get()"已执行完毕，已经生成好了动态代理的接口，于是可以继续往下执行。

当后续调用动态代理的接口时，便会进入InvokerInvocationHandler的invoke()方法中。

    public class Application {
        public static void main(String[] args) throws Exception {
            //Reference和ReferenceConfig是什么
            //Reference是一个引用，是对Provider端的一个服务实例的引用
            //ReferenceConfig这个服务实例的引用的一些配置
            //通过泛型传递了这个服务实例对外暴露的接口
            ReferenceConfig<DemoService> reference = new ReferenceConfig<>();

            //设置应用名称
            reference.setApplication(new ApplicationConfig("dubbo-demo-api-consumer"));

            //设置注册中心的地址，默认是ZooKeeper
            reference.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));

            //设置元数据上报地址
            reference.setMetadataReportConfig(new MetadataReportConfig("zookeeper://127.0.0.1:2181"));

            //设置要调用的服务的接口
            reference.setInterface(DemoService.class);

            //直接通过ReferenceConfig的get()方法来拿到一个DemoService接口
            //它是一个动态代理接口，只要被调用，便会通过底层调用Provider服务实例的对应接口
            DemoService service = reference.get();

            //下面调用动态代理的接口
            //会进入InvokerInvocationHandler的invoke()方法中
            String message = service.sayHello("dubbo");

            System.out.println(message);
            Thread.sleep(10000000L);
        }
    }

    //实现了JDK反射机制里的InvocationHandler接口
    public class InvokerInvocationHandler implements InvocationHandler {
        ...

        public InvokerInvocationHandler(Invoker<?> handler) {
            this.invoker = handler;
            URL url = invoker.getUrl();
            this.protocolServiceKey = url.getProtocolServiceKey();
            this.serviceModel = url.getServiceModel();
        }

        //通过这个代理对象可以拿到这个代理对象的接口
        //通过接口就可以定位到目标服务实例发布的服务接口，从而针对目标服务实例进行调用
        //拿到了调用目标服务实例的方法、传递进去的参数等信息，就可以进行完整的RPC调用
        //动态代理就是针对接口动态去生成该接口的实现类并进行调用

        //比如针对DemoService这个接口去生成一个实现类
        //由于这个实现类是动态生成的，那么它如何知道会有调用方去调用它的方法，以及又如何去执行？

        //为此，动态代理底层都会封装InvocationHandler
        //封装完后对动态代理所有方法的调用，都会跑到InvocationHandler这里来

        //这样，InvocationHandler的invoke方法就可以拿到：
        //Proxy动态代理对象、需要调用对象的哪个方法Method、以及被调用方法会传入的参数args

        //此时，对动态代理不同方法的调用以及具体方法被调用后会如何处理
        //都可以由如下invoke()方法发起远程调用由服务提供者来决定
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            //对于Object中定义的方法，直接调用Invoker对象的相应方法即可
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(invoker, args);
            }
            String methodName = method.getName();
            Class<?>[] parameterTypes = method.getParameterTypes();
            if (parameterTypes.length == 0) {
                if ("toString".equals(methodName)) {
                    //对toString()方法进行特殊处理
                    return invoker.toString();
                } else if ("$destroy".equals(methodName)) {
                    //对$destroy等方法的特殊处理
                    invoker.destroy();
                    return null;
                } else if ("hashCode".equals(methodName)) {
                    //对hashCode()方法进行特殊处理
                    return invoker.hashCode();
                }
            } else if (parameterTypes.length == 1 && "equals".equals(methodName)) {
                return invoker.equals(args[0]);
            }

            //创建RpcInvocation对象，后面会作为远程RPC调用的参数
            //Consumer端进行RPC调用时，必须要封装一个RpcInvocation
            //这个RpcInvocation会传递到Provider端
            //Provider端拿到请求数据后，也会封装一个RpcInvocation
            RpcInvocation rpcInvocation = new RpcInvocation(serviceModel, method.getName(), invoker.getInterface().getName(), protocolServiceKey, method.getParameterTypes(), args);

            if (serviceModel instanceof ConsumerModel) {
                rpcInvocation.put(Constants.CONSUMER_MODEL, serviceModel);
                rpcInvocation.put(Constants.METHOD_MODEL, ((ConsumerModel) serviceModel).getMethodModel(method));
            }

            //调用invoke()方法发起远程调用
            //拿到AsyncRpcResult之后，调用recreate()方法获取响应结果(或是Future)
            //下面的invoker其实就是装饰了MockClusterInvoker的MigrationInvoker
            return InvocationUtil.invoke(invoker, rpcInvocation);
        }
    }

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/86239b69-5d2e-42f3-a381-a51e9263cf24" />
