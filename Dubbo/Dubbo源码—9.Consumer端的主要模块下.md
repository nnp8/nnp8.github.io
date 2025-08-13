# Dubbo源码—9.Consumer端的主要模块下

**大纲**

**1.Consumer端动态代理构建源码**

**2.Dubbo动态代理调用+RPC调用执行链路分析**

**3.Netty网络客户端构建链路分析**

**4.Netty网络写数据同步和异步的分析**

**5.RPC调用链路中的Future异步分析**

**6.AsyncRpcResult等待结果的源码分析**

**7.Dubbo RPC请求对象如何进行封装**

**8.RPC调用链路中引入Filter链条部分**

**9.Dubbo自实现Threadlocal机制剖析**

**10.Consumer上下文处理过滤器深入剖析**

**11.RpcInvocation在调用链路中的流转分析**

**12.Dubbo中的Netty编码组件分析**

**13.基于Dubbo协议编码请求对象完成Dubbo序列化**

**14.基于Dubbo协议解码二进制数据完成Dubbo反序列化**

**15.请求Body解码与反序列化过程分析**

<br>

**1.Consumer端动态代理构建源码**

首先invoker的创建整个流程如下：

```
-> ReferenceConfig.get()
-> getScopeModel().getDeployer().start()
-> AbstractMethodConfig.getScopeModel()
-> ReferenceConfig.init()

-> AbstractConfig.refresh()
-> ReferenceConfigBase.preProcessRefresh()
-> AbstractConfig.assignProperties()
-> AbstractConfig.processExtraRefresh()
-> AbstractInterfaceConfig.processExtraRefresh()
-> ReferenceConfig.postProcessRefresh()
-> ReferenceConfig.checkAndUpdateSubConfigs()

-> AbstractInterfaceConfig.initServiceMetadata()
-> ReferenceConfig#repository.registerService()
-> ReferenceConfig#repository.registerConsumer()
-> ReferenceConfig.createProxy()

-> ReferenceConfig.shouldJvmRefer()
-> InjvmProtocol.isInjvmRefer()
-> ReferenceConfig.createInvokerForLocal()
-> ReferenceConfig.createInvokerForRemote()

-> ReferenceConfig#protocolSPI.refer(interfaceClass, curUrl)
-> Protocol$Adaptive.refer()
-> ProtocolSerializationWrapper.refer()
-> ProtocolFilterWrapper.refer()
-> ProtocolListenerWrapper.refer()
-> RegistryProtocol.refer()

-> RegistryProtocol#Cluster.getCluster()
-> RegistryProtocol.doRefer()
-> RegistryProtocol.getMigrationInvoker()
-> new ServiceDiscoveryMigrationInvoker<T>()
-> new MigrationInvoker()
-> RegistryProtocol.interceptInvoker()
-> RegistryProtocol.findRegistryProtocolListeners()
-> MigrationRuleListener.onRefer()
-> MigrationRuleHandler.refreshInvoker() / MigrationInvoker.refreshServiceDiscoveryInvoker()
-> MigrationInvoker.migrateToApplicationFirstInvoker()
-> MigrationInvoker.refreshInterfaceInvoker()
-> InterfaceCompatibleRegistryProtocol.getInvoker() / InterfaceCompatibleRegistryProtocol.getServiceDiscoveryInvoker()
-> RegistryProtocol.doCreateInvoker()

-> directory.buildRouterChain()
-> DynamicDirectory.buildRouterChain()
-> RouterChain.buildChain()
-> new RouterChain<>()
-> RouterChain.initWithRouters()
-> RouterChain.initWithStateRouters()

-> RegistryDirectory.subscribe()
-> DynamicDirectory.subscribe()
-> ListenerRegistryWrapper.subscribe()
-> FailbackRegistry.subscribe()
-> AbstractRegistry.subscribe() + FailbackRegistry.doSubscribe()
-> ZookeeperRegistry.doSubscribe() + create()/notify()
-> FailbackRegistry.notify()
-> AbstractRegistry.notify()
-> RegistryDirectory.notify()
-> RegistryDirectory.toInvokers()
-> DubboProtocol.refer()
-> DubboProtocol.protocolBindingRefer()
-> DubboProtocol.getClients()

-> DubboProtocol.initClient()
-> Exchangers.connect(url, requestHandler)
-> getExchanger(url).connect(url, handler) 
-> Exchanger.connect(url, handler)
-> HeaderExchanger.connect()
-> Transporters.connect()
-> Transporters.getTransporter(url).connect(url, handler)
-> NettyTransporter.connect()
-> new NettyClient(url, handler), 构建NettyClient发起连接
-> new AbstractClient(url, handler)
-> NettyClient.doOpen() + NettyClient.doConnect()
-> new DubboInvoker()

-> RegistryProtocol.doCreateInvoker()#cluster.join()
-> MockClusterWrapper.join()
-> AbstractCluster.join()
-> FailoverCluster.doJoin()
-> new FailoverClusterInvoker<>(directory)

-> ReferenceConfig.createInvokerForRemote()#new StaticDirectory()
-> Cluster.getCluster().join()
```

完成invoker的创建后，接下来便会根据invoker生成动态代理：

```
-> ReferenceConfig.createProxy()
-> ReferenceConfig.createInvokerForRemote()
-> ReferenceConfig.createInvokerForRemote()#protocolSPI.refer()
-> ReferenceConfig.createInvokerForRemote()#Cluster.getCluster().join()
-> ReferenceConfig.createProxy()#proxyFactory.getProxy()
```

在ReferenceConfig的createProxy()方法中，调用的proxyFactory.getProxy()其实是StubProxyFactoryWrapper的getProxy()方法，其中生成的动态代理类主要实现了如下三个接口：

```
一.org.apache.dubbo.demo.DemoService
二.org.apache.dubbo.rpc.service.EchoService
三.org.apache.dubbo.rpc.service.Destoryable
```

```
-> ReferenceConfig.createProxy()#proxyFactory.getProxy()
-> ProxyFactory$Adaptive.getProxy()
-> StubProxyFactoryWrapper.getProxy()
-> AbstractProxyFactory.getProxy()
-> JavassistProxyFactory.getProxy()
-> Proxy.getProxy()
-> new InvokerInvocationHandler()

public class StubProxyFactoryWrapper implements ProxyFactory {
    ...
    @Override
    public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
        //为什么它的名字叫stub打桩？
        //stub这个名字一般代表的是远程网络访问的动态代理；
        //通过这个stub可以对远程的机器进行网络访问；
        //stub打桩思想，是在网络远程访问里经常会使用的一种方法；
        //比如机器A和机器B之间进行的远程网络访问，必然要通过网络来进行连接和访问；
        //机器A在进行远程RPC访问时，其调用代码，最好不要直接写网络通信的逻辑，而是写针对某个接口类型对象的方法调用；
        //这样在机器A上面进行RPC调用时就像是在本地调用一样，都是直接调用某个对象的方法；
        //然后在这个对象的方法里，再去写网络通信的逻辑和目标机器B进行网络连接，并发送调用请求；
        //所以在机器A上会针对调用的那个接口，来动态去生成一个实现了该接口的类，也就是动态代理的代理类或者是stub打桩，
        //类似于在机器A上打下的一个伪装成目标类的桩，然后针对打的这个stub桩去进行调用，stub内部会编写网络通信代码实现跨机器的访问；
        //这就是远程网络连接和通信的stub打桩思想；
        //下面使用了抽象代理工厂来获取真正的动态代理，下面的proxyFactory.getProxy()会跑到AbstractProxyFactory.getProxy()中去
        T proxy = proxyFactory.getProxy(invoker, generic);
        ...
    }
    ...
}

public abstract class AbstractProxyFactory implements ProxyFactory {
    ...
    public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
        //when compiling with native image, ensure that the order of the interfaces remains unchanged
        //记录要代理的接口
        LinkedHashSet<Class<?>> interfaces = new LinkedHashSet<>();
        ClassLoader classLoader = getClassLoader(invoker);
        
        //获取URL中interfaces参数指定的接口
        String config = invoker.getUrl().getParameter(INTERFACES);
        if (StringUtils.isNotEmpty(config)) {
            //按照逗号切分interfaces参数，得到接口集合
            String[] types = COMMA_SPLIT_PATTERN.split(config);
            for (String type : types) {//遍历接口集合，并记录这些接口信息
                //对每个接口都拿出对应的class对象，放到interfaces集合里去
                interfaces.add(ReflectUtils.forName(classLoader, type));
            }
        }
        ...
        
        //获取Invoker中type字段指定的接口
        interfaces.add(invoker.getInterface());
        
        //添加EchoService、Destroyable两个默认接口
        interfaces.addAll(Arrays.asList(INTERNAL_INTERFACES));
        
        //调用抽象的getProxy()重载方法
        //dubbo的动态代理技术：javassist(动态拼接类的代码字符串，进行动态编译来动态生成一个类)、jdk(通过jdk提供的API利用反射来生成动态代理)
        //默认下面会调用子类JavassistProxyFactory的getProxy()方法
        return getProxy(invoker, interfaces.toArray(new Class<?>[0]));
    }
    ...
}

public class JdkProxyFactory extends AbstractProxyFactory {
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        //下面的Proxy是JDK反射机制里的Proxy
        return (T) Proxy.newProxyInstance(invoker.getInterface().getClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
    }
    ...
}

public class JavassistProxyFactory extends AbstractProxyFactory {
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        try {
            //下面的Proxy是Dubbo自己实现的Proxy
            return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
        } catch (Throwable fromJavassist) {
            T proxy = jdkProxyFactory.getProxy(invoker, interfaces);
            return proxy;
        }
    }
    ...
}

public class Proxy {
    ...
    public static Proxy getProxy(Class<?>... ics) {
        ...
        // ClassLoader from App Interface should support load some class from Dubbo
        ClassLoader cl = ics[0].getClassLoader();
        ProtectionDomain domain = ics[0].getProtectionDomain();
        
        //use interface class name list as key.
        //生成的动态代理类主要实现了如下三个接口：
        //1.org.apache.dubbo.demo.DemoService、
        //2.org.apache.dubbo.rpc.service.EchoService、
        //3.org.apache.dubbo.rpc.service.Destoryable
        String key = buildInterfacesKey(cl, ics);
        
        // get cache by class loader.
        final Map<String, Proxy> cache;
        synchronized (PROXY_CACHE_MAP) {
            cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new ConcurrentHashMap<>());
        }
        
        //接口列表将会作为第二层集合的Key
        Proxy proxy = cache.get(key);
        if (proxy == null) {
            synchronized (ics[0]) {
                proxy = cache.get(key);
                if (proxy == null) {
                    // create Proxy class.
                    proxy = new Proxy(buildProxyClass(cl, ics, domain));
                    cache.put(key, proxy);
                }
            }
        }
        return proxy;
    }
    
    private static Class<?> buildProxyClass(ClassLoader cl, Class<?>[] ics, ProtectionDomain domain) {
        ClassGenerator ccp = null;
        ccp = ClassGenerator.newInstance(cl);
        Set<String> worked = new HashSet<>();
        List<Method> methods = new ArrayList<>();
        String pkg = ics[0].getPackage().getName();
        Class<?> neighbor = ics[0];
        
        for (Class<?> ic : ics) {
            String npkg = ic.getPackage().getName();
            //向ClassGenerator中添加接口
            ccp.addInterface(ic);
            
            for (Method method : ic.getMethods()) {//遍历接口中的每个方法
                String desc = ReflectUtils.getDesc(method);
                //跳过已经重复方法以及static方法
                if (worked.contains(desc) || Modifier.isStatic(method.getModifiers())) {
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
                StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                for (int j = 0; j < pts.length; j++) {
                    code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(';');
                }
                code.append(" Object ret = handler.invoke(this, methods[").append(ix).append("], args);");
                
                //生成return语句
                if (!Void.TYPE.equals(rt)) {
                    code.append(" return ").append(asArgument(rt, "ret")).append(';');
                }
                
                //将生成好的方法添加到ClassGenerator中缓存
                methods.add(method);
                ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
            }
        }

        //create ProxyInstance class.
        //生成并设置代理类类名
        String pcn = neighbor.getName() + "DubboProxy" + PROXY_CLASS_COUNTER.getAndIncrement();
        ccp.setClassName(pcn);
        
        //添加字段，一个是前面生成的methods集合，另一个是InvocationHandler对象
        ccp.addField("public static java.lang.reflect.Method[] methods;");
        ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
        
        //添加构造方法
        ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
        
        //默认构造方法
        ccp.addDefaultConstructor();
        Class<?> clazz = ccp.toClass(neighbor, cl, domain);
        clazz.getField("methods").set(null, methods.toArray(new Method[0]));
        return clazz;
    }
    ...
}

//实现了JDK反射机制里的InvocationHandler接口
public class InvokerInvocationHandler implements InvocationHandler {
    ...
    public InvokerInvocationHandler(Invoker<?> handler) {
        this.invoker = handler;
        URL url = invoker.getUrl();
        this.protocolServiceKey = url.getProtocolServiceKey();
        this.serviceModel = url.getServiceModel();
    }
}
```

<br>

**2.Dubbo动态代理调用+RPC调用执行链路分析**

Dubbo动态代理调用入口如下service.sayHello("dubbo")：

```
public class Application {
    public static void main(String[] args) {
        //ReferenceConfig是什么，其实就是服务引用
        //Reference是拥有一个provider服务实例的引用
        //ReferenceConfig便是要调用的服务实例的引用配置
        //下面一行代码通过泛型传递了调用的服务实例对外暴露的接口
        ReferenceConfig<DemoService> reference = new ReferenceConfig<>();
        
        //应用名称
        reference.setApplication(new ApplicationConfig("dubbo-demo-api-consumer"));
        
        //设置注册中心的地址，默认是zookeeper
        reference.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
        
        //设置元数据上报地址
        reference.setMetadataReportConfig(new MetadataReportConfig("zookeeper://127.0.0.1:2181"));
        
        //设置要调用的服务的接口
        reference.setInterface(DemoService.class);
        
        //直接通过ReferenceConfig的get方法来拿到一个DemoService接口
        //它是一个动态代理接口，只要被调用，便会通过底层调用provider服务实例的对应接口
        DemoService service = reference.get();
        
        //下面的动态代理接口一旦被调用，首先就会跑到InvokerInvocationHandler.invoke()方法中
        String message = service.sayHello("dubbo");
        System.out.println(message);
    }
}
```

```
接着会进入InvokerInvocationHandler.invoke()方法.
接着会调用InvocationUtil.invoke()里的"invoker.invoke(rpcInvocation).recreate()".
这个invoker其实就是装饰了MockClusterInvoker的MigrationInvoker.
而MigrationInvoker.invoke()方法会调用MockClusterInvoker.invoke()方法.
接着会继续调用AbstractCluster内部类ClusterFilterInvoker的invoke()方法.
接着会继续调用FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法.
接着会继续调用ConsumerContextFilter的invoke()方法.
接着又会调用回FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法.
最终会调用AbstractClusterInvoker.invoke()方法.
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/834da0e4-7319-4aa5-bb18-210fca62926d" />

执行链路如下：

```
-> InvokerInvocationHandler.invoke()
-> InvocationUtil.invoke()
-> invoker.invoke(rpcInvocation).recreate()
-> MigrationInvoker.invoke()
-> MockClusterInvoker.invoke()
-> AbstractCluster.ClusterFilterInvoker.invoke()
-> FilterChainBuilder.CallbackRegistrationInvoker.invoke()
-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
-> ConsumerContextFilter.invoke()
-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
-> FutureFilter.invoke()
-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
-> MonitorFilter.invoke()
-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
-> RouterSnapshotFilter.invoke()
-> AbstractClusterInvoker.invoke()
(1)
-> AbstractClusterInvoker.list()
-> AbstractDirectory.list()
-> DynamicDirectory.list()
-> RouterChain.route()
(2)
-> AbstractClusterInvoker.initLoadBalance()
(3)
-> FailoverClusterInvoker.doInvoke()
-> AbstractClusterInvoker.select()
-> AbstractClusterInvoker.doSelect()
-> AbstractLoadBalance.select()
-> RandomLoadBalance.doSelect()
-> AbstractClusterInvoker.invokeWithContext()
-> ListenerInvokerWrapper.invoke()
-> AbstractInvoker.invoke()
-> AbstractInvoker.waitForResultIfSync()
-> AsyncRpcResult.get()

//实现了JDK反射机制里的InvocationHandler接口
public class InvokerInvocationHandler implements InvocationHandler {
    ...
    //通过这个代理对象可以拿到这个代理对象的接口，通过接口就可以定位到目标服务实例发布的服务接口，从而针对目标服务实例进行调用
    //拿到了调用目标服务实例的方法、传递进去的参数等信息，就可以进行完整的RPC调用
    //动态代理就是针对接口动态去生成该接口的实现类并进行调用
    //比如针对DemoService这个接口去生成一个实现类，由于这个实现类是动态生成的，那么它如何知道会有调用方去调用它的方法，以及又如何去执行？
    //为此，动态代理底层都会封装InvocationHandler，封装完后对动态代理所有方法的调用，都会跑到InvocationHandler这里来
    //这样，InvocationHandler的invoke方法就可以拿到proxy动态代理对象、需要调用对象的哪个方法Method、以及被调用方法会传入的参数args
    //此时，对动态代理不同方法的调用以及具体方法被调用后会如何处理，都可由如下invoke()方法发起远程调用由服务提供者来决定
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //对于Object中定义的方法，直接调用Invoker对象的相应方法即可
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (parameterTypes.length == 0) {
            if ("toString".equals(methodName)) {//对toString()方法进行特殊处理
                return invoker.toString();
            } else if ("$destroy".equals(methodName)) {//对$destroy等方法的特殊处理
                invoker.destroy();
                return null;
            } else if ("hashCode".equals(methodName)) {//对hashCode()方法进行特殊处理
                return invoker.hashCode();
            }
        } else if (parameterTypes.length == 1 && "equals".equals(methodName)) {
            return invoker.equals(args[0]);
        }
        
        //创建RpcInvocation对象，后面会作为远程RPC调用的参数
        //consumer端进行RPC调用时，必须要封装一个RpcInvocation
        //这个RpcInvocation会传递到provider端，provider端拿到请求数据后，也会封装一个RpcInvocation
        RpcInvocation rpcInvocation = new RpcInvocation(serviceModel, method.getName(), invoker.getInterface().getName(), protocolServiceKey, method.getParameterTypes(), args);
        if (serviceModel instanceof ConsumerModel) {
            rpcInvocation.put(Constants.CONSUMER_MODEL, serviceModel);
            rpcInvocation.put(Constants.METHOD_MODEL, ((ConsumerModel) serviceModel).getMethodModel(method));
        }
        
        //调用invoke()方法发起远程调用，拿到AsyncRpcResult之后，调用recreate()方法获取响应结果(或是Future)
        //下面的invoker其实就是装饰了MockClusterInvoker的MigrationInvoker
        return InvocationUtil.invoke(invoker, rpcInvocation);
    }
}

public class InvocationUtil {
    ...
    public static Object invoke(Invoker<?> invoker, RpcInvocation rpcInvocation) throws Throwable {
        ...
        //为什么要进行recreate？
        //recreate是为了对拿到的结果进行一个拷贝，将拷贝出来的结果对象返回给业务方去使用
        //这样Dubbo框架内部自己可以持有一个结果对象，避免了和业务方共享持有和访问，而产生相互的影响
        return invoker.invoke(rpcInvocation).recreate();
    }
}

public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
    private volatile ClusterInvoker<T> invoker;//服务引用时，它是一个MockClusterInvoker，参考RegistryProtocol.refer()的调用栈
    private volatile ClusterInvoker<T> serviceDiscoveryInvoker;//这是一个名为serviceDiscoveryInvoker的ClusterInvoker实现类的实例
    private volatile ClusterInvoker<T> currentAvailableInvoker;//表示当前使用的Invoker，服务引用时，它是一个MockClusterInvoker
    ...
    
    //真正在执行Invoker调用时，可以在这里看到MigrationInvoker里的逻辑
    //为什么会叫MigrationInvoker?
    //因为MigrationInvoker有两个Invoker：一个是invoker，一个是serviceDiscoveryInvoker；
    //执行这里的invoke()方法时会根据不同的条件去切换这两个invoker，将它们之一赋值给真正负责执行invoke()方法的currentAvailableInvoker；
    //也就是有一个针对currentAvailableInvoker进行切换的过程(根据不同的条件来切换是invoker还是serviceDiscoveryInvoker)，所以migration就是从这里来的
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        ...
        //下面会调用MockClusterInvoker.invoke()方法
        return currentAvailableInvoker.invoke(invocation);
    }
    ...
}

public class MockClusterInvoker<T> implements ClusterInvoker<T> {
    //核心其实是是否启用降级机制调用doMockInvoke()
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result result;
        //从URL中获取方法对应的mock配置
        String value = getUrl().getMethodParameter(invocation.getMethodName(), MOCK_KEY, Boolean.FALSE.toString()).trim();
        if (ConfigUtils.isEmpty(value)) {
            //no mock
            //若mock参数未配置或是配置为false，则不会开启Mock机制，直接调用底层的Invoker
            //这里会直接发起正常的调用
            //每一层Invoker都会去负责自己的事情，对Invoker的调用使用了严格的责任链模式，运用了责任链模式的思想
            //A Invoker->B Invoker->C Invoker->D Invoker
            //在发起RPC调用时，由于会涉及到很多的机制，比如降级调用机制(mock)，集群容错机制，负载均衡机制等
            //此时如果只有一个Invoker，那么它里面的代码就会很多很复杂，所以才用上了责任链模式
            //下面会调用AbstractCluster内部类ClusterFilterInvoker的invoke()方法，进行正常的RPC调用
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith(FORCE_KEY)) {
            //force:direct mock
            //若mock参数配置为force，则表示强制开始mock，直接调用doMockInvoke()方法
            //也就是对这个服务实例的调用使用了force-mock，强制性进行mock，根本就不会发起真正的RPC调用
            result = doMockInvoke(invocation, null);
        } else {
            //fail-mock
            //如果mock配置的不是force，则配置的是fail，会继续调用Invoker对象的invoke()方法进行请求
            try {
                //下面会调用AbstractCluster内部类ClusterFilterInvoker的invoke()方法，进行正常的RPC调用
                result = this.invoker.invoke(invocation);
                if (result.getException() != null && result.getException() instanceof RpcException) {
                    RpcException rpcException = (RpcException) result.getException();
                    if (rpcException.isBiz()) {
                        throw rpcException;
                    } else {
                        //如果RPC调用出现异常，那么会在这里直接进行mock调用
                        result = doMockInvoke(invocation, rpcException);
                    }
                }
            } catch (RpcException e) {
                //如果是业务异常，会直接抛出
                if (e.isBiz()) {
                    throw e;
                }
                //如果是非业务异常，会调用doMockInvoke()方法直接进行mock调用
                result = doMockInvoke(invocation, e);
            }
        }
        return result;
    }
    ...
}

public abstract class AbstractCluster implements Cluster {
    static class ClusterFilterInvoker<T> extends AbstractClusterInvoker<T> {
        ...
        @Override
        public Result invoke(Invocation invocation) throws RpcException {
            //下面会调用FilterChainBuilder的内部类CallbackRegistrationInvoker的invoke()方法
            return filterInvoker.invoke(invocation);
        }
        ...
    }
    ...
}

@SPI(value = "default", scope = APPLICATION)
public interface FilterChainBuilder {
    class CallbackRegistrationInvoker<T, FILTER extends BaseFilter> implements Invoker<T> {
        ...
        public Result invoke(Invocation invocation) throws RpcException {
            //下面会调用FilterChainBuilder$CopyOfFilterChainNode.invoke()
            Result asyncResult = filterInvoker.invoke(invocation);
            ...
    	}
     	...
    }
    
    class CopyOfFilterChainNode<T, TYPE extends Invoker<T>, FILTER extends BaseFilter> implements Invoker<T> {
        ...
        @Override
        public Result invoke(Invocation invocation) throws RpcException {
            Result asyncResult;
            try {
                InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Filter " + filter.getClass().getName() + " invoke.");
                //比如consumer端处理时下面会调用ConsumerContextFilter的invoke()方法
                //provider端处理时下面会最终调用AbstractProxyInvoker.invoke()方法
                asyncResult = filter.invoke(nextNode, invocation);
            } catch (Exception e) {
                ...
            }
            ...
        }
    }
    ...
}

@Activate(group = CONSUMER, order = Integer.MIN_VALUE)
public class ConsumerContextFilter implements ClusterFilter, ClusterFilter.Listener {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        ...
        //下面又会调用FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法
        return invoker.invoke(invocation);
    }
    ...
}

public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        //检测当前Invoker是否已销毁
        checkWhetherDestroyed();
        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Router route.");
        
        //下面会根据invocation信息列出所有的Invoker，也就是通过DynamicDirectory.list()方法进行服务发现
        //由于通过Directory获取Invoker对象列表，通过了解RegistryDirectory可知，其中已经调用了Router进行过滤
        //从而可以知道有哪些服务实例，有哪些Invoker，一个服务实例会对应一个Invoker，所以目标服务实例集群已变成invokers了
        List<Invoker<T>> invokers = list(invocation);
        InvocationProfilerUtils.releaseDetailProfiler(invocation);
        
        //通过SPI加载LoadBalance实例，也就是在这里选择出对应的负载均衡策略
        //比如下面默认会获取到一个RandomLoadBalance
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Cluster " + this.getClass().getName() + " invoke.");
        try {
            //调用由子类实现的抽象方法doInvoke()，下面会由子类FailoverClusterInvoker执行doInvoke()方法
            return doInvoke(invocation, invokers, loadbalance);
        } finally {
            InvocationProfilerUtils.releaseDetailProfiler(invocation);
        }
    }
    
    protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
        return getDirectory().list(invocation);//通过DynamicDirectory.list()进行服务发现
    }
    
    @Override
    public Directory<T> getDirectory() {
        return directory;
    }
    ...
}
```

AbstractClusterInvoker的invoke()方法首先会通过DynamicDirectory.list()方法列出所有的Invoker，然后该方法会调用AbstractClusterInvoker.initLoadBalance()初始化负载均衡，最后该方法会调用FailoverClusterInvoker的doInvoke()方法。

```
-> FailoverClusterInvoker.doInvoke()
-> AbstractClusterInvoker.select()
-> AbstractClusterInvoker.doSelect()
-> AbstractLoadBalance.select()
-> RandomLoadBalance.doSelect()
-> AbstractClusterInvoker.invokeWithContext()
-> ListenerInvokerWrapper.invoke()
-> AbstractInvoker.invoke()
-> AbstractInvoker.doInvokeAndReturn()
-> DubboInvoker.doInvoke()
-> AbstractInvoker.getCallbackExecutor()
-> DefaultExecutorRepository.getExecutor()
-> AbstractInvoker.waitForResultIfSync()
-> AsyncRpcResult.get()

public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {
    //dubbo有不同的集群容错策略，通过具体的配置可以改变使用不同的策略
    //还可以根据自己的需求，自定义集群容错策略，然后进行配置让dubbo采用自定义的集群容错策略
    //而这使用到了策略模式，dubbo会深度使用策略模式，它会针对某个环节设计出多种可替换的不同的策略
    //同时会提供一个默认的策略实现，然后通过配置可以使用其他策略
    ...

    //该方法会找一个Invoker，把invocation交给多个Invokers里的一个去发起RPC调用，其中loadbalance会实现负载均衡
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        //下面先做一个引用赋值
        List<Invoker<T>> copyInvokers = invokers;
        
        //检查Invokers
        checkInvokers(copyInvokers, invocation);
        
        //从RPC调用里提取一个method方法名称，从而确定需要调用的是哪个方法
        String methodName = RpcUtils.getMethodName(invocation);
        
        //计算最多调用次数；因为Failover策略是，如果发现调用不成功会进行重试，但默认会最多调用3次来让调用成功
        int len = calculateInvokeTimes(methodName);
        
        //retry loop.
        RpcException le = null;//last exception.
        
        //构建了一个跟invokers数量相等的一个list
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size());//invoked invokers.
        
        //基于计算出的调用次数，构建一个set；如果调用次数为3，那么意味着最多会调用3个provider服务实例
        Set<String> providers = new HashSet<String>(len);
        
        //对len次数进行循环
        for (int i = 0; i < len; i++) {
            //到i>0时，表示的是第一次调用失败，要开始进行重试了
            if (i > 0) {
                //检查当前服务实例是否被销毁
                checkWhetherDestroyed();
                
                //此时要调用DynamicDirectory进行一次invokers列表刷新
                //因为第一次调用都失败了，所以有可能invokers列表出现了变化，因而需要刷新一下invokers列表
                copyInvokers = list(invocation);
                
                //再次check一下invokers是否为空
                checkInvokers(copyInvokers, invocation);
            }
            
            //1.下面是AbstractClusterInvoker.select()方法，它会选择一个Invoker出来，具体逻辑是：
            //先尝试用负载均衡算法去选，如果选出的Invoker是选过的或者不可用的，那么就直接reselect
            //也就是对选过的找一个可用的，对选不出的直接挑选下一个
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);
            RpcContext.getServiceContext().setInvokers((List) invoked);
            
            boolean success = false;
            try {
                //2.AbstractClusterInvoker.invokeWithContext()方法会基于该invoker发起RPC调用，并拿到一个result
                Result result = invokeWithContext(invoker, invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("...", le);
                }
                success = true;
                return result;
            } catch (RpcException e) {
                //如果本次RPC调用失败了，那么就会有异常抛出来
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                if (!success) {
                    //下面会把出现RPC调用异常的进行设置，把本次调用失败的invoker地址添加到providers
                    providers.add(invoker.getUrl().getAddress());
                }
            }
        }
        
        //如果最后抛出如下这个异常，则说明本次RPC调用彻底失败了
        throw new RpcException(le.getCode(), "...", le.getCause() != null ? le.getCause() : le);
    }
}

public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    ...
    //第一个参数是此次使用的LoadBalance实现，第二个参数Invocation是此次服务调用的上下文信息
    //第三个参数是待选择的Invoker集合，第四个参数用来记录负载均衡已经选出来、尝试过的Invoker集合
    protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
        //invokers不能为空，为空的话就直接返回
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        
        //获取调用方法名，调用的方法名称处理逻辑是：RPC调用如果是null的话，method方法名就是一个空字符串；否则就是RPC调用里的方法名称
        String methodName = invocation == null ? StringUtils.EMPTY_STRING : invocation.getMethodName();
        
        //invokers代表了服务集群地址，下面会先获取第一个invoker，然后拿到它的URL，再去获取到对应到的sticky，其默认值是false
        //获取sticky配置，sticky表示粘滞连接，所谓粘滞连接是指Consumer会尽可能的调用同一个Provider节点，除非这个Provider无法提供服务
        boolean sticky = invokers.get(0).getUrl().getMethodParameter(methodName, CLUSTER_STICKY_KEY, DEFAULT_CLUSTER_STICKY);
        
        //ignore overloaded method
        //检测invokers列表是否包含stickyInvoker，如果不包含则说明stickyInvoker代表的服务提供者挂了，此时需要将其置空
        if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
            stickyInvoker = null;
        }
        
        //ignore concurrency problem
        //如果开启了粘滞连接特性，需要先判断这个Provider节点是否已经重试过了，下面的判断前半部分表示粘滞连接，后半部分表示stickyInvoker未重试过
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
            //检测当前stickyInvoker是否可用，如果可用，直接返回stickyInvoker
            if (availableCheck && stickyInvoker.isAvailable()) {
                return stickyInvoker;
            }
        }
        
        //执行到这里，说明前面的stickyInvoker为空，或者不可用
        //这里会继续调用doSelect选择新的Invoker对象，也就是基于LoadBalance去进行负载均衡选择一个Invoker出来
        Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);
        
        //是否开启粘滞，更新stickyInvoker为选择出来的invoker
        //sticky表示粘滞连接(粘滞调用)，所谓粘滞连接(粘滞调用)是指Consumer会尽可能的调用同一个Provider节点，除非这个Provider无法提供服务
        //也就是把一个Consumer端和一个Provider端粘在一起
        if (sticky) {
            stickyInvoker = invoker;
        }
        return invoker;
    }
    
    private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
        //判断是否需要进行负载均衡，Invoker集合为空，直接返回null
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
        
        //只有一个Invoker对象，直接返回即可
        //如果invokers的数量就1个，那么目标provider服务实例就一个，所以直接返回invokers里的第一个即可
        if (invokers.size() == 1) {
            Invoker<T> tInvoker = invokers.get(0);
            checkShouldInvalidateInvoker(tInvoker);
            return tInvoker;
        }
        
        //通过LoadBalance实现选择Invoker对象，即基于负载均衡的策略和算法选择出一个Invoker
        Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);
        
        //Invoke是否已经尝试调用过但是失败了
        boolean isSelected = selected != null && selected.contains(invoker);
        
        //Invoker是否不可用
        boolean isUnavailable = availableCheck && !invoker.isAvailable() && getUrl() != null;
        if (isUnavailable) {
            invalidateInvoker(invoker);
        }
        
        //如果LoadBalance选出的Invoker对象，已经尝试请求过了或不可用，则需要调用reselect()方法进行重选
        if (isSelected || isUnavailable) {
            //调用reselect()方法重选
            Invoker<T> rInvoker = reselect(loadbalance, invocation, invokers, selected, availableCheck);
            if (rInvoker != null) {
                //如果重选的Invoker对象不为空，则直接返回这个rInvoker
                invoker = rInvoker;
            } else {
                //如果选来选去都是空，那么对当前Invoker就直接选择它的下一个invoker即可
                int index = invokers.indexOf(invoker);
                
                //如果重选的Invoker对象为空，则返回该Invoker的下一个Invoker对象
                invoker = invokers.get((index + 1) % invokers.size());
            }
        }
        return invoker;
    }
    ...
}

public abstract class AbstractLoadBalance implements LoadBalance {
    ...
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        if (CollectionUtils.isEmpty(invokers)) {
            //Invoker集合为空，直接返回null
            return null;
        }
        
        //Invoker集合只包含一个Invoker，则直接返回该Invoker对象
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
        
        //Invoker集合包含多个Invoker对象时，交给doSelect()方法处理，这是个抽象方法，留给子类具体实现
        return doSelect(invokers, url, invocation);
    }
    
    protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
    ...
}

public class RandomLoadBalance extends AbstractLoadBalance {
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        //Number of invokers
        //先拿到目标服务实例集群的invokers数量
        int length = invokers.size();
        if (!needWeightLoadBalance(invokers, invocation)) {
            //举个例子，会直接基于一个随机的类，通过其nextInt()方法拿到invokers数量范围之内的机器对应的invoker
            return invokers.get(ThreadLocalRandom.current().nextInt(length));
        }
        
        //什么是权重？
        //对于权重越高的invoker它被调用的几率会越高一些，而这里随机负载均衡的invokers它们被调用到的机会/几率是相同的
        //Every invoker has the same weight?
        boolean sameWeight = true;
        
        //the maxWeight of every invokers, the minWeight = 0 or the maxWeight of the last invoker
        //计算每个Invoker对象对应的权重，并填充到weights[]数组中
        int[] weights = new int[length];
        
        //The sum of weights
        int totalWeight = 0;
        for (int i = 0; i < length; i++) {
            //计算每个Invoker的权重，以及总权重totalWeight
            int weight = getWeight(invokers.get(i), invocation);
            //Sum
            totalWeight += weight;
            //save for later use
            weights[i] = totalWeight;
            //检测每个Provider的权重是否相同
            if (sameWeight && totalWeight != weight * (i + 1)) {
                sameWeight = false;
            }
        }
        
        //各个Invoker权重值不相等时，计算随机数落在哪个区间上
        if (totalWeight > 0 && !sameWeight) {
            //If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            //随机获取一个[0, totalWeight) 区间内的数字
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            
            //Return a invoker based on the random value.
            for (int i = 0; i < length; i++) {
                if (offset < weights[i]) {
                    return invokers.get(i);
                }
            }
        }
        
        //If all invokers have the same weight value or totalWeight=0, return evenly.
        //各个Invoker权重值相同时，随机返回一个Invoker即可
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }
    ...
}

public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    ...
    protected Result invokeWithContext(Invoker<T> invoker, Invocation invocation) {
        setContext(invoker);
        Result result;
        try {
            if (ProfilerSwitch.isEnableSimpleProfiler()) {
                InvocationProfilerUtils.enterProfiler(invocation, "Invoker invoke. Target Address: " + invoker.getUrl().getAddress());
            }
            //下面会调用ListenerInvokerWrapper.invoke()方法
            result = invoker.invoke(invocation);
        } finally {
            clearContext(invoker);
            InvocationProfilerUtils.releaseSimpleProfiler(invocation);
        }
        return result;
    }
}

public class ListenerInvokerWrapper<T> implements Invoker<T> {
    private final Invoker<T> invoker;//底层被修饰的Invoker对象
    private final List<InvokerListener> listeners;//监听器集合
    ...

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        //下面会调用AbstractInvoker.invoke()方法
        return invoker.invoke(invocation);
    }
}

public abstract class AbstractInvoker<T> implements Invoker<T> {
    ...
    @Override
    public Result invoke(Invocation inv) throws RpcException {
        //首先将传入的Invocation转换为RpcInvocation
        RpcInvocation invocation = (RpcInvocation) inv;
        //prepare rpc invocation
        prepareInvocation(invocation);
        //do invoke rpc invocation and return async result，RPC调用返回的结果是异步的:async
        AsyncRpcResult asyncResult = doInvokeAndReturn(invocation);
        //wait rpc result if sync
        //默认情况下发起的RPC请求是异步化操作，但如果需要同步的话，那么是可以在这里等待同步的结果
        waitForResultIfSync(asyncResult, invocation);
        return asyncResult;
    }
    
    private AsyncRpcResult doInvokeAndReturn(RpcInvocation invocation) {
        AsyncRpcResult asyncResult;
        ...
        //调用子类实现的doInvoke()方法，比如DubboInvoker.doInvoke()方法
        asyncResult = (AsyncRpcResult) doInvoke(invocation);
        ...
        return asyncResult;
    }
}

public class DubboInvoker<T> extends AbstractInvoker<T> {
    //ExchangeClient底层封装的就是NettyClient
    private final ExchangeClient[] clients;
    ...

    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        
        //此次调用的方法名称
        final String methodName = RpcUtils.getMethodName(invocation);
        
        //向Invocation中添加附加信息，这里将URL的path和version添加到附加信息中
        inv.setAttachment(PATH_KEY, getUrl().getPath());
        inv.setAttachment(VERSION_KEY, version);
        
        //ExchangeClient和Exchange都是跟网络相关的，其底层是会去封装对应的Netty，NettyClient一般是被封装在里面
        ExchangeClient currentClient;
        if (clients.length == 1) {
            //选择一个ExchangeClient实例
            currentClient = clients[0];
        } else {
            //如果有多个用于网络通信的client，就会逐个去使用，这会是一个循环使用的过程
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
        //根据调用的方法名称和配置计算此次调用的超时时间，默认是1秒
        int timeout = calculateTimeout(invocation, methodName);
        invocation.setAttachment(TIMEOUT_KEY, timeout);
        if (isOneway) {
            //不需要关注返回值的请求
            boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
            currentClient.send(inv, isSent);
            return AsyncRpcResult.newDefaultAsyncResult(invocation);
        } else {
            //需要关注返回值的请求
            //1.获取处理响应的线程池
            //对于同步请求，会使用ThreadlessExecutor；对于异步请求，则会使用共享的线程池；
            ExecutorService executor = getCallbackExecutor(getUrl(), inv);

            //2.发起网络请求
            //currentClient.request()会使用上面选出的ExchangeClient执行request()方法，将请求发送出去，其实会调用ReferenceCountExchangeClient.request()方法
            //thenApply()会将AppResponse封装成AsyncRpcResult返回
            CompletableFuture<AppResponse> appResponseFuture = currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);

            //3.处理请求的响应结果
            //save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
            FutureContext.getContext().setCompatibleFuture(appResponseFuture);
            AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
            result.setExecutor(executor);
            
            //在外部想要拿到该结果，则必须要基于这个结果里的future同步的去进行等待
            //只要provider方返回一个结果，则肯定会写入到future里，此时就可以通过future拿到结果了
            return result;
        }
    }
}

public abstract class AbstractInvoker<T> implements Invoker<T> {
    protected ExecutorService getCallbackExecutor(URL url, Invocation inv) {
        //通过SPI机制拿到ExecutorRepository——线程池存储组件，Dubbo会把内部所有的线程池都存放在该组件里面
        //可以通过getCallbackExecutor()方法来获取已有的或者创建新的线程池
        if (InvokeMode.SYNC == RpcUtils.getInvokeMode(getUrl(), inv)) {
            return new ThreadlessExecutor();
        }
        
        //下面默认会先获取到ExecutorRepository扩展接口的实现类DefaultExecutorRepository的实例
        //然后调用DefaultExecutorRepository.getExecutor()方法
        return url.getOrDefaultApplicationModel().getExtensionLoader(ExecutorRepository.class)
            .getDefaultExtension()
            .getExecutor(url);
    }
    ...
}

public class DefaultExecutorRepository implements ExecutorRepository, ExtensionAccessorAware {
    ...
    public ExecutorService getExecutor(URL url) {
        //获取一个port->executor线程池的映射关系的缓存map
        Map<Integer, ExecutorService> executors = data.get(getExecutorKey(url));
        // Consumer's executor is sharing globally, key=Integer.MAX_VALUE. Provider's executor is sharing by protocol.
        Integer portKey = CONSUMER_SIDE.equalsIgnoreCase(url.getParameter(SIDE_KEY)) ? Integer.MAX_VALUE : url.getPort();
        ExecutorService executor = executors.get(portKey);
        if (executor != null && (executor.isShutdown() || executor.isTerminated())) {
            executors.remove(portKey);
            // Does not re-create a shutdown executor, use SHARED_EXECUTOR for downgrade.
            executor = null;
            logger.info("Executor for " + url + " is shutdown.");
        }
        if (executor == null) {
            return frameworkExecutorRepository.getSharedExecutor();
        } else {
            return executor;
        }
    }
}
```

<br>

**3.Netty网络客户端构建链路分析**

继续从DubboInvoker.doInvoke()的代码去看发起网络请求的过程：发起网络请求由DubboInvoker.doInvoke()里的currentClient.request()触发。

```
-> DubboInvoker.doInvoke()
-> DubboInvoker.doInvoke()#currentClient.request()
-> ReferenceCountExchangeClient.request()
-> HeaderExchangeClient.request()
-> HeaderExchangeChannel.request()
-> HeaderExchangeChannel.request()#channel.send()
-> AbstractPeer.send()
-> AbstractClient.send()
-> NettyChannel.send()
-> channel.writeAndFlush()

public class DubboInvoker<T> extends AbstractInvoker<T> {
    //ExchangeClient底层封装的就是NettyClient
    private final ExchangeClient[] clients;
    ...
    
    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        ...
        //1.获取处理响应的线程池
        //对于同步请求，会使用ThreadlessExecutor；对于异步请求，则会使用共享的线程池；
        ExecutorService executor = getCallbackExecutor(getUrl(), inv);

        //2.发起网络请求
        //currentClient.request()会使用上面选出的ExchangeClient执行request()方法，将请求发送出去
        //下面其实会调用ReferenceCountExchangeClient.request()方法，而thenApply()会将AppResponse封装成AsyncRpcResult返回
        CompletableFuture<AppResponse> appResponseFuture = currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);

        //3.处理请求的响应结果
        //save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter        	
        FutureContext.getContext().setCompatibleFuture(appResponseFuture);
        AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
        result.setExecutor(executor);
        
        //在外部想要拿到该结果，则必须要基于这个结果里的future同步的去进行等待
        //只要provider方返回一个结果，则肯定会写入到future里，此时就可以通过future拿到结果了
        return result;
    }
}

final class ReferenceCountExchangeClient implements ExchangeClient {
    ...
    @Override
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        //下面会调用HeaderExchangeClient.request()方法
        return client.request(request, timeout, executor);
    }
    ...
}

public class HeaderExchangeClient implements ExchangeClient {
    ...
    @Override
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        //下面会调用HeaderExchangeChannel.request()方法
        return channel.request(request, timeout, executor);
    }
    ...
}

final class HeaderExchangeChannel implements ExchangeChannel {
    ...
    private final Channel channel;
    
    @Override
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        
        //在consumer端最终发送请求时，是会到这里来执行的
        //这里首先会构建一个Request，它会把RpcInvocation对象封装为Request对象
        //也就是把一个业务语义的对象RpcInvocation，封装为了网络通信里的请求响应模型里的Request对象
        //因此从这一步开始，会引入网络通信的概念，比如Request这些概念，这是架构师多年经验设计出来的
        //如果直接将RpcInvocation交给Netty框架去进行处理，那么就不太符合网络通信的请求响应模型了
        //同步转异步的过程，其实就是直接返回了一个future，如果需要同步等待响应，那么可以调用future.get阻塞住
        
        //create request.
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);//双向请求，表示请求过去了还得返回对应的响应
        req.setData(request);//这个request就是我们的RpcInvocation
        
        //创建完一个future后，调用方后续在收到响应结果时会再来进行处理
        //所以返回的异步future结果，与发送请求是脱离开来的
        //下面会将NettyClient、request、timeout、线程池executor，封装成一个future
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
        
        //正常情况下，一般不会有超时问题
        try {
            //下面会调用NettyClient的send()方法，其实也就是调用AbstractPeer.send()方法，因为NettyClient没有覆盖其父类的send()方法
            //AbstractClient.send()又会调用NettyChannel.send()方法，NettyChannel.send()又会调用Netty底层的channel的writeAndFlush()方法
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
}

public abstract class AbstractPeer implements Endpoint, ChannelHandler {
    ...
    @Override
    public void send(Object message) throws RemotingException {
        //从入参为false可知，NettyChannel发送数据时，默认就是异步化的，false表示不会同步等待发送完毕后才返回
        //比如下面会调用AbstractClient.send()
        send(message, url.getParameter(Constants.SENT_KEY, false));
    }
    ...
}

public abstract class AbstractClient extends AbstractEndpoint implements Client {
    ...
    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        if (needReconnect && !isConnected()) {
            connect();
        }
        
        //比如下面会获取到NettyChannel
        Channel channel = getChannel();
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
        }
        
        //比如下面会调用NettyChannel.send()
        //因为NettyClient.doConnect()建立连接成功的时候会拿到一个channel，并将其赋值到NettyClient的channel属性中
        channel.send(message, sent);
    }
}
```

为了了解AbstractClient.send()发起网络请求的原理，下面来分析Netty网络客户端是如何进行构建的。Dubbo网络客户端构建入口就是DubboProtocol.initClient()方法里的Exchangers.connect(url, requestHandler)。

```
-> DubboProtocol.refer()
-> DubboProtocol.protocolBindingRefer()
-> DubboProtocol.getClients()
-> DubboProtocol.initClient()
-> Exchangers.connect(url, requestHandler)
-> getExchanger(url).connect(url, handler) 
-> Exchanger.connect(url, handler)
-> HeaderExchanger.connect()
-> Transporters.connect()
-> Transporters.getTransporter(url).connect(url, handler)
-> NettyTransporter.connect()
-> new NettyClient(url, handler) => 构建NettyClient发起连接
-> AbstractClient.wrapChannelHandler()
-> ChannelHandlers.wrapInternal()
-> new AbstractClient(url, handler)
-> NettyClient.doOpen()
-> NettyClient.doConnect()
-> new HeaderExchangeClient()

public class DubboProtocol extends AbstractProtocol {
    ...
    private ExchangeClient initClient(URL url) {
        ...
        //如果配置了延迟创建连接的特性，则创建LazyConnectExchangeClient
        return url.getParameter(LAZY_CONNECT_KEY, false)
            ? new LazyConnectExchangeClient(url, requestHandler)
            : Exchangers.connect(url, requestHandler);
            ...
        }
    }
}

public class Exchangers {
    ...
    public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        ...
        //下面会调用到HeaderExchanger.connect()方法，传入的handler是DubboProtocol的requestHandler
        return getExchanger(url).connect(url, handler);
    }
    
    public static Exchanger getExchanger(URL url) {
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        //根据SPI机制 + model组件体系，去拿到对应的SPI使用入口
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Exchanger.class).getExtension(type);
    }
    ...
}

public class HeaderExchanger implements Exchanger {
    ...
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        //传入的handler是DubboProtocol的requestHandler
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
    ...
}

public class Transporters {
    ...
    public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
        ...
        //下面会调用NettyTransporter.connect()方法，传入的handler装饰了DubboProtocol的requestHandler
        //返回一个NettyClient
        return getTransporter(url).connect(url, handler);
    }
    ...
}

public class NettyTransporter implements Transporter {
    ...
    public Client connect(URL url, ChannelHandler handler) throws RemotingException {
        //传入的handler装饰了DubboProtocol的requestHandler，返回一个NettyClient
        return new NettyClient(url, handler);
    }
    ...
}

public class NettyClient extends AbstractClient {
    ...
    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
        //传入的handler装饰了DubboProtocol的requestHandler
        super(url, wrapChannelHandler(url, handler));
    }
}

public abstract class AbstractClient extends AbstractEndpoint implements Client {
    ...
    protected static ChannelHandler wrapChannelHandler(URL url, ChannelHandler handler) {
        //传入的handler装饰了DubboProtocol的requestHandler
        return ChannelHandlers.wrap(handler, url);
    }
}

public class ChannelHandlers {
    public static ChannelHandler wrap(ChannelHandler handler, URL url) {
        return ChannelHandlers.getInstance().wrapInternal(handler, url);
    }
    
    //MultiMessageHandler->HeartbeatHandler->AllChannelHandler->DecodeHandler->HeaderExchangeHandler->DubboProtocol的requestHandler
    //其中AllChannelHandler是由下面代码通过SPI获取到的自适应实现类AllDispatcher的dispatch()方法返回的
    protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
        return new MultiMessageHandler(new HeartbeatHandler(url.getOrDefaultFrameworkModel().getExtensionLoader(Dispatcher.class)
            .getAdaptiveExtension().dispatch(handler, url)));
    }
    ...
}

public abstract class AbstractClient extends AbstractEndpoint implements Client {
    ...
    public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
        //调用父类的构造方法
        super(url, handler);
        //解析URL，初始化executor线程池
        initExecutor(url);
        //初始化底层的NIO库的相关组件
        doOpen();
        //connect. 创建底层连接
        connect();
        ...
    }
    
    protected void connect() throws RemotingException {
        ...
        doConnect();
    }
    ...
}

public class NettyClient extends AbstractClient {
    ...
    protected void doOpen() throws Throwable {
        //创建NettyClientHandler，handler一般来说是用来处理网络请求的
        final NettyClientHandler nettyClientHandler = createNettyClientHandler();
        //创建Bootstrap，企业级Netty编程示范如下，对于NettyClient必须构建一个Bootstrap
        bootstrap = new Bootstrap();
        initBootstrap(nettyClientHandler);
    }
    
    protected void doConnect() throws Throwable {
        long start = System.currentTimeMillis();
        //bootstrap.connect会对server端发起一个网络连接，但这个网络连接，并不是立刻就可以做好的
        ChannelFuture future = bootstrap.connect(getConnectAddress());
        ...
        
        boolean ret = future.awaitUninterruptibly(getConnectTimeout(), MILLISECONDS);
        ...
        
        //如果连接成功，此时就可以通过future拿到一个channel，它代表着client端和server端provider服务实例建立好的网络连接
        Channel newChannel = future.channel();
        NettyClient.this.channel = newChannel;
        ...
    }
    ...
}

public class HeaderExchangeClient implements ExchangeClient {
    //NettyClient的handler属性是经过层层装饰的：MultiMessageHandler -> HeartbeatHandler -> AllChannelHandler -> DecodeHandler -> HeaderExchangeHandler -> DubboProtocol.requestHandler
    //传入的client是NettyClient，HeaderExchangeClient装饰了NettyClient，HeaderExchangeClient的channel属性也是一个装饰了NettyClient的HeaderExchangeChannel
    public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);
        ...
    }
    ...
}
```

总结：注意Dubbo网络客户端构建最后会构建出一个如下层层wrap的handler，这个handler会成为NettyClient的handler属性。

```
-> MultiMessageHandler
-> HeartbeatHandler
-> AllChannelHandler
-> DecodeHandler
-> HeaderExchangeHandler
-> DubboProtocol.requestHandler
```

同时HeaderExchangeClient装饰了NettyClient，即HeaderExchangeClient的client属性是一个NettyClient。并且HeaderExchangeClient的channel属性也是一个装饰了NettyClient的HeaderExchangeChannel。

这就呼应了发起请求时调用的HeaderExchangeChannel的channel.send()方法会调用NettyClient.send()方法，而且由于NettyClient.doConnect()建立连接成功的时候会拿到一个channel，并将其赋值到NettyClient的channel属性中。

所以NettyClient.send() -> AbstractClient.send() -> NettyChannel.send() -> channel.writeAndFlush()。

```
final class HeaderExchangeChannel implements ExchangeChannel {
    ...
    private final Channel channel;
    
    @Override
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        ...
        try {
            //下面会调用NettyClient的send()方法，其实也就是调用AbstractPeer.send()方法，因为NettyClient没有覆盖其父类的send()方法
            //AbstractClient.send()又会调用NettyChannel.send()方法，NettyChannel.send()又会调用Netty底层的channel的writeAndFlush()方法
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
}

public abstract class AbstractPeer implements Endpoint, ChannelHandler {
    ...
    @Override
    public void send(Object message) throws RemotingException {
        //从入参为false可知，NettyChannel发送数据时，默认就是异步化的，false表示不会同步等待发送完毕后才返回
        //比如下面会调用AbstractClient.send()
        send(message, url.getParameter(Constants.SENT_KEY, false));
    }
    ...
}

public abstract class AbstractClient extends AbstractEndpoint implements Client {
    ...
    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        if (needReconnect && !isConnected()) {
            connect();
        }

        //比如下面会获取到NettyChannel
        Channel channel = getChannel();
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
        }
        
        //比如下面会调用NettyChannel.send()
        //因为NettyClient.doConnect()建立连接成功的时候会拿到一个channel，并将其赋值到NettyClient的channel属性中
        channel.send(message, sent);
    }
}
```

<br>

**4.Netty网络写数据同步和异步的分析**

Dubbo是客户端发起网络请求时，是通过NettyChannel.send()方法来进行网络写数据的。传入的参数sent如果为false则是异步，如果为true则是同步，即同步阻塞直到网络操作完成，并且有超时限制。

```
final class NettyChannel extends AbstractChannel {
    ...
    //传入的参数sent如果为false则说明网络操作是异步的，如果为true则网络操作需要同步等待
    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        // whether the channel is closed
        super.send(message, sent);
        boolean success = true;
        int timeout = 0;
        try {
            //下面这行代码的channel是一个NioSocketChannel，基于Netty的channel来进行写数据
            //Netty的channel.writeAndFlush()操作，本身既可以支持同步、也可以支持异步，如果需要同步，则可以通过future.await(timeout)来实现
            //Netty的channel.writeAndFlush()操作，默认是异步的
            ChannelFuture future = channel.writeAndFlush(message);
            if (sent) {
                //wait timeout ms
                timeout = getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
                //必须在这里等待Netty的网络写操作全部完成了才可以返回，等待超时时间是timeout
                success = future.await(timeout);
            }
            Throwable cause = future.cause();
            if (cause != null) {
                throw cause;
            }
        } catch (Throwable e) {
            removeChannelIfDisconnected(channel);
            throw new RemotingException(this, "Failed to send message " + PayloadDropper.getRequestWithoutData(message) + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
        }
        
        if (!success) {
            throw new RemotingException(this, "Failed to send message " + PayloadDropper.getRequestWithoutData(message) + " to " + getRemoteAddress() + "in timeout(" + timeout + "ms) limit");
        }
    }
    ...
}
```

<br>

**5.RPC调用链路中的Future异步分析**

thenApply()会将AppResponse封装成AsyncRpcResult返回：

```
public class DubboInvoker<T> extends AbstractInvoker<T> {
    //ExchangeClient底层封装的就是NettyClient
    private final ExchangeClient[] clients;
    ...
    
    @Override
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        ...
        //1.获取处理响应的线程池
        //对于同步请求，会使用ThreadlessExecutor；对于异步请求，则会使用共享的线程池；
        ExecutorService executor = getCallbackExecutor(getUrl(), inv);

        //2.发起网络请求
        //currentClient.request()会使用上面选出的ExchangeClient执行request()方法，将请求发送出去
        //下面其实会调用ReferenceCountExchangeClient.request()方法
        //而thenApply()会将AppResponse封装成AsyncRpcResult返回
        CompletableFuture<AppResponse> appResponseFuture = currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);

        //3.处理请求的响应结果
        //save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter        	
        FutureContext.getContext().setCompatibleFuture(appResponseFuture);
        AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
        result.setExecutor(executor);
        
        //在外部想要拿到该结果，则必须要基于这个结果里的future同步的去进行等待
        //只要provider方返回一个结果，则肯定会写入到future里，此时就可以通过future拿到结果了
        return result;
    }
}
```

HeaderExchangeChannel.request()返回的异步future结果，与发送请求是脱离开来的。会同步发起请求，但默认不会等待请求完成了才返回，而是直接返回future对象。

```
final class HeaderExchangeChannel implements ExchangeChannel {
    ...
    private final Channel channel;
    
    @Override
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        
        //在consumer端最终发送请求时，是会到这里来执行的
        //这里首先会构建一个Request，它会把RpcInvocation对象封装为Request对象
        //也就是把一个业务语义的对象RpcInvocation，封装为了网络通信里的请求响应模型里的Request对象
        //因此从这一步开始，会引入网络通信的概念，比如Request这些概念，这是架构师多年经验设计出来的
        //如果直接将RpcInvocation交给Netty框架去进行处理，那么就不太符合网络通信的请求响应模型了
        //同步转异步的过程，其实就是直接返回了一个future，如果需要同步等待响应，那么可以调用future.get阻塞住
        
        //create request.
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);//双向请求，表示请求过去了还得返回对应的响应
        req.setData(request);//这个request就是我们的RpcInvocation
        
        //创建完一个future后，调用方后续在收到响应结果时会再来进行处理
        //所以返回的异步future结果，与发送请求是脱离开来的
        //下面newFuture()会将NettyClient、request、timeout、线程池executor，封装成一个future
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);
        
        //正常情况下，一般不会有超时问题
        try {
            //下面会调用NettyClient的send()方法，其实也就是调用AbstractPeer.send()方法，因为NettyClient没有覆盖其父类的send()方法
            //AbstractClient.send()又会调用NettyChannel.send()方法，NettyChannel.send()又会调用Netty底层的channel的writeAndFlush()方法
            //会同步发起请求，但默认不会等待请求完成了才返回
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        
        return future;
    }
}
```

从AbstractPeer.send()方法可知，NettyChannel发送数据时，默认就是异步化的，即不会同步等待发送完毕后才返回。

```
public abstract class AbstractPeer implements Endpoint, ChannelHandler {
    ...
    @Override
    public void send(Object message) throws RemotingException {
        //从入参为false可知，NettyChannel发送数据时，默认就是异步化的，false表示不会同步等待发送完毕后才返回
        //比如下面会调用AbstractClient.send()
        send(message, url.getParameter(Constants.SENT_KEY, false));
    }
    ...
}
```

DefaultFuture.newFuture()会启动一个定时任务进行task超时检查：

```
public class DefaultFuture extends CompletableFuture<Object> {
    ...
    public static DefaultFuture newFuture(Channel channel, Request request, int timeout, ExecutorService executor) {
        //1.根据channel、request、timeout创建DefaultFuture对象，然后设置其线程池为executor，该过程只是简单的赋值操作
        final DefaultFuture future = new DefaultFuture(channel, request, timeout);
        future.setExecutor(executor);
        
        //ThreadlessExecutor needs to hold the waiting future in case of circuit return.
        if (executor instanceof ThreadlessExecutor) {
            ((ThreadlessExecutor) executor).setWaitingFuture(future);
        }
        
        //2.timeout check
        timeoutCheck(future);
        return future;
    }
    
    //利用TIME_OUT_TIMER时间轮启动一个定时任务进行task超时检查
    private static void timeoutCheck(DefaultFuture future) {
        TimeoutCheckTask task = new TimeoutCheckTask(future.getId());
        future.timeoutCheckTask = TIME_OUT_TIMER.get().newTimeout(task, future.getTimeout(), TimeUnit.MILLISECONDS);
    }
    
    private static class TimeoutCheckTask implements TimerTask {
        private final Long requestID;

        TimeoutCheckTask(Long requestID) {
            this.requestID = requestID;
        }
        
        @Override
        public void run(Timeout timeout) {
            DefaultFuture future = DefaultFuture.getFuture(requestID);
            if (future == null || future.isDone()) {
                //检查该任务关联的DefaultFuture对象是否已经完成
                return;
            }
            
            if (future.getExecutor() != null) {
                //提交到线程池执行，注意ThreadlessExecutor的情况
                future.getExecutor().execute(() -> notifyTimeout(future));
            } else {
                //通知已经超时
                notifyTimeout(future);
            }
        }
        
        //通知已经超时
        private void notifyTimeout(DefaultFuture future) {
            //没有收到对端的响应，这里会创建一个Response，表示超时的响应
            //create exception response.
            Response timeoutResponse = new Response(future.getId());
            //set timeout status.
            timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
            timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
            //handle response.
            DefaultFuture.received(future.getChannel(), timeoutResponse, true);
    	}
    }
}
```

future.isDone()是会在HeaderExchangeHandler.received()处理返回响应时变为true。

```
-> HeaderExchangeHandler.received()
-> HeaderExchangeHandler.handleResponse()
-> DefaultFuture.received()
-> DefaultFuture.doReceived()

public class HeaderExchangeHandler implements ChannelHandlerDelegate {
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        if (message instanceof Request) {//收到Request请求
            // handle request.
            Request request = (Request) message;
            if (request.isEvent()) {//事件类型的请求
                handlerEvent(channel, request);
            } else {//非事件的请求
                if (request.isTwoWay()) {//twoway
                    //下面会对请求进行处理
                    handleRequest(exchangeChannel, request);
                } else {//oneway
                    handler.received(exchangeChannel, request.getData());
                }
            }
        } else if (message instanceof Response) {//收到Response响应
            handleResponse(channel, (Response) message);
        } else if (message instanceof String) {
            if (isClientSide(channel)) {
                Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                logger.error(e.getMessage(), e);
            } else {
                String echo = handler.telnet(channel, (String) message);
                if (StringUtils.isNotEmpty(echo)) {
                    channel.send(echo);
                }
            }
        } else {
            handler.received(exchangeChannel, message);
        }
    }
    
    static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {
            DefaultFuture.received(channel, response);
        }
    }
    ...
}

public class DefaultFuture extends CompletableFuture<Object> {
    private static final Map<Long, Channel> CHANNELS = new ConcurrentHashMap<>();//请求ID和Channel的映射关系
    private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();//请求ID和DefaultFuture的映射关系
    ...
    
    public static void received(Channel channel, Response response) {
        //下面的方法可以根据响应response利用FUTURES映射关系去获取当初发起请求的future
        received(channel, response, false);
    }
    
    //根据响应response利用FUTURES映射关系去获取当初发起请求的future
    public static void received(Channel channel, Response response, boolean timeout) {
        try {
            //根据响应ID即请求ID去获取该请求对应的DefaultFuture
            //同时清理FUTURES中记录的请求ID与DefaultFuture之间的映射关系
            DefaultFuture future = FUTURES.remove(response.getId());
            if (future != null) {
                Timeout t = future.timeoutCheckTask;
                //未超时，取消定时任务
                if (!timeout) {
                    //decrease Time
                    t.cancel();
                }
                //调用doReceived()方法，设置future的result值
                future.doReceived(response);
            } else {
                logger.warn("...");
            }
        } finally {
            //清理CHANNELS中记录的请求ID与Channel之间的映射关系
            CHANNELS.remove(response.getId());
    	}
    }
    ...
}
```

<br>

**6.AsyncRpcResult等待结果的源码分析**

DubboInvoker的doInvoke()方法返回的AsyncRpcResult经过层层传递，会返回到FailoverClusterInvoker的doInvoke()方法里。接着该AsyncRpcResult又会返回到MockClusterInvoker的invoke()方法 -> MigrationInvoker的invoke()方法里。最后该AsyncRpcResult就会返回到InvocationUtil的invoke()方法里，从而进行AsyncRpcResult的recreate()方法操作将结果返回。

```
-> InvokerInvocationHandler.invoke()
-> InvocationUtil.invoke()
-> invoker.invoke(rpcInvocation).recreate()
-> MigrationInvoker.invoke()
-> MockClusterInvoker.invoke()

-> AbstractCluster.ClusterFilterInvoker.invoke()
-> FilterChainBuilder.ClusterFilterChainNode.invoke()
-> AbstractClusterInvoker.invoke()

-> FailoverClusterInvoker.doInvoke()
-> AbstractClusterInvoker.select()
-> AbstractClusterInvoker.doSelect()
-> AbstractLoadBalance.select()
-> RandomLoadBalance.doSelect()
-> AbstractClusterInvoker.invokeWithContext()
-> ListenerInvokerWrapper.invoke()
-> AbstractInvoker.invoke()
-> AbstractInvoker.doInvokeAndReturn()
-> DubboInvoker.doInvoke()

public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {
    //该方法会找一个Invoker，把invocation交给多个Invokers里的一个去发起RPC调用，其中loadbalance会实现负载均衡
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        ...
        //2.AbstractClusterInvoker.invokeWithContext()方法会基于该invoker发起RPC调用，并拿到一个result
        Result result = invokeWithContext(invoker, invocation);
        if (le != null && logger.isWarnEnabled()) {
            logger.warn("...", le);
        }
        success = true;
        ...
    }
}

public class MockClusterInvoker<T> implements ClusterInvoker<T> {
    //核心其实是是否启用降级机制调用doMockInvoke()
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        ...
        //下面会调用AbstractCluster内部类ClusterFilterInvoker的invoke()方法，进行正常的RPC调用
        result = this.invoker.invoke(invocation);
        ...
        return result;
    }
    ...
}

public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
    //为什么会叫MigrationInvoker?
    //因为MigrationInvoker有两个Invoker：一个是invoker，一个是serviceDiscoveryInvoker；
    //执行这里的invoke()方法时会根据不同的条件去切换这两个invoker，将它们之一赋值给真正负责执行invoke()方法的currentAvailableInvoker；
    //也就是有一个针对currentAvailableInvoker进行切换的过程(根据不同的条件来切换是invoker还是serviceDiscoveryInvoker)，所以migration就是从这里来的
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
    	...
     	//下面会调用MockClusterInvoker.invoke()方法
    	return currentAvailableInvoker.invoke(invocation);
    }
    ...
}

public class InvocationUtil {
    ...
    public static Object invoke(Invoker<?> invoker, RpcInvocation rpcInvocation) throws Throwable {
    	...
    	//为什么要进行recreate？
    	//recreate是为了对拿到的结果进行一个拷贝，将拷贝出来的结果对象返回给业务方去使用
    	//这样Dubbo框架内部自己可以持有一个结果对象，避免了和业务方共享持有和访问，而产生相互的影响
    	return invoker.invoke(rpcInvocation).recreate();
    }
}
```

AsyncRpcResult的recreate()操作的逻辑如下，注意最后从CompletableFuture获取AppResponse时，会进行阻塞循环式获取，获取到值才返回。

```
public class AsyncRpcResult implements Result {
    ...
    @Override
    public Object recreate() throws Throwable {
        RpcInvocation rpcInvocation = (RpcInvocation) invocation;
        //对InvokeMode模式进行判断
        if (InvokeMode.FUTURE == rpcInvocation.getInvokeMode()) {
            //如果模式为FUTURE，则表示支持异步化，所以返回的就是一个future对象
            //而这个future异步对象代表的就是一个异步化结果，此时可能有结果了也可能没有结果，这需要自己去获取
            return RpcContext.getClientAttachment().getFuture();
        } else if (InvokeMode.ASYNC == rpcInvocation.getInvokeMode()) {
            //如果模式是ASYNC，则创建默认结果返回
            return createDefaultValue(invocation).recreate();
        }
        //如果模式SYNC
        return getAppResponse().recreate();
    }
    
    public Result getAppResponse() {
        try {
            //DubboInvoker.doInvoke()方法返回的AsyncRpcResult会封装一个CompletableFuture进去
            //这里首先会做一个判断，如果CompletableFuture的isDone()方法返回true，则表示已经完成请求并拿到了响应
            //响应结果会通过HeaderExchangeHandler.received()被放到CompletableFuture里面
            if (responseFuture.isDone()) {//检测responseFuture是否已完成
                //从CompletableFuture进行阻塞式循环获取AppResponse并进行返回
                return responseFuture.get();
            }
        } catch (Exception e) {
            // This should not happen in normal request process;
            logger.error("Got exception when trying to fetch the underlying result from AsyncRpcResult.");
            throw new RpcException(e);
        }
        //根据调用方法的返回值，生成默认值
        return createDefaultValue(invocation);
    }
    ...
}

//这是JDK的CompletableFuture
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
    //Waits if necessary for this future to complete, and then returns its result.
    public T get() throws InterruptedException, ExecutionException {
        Object r;
        return reportGet((r = result) == null ? waitingGet(true) : r);
    }
    
    private Object waitingGet(boolean interruptible) {
        ...
        while ((r = result) == null) {
            ...
        }
        ...
    }
    ...
}
```

此外，在FilterChainBuilder的ClusterFilterChainNode的invoke()方法中，会对AsyncRpcResult添加一个回调。

```
@SPI(value = "default", scope = APPLICATION)
public interface FilterChainBuilder {
    class FilterChainNode<T, TYPE extends Invoker<T>, FILTER extends BaseFilter> implements Invoker<T> {
        ...
        @Override
        public Result invoke(Invocation invocation) throws RpcException {
            Result asyncResult;
            try {
                InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Filter " + filter.getClass().getName() + " invoke.");
                //比如consumer端处理时下面会调用ConsumerContextFilter的invoke()方法
                //provider端处理时下面会最终调用AbstractProxyInvoker.invoke()方法
                asyncResult = filter.invoke(nextNode, invocation);
            } catch (Exception e) {
                ...
            }
            
            return asyncResult.whenCompleteWithContext((r, t) -> {
                InvocationProfilerUtils.releaseDetailProfiler(invocation);
                if (filter instanceof ListenableFilter) {
                    ListenableFilter listenableFilter = ((ListenableFilter) filter);
                    Filter.Listener listener = listenableFilter.listener(invocation);
                    try {
                        if (listener != null) {
                            if (t == null) {
                                listener.onResponse(r, originalInvoker, invocation);
                            } else {
                                listener.onError(t, originalInvoker, invocation);
                            }
                        }
                    } finally {
                        listenableFilter.removeListener(invocation);
                    }
                } else if (filter instanceof FILTER.Listener) {
                    FILTER.Listener listener = (FILTER.Listener) filter;
                    if (t == null) {
                        listener.onResponse(r, originalInvoker, invocation);
                    } else {
                        listener.onError(t, originalInvoker, invocation);
                    }
                }
            });
        }
    }
    ...
}

public interface Result extends Serializable {
    ...
    //Add a callback which can be triggered when the RPC call finishes.
    //Just as the method name implies, this method will guarantee the callback being triggered under the same context as when the call was started
    Result whenCompleteWithContext(BiConsumer<Result, Throwable> fn);
}
```

<br>

**7.Dubbo RPC请求对象如何进行封装**

```
//实现了JDK反射机制里的InvocationHandler接口
public class InvokerInvocationHandler implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        ...
        //创建RpcInvocation对象，后面会作为远程RPC调用的参数
        //在consumer端进行RPC调用时，必须要封装一个RpcInvocation
        //这个RpcInvocation会传递到provider端，provider端拿到请求数据后，也会封装一个RpcInvocation
        RpcInvocation rpcInvocation = new RpcInvocation(
            serviceModel, 
            method.getName(), 
            invoker.getInterface().getName(), 
            protocolServiceKey, 
            method.getParameterTypes(), 
            args
        );
        if (serviceModel instanceof ConsumerModel) {
            rpcInvocation.put(Constants.CONSUMER_MODEL, serviceModel);
            rpcInvocation.put(Constants.METHOD_MODEL, ((ConsumerModel) serviceModel).getMethodModel(method));
        }

        //调用invoke()方法发起远程调用，拿到AsyncRpcResult之后，调用recreate()方法获取响应结果(或是Future)
        //下面的invoker其实就是装饰了MockClusterInvoker的MigrationInvoker
        return InvocationUtil.invoke(invoker, rpcInvocation);
    }
}

public class RpcInvocation implements Invocation, Serializable {
    public RpcInvocation(ServiceModel serviceModel, String methodName, String interfaceName, String protocolServiceKey, Class<?>[] parameterTypes, Object[] arguments) {
        this(null, serviceModel, methodName, interfaceName, protocolServiceKey, 
        parameterTypes, arguments, null, null, null, null);
    }
    
    public RpcInvocation(String targetServiceUniqueName, ServiceModel serviceModel, String methodName, String interfaceName, String protocolServiceKey, Class<?>[] parameterTypes, Object[] arguments, Map<String, Object> attachments, Invoker<?> invoker, Map<Object, Object> attributes, InvokeMode invokeMode) {
        this.targetServiceUniqueName = targetServiceUniqueName;
        this.serviceModel = serviceModel;
        this.methodName = methodName;
        this.interfaceName = interfaceName;
        this.protocolServiceKey = protocolServiceKey;
        this.parameterTypes = parameterTypes == null ? new Class<?>[0] : parameterTypes;
        this.arguments = arguments == null ? new Object[0] : arguments;
        this.attachments = attachments == null ? new HashMap<>() : attachments;
        this.attributes = attributes == null ? Collections.synchronizedMap(new HashMap<>()) : attributes;
        this.invoker = invoker;
        initParameterDesc();
        this.invokeMode = invokeMode;
    }
    ...
}
```

<br>

**8.RPC调用链路中引入Filter链条部分**

Filter会在ProtocolFilterWrapper中通过DefaultFilterChainBuilder的buildInvokerChain()方法构建好。

```
-> ReferenceConfig#protocolSPI.refer(interfaceClass, curUrl)
-> Protocol$Adaptive.refer()
-> ProtocolSerializationWrapper.refer()
-> ProtocolFilterWrapper.refer()
-> builder.buildInvokerChain()
-> DefaultFilterChainBuilder.buildInvokerChain()

@Activate(order = 100)
public class ProtocolFilterWrapper implements Protocol {
    @Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        if (UrlUtils.isRegistry(url)) {
            //下面会调用ProtocolListenerWrapper.refer()方法
            return protocol.refer(type, url);
        }
        FilterChainBuilder builder = getFilterChainBuilder(url);
        return builder.buildInvokerChain(protocol.refer(type, url), REFERENCE_FILTER_KEY, CommonConstants.CONSUMER);
    }
    ...
}

@Activate
public class DefaultFilterChainBuilder implements FilterChainBuilder {
    //build consumer/provider filter chain
    @Override
    public <T> Invoker<T> buildInvokerChain(final Invoker<T> originalInvoker, String key, String group) {
        Invoker<T> last = originalInvoker;
        URL url = originalInvoker.getUrl();
        List<ModuleModel> moduleModels = getModuleModelsFromUrl(url);
        List<Filter> filters;
        
        //通过SPI机制来获取Filter
        if (moduleModels != null && moduleModels.size() == 1) {
            filters = ScopeModelUtil.getExtensionLoader(Filter.class, moduleModels.get(0)).getActivateExtension(url, key, group);
        } else if (moduleModels != null && moduleModels.size() > 1) {
            filters = new ArrayList<>();
            List<ExtensionDirector> directors = new ArrayList<>();
            for (ModuleModel moduleModel : moduleModels) {
                List<Filter> tempFilters = ScopeModelUtil.getExtensionLoader(Filter.class, moduleModel).getActivateExtension(url, key, group);
                filters.addAll(tempFilters);
                directors.add(moduleModel.getExtensionDirector());
            }
            filters = sortingAndDeduplication(filters, directors);
        } else {
            filters = ScopeModelUtil.getExtensionLoader(Filter.class, null).getActivateExtension(url, key, group);
        }

        //构建Filter链条
        if (!CollectionUtils.isEmpty(filters)) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new CopyOfFilterChainNode<>(originalInvoker, next, filter);
            }
            return new CallbackRegistrationInvoker<>(last, filters);
        }
        return last;
    }
    ...
}
```

Filter会在AbstractCluster的ClusterFilterInvoker的invoke()方法中被调用：

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/4003aace-a334-4e4c-9672-85c95ca46128" />

比如ActiveLimitFilter是放在consumer端对并发进行控制的过滤器；

```
-> InvokerInvocationHandler.invoke()
-> InvocationUtil.invoke()
-> invoker.invoke(rpcInvocation).recreate()
-> MigrationInvoker.invoke()
-> MockClusterInvoker.invoke()

-> AbstractCluster.ClusterFilterInvoker.invoke()
-> FilterChainBuilder.CallbackRegistrationInvoker.invoke()
-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
-> ConsumerContextFilter.invoke()
...
-> ActiveLimitFilter.invoke()
...
-> AbstractClusterInvoker.invoke()
-> AbstractClusterInvoker.list()
-> AbstractDirectory.list()
-> DynamicDirectory.list()
-> RouterChain.route()

@Activate(group = CONSUMER, order = Integer.MIN_VALUE)
public class ConsumerContextFilter implements ClusterFilter, ClusterFilter.Listener {
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        //RpcInvocation代表了一次RPC调用
        //RpcStatus代表了对指定服务、方法调用的统计状态数据
        //RpcContext代表了RPC调用过程中的一个上下文，也就是RPC相关的一些信息
        RpcContext.RestoreServiceContext originServiceContext = RpcContext.storeServiceContext();
        RpcContext.getServiceContext()//通过由当前线程绑定的一个数据空间作为它的RPC上下文
            .setInvoker(invoker)//记录Invoker
            .setInvocation(invocation)//记录Invocation，把当前这个RPC调用放到里面去，因为可能会有很多线程都在同时发起访问
            .setLocalAddress(NetUtils.getLocalHost(), 0);//记录本地地址
        RpcContext context = RpcContext.getClientAttachment();
        context.setAttachment(REMOTE_APPLICATION_KEY, invoker.getUrl().getApplication());
        if (invocation instanceof RpcInvocation) {
            ((RpcInvocation) invocation).setInvoker(invoker);//对RpcInvocation设置Invoker
        }
        ...
        //下面又会调用FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法
        return invoker.invoke(invocation);
    }
    ...
}

//ActiveLimitFilter是放在consumer端对并发进行控制的过滤器
//在框架设计层面，或者对很多系统或者框架而言，都是必须要设计一个filter机制的
//这样通过filter机制，对一些核心操作，进行过滤和拦截，并在此基础上做一些增强性的操作
//以及允许我们自己去实现一些filter，通过配置项的配置，加入到它的filter链条去
@Activate(group = CONSUMER, value = ACTIVES_KEY)
public class ActiveLimitFilter implements Filter, Filter.Listener {
    private static final String ACTIVE_LIMIT_FILTER_START_TIME = "active_limit_filter_start_time";
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        //获得url对象
        URL url = invoker.getUrl();
        //获得方法名称
        String methodName = invocation.getMethodName();
        //计算获取最大并发数
        int max = invoker.getUrl().getMethodParameter(methodName, ACTIVES_KEY, 0);
        //获取该方法的状态信息，RpcStatus是RPC调用的状态数据，RpcStatus可以统计和获取指定接口方法的当前调用的状态数据
        final RpcStatus rpcStatus = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName());
        
        //如果设置了consumer端的ActiveLimit数值，而此时针对目标接口的方法活跃调用次数已超过了设置值，则RpcStatus.beginCount()为false
        //尝试并发度加一
        if (!RpcStatus.beginCount(url, methodName, max)) {
            //如果活跃调用次数超过了设置的阈值，此时只能进行wait等待，超时时间
            long timeout = invoker.getUrl().getMethodParameter(invocation.getMethodName(), TIMEOUT_KEY, 0);
            long start = System.currentTimeMillis();
            long remain = timeout;
            //加锁
            synchronized (rpcStatus) {
                //再次尝试并发度加一
                while (!RpcStatus.beginCount(url, methodName, max)) {
                    try {
                        //等待并发度降低
                        rpcStatus.wait(remain);
                    } catch (InterruptedException e) {
                        //ignore
                    }
                    //检测是否超时
                    long elapsed = System.currentTimeMillis() - start;
                    remain = timeout - elapsed;
                    if (remain <= 0) {
                        throw new RpcException(RpcException.LIMIT_EXCEEDED_EXCEPTION, "... " + max);
                    }
                }
            }
        }
        
        //添加一个attribute
        invocation.put(ACTIVE_LIMIT_FILTER_START_TIME, System.currentTimeMillis());
        
        return invoker.invoke(invocation);
    }
    ...
}
```

<br>

**9.Dubbo自实现ThreadLocal机制剖析**

RpcContext中就大量使用了Dubbo自实现的Threadlocal机制，也就是InternalThreadLocal类，这个其实和Netty的FastThreadLocal一样的。

```
public class RpcContext {
    private static final RpcContext AGENT = new RpcContext();
    
    //use internal thread local to improve performance
    private static final InternalThreadLocal<RpcContextAttachment> SERVER_LOCAL = new InternalThreadLocal<RpcContextAttachment>() {
        protected RpcContextAttachment initialValue() {
            return new RpcContextAttachment();
        }
    };
    
    private static final InternalThreadLocal<RpcContextAttachment> CLIENT_ATTACHMENT = new InternalThreadLocal<RpcContextAttachment>() {
        protected RpcContextAttachment initialValue() {
            return new RpcContextAttachment();
        }
    };
    
    private static final InternalThreadLocal<RpcContextAttachment> SERVER_ATTACHMENT = new InternalThreadLocal<RpcContextAttachment>() {
        protected RpcContextAttachment initialValue() {
            return new RpcContextAttachment();
        }
    };
    
    private static final InternalThreadLocal<RpcServiceContext> SERVICE_CONTEXT = new InternalThreadLocal<RpcServiceContext>() {
        protected RpcServiceContext initialValue() {
            return new RpcServiceContext();
        }
    };
    
    //use by cancel call
    private static final InternalThreadLocal<CancellationContext> CANCELLATION_CONTEXT = new InternalThreadLocal<CancellationContext>() {
        protected CancellationContext initialValue() {
            return new CancellationContext();
        }
    };
    ...
}
```

<br>

**10.Consumer上下文处理过滤器深入剖析**

Consumer上下文处理过滤也就是ConsumerContextFilter：

```
@Activate(group = CONSUMER, order = Integer.MIN_VALUE)
public class ConsumerContextFilter implements ClusterFilter, ClusterFilter.Listener {
    private Set<PenetrateAttachmentSelector> supportedSelectors;
    
    public ConsumerContextFilter(ApplicationModel applicationModel) {
        //初始化Consumer上下文过滤器时，就会在这里通过SPI机制先拿到接口ExtensionLoader
        //然后再通过ExtensionLoader拿到该接口的所有实现类，放到supportedSelectors集合
        ExtensionLoader<PenetrateAttachmentSelector> selectorExtensionLoader = applicationModel.getExtensionLoader(PenetrateAttachmentSelector.class);
        supportedSelectors = selectorExtensionLoader.getSupportedExtensionInstances();
    }
    
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        //RpcInvocation代表了一次RPC调用
        //RpcStatus代表了对指定服务、方法调用的统计状态数据
        //RpcContext代表了RPC调用过程中的一个上下文，也就是RPC相关的一些信息
        RpcContext.RestoreServiceContext originServiceContext = RpcContext.storeServiceContext();
        try {
            RpcContext.getServiceContext()//通过由当前线程绑定的一个数据空间作为它的RPC上下文
                .setInvoker(invoker)//记录Invoker
                .setInvocation(invocation)//记录Invocation，把当前这个RPC调用放到里面去，因为可能会有很多线程都在同时发起访问
                .setLocalAddress(NetUtils.getLocalHost(), 0);//记录本地地址
            RpcContext context = RpcContext.getClientAttachment();
            context.setAttachment(REMOTE_APPLICATION_KEY, invoker.getUrl().getApplication());
            
            if (invocation instanceof RpcInvocation) {
                ((RpcInvocation) invocation).setInvoker(invoker);//对RpcInvocation设置Invoker
            }
            
            if (CollectionUtils.isNotEmpty(supportedSelectors)) {
                //遍历supportedSelectors集合，集合的元素就是PenetrateAttachmentSelector接口的一个实现类的实例
                for (PenetrateAttachmentSelector supportedSelector : supportedSelectors) {
                    Map<String, Object> selected = supportedSelector.select();
                    if (CollectionUtils.isNotEmptyMap(selected)) {
                        //给RpcInvocation添加一个ObjectAttachments
                        ((RpcInvocation) invocation).addObjectAttachments(selected);
                    }
                }
            } else {
                ((RpcInvocation) invocation).addObjectAttachments(RpcContext.getServerAttachment().getObjectAttachments());
            }

            Map<String, Object> contextAttachments = RpcContext.getClientAttachment().getObjectAttachments();
            if (CollectionUtils.isNotEmptyMap(contextAttachments)) {
                ((RpcInvocation) invocation).addObjectAttachments(contextAttachments);
            }
            
            //pass default timeout set by end user (ReferenceConfig)
            //检测是否超时
            Object countDown = context.getObjectAttachment(TIME_COUNTDOWN_KEY);
            if (countDown != null) {
                TimeoutCountDown timeoutCountDown = (TimeoutCountDown) countDown;
                if (timeoutCountDown.isExpired()) {
                    return AsyncRpcResult.newDefaultAsyncResult(new RpcException(RpcException.TIMEOUT_TERMINATE, "..."), invocation);
                }
            }
            
            //进行removeServerContext，这与上面的serviceContext不一样
            RpcContext.removeServerContext();
            
            //下面会调用FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法
            return invoker.invoke(invocation);
        } finally {
            //执行完毕后会对线程对应的RpcContent进行释放和清空处理，相当于对RPC调用的数据和资源的清理和释放
            RpcContext.restoreServiceContext(originServiceContext);
        }
    }
    ...
}
```

<br>

**11.RpcInvocation在调用链路中的流转分析**

其实就是如下所示，其中HeaderExchangeChannel的request()方法里的request就是RpcInvocation。

```
-> InvokerInvocationHandler.invoke()
-> InvocationUtil.invoke()
-> invoker.invoke(rpcInvocation).recreate()
-> MigrationInvoker.invoke()
-> MockClusterInvoker.invoke()

-> AbstractCluster.ClusterFilterInvoker.invoke()
-> FilterChainBuilder.CallbackRegistrationInvoker.invoke()
-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
-> ConsumerContextFilter.invoke()

-> ActiveLimitFilter.invoke()
...
-> AbstractClusterInvoker.invoke()

-> FailoverClusterInvoker.doInvoke()
-> AbstractClusterInvoker.select()
-> AbstractClusterInvoker.doSelect()
-> AbstractLoadBalance.select()
-> RandomLoadBalance.doSelect()
-> AbstractClusterInvoker.invokeWithContext()
-> ListenerInvokerWrapper.invoke()
-> AbstractInvoker.invoke()
-> AbstractInvoker.doInvokeAndReturn()
-> DubboInvoker.doInvoke()

-> DubboInvoker.doInvoke()#currentClient.request()
-> ReferenceCountExchangeClient.request()
-> HeaderExchangeClient.request()
-> HeaderExchangeChannel.request()
-> HeaderExchangeChannel.request()#channel.send()
-> AbstractPeer.send()
-> AbstractClient.send()
-> NettyChannel.send()
-> channel.writeAndFlush()
```

<br>

**12.Dubbo中的Netty编码组件分析**

Netty编码组件可以参考NettyClient的initBootstrap()方法里的bootstrap.handler()，其中encoder就是用来负责进行数据编码、处理序列化的，把对象转换为二进制数据才可以通过网络发送出去。

```
public class NettyClient extends AbstractClient {
    ...
    @Override
    protected void doOpen() throws Throwable {
        //创建NettyClientHandler，handler一般来说是用来处理网络请求的
        final NettyClientHandler nettyClientHandler = createNettyClientHandler();
        //创建Bootstrap，企业级Netty编程示范如下，对于NettyClient必须构建一个Bootstrap
        bootstrap = new Bootstrap();
        initBootstrap(nettyClientHandler);
    }
    
    protected void initBootstrap(NettyClientHandler nettyClientHandler) {
        ...
        bootstrap.handler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                //心跳请求的时间间隔
                int heartbeatInterval = UrlUtils.getHeartbeat(getUrl());
                ...
                //通过NettyCodecAdapter创建Netty中的编解码器
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                    //注册ChannelHandler
                    .addLast("decoder", adapter.getDecoder())
                    .addLast("encoder", adapter.getEncoder())//encoder就是用来负责进行数据编码、处理序列化的，把对象转换为二进制数据才可以通过网络发送出去
                    .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
                    .addLast("handler", nettyClientHandler);
                ...
            }
        });
    }
    ...
}
```

进行对象编码时会调用NettyCodecAdapter的getEncoder()的encode()方法进行处理，而接着又会调用到codec.encode()方法进行具体编码。

```
//Dubbo客户端发送请求
-> NettyCodecAdapter.encoder.encode()
-> ExchangeCodec.encode()
-> ExchangeCodec.encodeRequest()
-> DubboCodec.getSerialization()
-> DubboCodecSupport.getRequestSerialization()
-> DubboCodec.encodeRequestData()

//Dubbo服务端接收请求
-> NettyCodecAdapter.decoder.decode()
-> ExchangeCodec.decode()
-> DubboCodec.decodeBody()
-> DecodeableRpcInvocation.decode()

final public class NettyCodecAdapter {
    private final ChannelHandler encoder = new InternalEncoder();
    private final ChannelHandler decoder = new InternalDecoder();
    private final Codec2 codec;
    
    public ChannelHandler getEncoder() {
        return encoder;
    }
    
    private class InternalEncoder extends MessageToByteEncoder {
        @Override
        protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
            ChannelBuffer buffer = new NettyBackedChannelBuffer(out);
            Channel ch = ctx.channel();
            NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
            //比如下面会调用DubboCodec父类ExchangeCodec.encode()方法
            codec.encode(channel, buffer, msg);
        }
    }
    
    private class InternalDecoder extends ByteToMessageDecoder {
        @Override
        protected void decode(ChannelHandlerContext ctx, ByteBuf input, List<Object> out) throws Exception {
            //将ByteBuf封装成统一的ChannelBuffer
            ChannelBuffer message = new NettyBackedChannelBuffer(input);
            //拿到关联的Channel
            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
            //decode object.
            do {
                //记录当前readerIndex的位置
                int saveReaderIndex = message.readerIndex();
                //委托给Codec2进行解码
                Object msg = codec.decode(channel, message);
                //当前接收到的数据不足一个消息的长度，会返回NEED_MORE_INPUT，这里会重置readerIndex，继续等待接收更多的数据
                if (msg == Codec2.DecodeResult.NEED_MORE_INPUT) {
                    message.readerIndex(saveReaderIndex);
                    break;
                } else {
                    //is it possible to go here ?
                    if (saveReaderIndex == message.readerIndex()) {
                        throw new IOException("Decode without read data.");
                    }
                    //将读取到的消息传递给后面的Handler处理
                    if (msg != null) {
                        out.add(msg);
                    }
                }
            } while (message.readable());
        }
    }
    ...
}

public class ExchangeCodec extends TelnetCodec {
    @Override
    public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
        if (msg instanceof Request) {
            encodeRequest(channel, buffer, (Request) msg);
        } else if (msg instanceof Response) {
            encodeResponse(channel, buffer, (Response) msg);
        } else {
            super.encode(channel, buffer, msg);
        }
    }
    
    @Override
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
        int readable = buffer.readableBytes();
        byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
        //先把header请求头的数据读取出来
        buffer.readBytes(header);
        return decode(channel, buffer, readable, header);
    }
    
    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
        //首先获取序列化组件，比如下面会调用DubboCodec.getSerialization()方法
        Serialization serialization = getSerialization(channel, req);
        //接下来便是按照Dubbo协议进行二进制数据格式进行组织的过程了
        ...
        //对Dubbo请求进行序列化，具体在DubboCodec中实现
        encodeRequestData(channel, out, req.getData(), req.getVersion());
        ...
    }
    
    @Override
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
        ...
        return decodeBody(channel, is, header);
        ...
    }
}

public class DubboCodec extends ExchangeCodec {
    ...
    @Override
    protected Serialization getSerialization(Channel channel, Request req) {
        if (!(req.getData() instanceof Invocation)) {
            return super.getSerialization(channel, req);
        }
        return DubboCodecSupport.getRequestSerialization(channel.getUrl(), (Invocation) req.getData());
    }
}

public class DubboCodecSupport {
    public static Serialization getRequestSerialization(URL url, Invocation invocation) {
        Object serializationTypeObj = invocation.get(SERIALIZATION_ID_KEY);
        if (serializationTypeObj != null) {
            return CodecSupport.getSerializationById((byte) serializationTypeObj);
        }
        
        //默认会获取hessian2序列化组件Hessian2Serialization
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Serialization.class).getExtension(
            url.getParameter(org.apache.dubbo.remoting.Constants.SERIALIZATION_KEY, Constants.DEFAULT_REMOTING_SERIALIZATION));
    }
    ...
}

public class DubboCodec extends ExchangeCodec {
    ...
    @Override
    protected void encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
        //请求体相关的内容，都封装在了RpcInvocation
        RpcInvocation inv = (RpcInvocation) data;
        ...
    }
    
    @Override
    protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
        ...
        inv = new DecodeableRpcInvocation(frameworkModel, channel, req, is, proto);
        inv.decode();
        ...
    }
}

public class DecodeableRpcInvocation extends RpcInvocation implements Codec, Decodeable {
    ...
    @Override
    public void decode() throws Exception {
    	...
    }
}
```

<br>

**13.基于Dubbo协议编码请求对象完成Dubbo序列化**

详细过程位于ExchangeCodec的encodeRequest()方法中。需要注意的是：对RpcInvocation序列化后的数据，会写到这里buffer的第16个字节后面，而且是先写RpcInvocation二进制数据的Body的，写完之后后面才写header到buffer的前面16字节中。

Dubbo的网络协议的设计规则：2个字节的魔数 + 1个字节的标识(通过位运算能包含多种信息) + 1个字节的空位置 + 8个字节的请求ID + 4个字节的Body长度 ==> 组成16字节的Header。16字节后的就是Body，仅仅用一个字节当通过和不同的字节(FLAG_REQUEST、FLAG_TWOWAY、FLAG_EVENT)进行位或运算后可以表示成3个含义。

```
public class ExchangeCodec extends TelnetCodec {
    ...
    //Dubbo的网络协议的设计思想：
    //(1)2个字节的魔数 + 1个字节的标识(通过位运算能包含多种信息) + 1个字节的空位置 + 8个字节的请求ID + 4个字节的Body长度 ==> 组成16字节的Header
    //(2)16字节后的就是Body；
    protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
        //首先获取序列化组件，比如下面会调用DubboCodec.getSerialization()方法
        Serialization serialization = getSerialization(channel, req);
        
        //接下来便是按照Dubbo协议进行二进制数据格式进行组织的过程了
        //header.
        //该数组用来暂存协议头，也就是请求头，一共是16字节，这里面会放16个字节的数据
        byte[] header = new byte[HEADER_LENGTH];
        
        //set magic number.
        //在header数组的前两个字节中写入魔数
        Bytes.short2bytes(MAGIC, header);
        
        //set request and serialization flag.
        //仅仅用一个字节当通过和不同的字节(FLAG_REQUEST、FLAG_TWOWAY、FLAG_EVENT)进行位或运算后可以表示成3个含义
        //根据当前使用的序列化设置协议头中的序列化标志位，表示数据属于请求的还是响应的或其他，这里占1个字节
        header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());
        
        //设置协议头中的2Way标志位
        if (req.isTwoWay()) {
            header[2] |= FLAG_TWOWAY;//进行位或运算，这样通过1个字节就能表达出来序列化标志位和2Way标志位
        }
        
        //设置协议头中的Event标志位
        if (req.isEvent()) {
            header[2] |= FLAG_EVENT;//进行位或运算，这样通过1个字节就能表达出来序列化标志位和Event标志位
        }
        
        //set request id.
        //将请求ID记录到请求头header中，long型占了8个字节(byte是1字节、short是2字节、int是4字节、long是8字节)
        Bytes.long2bytes(req.getId(), header, 4);
        
        //encode request data.
        //下面开始序列化请求，并统计序列化后的字节数
        //首先使用savedWriteIndex记录ChannelBuffer当前的写入位置
        int savedWriteIndex = buffer.writerIndex();
        
        //将写入位置后移16字节
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
        
        //根据选定的序列化方式对请求进行序列化
        ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
        if (req.isHeartbeat()) {
            //heartbeat request data is always null
            bos.write(CodecSupport.getNullBytesOf(serialization));
        } else {
            //下面这一行并没有开始去进行序列化，只是把hessian2的out流 ->(嫁接) bos ->(嫁接) ChannelBuffer
            ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
            if (req.isEvent()) {
                //对事件进行序列化
                encodeEventData(channel, out, req.getData());
            } else {
                //对Dubbo请求进行序列化，具体在DubboCodec中实现
                //下面传入的out已经是hessian2的out流了，req.getData()可以获取出来对应的RpcInvocation
                //并且序列化后的数据，会写到这里buffer的第16个字节后面，而且是先写RpcInvocation二进制数据的Body的，写完之后后面才写header到buffer的前面16字节中
                encodeRequestData(channel, out, req.getData(), req.getVersion());
            }
            out.flushBuffer();
            if (out instanceof Cleanable) {
                ((Cleanable) out).cleanup();
            }
        }

        bos.flush();
        //完成序列化
        bos.close();
        
        //统计请求序列化之后，得到的字节数
        int len = bos.writtenBytes();
        
        //限制一下请求的字节长度
        checkPayload(channel, len);
        
        //将字节数写入header数组中
        Bytes.int2bytes(len, header, 12);
        
        //write
        //下面调整ChannelBuffer当前的写入位置到savedWriteIndex=0，也就是重新定位buffer到起始位置，然后将协议头header写入Buffer中
        buffer.writerIndex(savedWriteIndex);
        
        //write header.
        //将header内容写入buffer
        buffer.writeBytes(header);
        
        //最后，将ChannelBuffer的写入位置移动到正确的位置
        buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
    }
    ...
}
```

<br>

**14.基于Dubbo协议解码二进制数据完成Dubbo反序列化**

详细入口位于ExchangeCodec.decode()方法中：

```
 public class ExchangeCodec extends TelnetCodec {
    ...
    @Override
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
        int readable = buffer.readableBytes();
        byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
        //先把header请求头的数据读取出来
        buffer.readBytes(header);
        return decode(channel, buffer, readable, header);
    }
    
    @Override
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
        //check magic number.
        //检查魔数
        if (readable > 0 && header[0] != MAGIC_HIGH || readable > 1 && header[1] != MAGIC_LOW) {
            int length = header.length;
            if (header.length < readable) {
                header = Bytes.copyOf(header, readable);
                buffer.readBytes(header, length, readable - length);
            }
            
            for (int i = 1; i < header.length - 1; i++) {
                if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                    buffer.readerIndex(buffer.readerIndex() - header.length + i);
                    header = Bytes.copyOf(header, i);
                    break;
                }
            }
            return super.decode(channel, buffer, readable, header);
        }
        
        //check length.
        //检查长度
        if (readable < HEADER_LENGTH) {
            return DecodeResult.NEED_MORE_INPUT;
        }
        
        //get data length.
        //获取数据长度
        int len = Bytes.bytes2int(header, 12);
        Object obj = finishRespWhenOverPayload(channel, len, header);
        if (null != obj) {
            return obj;
        }
        
        int tt = len + HEADER_LENGTH;
        if (readable < tt) {
            return DecodeResult.NEED_MORE_INPUT;
        }
        
        //limit input stream.
        //构建长度受限的输入流
        ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);
        //完成header的检查后，便对body进行反序列化处理
        return decodeBody(channel, is, header);
        ...
    }
    ...
}
```

<br>

**15.请求Body解码与反序列化过程分析**

```
@Override
protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
    //header[2]就是serialization flag，在ExchangeCodec.encodeRequest()方法中可知
    //接下来会通过一个字节的flag和其他不同字节进行与运算得出的是否为0的结果来表达不同的意思
    byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
    
    //get request id.
    //从header提取出请求ID
    long id = Bytes.bytes2long(header, 4);
    
    //flag与FLAG_REQUEST进行与运算的结果是0，说明是response
    if ((flag & FLAG_REQUEST) == 0) {
        ...
    } else {
        //flag与FLAG_REQUEST进行与运算的结果不是0，说明是request
        //decode request.
        Request req = new Request(id);
        req.setVersion(Version.getProtocolVersion());
        
        //flag与FLAG_TWOWAY进行与运算的结果不是0，说明是2way
        req.setTwoWay((flag & FLAG_TWOWAY) != 0);
        
        //flag与FLAG_EVENT进行与运算的结果不是0，说明是event
        if ((flag & FLAG_EVENT) != 0) {
            req.setEvent(true);
        }
        
        try {
            Object data;
            if (req.isEvent()) {
                byte[] eventPayload = CodecSupport.getPayload(is);
                if (CodecSupport.isHeartBeat(eventPayload, proto)) {
                    // heart beat response data is always null;
                    data = null;
                } else {
                    ObjectInput in = CodecSupport.deserialize(channel.getUrl(), new ByteArrayInputStream(eventPayload), proto);
                    data = decodeEventData(channel, in, eventPayload);
                }
            } else {
                DecodeableRpcInvocation inv;
                //这里会检查DECODE_IN_IO_THREAD_KEY参数
                if (channel.getUrl().getParameter(DECODE_IN_IO_THREAD_KEY, DEFAULT_DECODE_IN_IO_THREAD)) {
                    inv = new DecodeableRpcInvocation(frameworkModel, channel, req, is, proto);
                    //直接调用decode()方法在当前IO线程中解码
                    inv.decode();
                } else {
                    //这里只是读取数据，不会调用decode()方法在当前IO线程中进行解码
                    inv = new DecodeableRpcInvocation(frameworkModel, channel, req, new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                }
                data = inv;
            }
            req.setData(data);
        } catch (Throwable t) {
            if (log.isWarnEnabled()) {
                log.warn("Decode request failed: " + t.getMessage(), t);
            }
            // bad request
            req.setBroken(true);
            req.setData(t);
        }
        return req;
    }
}

public class DecodeableRpcInvocation extends RpcInvocation implements Codec, Decodeable {
    ...
    public void decode() throws Exception {
        ...
        decode(channel, inputStream);
        ...
    }
    
    public Object decode(Channel channel, InputStream input) throws IOException {
    	//下面的过程是DubboCodec.encodeRequestData()方法编码过程的逆向
    	//基于hessian2构建一个支持反序列化的输入流，嫁接了底层的输入流
    	ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType).deserialize(channel.getUrl(), input);
    	this.put(SERIALIZATION_ID_KEY, serializationType);
    	
    	//首先读取Dubbo Version
    	String dubboVersion = in.readUTF();
    	request.setVersion(dubboVersion);
    	setAttachment(DUBBO_VERSION_KEY, dubboVersion);
    	
    	//读取path
    	String path = in.readUTF();
    	setAttachment(PATH_KEY, path);
    	
    	//读取版本version
    	String version = in.readUTF();
    	setAttachment(VERSION_KEY, version);
    	
    	//读取方法名称并设置
    	setMethodName(in.readUTF());
    	
    	//读取参数类型列表
    	String desc = in.readUTF();
    	setParameterTypesDesc(desc);
     	...
    }
}
```
