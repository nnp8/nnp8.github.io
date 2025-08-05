# Dubbo源码—3.服务引用时执行RPC的流程

**大纲**

**1.对动态代理接口的方法进行调用时的入口**

**2.动态代理基于Invoker执行RPC调用**

**3.服务发现获取目标服务实例集群Invokers**

**4.DynamicDirectory进行服务发现的流程**

**5.构造DubboInvoker并建立网络连接的过程**

**6.LoadBalance的负载均衡机制**

**7.Dubbo进行RPC调用时的异步执行过程**

**8.Dubbo进行RPC调用后会如何等待响应**

**9.NettyServer会如何调用本地代码**

**10.Dubbo的分层架构原理**

<br>

**1.对动态代理接口的方法进行调用时的入口**

当执行完Reference的get()方法生成动态代理的接口后，就可以调用接口的方法了。

当动态代理的接口方法被调用时，会进入InvokerInvocationHandler的invoke()方法中。

```
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

public class InvocationUtil {
    public static Object invoke(Invoker<?> invoker, RpcInvocation rpcInvocation) throws Throwable {
        ...
        //为什么要进行recreate？
        //recreate是为了对拿到的结果进行一个拷贝，将拷贝出来的结果对象返回给业务方去使用
        //这样Dubbo框架内部自己可以持有一个结果对象，避免了和业务方共享持有和访问，而产生相互的影响
        return invoker.invoke(rpcInvocation).recreate();
    }
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/7261db06-ac7c-4468-bc01-d0bb01d3ebde" />

<br>

**2.动态代理基于Invoker执行RPC调用**

InvokerInvocationHandler的invoke()方法会调用InvocationUtil的invoke()方法，而后者又会返回"invoker.invoke(rpcInvocation).recreate()"的结果。其中，这个invoker是装饰了MockClusterInvoker的MigrationInvoker。所以，调用MigrationInvoker的invoke()方法时，会调用MockClusterInvoker的invoke()方法。

最终会调用AbstractClusterInvoker的invoke()方法，该方法会根据invocation信息列出所有的Invoker，也就是通过DynamicDirectory的list()方法进行服务发现，从而列出有哪些服务实例和有哪些Invoker，其中一个服务实例会对应一个Invoker。

```
//-> InvokerInvocationHandler.invoke()
//-> InvocationUtil.invoke()
//   invoker.invoke(rpcInvocation).recreate()
//-> MigrationInvoker.invoke()
//-> MockClusterInvoker.invoke()
//-> AbstractCluster.ClusterFilterInvoker.invoke()
//-> FilterChainBuilder.CallbackRegistrationInvoker.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> ConsumerContextFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> FutureFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> MonitorFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> RouterSnapshotFilter.invoke()
//-> AbstractClusterInvoker.invoke()
//-> AbstractClusterInvoker.list()
//-> DynamicDirectory.list()
//-> FailoverClusterInvoker.doInvoke()
```

```
public class InvocationUtil {
    public static Object invoke(Invoker<?> invoker, RpcInvocation rpcInvocation) throws Throwable {
        ...
        //调用invoke()方法发起远程调用
        //拿到AsyncRpcResult之后，调用recreate()方法获取响应结果(或是Future)
        //下面的invoker其实就是装饰了MockClusterInvoker的MigrationInvoker
        return invoker.invoke(rpcInvocation).recreate();
    }
}

public class MigrationInvoker<T> implements MigrationClusterInvoker<T> {
    //服务引用时，它是一个MockClusterInvoker
    private volatile ClusterInvoker<T> invoker;
    //这是一个名为serviceDiscoveryInvoker的ClusterInvoker实现类的实例，也是一个MockClusterInvoker
    private volatile ClusterInvoker<T> serviceDiscoveryInvoker;
    //服务引用时，它是一个MockClusterInvoker
    private volatile ClusterInvoker<T> currentAvailableInvoker;
    ...

    //真正在执行Invoker调用时，可以在这里看到MigrationInvoker里的逻辑
    //为什么会叫MigrationInvoker？因为MigrationInvoker有两个Invoker：一个是invoker，一个是serviceDiscoveryInvoker
    //执行这里的invoke()方法时会根据不同的条件去切换这两个invoker，将它们之一赋值给真正负责执行invoke()方法的currentAvailableInvoker
    //也就是有一个针对currentAvailableInvoker进行切换的过程(根据不同的条件来切换是invoker还是serviceDiscoveryInvoker)，所以migration就是从这里来的
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        ...
        //check if invoker available for each time
        return decideInvoker().invoke(invocation);
    }

    private ClusterInvoker<T> decideInvoker() {
        if (currentAvailableInvoker == serviceDiscoveryInvoker) {
            if (checkInvokerAvailable(serviceDiscoveryInvoker)) {
                return serviceDiscoveryInvoker;
            }
            return invoker;
        } else {
            return currentAvailableInvoker;
        }
    }

    public boolean checkInvokerAvailable(ClusterInvoker<T> invoker) {
        //其实就是调用MockClusterInvoker的isAvailable()来检查是否可用
        return invoker != null && !invoker.isDestroyed() && invoker.isAvailable();
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
        ///传入的invoker为AbstractCluster的内部类ClusterFilterInvoker
        //其filterInvoker属性的originalInvoker属性便为封装了DynamicDirectory的FailoverClusterInvoker
        this.invoker = invoker;
    }

    @Override
    public boolean isAvailable() {
        return directory.isAvailable();
    }

    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        ...
        //这里会直接发起正常的调用
        //每一层Invoker都会去负责自己的事情，对Invoker的调用使用了严格的责任链模式，运用了责任链模式的思想
        //A Invoker->B Invoker->C Invoker->D Invoker
        //在发起RPC调用时，由于会涉及到很多的机制，比如降级调用机制(mock)，集群容错机制，负载均衡机制等
        //此时如果只有一个Invoker，那么它里面的代码就会很多很复杂，所以才用上了责任链模式
        //下面会调用AbstractCluster内部类ClusterFilterInvoker的invoke()方法，进行正常的RPC调用
        result = this.invoker.invoke(invocation);
        ...
        return result;
    }
    ...
}

public abstract class AbstractCluster implements Cluster {
    ...
    static class ClusterFilterInvoker<T> extends AbstractClusterInvoker<T> {
        ...
        @Override
        public Result invoke(Invocation invocation) throws RpcException {
            //下面会调用FilterChainBuilder的内部类CallbackRegistrationInvoker的invoke()方法
            return filterInvoker.invoke(invocation);
        }
        ...
    }
    ...
}

@SPI(value = "default", scope = APPLICATION)
public interface FilterChainBuilder {
    ...
    class CallbackRegistrationInvoker<T, FILTER extends BaseFilter> implements Invoker<T> {
        final Invoker<T> filterInvoker;
        final List<FILTER> filters;

        public CallbackRegistrationInvoker(Invoker<T> filterInvoker, List<FILTER> filters) {
            //这是一个CopyOfFilterChainNode
            this.filterInvoker = filterInvoker;
            //[ConsumerContextFilter, FutureFilter, MonitorFilter, RouterSnapshotFilter]
            this.filters = filters;
        }

        public Result invoke(Invocation invocation) throws RpcException {
            //下面首先会调用FilterChainBuilder$CopyOfFilterChainNode.invoke()
            Result asyncResult = filterInvoker.invoke(invocation);
            ...
        }
        ...
    }

    class CopyOfFilterChainNode<T, TYPE extends Invoker<T>, FILTER extends BaseFilter> implements Invoker<T> {
        ...
        @Override
        public Result invoke(Invocation invocation) throws RpcException {
            Result asyncResult;
            try {
                InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Filter " + filter.getClass().getName() + " invoke.");
                //比如Consumer端处理时下面会调用ConsumerContextFilter的invoke()方法
                //Provider端处理时下面会最终调用AbstractProxyInvoker.invoke()方法
                asyncResult = filter.invoke(nextNode, invocation);
            } catch (Exception e) {
                ...
            }
            ...
        }
        ...
    }
    ...
}

@Activate(group = CONSUMER, order = Integer.MIN_VALUE)
public class ConsumerContextFilter implements ClusterFilter, ClusterFilter.Listener {
    ...
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        ...
        //下面又会调用FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法
        return invoker.invoke(invocation);
    }
    ...
}

@Activate(group = CommonConstants.CONSUMER)
public class FutureFilter implements ClusterFilter, ClusterFilter.Listener {
    ...
    @Override
    public Result invoke(final Invoker<?> invoker, final Invocation invocation) throws RpcException {
        //触发invoker调用的回调
        fireInvokeCallback(invoker, invocation);
        //下面又会调用FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法
        return invoker.invoke(invocation);
    }
    ...
}

@Activate(group = {PROVIDER})
public class MonitorFilter implements Filter, Filter.Listener {
    ...
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        if (invoker.getUrl().hasAttribute(MONITOR_KEY)) {
            invocation.put(MONITOR_FILTER_START_TIME, System.currentTimeMillis());
            invocation.put(MONITOR_REMOTE_HOST_STORE, RpcContext.getServiceContext().getRemoteHost());
            //count up
            getConcurrent(invoker, invocation).incrementAndGet();
        }
        //proceed invocation chain
        //下面又会调用FilterChainBuilder的内部类CopyOfFilterChainNode的invoke()方法
        return invoker.invoke(invocation);
    }
    ...
}

@Activate(group = {CONSUMER})
public class RouterSnapshotFilter implements ClusterFilter, BaseFilter.Listener {
    ...
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        //接下来会调用AbstractClusterInvoker的invoke()方法
        if (!switcher.isEnable()) {
            return invoker.invoke(invocation);
        }
        if (!logger.isInfoEnabled()) {
            return invoker.invoke(invocation);
        }
        if (!switcher.isEnable(invocation.getServiceModel().getServiceKey())) {
            return invoker.invoke(invocation);
        }
        RpcContext.getServiceContext().setNeedPrintRouterSnapshot(true);
        return invoker.invoke(invocation);
    }
    ...
}

public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    ...
    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        ...
        //下面会根据invocation信息列出所有的Invoker
        //也就是通过DynamicDirectory.list()方法进行服务发现
        //由于通过Directory获取Invoker对象列表
        //通过了解RegistryDirectory可知，其中已经调用了Router进行过滤
        //从而可以知道有哪些服务实例、有哪些Invoker
        //由于一个服务实例会对应一个Invoker，所以目标服务实例集群已变成invokers了
        List<Invoker<T>> invokers = list(invocation);
        InvocationProfilerUtils.releaseDetailProfiler(invocation);

        //通过SPI加载LoadBalance实例，也就是在这里选择出对应的负载均衡策略
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Cluster " + this.getClass().getName() + " invoke.");
        try {
            //调用由子类实现的抽象方法doInvoke()
            //下面会由子类FailoverClusterInvoker执行doInvoke()方法
            return doInvoke(invocation, invokers, loadbalance);
        } finally {
            InvocationProfilerUtils.releaseDetailProfiler(invocation);
        }
    }

    protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
        //通过DynamicDirectory.list()进行服务发现
        return getDirectory().list(invocation);
    }

    @Override
    public Directory<T> getDirectory() {
        return directory;
    }
    ...
}

public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {
    ...
    //该方法会找出一个Invoker
    //把invocation交给多个Invokers里的一个去发起RPC调用
    //其中会根据传入的loadbalance来实现负载均衡
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        //下面先做一个引用赋值
        List<Invoker<T>> copyInvokers = invokers;
        //检查Invokers
        checkInvokers(copyInvokers, invocation);
        //从RPC调用里提取一个method方法名称，从而确定需要调用的是哪个方法
        String methodName = RpcUtils.getMethodName(invocation);
        //计算最多调用次数；因为Failover策略是，如果发现调用不成功会进行重试，但默认会最多调用3次来让调用成功
        int len = calculateInvokeTimes(methodName);
        //last exception.
        RpcException le = null;
        //构建了一个跟invokers数量相等的一个list
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size());//invoked invokers.
        //基于计算出的调用次数，构建一个set；如果调用次数为3，那么意味着最多会调用3个provider服务实例
        Set<String> providers = new HashSet<String>(len);

        //对len次数进行循环
        for (int i = 0; i < len; i++) {
            //到i>0时，表示的是第一次调用失败，要开始进行重试了
            if (i > 0) {
                //检查当前服务实例是否被销毁
                checkWhetherDestroyed();
                //此时要调用DynamicDirectory进行一次invokers列表刷新
                //因为第一次调用都失败了，所以有可能invokers列表出现了变化，因而需要刷新一下invokers列表
                copyInvokers = list(invocation);
                //check again
                //再次check一下invokers是否为空
                checkInvokers(copyInvokers, invocation);
            }

            //1.下面会调用AbstractClusterInvoker的select()方法
            //select()方法会选择一个Invoker出来，具体逻辑是：
            //首先尝试用负载均衡算法去选
            //如果选出的Invoker是选过的或者不可用的，那么就直接reselect
            //也就是对选过的找一个可用的，对选不出的直接挑选下一个
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);

            RpcContext.getServiceContext().setInvokers((List) invoked);
            boolean success = false;
            try {
                //2.调用AbstractClusterInvoker的invokeWithContext()方法
                //基于该Invoker发起RPC调用，并拿到一个result
                Result result = invokeWithContext(invoker, invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("...");
                }
                success = true;
                return result;
            } catch (RpcException e) {
                //如果本次RPC调用失败了，那么就会有异常抛出来
                if (e.isBiz()) {
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                if (!success) {
                    //下面会把出现RPC调用异常的进行设置，把本次调用失败的invoker地址添加到providers
                    providers.add(invoker.getUrl().getAddress());
                }
            }
        }

        //如果最后抛出如下这个异常，则说明本次RPC调用彻底失败了
        throw new RpcException("...");
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/ad9e89dc-77da-4ade-a0bf-b76755f19e5f" />

<br>

**3.服务发现获取目标服务实例集群Invokers**

服务发现获取目标服务实例集群Invokers的入口便是AbstractClusterInvoker的list()方法，该方法会调用AbstractDirectory的list()方法来获取目标服务实例集群。

在AbstractDirectory的list()方法中，可以发现目标服务实例集群invokers其实已经缓存在内存了，所以并没有直接去zk中进行获取。同时，该方法会根据缓存的目标服务实例集群invokers，调用AbstractDirectory的子类DynamicDirectory的doList()方法来获取合适的目标服务实例集群。

而在DynamicDirectory的doList()方法中，则会调用RouterChain的route()方法对目标服务实例集群进行过滤。

```
//-> AbstractClusterInvoker.invoke()
//-> AbstractClusterInvoker.list()
//-> AbstractDirectory.list()
//-> DynamicDirectory.doList()
//-> RouterChain.route()
```

```
public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    protected Directory<T> directory;

    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        ...
        //下面会根据invocation信息列出所有的Invoker
        //也就是通过DynamicDirectory.list()方法进行服务发现
        //由于通过Directory获取Invoker对象列表
        //通过了解RegistryDirectory可知，其中已经调用了Router进行过滤
        //从而可以知道有哪些服务实例、有哪些Invoker
        //由于一个服务实例会对应一个Invoker，所以目标服务实例集群已变成invokers了
        List<Invoker<T>> invokers = list(invocation);
        InvocationProfilerUtils.releaseDetailProfiler(invocation);

        //通过SPI加载LoadBalance实例，也就是在这里选择出对应的负载均衡策略
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Cluster " + this.getClass().getName() + " invoke.");
        try {
            //调用由子类实现的抽象方法doInvoke()
            //下面会由子类FailoverClusterInvoker执行doInvoke()方法
            return doInvoke(invocation, invokers, loadbalance);
        } finally {
            InvocationProfilerUtils.releaseDetailProfiler(invocation);
        }
    }

    protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
        //通过DynamicDirectory.list()进行服务发现
        return getDirectory().list(invocation);
    }

    @Override
    public Directory<T> getDirectory() {
        return directory;
    }
    ...
}

public abstract class AbstractDirectory<T> implements Directory<T> {
    //All invokers from registry
    private volatile BitList<Invoker<T>> invokers = BitList.emptyList();
    //Valid Invoker. All invokers from registry exclude unavailable and disabled invokers.
    private volatile BitList<Invoker<T>> validInvokers = BitList.emptyList();
    ...

    @Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        ...
        BitList<Invoker<T>> availableInvokers;
        //use clone to avoid being modified at doList().
        if (invokersInitialized) {
            availableInvokers = validInvokers.clone();
        } else {
            availableInvokers = invokers.clone();
        }
        List<Invoker<T>> routedResult = doList(availableInvokers, invocation);
        ...
    }

    protected abstract List<Invoker<T>> doList(BitList<Invoker<T>> invokers, Invocation invocation) throws RpcException;
    ...
}

public abstract class DynamicDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
    ...
    @Override
    public List<Invoker<T>> doList(BitList<Invoker<T>> invokers, Invocation invocation) {
        if (forbidden && shouldFailFast) {
            //1. No service provider
            //2. Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "...");
        }
        if (multiGroup) {
            return this.getInvokers();
        }
        try {
            //Get invokers from cache, only runtime routers will be executed.
            //从缓存中获取invokers，只有运行中的routers才会被执行
            List<Invoker<T>> result = routerChain.route(getConsumerUrl(), invokers, invocation);
            return result == null ? BitList.emptyList() : result;
        } catch (Throwable t) {
            //2-1 - Failed to execute routing.
            logger.error(CLUSTER_FAILED_SITE_SELECTION, "", "", "Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
            return BitList.emptyList();
        }
    }
    ...
}

public class RouterChain<T> {
    ...
    public List<Invoker<T>> route(URL url, BitList<Invoker<T>> availableInvokers, Invocation invocation) {
        if (RpcContext.getServiceContext().isNeedPrintRouterSnapshot()) {
            return routeAndPrint(url, availableInvokers, invocation);
        } else {
            return simpleRoute(url, availableInvokers, invocation);
        }
    }
    ...
}
```

在AbstractDirectory的list()方法中，会获取缓存中的invokers进行处理，那么缓存中的invokers究竟是什么时候从zk中获取并进行缓存的呢？

其实就是在服务引用时创建动态代理的过程中，通过调用DynamicDirectory的subscribe()方法，从zk中获取服务实例并进行缓存的，其中的调用链如下所示：

```
//-> ReferenceConfig.get()
//-> ReferenceConfig.init()
//-> ReferenceConfig.createProxy() 
//-> ReferenceConfig.createInvokerForRemote() 
//-> protocolSPI.refer() 
//-> RegistryProtocol.refer()
//-> RegistryProtocol.doRefer()
//-> RegistryProtocol.interceptInvoker() 
//-> listener.onRefer()
//-> RegistryProtocol.getInvoker()
//-> RegistryProtocol.doCreateInvoker() 
//-> directory.subscribe(toSubscribeUrl(urlToRegistry))
```

```
public class RegistryProtocol implements Protocol, ScopeModelAware {
    ...
    protected <T> ClusterInvoker<T> doCreateInvoker(DynamicDirectory<T> directory, Cluster cluster, Registry registry, Class<T> type) {
        ...
        //订阅服务，toSubscribeUrl()方法会将urlToRegistry中category参数修改为"providers,configurators,routers"
        //下面会调用DynamicDirectory子类RegistryDirectory的subscribe()会进行服务发现，同时还会添加相应的监听器
        //进行服务发现时，会把从注册中心拿到的服务实例集群invokers都初始化和缓存完毕
        //并对每个服务实例都新建NettyClient来和provider建立好网络连接
        //后面通过directory.list()获取服务实例集群invokers就会从缓存中获取了
        directory.subscribe(toSubscribeUrl(urlToRegistry));
        ...
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/e3ff07a3-01dc-4847-b315-6efd70f8b064" />

<br>

**4.DynamicDirectory进行服务发现的流程**

**(1)subscribe()方法对服务订阅和发现**

**(2)doList()方法进行获取服务**

**(1)subscribe()方法对服务订阅和发现**

```
//-> ReferenceConfig.get()
//-> RegistryProtocol.doCreateInvoker()
//-> directory.subscribe(toSubscribeUrl(urlToRegistry))
//-> RegistryDirectory.subscribe()
//-> DynamicDirectory.subscribe()
//-> ListenerRegistryWrapper.subscribe()
//-> FailbackRegistry.subscribe()
//-> AbstractRegistry.subscribe() + FailbackRegistry.doSubscribe()
//-> ZookeeperRegistry.doSubscribe() + create()/notify() [通过zkClient对服务进行订阅和监听]
//-> FailbackRegistry.notify() [通知NotifyListener处理当前已有的URL等注册数据]
//-> AbstractRegistry.notify()
//-> RegistryDirectory.notify() [期间会通过NettyClient和provider建立网络连接]
//-> RegistryDirectory.toInvokers()
//-> DubboProtocol.refer()
//-> DubboProtocol.protocolBindingRefer() [构建Invoker即封装一个DubboInvoker]
//-> DubboProtocol.getClients() [DubboInvoker的构建过程便会通过NettyClient对服务实例建立网络连接]
//-> DubboProtocol.initClient()
//-> Exchangers.connect()
//-> new DubboInvoker()
```

```
public class RegistryDirectory<T> extends DynamicDirectory<T> {
    private final ModuleModel moduleModel;
    private final ConsumerConfigurationListener consumerConfigurationListener;
    private ReferenceConfigurationListener referenceConfigurationListener;
    ...

    @Override
    public void subscribe(URL url) {
        //调用DynamicDirectory.subscribe()方法
        super.subscribe(url);
        if (moduleModel.getModelEnvironment().getConfiguration().convert(Boolean.class, org.apache.dubbo.registry.Constants.ENABLE_CONFIGURATION_LISTEN, true)) {
            //将当前RegistryDirectory对象作为ConfigurationListener记录到consumerConfigurationListener中
            consumerConfigurationListener.addNotifyListener(this);
            referenceConfigurationListener = new ReferenceConfigurationListener(moduleModel, this, url);
        }
    }
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

//DynamicDirectory有两个重要的方法：
//1.subscribe()
//2.doList()
public abstract class DynamicDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
    ...
    public void subscribe(URL url) {
        setSubscribeUrl(url);
        //实现服务发现靠的就是下面这个方法
        //此时会调用ListenerRegistryWrapper.subscribe()方法
        registry.subscribe(url, this);
    }
    ...
}

public class ListenerRegistryWrapper implements Registry {
    //RegistryProtocol.export()方法中获取Registry时，这里会是一个ServiceDiscoveryRegistry
    //RegistryProtocol.refer() -> RegistryProtocol.doCreateInvoker()时，这里会是一个ZookeeperRegistry
    private final Registry registry;

    ...
    public void subscribe(URL url, NotifyListener listener) {
        ...
        //服务引用时下面会调用ZookeeperRegistry的父类FailbackRegistry的subscribe()方法
        //服务发布时下面会调用ServiceDiscoveryRegistry的subscribe()方法
        registry.subscribe(url, listener);
        ...
    }
    ...
}

public abstract class FailbackRegistry extends AbstractRegistry {
    ...
    public void subscribe(URL url, NotifyListener listener) {
        //先执行父类的订阅过程，也就是调用AbstractRegistry.subscribe()
        super.subscribe(url, listener);
        ...
        //然后调用ZookeeperRegistry.doSubscribe()方法
        doSubscribe(url, listener);
        ...
    }

    public abstract void doSubscribe(URL url, NotifyListener listener);
    ...
}

public abstract class AbstractRegistry implements Registry {
    ...
    @Override
    public void subscribe(URL url, NotifyListener listener) {
        ...
        //这个url是当前要订阅的地址，也就是要关注的那个服务接口
        //listener是那个服务的订阅监听，可能会施加多个监听器
        //这些监听器会被加入到subscribed中该url所对应的set里面
        //订阅和监听时，需要添加一个监听器，但这个NotifyListener并非是zk里面的监听器
        Set<NotifyListener> listeners = subscribed.computeIfAbsent(url, n -> new ConcurrentHashSet<>());
        listeners.add(listener);
    }
    ...
}

public class ZookeeperRegistry extends CacheableFailbackRegistry {
    ...
    public void doSubscribe(final URL url, final NotifyListener listener) {
        ...
        //create "directories".
        //尝试创建持久节点，主要是为了确保当前path在Zookeeper上存在
        //比如会对"/dubbo/服务接口/providers"进行检查，如果存在就不处理，如果不存在就去进行创建
        zkClient.create(root, false);

        //Add children (i.e. service items).
        //针对要订阅和发现的那个服务节点添加一个监听器
        //而且第一次加监听器，就会直接把子节点列表返回
        //这个子节点列表，就是指定的Provider服务实例集群地址列表
        List<String> children = zkClient.addChildListener(path, zkListener);
        ...

        //初次订阅时，会主动调用一次notify()方法，通知NotifyListener处理当前已有的URL等注册数据
        //这里会调用FailbackRegistry.notify()方法
        notify(url, listener, urls);
    }
    ...
}

//这是一个支持故障和重试的抽象类
public abstract class FailbackRegistry extends AbstractRegistry {
    ...
    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        //在订阅和发现时，必然需要直接定位到对应的Providers集群地址
        //所以这些地址需要存储在本地，这样才能方便后续通过directory获取，保证随时可以拿到对应的集群地址
        ...
        //FailbackRegistry的doNotify()方法实际上就是调用父类AbstractRegistry.notify()方法，没有其他逻辑
        doNotify(url, listener, urls);
        ...
    }

    protected void doNotify(URL url, NotifyListener listener, List<URL> urls) {
        //FailbackRegistry的doNotify()方法实际上就是调用父类AbstractRegistry.notify()方法，没有其他逻辑
        super.notify(url, listener, urls);
    }
    ...
}

public abstract class AbstractRegistry implements Registry {
    ...
    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        ...
        for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
            String category = entry.getKey();
            List<URL> categoryList = entry.getValue();
            categoryNotified.put(category, categoryList);
            //下面会调用RegistryDirectory的notify()方法，通过NettyClient建立网络连接
            listener.notify(categoryList);
            ...
        }
    }
    ...
}

public class RegistryDirectory<T> extends DynamicDirectory<T> {
    ...
    public synchronized void notify(List<URL> urls) {
        //按照category进行分类，分成configurators、routers、providers三类
        Map<String, List<URL>> categoryUrls = urls.stream()
            .filter(Objects::nonNull)
            .filter(this::isValidCategory)
            .filter(this::isNotCompatibleFor26x)
            .collect(Collectors.groupingBy(this::judgeCategory));
        //获取configurators类型的URL，并转换成Configurator对象
        List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
        this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);
        //获取routers类型的URL，并转成Router对象，添加到RouterChain中
        List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
        toRouters(routerURLs).ifPresent(this::addRouters);
        //providers
        //获取providers类型的URL，调用refreshOverrideAndInvoker()方法进行处理
        List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
        ...

        refreshOverrideAndInvoker(providerURLs);
    }

    private synchronized void refreshOverrideAndInvoker(List<URL> urls) {
        refreshInvoker(urls);
    }

    private void refreshInvoker(List<URL> invokerUrls) {
        ...
        //将invokerUrls转换为对应的Invoker映射关系
        Map<URL, Invoker<T>> newUrlInvokerMap = toInvokers(oldUrlInvokerMap, invokerUrls);
        ...
    }

    private Map<URL, Invoker<T>> toInvokers(Map<URL, Invoker<T>> oldUrlInvokerMap, List<URL> urls) {
        ...
        //这里通过Protocol.refer()方法创建对应的Invoker对象
        //比如首先会调用Protocol$Adaptive.refer()
        //然后调用ProtocolSerializationWrapper.refer()
        //接着调用ProtocolFilterWrapper.refer()
        //然后调用ProtocolListenerWrapper.refer()
        //最后调用DubboProtocol.refer()
        invoker = protocol.refer(serviceType, url);
        ...
    }
    ...
}

public class DubboProtocol extends AbstractProtocol {
    ...
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        checkDestroyed();
        return protocolBindingRefer(type, url);
    }

    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        checkDestroyed();
        //进行序列化优化，注册需要优化的类
        optimizeSerialization(url);
        //create rpc invoker，创建DubboInvoker对象
        //首先将url传入getClients()方法中，针对目标服务实例进行网络连接
        //然后再根据这个目标服务实例的url，构建一个DubboInvoker
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        //将上面创建的DubboInvoker对象添加到invoker集合中
        invokers.add(invoker);
        return invoker;
    }

    private ExchangeClient[] getClients(URL url) {
        //CONNECTIONS_KEY参数值决定了后续建立连接的数量
        int connections = url.getParameter(CONNECTIONS_KEY, 0);
        //如果没有连接数的相关配置，默认使用共享连接的方式
        if (connections == 0) {
            //确定建立共享连接的条数，默认只建立一条共享连接
            String shareConnectionsStr = StringUtils.isBlank(url.getParameter(SHARE_CONNECTIONS_KEY, (String) null))
                ? ConfigurationUtils.getProperty(url.getOrDefaultApplicationModel(), SHARE_CONNECTIONS_KEY, DEFAULT_SHARE_CONNECTIONS)
                : url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
            connections = Integer.parseInt(shareConnectionsStr);

            //创建公共ExchangeClient集合
            List<ReferenceCountExchangeClient> shareClients = getSharedClient(url, connections);
            //整理要返回的ExchangeClient集合
            ExchangeClient[] clients = new ExchangeClient[connections];
            Arrays.setAll(clients, shareClients::get);
            return clients;
        }

        //整理要返回的ExchangeClient集合
        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            //不使用公共连接的情况下，会创建单独的ExchangeClient实例
            clients[i] = initClient(url);
        }
        return clients;
    }

    private ExchangeClient initClient(URL url) {
        //获取客户端类型，并检查
        String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));
        if (StringUtils.isNotEmpty(str) && !url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                " supported client type is " + StringUtils.join(url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }
        try {
            url = new ServiceConfigURL(DubboCodec.NAME, url.getUsername(), url.getPassword(), url.getHost(), url.getPort(), url.getPath(), url.getAllParameters());
            //设置Codec2的扩展名
            url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
            //设置默认的心跳间隔
            url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));
            //如果配置了延迟创建连接的特性，则创建LazyConnectExchangeClient
            //否则，调用Exchangers的connect()方法建立网络连接
            return url.getParameter(LAZY_CONNECT_KEY, false)
                ? new LazyConnectExchangeClient(url, requestHandler)
                : Exchangers.connect(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }
    }
    ...
}
```

<br>

**(2)doList()方法进行获取服务**

```
public abstract class DynamicDirectory<T> extends AbstractDirectory<T> implements NotifyListener {
    ...
    @Override
    public List<Invoker<T>> doList(BitList<Invoker<T>> invokers, Invocation invocation) {
        if (forbidden && shouldFailFast) {
            //检测forbidden字段，当该字段在refreshInvoker()过程中设置为true时，表示无Provider可用，直接抛出异常
            //1.No service provider
            //2.Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "...");
        }

        if (multiGroup) {
            //multiGroup为true时的特殊处理
            //在refreshInvoker()方法中针对multiGroup为true的场景，已经使用Router进行了筛选，所以这里直接返回接口
            return this.getInvokers();
        }

        try {
            //会有很多种Router路由策略，比如TagRouter、ServiceRouter、AppRouter等；
            //当调用Provider时，就可以根据tag、service、app等各种配置进行筛选，把一些特定实例的provider筛选出来进行访问
            //比如灰度发布，对于Provider现在需要灰度发布一台新机器，跑的是新版本，而另外2台机器依旧是旧版本
            //这时就需要所提供的Provider服务，必须要能把请求分发给老版本的旧机器，而新版本的新机器则暂时还不能访问或者只分发一点点流量

            //Get invokers from cache, only runtime routers will be executed.
            //从缓存中获取invokers，只有运行中的routers才会被执行
            //也就是通过RouterChain.route()方法路由Invoker集合，最终得到符合路由条件的Invoker集合
            List<Invoker<T>> result = routerChain.route(getConsumerUrl(), invokers, invocation);
            return result == null ? BitList.emptyList() : result;
        } catch (Throwable t) {
            logger.error("Failed to execute router: " + getUrl() + ", because: " + t.getMessage(), t);
            return BitList.emptyList();
        }
    }
    ...
}

//Router chain
//路由链条其实就是一些路由规则
//通过这些路由规则的过滤，能够把可以用来访问的目标服务实例invokers筛选出来
//使用了责任链模式的链条有：filter链条，invoker链条，router链条
//会有很多种Router路由策略，比如TagRouter、ServiceRouter、AppRouter等
//当调用Provider时，就可以根据tag、service、app等各种配置进行筛选，把一些特定实例的provider筛选出来进行访问
//比如灰度发布，对于Provider现在需要灰度发布一台新机器，跑的是新版本，而另外2台机器依旧是旧版本
//这时就需要所提供的Provider服务，必须要能把请求分发给老版本的旧机器，而新版本的新机器则暂时还不能访问或者只分发一点点流量
public class RouterChain<T> {
    //当前RouterChain中要使用的Router集合
    private volatile List<Router> routers = Collections.emptyList();
    //RouterChain中的链表头节点
    private volatile StateRouter<T> headStateRouter;
    ...

    public List<Invoker<T>> route(URL url, BitList<Invoker<T>> availableInvokers, Invocation invocation) {
        if (RpcContext.getServiceContext().isNeedPrintRouterSnapshot()) {
            return routeAndPrint(url, availableInvokers, invocation);
        } else {
            return simpleRoute(url, availableInvokers, invocation);
        }
    }

    public List<Invoker<T>> simpleRoute(URL url, BitList<Invoker<T>> availableInvokers, Invocation invocation) {
        BitList<Invoker<T>> resultInvokers = availableInvokers.clone();

        //1.route state router
        //从头开始执行StateRouter链里的每个StateRouter的route()方法
        resultInvokers = headStateRouter.route(resultInvokers, url, invocation, false, null);
        if (resultInvokers.isEmpty() && (shouldFailFast || routers.isEmpty())) {
            printRouterSnapshot(url, availableInvokers, invocation);
            return BitList.emptyList();
        }

        if (routers.isEmpty()) {
            return resultInvokers;
        }
        List<Invoker<T>> commonRouterResult = resultInvokers.cloneToArrayList();

        //2.route common router
        //遍历routers字段，逐个调用Router对象的route()方法，对invokers集合进行过滤
        for (Router router : routers) {
            //Copy resultInvokers to a arrayList. BitList not support
            RouterResult<Invoker<T>> routeResult = router.route(commonRouterResult, url, invocation, false);
            commonRouterResult = routeResult.getResult();
            if (CollectionUtils.isEmpty(commonRouterResult) && shouldFailFast) {
                printRouterSnapshot(url, availableInvokers, invocation);
                return BitList.emptyList();
            }

            //stop continue routing
            if (!routeResult.isNeedContinueRoute()) {
                return commonRouterResult;
            }
        }

        if (commonRouterResult.isEmpty()) {
            printRouterSnapshot(url, availableInvokers, invocation);
            return BitList.emptyList();
        }

        return commonRouterResult;
    }
    ...
}
```

<br>

**5.构造DubboInvoker并建立网络连接的过程**

DubboProtocol的protocolBindingRefer()方法会先通过getClients()方法建立网络连接，然后才封装一个DubboInvoker对象进行返回。

```
//-> ReferenceConfig.get()
//...
//-> DubboProtocol.refer()
//-> DubboProtocol.protocolBindingRefer()
//-> DubboProtocol.getClients()
//-> DubboProtocol.getSharedClient()
//-> DubboProtocol.buildReferenceCountExchangeClientList()
//-> DubboProtocol.buildReferenceCountExchangeClient()
//-> DubboProtocol.initClient()
//-> Exchangers.connect(url, requestHandler)
//-> getExchanger(url).connect(url, handler) 
//-> Exchanger.connect(url, handler)
//-> HeaderExchanger.connect()
//-> Transporters.connect()
//-> Transporters.getTransporter(url).connect(url, handler)
//-> NettyTransporter.connect()
//-> new NettyClient(url, handler) [构建NettyClient发起连接]
//-> new AbstractClient(url, handler)
//-> NettyClient.doOpen() + NettyClient.doConnect()
```

```
public class DubboProtocol extends AbstractProtocol {
    //<host:port,Exchanger>
    //Map<String, List<ReferenceCountExchangeClient>
    private final Map<String, Object> referenceClientMap = new ConcurrentHashMap<>();
    ...

    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        checkDestroyed();
        return protocolBindingRefer(type, url);
    }

    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        checkDestroyed();
        //进行序列化优化，注册需要优化的类
        optimizeSerialization(url);
        //create rpc invoker，创建DubboInvoker对象
        //首先将url传入getClients()方法中，针对目标服务实例进行网络连接
        //然后再根据这个目标服务实例的url，构建一个DubboInvoker
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        //将上面创建的DubboInvoker对象添加到invoker集合中
        invokers.add(invoker);
        return invoker;
    }
        
    private ExchangeClient[] getClients(URL url) {
        //CONNECTIONS_KEY参数值决定了后续建立连接的数量
        int connections = url.getParameter(CONNECTIONS_KEY, 0);
        //如果没有连接数的相关配置，默认使用共享连接的方式
        if (connections == 0) {
            //确定建立共享连接的条数，默认只建立一条共享连接
            String shareConnectionsStr = StringUtils.isBlank(url.getParameter(SHARE_CONNECTIONS_KEY, (String) null))
                ? ConfigurationUtils.getProperty(url.getOrDefaultApplicationModel(), SHARE_CONNECTIONS_KEY, DEFAULT_SHARE_CONNECTIONS)
                : url.getParameter(SHARE_CONNECTIONS_KEY, (String) null);
            connections = Integer.parseInt(shareConnectionsStr);

            //创建公共ExchangeClient集合
            List<ReferenceCountExchangeClient> shareClients = getSharedClient(url, connections);
            //整理要返回的ExchangeClient集合
            ExchangeClient[] clients = new ExchangeClient[connections];
            Arrays.setAll(clients, shareClients::get);
            return clients;
        }

        //整理要返回的ExchangeClient集合
        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            //不使用公共连接的情况下，会创建单独的ExchangeClient实例
            clients[i] = initClient(url);
        }
        return clients;
    }

    private List<ReferenceCountExchangeClient> getSharedClient(URL url, int connectNum) {
        //获取对端的地址(host:port)
        String key = url.getAddress();
        //从referenceClientMap集合中，获取与该地址连接的ReferenceCountExchangeClient集合
        Object clients = referenceClientMap.get(key);
        if (clients instanceof List) {
            List<ReferenceCountExchangeClient> typedClients = (List<ReferenceCountExchangeClient>) clients;
            //检测上述客户端集合是否全部可用
            if (checkClientCanUse(typedClients)) {
                //客户端全部可用时
                batchClientRefIncr(typedClients);
                return typedClients;
            }
        }

        List<ReferenceCountExchangeClient> typedClients = null;
        synchronized (referenceClientMap) {
            for (; ;) {
                //guarantee just one thread in loading condition. 
                //And Other is waiting It had finished.
                clients = referenceClientMap.get(key);
                if (clients instanceof List) {
                    typedClients = (List<ReferenceCountExchangeClient>) clients;
                    //double check，再次检测客户端是否可用
                    if (checkClientCanUse(typedClients)) {
                        batchClientRefIncr(typedClients);
                        return typedClients;
                    } else {
                        referenceClientMap.put(key, PENDING_OBJECT);
                        break;
                    }
                } else if (clients == PENDING_OBJECT) {
                    try {
                        referenceClientMap.wait();
                    } catch (InterruptedException ignored) {
                    }
                } else {
                    referenceClientMap.put(key, PENDING_OBJECT);
                    break;
                }
            }
        }

        try {
            //connectNum must be greater than or equal to 1
            //至少一个共享连接
            connectNum = Math.max(connectNum, 1);
            //If the clients is empty, then the first initialization is
            if (CollectionUtils.isEmpty(typedClients)) {
                //如果当前Clients集合为空，则直接通过initClient()方法初始化所有共享客户端
                typedClients = buildReferenceCountExchangeClientList(url, connectNum);
            } else {
                //如果只有部分共享客户端不可用，则只需要处理这些不可用的客户端
                for (int i = 0; i < typedClients.size(); i++) {
                    ReferenceCountExchangeClient referenceCountExchangeClient = typedClients.get(i);
                    //If there is a client in the list that is no longer available, create a new one to replace him.
                    if (referenceCountExchangeClient == null || referenceCountExchangeClient.isClosed()) {
                        typedClients.set(i, buildReferenceCountExchangeClient(url));
                        continue;
                    }
                    //增加引用
                    referenceCountExchangeClient.incrementAndGetCount();
                }
            }
        } finally {
            synchronized (referenceClientMap) {
                if (typedClients == null) {
                    referenceClientMap.remove(key);
                } else {
                    referenceClientMap.put(key, typedClients);
                }
                referenceClientMap.notifyAll();
            }
        }
        return typedClients;
    }

    private List<ReferenceCountExchangeClient> buildReferenceCountExchangeClientList(URL url, int connectNum) {
        List<ReferenceCountExchangeClient> clients = new ArrayList<>();
        for (int i = 0; i < connectNum; i++) {
            clients.add(buildReferenceCountExchangeClient(url));
        }
        return clients;
    }

    private ReferenceCountExchangeClient buildReferenceCountExchangeClient(URL url) {
        ExchangeClient exchangeClient = initClient(url);
        ReferenceCountExchangeClient client = new ReferenceCountExchangeClient(exchangeClient, DubboCodec.NAME);
        //read configs
        int shutdownTimeout = ConfigurationUtils.getServerShutdownTimeout(url.getScopeModel());
        client.setShutdownWaitTime(shutdownTimeout);
        return client;
    }

    private ExchangeClient initClient(URL url) {
        //获取客户端类型，并检查
        String str = url.getParameter(CLIENT_KEY, url.getParameter(SERVER_KEY, DEFAULT_REMOTING_CLIENT));
        if (StringUtils.isNotEmpty(str) && !url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported client type: " + str + "," +
                " supported client type is " + StringUtils.join(url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).getSupportedExtensions(), " "));
        }
        try {
            url = new ServiceConfigURL(DubboCodec.NAME, url.getUsername(), url.getPassword(), url.getHost(), url.getPort(), url.getPath(), url.getAllParameters());
            //设置Codec2的扩展名
            url = url.addParameter(CODEC_KEY, DubboCodec.NAME);
            //设置默认的心跳间隔
            url = url.addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT));
            //如果配置了延迟创建连接的特性，则创建LazyConnectExchangeClient
            //否则，调用Exchangers的connect()方法建立网络连接
            return url.getParameter(LAZY_CONNECT_KEY, false)
                ? new LazyConnectExchangeClient(url, requestHandler)
                : Exchangers.connect(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to create remoting client for service(" + url + "): " + e.getMessage(), e);
        }
    }
    ...
}

public class Exchangers {
    ...
    public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        ...
        //下面会调用到HeaderExchanger.connect()方法
        //传入的handler是DubboProtocol的requestHandler
        return getExchanger(url).connect(url, handler);
    }

    public static Exchanger getExchanger(URL url) {
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        //根据SPI机制，通过model组件体系去拿到对应的SPI扩展实现类实例
        //比如获取到的是一个HeaderExchanger
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Exchanger.class).getExtension(type);
    }
    ...
}

public class HeaderExchanger implements Exchanger {
    public static final String NAME = "header";

    //NettyClient -> MultiMessageHandler -> HeartbeatHandler -> AllChannelHandler -> DecodeHandler -> HeaderExchangeHandler -> requestHandler
    //HeaderExchangeClient -> NettyClient
    @Override
    public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
        //传入的handler是DubboProtocol的requestHandler
        return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
    }
    ...
}

public class Transporters {
    ...
    public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
        ...
        //下面会调用NettyTransporter.connect()方法
        //传入的handler为装饰了DubboProtocol的requestHandler
        //返回一个NettyClient
        return getTransporter(url).connect(url, handler);
    }

    public static Transporter getTransporter(URL url) {
        //下面使用了getAdaptiveExtension()的自适应机制，针对接口动态生成代码然后创建代理类
        //代理类的方法，会根据url的参数动态提取对应的实现类的name名称，以及获取真正的需要使用的实现类
        //有了真正的实现类后，就可以去调用实现类的extension实例的方法了
        //比如下面会获取到一个NettyTransporter实例
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
    ...
}

public class NettyTransporter implements Transporter {
    ...
    public Client connect(URL url, ChannelHandler handler) throws RemotingException {
        //传入的handler为装饰了DubboProtocol的requestHandler
        //返回一个NettyClient
        return new NettyClient(url, handler);
    }
    ...
}

public abstract class AbstractClient extends AbstractEndpoint implements Client {
    ...
    public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
        //调用父类的构造方法
        super(url, handler);
        //解析URL，初始化executor
        initExecutor(url);
        //初始化底层的NIO库的相关组件
        doOpen();
        //connect. 创建底层连接
        connect();
        ...
    }

    protected void connect() throws RemotingException {
        ...
        doConnect();
        ...
    }
    ...
}

public class NettyClient extends AbstractClient {
    ...
    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
        super(url, wrapChannelHandler(url, handler));
    }

    protected void doOpen() throws Throwable {
        //创建NettyClientHandler，handler一般来说是用来处理网络请求的
        final NettyClientHandler nettyClientHandler = createNettyClientHandler();
        //创建Bootstrap，对于NettyClient必须构建一个Bootstrap
        bootstrap = new Bootstrap();
        initBootstrap(nettyClientHandler);
    }

    protected void doConnect() throws Throwable {
        long start = System.currentTimeMillis();
        //bootstrap.connect会对server端发起一个网络连接
        //但这个网络连接，并不是立刻就可以做好的
        ChannelFuture future = bootstrap.connect(getConnectAddress());
        ...
        boolean ret = future.awaitUninterruptibly(getConnectTimeout(), MILLISECONDS);
        ...
        //如果连接成功，此时就可以通过future拿到一个Channel
        //这个Channel代表着Client端和Server端Provider服务实例建立好的网络连接
        Channel newChannel = future.channel();
        ...
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/9f7bd97e-27ba-4a4d-82da-12081703bae5" />

<br>

**6.LoadBalance的负载均衡机制**

处理负载均衡的入口是AbstractClusterInvoker的initLoadBalance()方法。

```
//-> InvokerInvocationHandler.invoke()
//-> InvocationUtil.invoke()
//-> MigrationInvoker.invoke()
//-> MockClusterInvoker.invoke()
//-> AbstractCluster.ClusterFilterInvoker.invoke()
//-> FilterChainBuilder.CallbackRegistrationInvoker.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> ConsumerContextFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> FutureFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> MonitorFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> RouterSnapshotFilter.invoke()
//-> AbstractClusterInvoker.invoke()
//-> AbstractClusterInvoker.initLoadBalance()
```

具体如下：

```
//-> AbstractClusterInvoker.invoke()
//-> AbstractClusterInvoker.initLoadBalance()
//-> FailoverClusterInvoker.doInvoke()
//-> AbstractClusterInvoker.select()
//-> AbstractClusterInvoker.doSelect()
//-> AbstractLoadBalance.select()
//-> RandomLoadBalance.doSelect()

//如果要对一个目标服务实例进行RPC调用，那么负责调用的这个组件就可以叫做Invoker
public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    ...
    public Result invoke(final Invocation invocation) throws RpcException {
        //检测当前Invoker是否已销毁
        checkWhetherDestroyed();
        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Router route.");

        //下面会根据invocation信息列出所有的Invoker
        //也就是通过DynamicDirectory.list()方法进行服务发现
        //由于通过Directory获取Invoker对象列表
        //通过了解RegistryDirectory可知，其中已经调用了Router进行过滤
        //从而可以知道有哪些服务实例、有哪些Invoker
        //由于一个服务实例会对应一个Invoker，所以目标服务实例集群已变成invokers了
        List<Invoker<T>> invokers = list(invocation);
        InvocationProfilerUtils.releaseDetailProfiler(invocation);

        //通过SPI加载LoadBalance实例，也就是在这里选择出对应的负载均衡策略
        //比如下面默认会获取到一个RandomLoadBalance
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        InvocationProfilerUtils.enterDetailProfiler(invocation, () -> "Cluster " + this.getClass().getName() + " invoke.");
        try {
            //调用由子类实现的抽象方法doInvoke()
            //下面会由子类FailoverClusterInvoker执行doInvoke()方法
            return doInvoke(invocation, invokers, loadbalance);
        } finally {
            InvocationProfilerUtils.releaseDetailProfiler(invocation);
        }
    }

    //Init LoadBalance.
    //if invokers is not empty, init from the first invoke's url and invocation
    //if invokes is empty, init a default LoadBalance(RandomLoadBalance)
    protected LoadBalance initLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {
        ApplicationModel applicationModel = ScopeModelUtil.getApplicationModel(invocation.getModuleModel());
        if (CollectionUtils.isNotEmpty(invokers)) {
            //通过SPI获取负载均衡组件
            return applicationModel.getExtensionLoader(LoadBalance.class).getExtension(
                invokers.get(0).getUrl().getMethodParameter(
                    RpcUtils.getMethodName(invocation), LOADBALANCE_KEY, DEFAULT_LOADBALANCE
                )
            );
        } else {
            return applicationModel.getExtensionLoader(LoadBalance.class).getExtension(DEFAULT_LOADBALANCE);
        }
    }
    ...
}

public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {
    ...
    //该方法会找出一个Invoker
    //把invocation交给多个Invokers里的一个去发起RPC调用
    //其中会根据传入的loadbalance来实现负载均衡
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        //下面先做一个引用赋值
        List<Invoker<T>> copyInvokers = invokers;
        //检查Invokers
        checkInvokers(copyInvokers, invocation);
        //从RPC调用里提取一个method方法名称，从而确定需要调用的是哪个方法
        String methodName = RpcUtils.getMethodName(invocation);
        //计算最多调用次数；因为Failover策略是，如果发现调用不成功会进行重试，但默认会最多调用3次来让调用成功
        int len = calculateInvokeTimes(methodName);
        //retry loop.
        RpcException le = null;//last exception.
        //构建了一个跟invokers数量相等的一个list
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size());//invoked invokers.
        //基于计算出的调用次数，构建一个set；如果调用次数为3，那么意味着最多会调用3个provider服务实例
        Set<String> providers = new HashSet<String>(len);

        //对len次数进行循环
        for (int i = 0; i < len; i++) {
            //Reselect before retry to avoid a change of candidate `invokers`.
            //NOTE: if `invokers` changed, then `invoked` also lose accuracy.
            //到i>0时，表示的是第一次调用失败，要开始进行重试了
            if (i > 0) {
                //检查当前服务实例是否被销毁
                checkWhetherDestroyed();
                //此时要调用DynamicDirectory进行一次invokers列表刷新
                //因为第一次调用都失败了，所以有可能invokers列表出现了变化，因而需要刷新一下invokers列表
                copyInvokers = list(invocation);
                //check again
                //再次check一下invokers是否为空
                checkInvokers(copyInvokers, invocation);
            }

            //1.下面会调用AbstractClusterInvoker的select()方法
            //select()方法会选择一个Invoker出来，具体逻辑是：
            //首先尝试用负载均衡算法去选
            //如果选出的Invoker是选过的或者不可用的，那么就直接reselect
            //也就是对选过的找一个可用的，对选不出的直接挑选下一个
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);

            RpcContext.getServiceContext().setInvokers((List) invoked);
            boolean success = false;
            try {
                //2.调用AbstractClusterInvoker的invokeWithContext()方法
                //基于该Invoker发起RPC调用，并拿到一个result
                Result result = invokeWithContext(invoker, invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("...");
                }
                success = true;
                return result;
            } catch (RpcException e) {
                //如果本次RPC调用失败了，那么就会有异常抛出来
                if (e.isBiz()) {
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                if (!success) {
                    //下面会把出现RPC调用异常的进行设置，把本次调用失败的invoker地址添加到providers
                    providers.add(invoker.getUrl().getAddress());
                }
            }
        }

        //如果最后抛出如下这个异常，则说明本次RPC调用彻底失败了
        throw new RpcException("...");
    }
    ...
}

public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    ...
    //第一个参数是此次使用的LoadBalance实现
    //第二个参数Invocation是此次服务调用的上下文信息
    //第三个参数是待选择的Invoker集合
    //第四个参数用来记录负载均衡已经选出来、尝试过的Invoker集合
    protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
        //invokers不能为空，为空的话就直接返回
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }

        //获取调用方法名，调用的方法名称处理逻辑是：
        //如果RPC调用是null，则method方法名就是一个空字符串，否则就是RPC调用里的方法名称
        String methodName = invocation == null ? StringUtils.EMPTY_STRING : invocation.getMethodName();

        //下面会获取sticky的配置，其中sticky表示粘滞连接
        //所谓粘滞连接是指Consumer会尽可能的调用同一个Provider节点，除非这个Provider无法提供服务
        //invokers代表了服务集群地址，首先会根据invokers获取第一个invoker
        //然后拿到这个invoker的URL，接着去获取URL中对应的sticky，其默认值是false
        boolean sticky = invokers.get(0).getUrl().getMethodParameter(methodName, CLUSTER_STICKY_KEY, DEFAULT_CLUSTER_STICKY);

        //ignore overloaded method
        //检测invokers列表是否包含stickyInvoker
        //如果不包含则说明stickyInvoker代表的服务提供者挂了，此时需要将其置空
        if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
            stickyInvoker = null;
        }

        //ignore concurrency problem
        //如果开启了粘滞连接特性，则需要先判断这个Provider节点是否已经重试过了
        //下面的判断前半部分表示粘滞连接，后半部分表示stickyInvoker未重试过
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
            //检测当前stickyInvoker是否可用，如果可用，直接返回stickyInvoker
            if (availableCheck && stickyInvoker.isAvailable()) {
                return stickyInvoker;
            }
        }

        //执行到这里，说明前面的stickyInvoker为空，或者不可用
        //这里会继续调用doSelect选择新的Invoker对象
        //也就是基于LoadBalance去进行负载均衡选择一个Invoker出来
        Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);

        //是否开启粘滞，更新stickyInvoker为选择出来的invoker
        //sticky表示粘滞连接(粘滞调用)
        //所谓粘滞连接(粘滞调用)是指Consumer会尽可能的调用同一个Provider节点，除非这个Provider无法提供服务
        //也就是把一个Consumer端和一个Provider端粘在一起
        if (sticky) {
            stickyInvoker = invoker;
        }
        return invoker;
    }

    private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
        //判断是否需要进行负载均衡，Invoker集合为空，直接返回null
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }

        //只有一个Invoker对象，直接返回即可
        //如果invokers的数量就1个，那么目标Provider服务实例就一个
        //所以直接返回invokers里的第一个即可
        if (invokers.size() == 1) {
            Invoker<T> tInvoker = invokers.get(0);
            checkShouldInvalidateInvoker(tInvoker);
            return tInvoker;
        }

        //通过LoadBalance实现选择Invoker对象
        //即基于负载均衡的策略和算法选择出一个Invoker
        Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

        //Invoke是否已经尝试调用过但是失败了
        boolean isSelected = selected != null && selected.contains(invoker);

        //Invoker是否不可用
        boolean isUnavailable = availableCheck && !invoker.isAvailable() && getUrl() != null;
        if (isUnavailable) {
            invalidateInvoker(invoker);
        }

        //如果LoadBalance选出的Invoker对象，已经尝试请求过了或不可用，则需要调用reselect()方法进行重选
        if (isSelected || isUnavailable) {
            try {
                //调用reselect()方法重选
                Invoker<T> rInvoker = reselect(loadbalance, invocation, invokers, selected, availableCheck);
                if (rInvoker != null) {
                    //如果重选的Invoker对象不为空，则直接返回这个rInvoker
                    invoker = rInvoker;
                } else {
                    //Check the index of current selected invoker, if it's not the last one, choose the one at index+1.
                    //如果选来选去都是空，那么对当前Invoker就直接选择它的下一个invoker即可
                    int index = invokers.indexOf(invoker);
                    try {
                        //Avoid collision
                        //如果重选的Invoker对象为空，则返回该Invoker的下一个Invoker对象
                        invoker = invokers.get((index + 1) % invokers.size());
                    } catch (Exception e) {
                        logger.warn(e.getMessage() + " may because invokers list dynamic change, ignore.", e);
                    }
                }
            } catch (Throwable t) {
                logger.error("cluster reselect fail reason is :" + t.getMessage() + " if can not solve, you can set cluster.availablecheck=false in url", t);
            }
        }
        return invoker;
    }
    ...
}

public abstract class AbstractLoadBalance implements LoadBalance {
    ...
    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        if (CollectionUtils.isEmpty(invokers)) {
            //Invoker集合为空，直接返回null
            return null;
        }

        //Invoker集合只包含一个Invoker，则直接返回该Invoker对象
        if (invokers.size() == 1) {
            return invokers.get(0);
        }

        //Invoker集合包含多个Invoker对象时，交给doSelect()方法处理
        //交给doSelect()方法是个抽象方法，留给子类具体实现
        return doSelect(invokers, url, invocation);
    }

    protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
    ...
}

public class RandomLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "random";

    //Select one invoker between a list using a random criteria
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        //先拿到目标服务实例集群的invokers数量
        int length = invokers.size();
        if (!needWeightLoadBalance(invokers, invocation)) {
            //基于一个随机的类，通过其nextInt()方法拿到invokers数量范围之内的机器对应的invoker
            return invokers.get(ThreadLocalRandom.current().nextInt(length));
        }

        //什么是权重？
        //对于权重越高的invoker它被调用的几率会越高一些
        //而这里随机负载均衡的invokers它们被调用到的机会/几率是相同的
        //Every invoker has the same weight?
        boolean sameWeight = true;

        //the maxWeight of every invokers, the minWeight = 0 or the maxWeight of the last invoker
        //计算每个Invoker对象对应的权重，并填充到weights[]数组中
        int[] weights = new int[length];

        //The sum of weights
        int totalWeight = 0;
        for (int i = 0; i < length; i++) {
            //计算每个Invoker的权重，以及总权重totalWeight
            int weight = getWeight(invokers.get(i), invocation);
            //Sum
            totalWeight += weight;
            //save for later use
            weights[i] = totalWeight;
            //检测每个Provider的权重是否相同
            if (sameWeight && totalWeight != weight * (i + 1)) {
                sameWeight = false;
            }
        }

        //各个Invoker权重值不相等时，计算随机数落在哪个区间上
        if (totalWeight > 0 && !sameWeight) {
            //If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            //随机获取一个[0, totalWeight) 区间内的数字
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            //Return a invoker based on the random value.
            for (int i = 0; i < length; i++) {
                if (offset < weights[i]) {
                    return invokers.get(i);
                }
            }
        }

        //If all invokers have the same weight value or totalWeight=0, return evenly.
        //各个Invoker权重值相同时，随机返回一个Invoker即可
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }

    private <T> boolean needWeightLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {
        Invoker invoker = invokers.get(0);
        URL invokerUrl = invoker.getUrl();
        if (invoker instanceof ClusterInvoker) {
            invokerUrl = ((ClusterInvoker<?>) invoker).getRegistryUrl();
        }

        // Multiple registry scenario, load balance among multiple registries.
        if (REGISTRY_SERVICE_REFERENCE_PATH.equals(invokerUrl.getServiceInterface())) {
            String weight = invokerUrl.getParameter(WEIGHT_KEY);
            if (StringUtils.isNotEmpty(weight)) {
                return true;
            }
        } else {
            String weight = invokerUrl.getMethodParameter(invocation.getMethodName(), WEIGHT_KEY);
            if (StringUtils.isNotEmpty(weight)) {
                return true;
            } else {
                String timeStamp = invoker.getUrl().getParameter(TIMESTAMP_KEY);
                if (StringUtils.isNotEmpty(timeStamp)) {
                    return true;
                }
            }
        }
        return false;
    }
    ...
}
```

通过负载均衡机制选出一个Invoker之后，就可以在FailoverClusterInvoker的doInvoke()方法中，通过FailoverClusterInvoker父类AbstractClusterInvoker的invokeWithContext()方法去执行Invoker的invoke()方法进行RPC调用。

```
//-> FailoverClusterInvoker.doInvoke()
//-> AbstractClusterInvoker.invokeWithContext()
//-> ListenerInvokerWrapper.invoke()
//-> AbstractInvoker.invoke()

public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {
    ...
    //该方法会找一个Invoker，把invocation交给多个Invokers里的一个去发起RPC调用
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        ...
        //1.下面会调用AbstractClusterInvoker的select()方法
        //select()方法会选择一个Invoker出来，具体逻辑是：
        //首先尝试用负载均衡算法去选
        //如果选出的Invoker是选过的或者不可用的，那么就直接reselect
        //也就是对选过的找一个可用的，对选不出的直接挑选下一个
        Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
        ...
        //2.调用AbstractClusterInvoker的invokeWithContext()方法
        //基于该Invoker发起RPC调用，并拿到一个result
        Result result = invokeWithContext(invoker, invocation);
        ...
    }
    ...
}

public abstract class AbstractClusterInvoker<T> implements ClusterInvoker<T> {
    ...
    protected Result invokeWithContext(Invoker<T> invoker, Invocation invocation) {
        setContext(invoker);
        Result result;
        try {
            if (ProfilerSwitch.isEnableSimpleProfiler()) {
                InvocationProfilerUtils.enterProfiler(invocation, "Invoker invoke. Target Address: " + invoker.getUrl().getAddress());
            }
            //下面会调用ListenerInvokerWrapper.invoke()方法
            result = invoker.invoke(invocation);
        } finally {
            clearContext(invoker);
            InvocationProfilerUtils.releaseSimpleProfiler(invocation);
        }
        return result;
    }
    ...
}

public class ListenerInvokerWrapper<T> implements Invoker<T> {
    //这是一个DubboInvoker
    private final Invoker<T> invoker;
    ...

    public Result invoke(Invocation invocation) throws RpcException {
        //下面会调用DubboInvoker的父类AbstractInvoker.invoke()方法
        return invoker.invoke(invocation);
    }
    ...
}

public abstract class AbstractInvoker<T> implements Invoker<T> {
    ...
    public Result invoke(Invocation inv) throws RpcException {
        ...
        //首先将传入的Invocation转换为RpcInvocation
        RpcInvocation invocation = (RpcInvocation) inv;
        //prepare rpc invocation
        prepareInvocation(invocation);
        //do invoke rpc invocation and return async result
        //RPC调用返回的结果是异步的:async
        AsyncRpcResult asyncResult = doInvokeAndReturn(invocation);
        //wait rpc result if sync
        //默认情况下发起的RPC请求是异步化操作
        //但如果需要同步的话，那么是可以在这里等待同步的结果
        waitForResultIfSync(asyncResult, invocation);
        return asyncResult;
    }

    private AsyncRpcResult doInvokeAndReturn(RpcInvocation invocation) {
        ...
        //调用子类实现的doInvoke()方法
        //比如DubboInvoker的doInvoke()方法
        asyncResult = (AsyncRpcResult) doInvoke(invocation);
        ...
    }

    protected abstract Result doInvoke(Invocation invocation) throws Throwable;
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/73715169-eb1b-44b5-9f6a-d6a4744e0701" />

<br>

**7.Dubbo进行RPC调用时的异步执行过程**

在AbstractInvoker的invoke()方法中，会通过调用AbstractInvoker的doInvokeAndReturn()方法来开始异步执行RPC调用。

```
//-> InvokerInvocationHandler.invoke()
//-> InvocationUtil.invoke() [invoker.invoke(rpcInvocation).recreate()]
//...
//-> AbstractInvoker.invoke()
//-> AbstractInvoker.doInvokeAndReturn()
//-> DubboInvoker.doInvoke()
//-> AbstractInvoker.getCallbackExecutor()
//-> 配合线程池使用currentClient.request()发起RPC请求
//-> ReferenceCountExchangeClient.request()
//-> HeaderExchangeClient.request()
//-> HeaderExchangeChannel.request()
//-> DefaultFuture.newFuture()
//-> NettyClient.send() => AbstractPeer.send() => AbstractClient.send()
//-> NettyChannel.send() => channel.writeAndFlush()
```

```
public abstract class AbstractInvoker<T> implements Invoker<T> {
    ...
    public Result invoke(Invocation inv) throws RpcException {
        ...
        //首先将传入的Invocation转换为RpcInvocation
        RpcInvocation invocation = (RpcInvocation) inv;
        //prepare rpc invocation
        prepareInvocation(invocation);
        //do invoke rpc invocation and return async result，RPC调用返回的结果是异步的:async
        AsyncRpcResult asyncResult = doInvokeAndReturn(invocation);
        //wait rpc result if sync
        //默认情况下发起的RPC请求是异步化操作，但如果需要同步的话，那么是可以在这里等待同步的结果
        waitForResultIfSync(asyncResult, invocation);
        return asyncResult;
    }

    private AsyncRpcResult doInvokeAndReturn(RpcInvocation invocation) {
        ...
        //调用子类实现的doInvoke()方法
        //比如DubboInvoker的doInvoke()方法
        asyncResult = (AsyncRpcResult) doInvoke(invocation);
        ...
    }

    protected abstract Result doInvoke(Invocation invocation) throws Throwable;
   
    protected ExecutorService getCallbackExecutor(URL url, Invocation inv) {
        //首先通过SPI机制拿到ExecutorRepository——线程池存储组件
        //Dubbo会把内部所有的线程池都存放在该线程池存储组件里，或者要创建新的线程池也是通过它
        //model组件体系，其实就是封装了dubbo内部所有的公共的组件体系，可以用设计模式来形容
        //model组件设计思想就是门面模式，Model(本身是没有意义的) -> 门面，它封装了很多的组件如SPI、service数据、配置、Repository组件、BeanFactory
        //model就成为了一个门面，在整个dubbo框架里，如果要用到一些公共组件，就直接找model去获取就可以了
        if (InvokeMode.SYNC == RpcUtils.getInvokeMode(getUrl(), inv)) {
            return new ThreadlessExecutor();
        }
        return url.getOrDefaultApplicationModel()
            .getExtensionLoader(ExecutorRepository.class)
            .getDefaultExtension()
            .getExecutor(url);
    }
    ...
}

public class DubboInvoker<T> extends AbstractInvoker<T> {
    //ExchangeClient底层封装的就是NettyClient
    private final ExchangeClient[] clients;
    private final Set<Invoker<?>> invokers;
    ...

    public DubboInvoker(Class<T> serviceType, URL url, ExchangeClient[] clients, Set<Invoker<?>> invokers) {
        super(serviceType, url, new String[]{INTERFACE_KEY, GROUP_KEY, TOKEN_KEY});
        //DubboProtocol的protocolBindingRefer()方法在构建DubboInvoker时
        //会先通过DubboProtocol的getClients()方法获取ReferenceCountExchangeClient
        //然后再将这些ReferenceCountExchangeClient传入这里去构建DubboInvoker
        this.clients = clients;
        this.invokers = invokers;
        ...
    }

    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        //此次调用的方法名称
        final String methodName = RpcUtils.getMethodName(invocation);
        //向Invocation中添加附加信息
        //这里将URL的path和version添加到附加信息中
        inv.setAttachment(PATH_KEY, getUrl().getPath());
        inv.setAttachment(VERSION_KEY, version);

        //ExchangeClient和Exchange都是跟网络相关的
        ExchangeClient currentClient;
        if (clients.length == 1) {
            //选择一个ExchangeClient实例
            currentClient = clients[0];
        } else {
            //如果有多个用于网络通信的Client，就会逐个去使用，这会是一个循环使用的过程
            currentClient = clients[index.getAndIncrement() % clients.length];
        }

        try {
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            //根据调用的方法名称和配置计算此次调用的超时时间，默认是1秒
            int timeout = calculateTimeout(invocation, methodName);
            invocation.setAttachment(TIMEOUT_KEY, timeout);
            if (isOneway) {
                //不需要关注返回值的请求
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                return AsyncRpcResult.newDefaultAsyncResult(invocation);
            } else {
                //需要关注返回值的请求

                //1.获取处理响应的线程池
                //下面会调用AbstractInvoker的getCallbackExecutor()方法
                //对于同步请求，会使用ThreadlessExecutor
                //对于异步请求，则会使用共享的线程池
                ExecutorService executor = getCallbackExecutor(getUrl(), inv);

                //2.发起网络请求
                //currentClient.request()会使用上面选出的ExchangeClient执行request()方法将请求发送出去
                //下面其实会调用ReferenceCountExchangeClient的request()方法
                //thenApply()会将AppResponse封装成AsyncRpcResult返回
                CompletableFuture<AppResponse> appResponseFuture = 
                    currentClient.request(inv, timeout, executor).thenApply(obj -> (AppResponse) obj);

                //3.处理请求的响应结果
                //save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
                FutureContext.getContext().setCompatibleFuture(appResponseFuture);
                AsyncRpcResult result = new AsyncRpcResult(appResponseFuture, inv);
                result.setExecutor(executor);

                //要想拿到AppResponse结果，就需要基于CompletableFuture进行同步等待
                //只要Provider返回响应结果，就必然会写入到CompletableFuture里
                //此时就可以通过CompletableFuture取出AppResponse结果了
                //比如InvocationUtil的invoke()方法在执行"invoker.invoke(rpcInvocation).recreate()"时
                //就会调用AsyncRpcResult的recreate()方法来获取Provider返回的响应结果
                //也就是会调用到CompletableFuture的get()方法获取Provider返回的响应结果
                return result;
            }
        } catch (TimeoutException e) {
            ...
        }
    }
    ...
}

public class AsyncRpcResult implements Result {
    private CompletableFuture<AppResponse> responseFuture;
    private Invocation invocation;
    private final boolean async;
    private Executor executor;
    private RpcContext.RestoreContext storedContext;
    ...

    public AsyncRpcResult(CompletableFuture<AppResponse> future, Invocation invocation) {
        this.responseFuture = future;
        this.invocation = invocation;
        RpcInvocation rpcInvocation = (RpcInvocation) invocation;
        if ((rpcInvocation.get(PROVIDER_ASYNC_KEY) != null || InvokeMode.SYNC != rpcInvocation.getInvokeMode()) && !future.isDone()) {
            async = true;
            this.storedContext = RpcContext.clearAndStoreContext();
        } else {
            async = false;
        }
    }

    @Override
    public Object recreate() throws Throwable {
        RpcInvocation rpcInvocation = (RpcInvocation) invocation;
        //对InvokeMode模式进行判断
        if (InvokeMode.FUTURE == rpcInvocation.getInvokeMode()) {
            //如果模式为FUTURE，则表示支持异步化，所以返回的就是一个future对象
            //而这个future异步对象代表的就是一个异步化结果，此时可能有结果了也可能没有结果，这需要自己去获取
            return RpcContext.getClientAttachment().getFuture();
        } else if (InvokeMode.ASYNC == rpcInvocation.getInvokeMode()) {
            //如果模式是ASYNC，则创建默认结果返回
            return createDefaultValue(invocation).recreate();
        }
        //如果模式SYNC
        return getAppResponse().recreate();
    }

    public Result getAppResponse() {
        try {
            //DubboInvoker.doInvoke()方法返回的AsyncRpcResult会封装一个CompletableFuture进去
            //这里首先会做一个判断
            //如果CompletableFuture的isDone()方法返回true，则表示已经完成请求并拿到了响应
            //响应结果会通过HeaderExchangeHandler.received()被放到CompletableFuture里面
            if (responseFuture.isDone()) {//检测responseFuture是否已完成
                //从CompletableFuture进行阻塞式循环获取AppResponse并进行返回
                return responseFuture.get();
            }
        } catch (Exception e) {
            //This should not happen in normal request process;
            logger.error("Got exception when trying to fetch the underlying result from AsyncRpcResult.");
            throw new RpcException(e);
        }
        //根据调用方法的返回值，生成默认值
        return createDefaultValue(invocation);
    }
    
    @Override
    public Result get() throws InterruptedException, ExecutionException {
        if (executor != null && executor instanceof ThreadlessExecutor) {
            //针对ThreadlessExecutor的特殊处理，这里调用waitAndDrain()等待响应
            ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
            threadlessExecutor.waitAndDrain();
        }
        //非ThreadlessExecutor线程池的场景中
        //则直接调用Future(最底层是DefaultFuture)的get()方法阻塞
        return responseFuture.get();
    }

    @Override
    public Result get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        if (executor != null && executor instanceof ThreadlessExecutor) {
            //针对ThreadlessExecutor的特殊处理，这里调用waitAndDrain()等待响应
            ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
            threadlessExecutor.waitAndDrain();
        }
        //非ThreadlessExecutor线程池的场景中
        //则直接调用Future(最底层是DefaultFuture)的get()方法阻塞
        return responseFuture.get(timeout, unit);
    }
    ...
}
```

```
final class ReferenceCountExchangeClient implements ExchangeClient {
    private ExchangeClient client;
    ...

    //ReferenceCountExchangeClient的构入口是DubboProtocol的initClient()方法
    public ReferenceCountExchangeClient(ExchangeClient client, String codec) {
        this.client = client;
        this.referenceCount.incrementAndGet();
        this.url = client.getUrl();
    }

    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        //下面会调用HeaderExchangeClient.request()方法
        return client.request(request, timeout, executor);
    }
    ...
}

public class HeaderExchangeClient implements ExchangeClient {
    ...
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        //下面会调用HeaderExchangeChannel.request()  
        return channel.request(request, timeout, executor);
    }
    ...
}

final class HeaderExchangeChannel implements ExchangeChannel {
    ...
    public CompletableFuture<Object> request(Object request, int timeout, ExecutorService executor) throws RemotingException {
        ...
        //为什么Dubbo中要设计出来Exchange这一层
        //官方的解释是Exchange这一层会负责同步转异步
        //也就是进行Request/Response网络请求响应模型的封装
        //当Consumer端最终发送请求时，最终会执行到这里

        //这里首先会构建一个Request，把RpcInvocation对象封装为Request对象
        //也就是把一个业务语义的对象RpcInvocation，封装为网络通信里的请求响应模型里的Request对象
        //因此从这一步开始，会进入网络通信的范围，引入Request这些概念，便是多年架构经验设计出来的
        //如果直接将RpcInvocation交给Netty框架去进行处理，那么就不太符合网络通信的请求响应模型了

        //然后会创建future，并调用channel.send()进行请求发送，而这又涉及了同步转异步的过程
        //也就是最后会直接返回一个future，如果需要同步等待响应，那么可以调用future.get()方法进行阻塞

        //create request.
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        //双向请求，表示请求过去了还得返回对应的响应
        req.setTwoWay(true);
        //这个request就是我们的RpcInvocation
        req.setData(request);

        //创建完一个future后，调用方后续在收到响应结果时会再来进行处理
        //所以返回的异步future结果，与发送请求是脱离开来的
        //下面会将NettyClient、request、timeout、线程池executor，封装成一个future
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout, executor);

        //正常情况下，一般不会有超时问题
        try {
            //下面会调用NettyClient的send()方法
            //也就是调用AbstractPeer.send()方法，因为NettyClient没有覆盖其父类的send()方法
            //AbstractClient.send()又会调用NettyChannel.send()方法
            //NettyChannel.send()又会调用Netty底层的channel的writeAndFlush()方法
            //会同步发起请求，但默认不会等待请求完成了才返回
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
    ...
}

public class DefaultFuture extends CompletableFuture<Object> {
    private final Channel channel;
    private final Request request;
    private final Long id;
    private final int timeout;
    //请求ID和DefaultFuture的映射关系
    private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();
    //请求ID和Channel的映射关系
    private static final Map<Long, Channel> CHANNELS = new ConcurrentHashMap<>();

    private static final GlobalResourceInitializer<Timer> TIME_OUT_TIMER = 
        new GlobalResourceInitializer<>(
            () -> new HashedWheelTimer(new NamedThreadFactory("dubbo-future-timeout", true), 30, TimeUnit.MILLISECONDS), 
            DefaultFuture::destroy
        );
    ...

    private DefaultFuture(Channel channel, Request request, int timeout) {
        this.channel = channel;
        this.request = request;
        this.id = request.getId();
        this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
        //put into waiting map.
        FUTURES.put(id, this);
        CHANNELS.put(id, channel);
    }

    public static DefaultFuture newFuture(Channel channel, Request request, int timeout, ExecutorService executor) {
        //1.根据channel、request、timeout创建DefaultFuture对象
        //然后设置其线程池为executor，该过程只是简单的赋值操作
        final DefaultFuture future = new DefaultFuture(channel, request, timeout);
        future.setExecutor(executor);
        //ThreadlessExecutor needs to hold the waiting future in case of circuit return.
        if (executor instanceof ThreadlessExecutor) {
            ((ThreadlessExecutor) executor).setWaitingFuture(future);
        }
        //2.timeout check
        timeoutCheck(future);
        return future;
    }

    //利用TIME_OUT_TIMER时间轮启动一个定时任务进行task超时检查
    private static void timeoutCheck(DefaultFuture future) {
        TimeoutCheckTask task = new TimeoutCheckTask(future.getId());
        future.timeoutCheckTask = TIME_OUT_TIMER.get().newTimeout(task, future.getTimeout(), TimeUnit.MILLISECONDS);
    }
    ...
}

public class NettyClient extends AbstractClient {
    ...
    ...
}

public abstract class AbstractPeer implements Endpoint, ChannelHandler {
    ...
    public void send(Object message) throws RemotingException {
        //false表示不会同步等待发送完毕后才返回
        //从入参为false可知，NettyChannel发送数据时，默认就是异步化的
        //比如下面会调用AbstractClient.send()
        send(message, url.getParameter(Constants.SENT_KEY, false));
    }
    ...
}

public abstract class AbstractClient extends AbstractEndpoint implements Client {
    private final Lock connectLock = new ReentrantLock();
    private final boolean needReconnect;
    protected volatile ExecutorService executor;
    ...

    public AbstractClient(URL url, ChannelHandler handler) throws RemotingException {
        //调用父类的构造方法
        super(url, handler);
        //解析URL，初始化needReconnect值
        needReconnect = url.getParameter(Constants.SEND_RECONNECT_KEY, true);
        //解析URL，初始化executor线程池
        initExecutor(url);

        try {
            //初始化底层的NIO库的相关组件
            doOpen();
        } catch (Throwable t) {
            close();
            throw new RemotingException("...");
        }

        try {
            //connect创建底层连接
            connect();
        } catch (Throwable t) {
            close();
            throw new RemotingException("...");
        }
    }

    private void initExecutor(URL url) {
        ExecutorRepository executorRepository = url.getOrDefaultApplicationModel()
            .getExtensionLoader(ExecutorRepository.class).getDefaultExtension();
        url = url.addParameter(THREAD_NAME_KEY, CLIENT_THREAD_POOL_NAME)
            .addParameterIfAbsent(THREADPOOL_KEY, DEFAULT_CLIENT_THREADPOOL);
        executor = executorRepository.createExecutorIfAbsent(url);
    }

    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        if (needReconnect && !isConnected()) {
            connect();
        }
        //比如下面会获取到NettyChannel
        Channel channel = getChannel();
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
        }
        //比如下面会调用NettyChannel.send()
        //因为NettyClient的doConnect()方法建立连接时会拿到一个Channel
        //然后将这个Channel赋值到NettyClient的channel属性中
        channel.send(message, sent);
    }

    protected void connect() throws RemotingException {
        connectLock.lock();
        try {
            doConnect();
        } finally {
            connectLock.unlock();
        }
    }
    ...
}

final class NettyChannel extends AbstractChannel {
    ...
    public void send(Object message, boolean sent) throws RemotingException {
        ...
        //下面这行代码的channel是一个NioSocketChannel
        ChannelFuture future = channel.writeAndFlush(message);
        if (sent) {
            //wait timeout ms
            timeout = getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        ...
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/503ab325-7b26-4b20-bcfd-ea5d975719d5" />

<br>

**8.Dubbo进行RPC调用后会如何等待响应**

AbstractInvoker的waitForResultIfSync()方法是等待响应的入口。

```
//-> InvokerInvocationHandler.invoke()
//-> InvocationUtil.invoke() [invoker.invoke(rpcInvocation).recreate()]
//...
//-> MigrationInvoker.invoke()
//-> MockClusterInvoker.invoke()
//-> AbstractCluster.ClusterFilterInvoker.invoke()
//-> FilterChainBuilder.CallbackRegistrationInvoker.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> ConsumerContextFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> FutureFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> MonitorFilter.invoke()
//-> FilterChainBuilder.CopyOfFilterChainNode.invoke()
//-> RouterSnapshotFilter.invoke()
//-> AbstractClusterInvoker.invoke()
//-> AbstractClusterInvoker.initLoadBalance()
//-> FailoverClusterInvoker.doInvoke()
//-> AbstractClusterInvoker.select()
//-> AbstractClusterInvoker.doSelect()
//-> AbstractLoadBalance.select()
//-> RandomLoadBalance.doSelect()
//-> AbstractClusterInvoker.invokeWithContext()
//-> ListenerInvokerWrapper.invoke()
//-> AbstractInvoker.invoke()
//-> AbstractInvoker.doInvokeAndReturn()
//-> DubboInvoker.doInvoke()
//-> AbstractInvoker.waitForResultIfSync()
//-> AsyncRpcResult.get()

public abstract class AbstractInvoker<T> implements Invoker<T> {
    ...
    public Result invoke(Invocation inv) throws RpcException {
        ...
        //首先将传入的Invocation转换为RpcInvocation
        RpcInvocation invocation = (RpcInvocation) inv;
        //prepare rpc invocation
        prepareInvocation(invocation);
        //do invoke rpc invocation and return async result，RPC调用返回的结果是异步的:async
        AsyncRpcResult asyncResult = doInvokeAndReturn(invocation);
        //wait rpc result if sync
        //默认情况下发起的RPC请求是异步化操作，但如果需要同步的话，那么是可以在这里等待同步的结果
        waitForResultIfSync(asyncResult, invocation);
        return asyncResult;
    }

    private AsyncRpcResult doInvokeAndReturn(RpcInvocation invocation) {
        ...
        //调用子类实现的doInvoke()方法
        //比如DubboInvoker的doInvoke()方法
        asyncResult = (AsyncRpcResult) doInvoke(invocation);
        ...
    }

    private void waitForResultIfSync(AsyncRpcResult asyncResult, RpcInvocation invocation) {
        ...
        Object timeout = invocation.getObjectAttachmentWithoutConvert(TIMEOUT_KEY);
        if (timeout instanceof Integer) {
            asyncResult.get((Integer) timeout, TimeUnit.MILLISECONDS);
        } else {
            asyncResult.get(Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
        }
        ...
    }
    ...
}

public class AsyncRpcResult implements Result {
    private CompletableFuture<AppResponse> responseFuture;
    ...

    public Result get() throws InterruptedException, ExecutionException {
        if (executor != null && executor instanceof ThreadlessExecutor) {
            //针对ThreadlessExecutor的特殊处理，这里调用waitAndDrain()等待响应
            ThreadlessExecutor threadlessExecutor = (ThreadlessExecutor) executor;
            threadlessExecutor.waitAndDrain();
        }
        //非ThreadlessExecutor线程池的场景中，则直接调用Future(最底层是DefaultFuture)的get()方法阻塞
        return responseFuture.get();
    }
    ...
}
```

<br>

**9.NettyServer会如何调用本地代码**

Consumer端发送过来的RPC请求，会由Provider端的NettyServer进行处理。

```
//-> NettyServer
//-> NettyServerHandler.channelRead()
//-> NettyChannel.getOrAddChannel() [读取请求，获取到NettyChannel]
//-> AbstractPeer.received()
//-> MultiMessageHandler.received()
//-> HeartbeatHandler.received()
//-> AllChannelHandler.received()
//-> WrappedChannelHandler.getPreferredExecutorService() [获取线程池]
//-> new ChannelEventRunnable()
//-> AllChannelHandler.received()#executor.execute() [提交一个异步任务给线程池进行处理]
//-> ChannelEventRunnable.run() [启动任务处理请求]
//-> DecodeHandler.received()
//-> HeaderExchangeHandler.received()
//-> HeaderExchangeHandler.handleRequest() [对请求进行处理]
//-> DubboProtocol.requestHandler.reply()
//-> FilterChainBuilder.FilterChainNode.invoke()
//-> AbstractProxyInvoker.invoke() [在FilterChain中最终会跑到AbstractProxyInvoker.invke()]
//-> JavassistProxyFactory.getInvoker().doInvoke()
//-> 目标实现类方法
```

```
public class NettyServer extends AbstractServer {
    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;
    private Map<String, Channel> channels;
    private io.netty.channel.Channel channel;
    private final int serverShutdownTimeoutMills;

    //NettyServer在构建的过程中，会构建和打开真正的网络服务器
    //这里是基于netty4技术去实现了网络服务器构建和打开的，一旦打开后，Netty Server就开始监听指定的端口号
    //当发现有请求过来就可以去进行处理，也就是通过ProxyInvoker去调用本地实现类的目标方法
    //入参handler其实就是DubboProtocol中的requestHandler
    //入参handler会先被HeartbeatHandler装饰，再被MultiMessageHandler装饰，再被其他修饰等等
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        //you can customize name and type of client thread pool by THREAD_NAME_KEY and THREAD_POOL_KEY in CommonConstants.
        //the handler will be wrapped: MultiMessageHandler -> HeartbeatHandler -> handler
        super(ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME), ChannelHandlers.wrap(handler, url));
        //read config before destroy
        serverShutdownTimeoutMills = ConfigurationUtils.getServerShutdownTimeout(getUrl().getOrDefaultModuleModel());
    }

    //Init and start netty server
    @Override
    protected void doOpen() throws Throwable {
        //创建ServerBootstrap
        bootstrap = new ServerBootstrap();

        //EventLoop，也可以理解为网络服务器，它会监听一个本地的端口号
        //外部系统针对本地服务器端口号发起连接、通信、网络事件时，监听的端口号就会不停的产生网络事件
        //EventLoop网络服务器，还会不停轮询监听到的网络事件
        //boss的意思是负责监听端口号是否有外部系统的连接请求，它是一个EventLoopGroup线程池
        //如果发现了网络事件，就需要进行请求处理，可以通过workerGroup里的多个线程进行并发处理
        //创建boss EventLoopGroup，线程数是1
        bossGroup = createBossGroup();

        //创建worker EventLoopGroup，线程数是CPU核数 + 1，但最多不会超过32个线程
        workerGroup = createWorkerGroup();

        //创建NettyServerHandler
        //它是一个Netty中的ChannelHandler实现，不是Dubbo Remoting层的ChannelHandler接口的实现
        final NettyServerHandler nettyServerHandler = createNettyServerHandler();

        //获取当前NettyServer创建的所有Channel
        //channels集合中的Channel不是Netty中的Channel对象，而是Dubbo Remoting层的Channel对象
        channels = nettyServerHandler.getChannels();

        //初始化ServerBootstrap，指定boss和worker EventLoopGroup
        initServerBootstrap(nettyServerHandler);

        //绑定指定的地址和端口
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());

        //等待bind操作完成
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();
    }

    protected EventLoopGroup createBossGroup() {
        return NettyEventLoopFactory.eventLoopGroup(1, EVENT_LOOP_BOSS_POOL_NAME);
    }

    protected EventLoopGroup createWorkerGroup() {
        return NettyEventLoopFactory.eventLoopGroup(
            getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
            EVENT_LOOP_WORKER_POOL_NAME
        );
    }

    //NettyServer本身就是一个handler，它的顶级父类实现了ChannelHandler接口
    //这个方法会将NettyServer自己作为参数传入NettyServerHandler之中
    protected NettyServerHandler createNettyServerHandler() {
        return new NettyServerHandler(getUrl(), this);
    }

    protected void initServerBootstrap(NettyServerHandler nettyServerHandler) {
        boolean keepalive = getUrl().getParameter(KEEP_ALIVE_KEY, Boolean.FALSE);
        bootstrap.group(bossGroup, workerGroup)
        .channel(NettyEventLoopFactory.serverSocketChannelClass())
        .option(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
        .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
        .childOption(ChannelOption.SO_KEEPALIVE, keepalive)
        .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            protected void initChannel(SocketChannel ch) throws Exception {
                //连接空闲超时时间
                int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                //NettyCodecAdapter中会创建Decoder和Encoder
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                if (getUrl().getParameter(SSL_ENABLED_KEY, false)) {
                    ch.pipeline().addLast("negotiation", new SslServerTlsHandler(getUrl()));
                }
                ch.pipeline()
                    //注册Decoder和Encoder
                    .addLast("decoder", adapter.getDecoder())
                    .addLast("encoder", adapter.getEncoder())
                    //注册IdleStateHandler
                    .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
                    //注册NettyServerHandler
                    .addLast("handler", nettyServerHandler);
                }
        });
    }
}

public class ChannelHandlers {
    private static ChannelHandlers INSTANCE = new ChannelHandlers();

    protected ChannelHandlers() {
    }

    public static ChannelHandler wrap(ChannelHandler handler, URL url) {
        return ChannelHandlers.getInstance().wrapInternal(handler, url);
    }

    protected static ChannelHandlers getInstance() {
        return INSTANCE;
    }

    static void setTestingChannelHandlers(ChannelHandlers instance) {
        INSTANCE = instance;
    }

    //MultiMessageHandler -> HeartbeatHandler -> AllChannelHandler -> DecodeHandler -> HeaderExchangeHandler -> DubboProtocol的requestHandler
    //其中AllChannelHandler是由下面代码通过SPI获取到的自适应实现类AllDispatcher的dispatch()方法返回的
    protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
        return new MultiMessageHandler(new HeartbeatHandler(url.getOrDefaultFrameworkModel().getExtensionLoader(Dispatcher.class)
            .getAdaptiveExtension().dispatch(handler, url)));
    }
}

public class NettyServerHandler extends ChannelDuplexHandler {
    private final URL url;
    private final ChannelHandler handler;
    ...

    //NettyServer会将自己作为参数传入NettyServerHandler的构造函数之中
    public NettyServerHandler(URL url, ChannelHandler handler) {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        this.url = url;
        this.handler = handler;
    }

    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        //下面会调用AbstractPeer.received()方法
        handler.received(channel, msg);
    }
    ...
}

public abstract class AbstractPeer implements Endpoint, ChannelHandler {
    ...
    public void received(Channel ch, Object msg) throws RemotingException {
        if (closed) {
            return;
        }
        //这里的handler会被如下handler先后装饰
        //MultiMessageHandler -> HeartbeatHandler -> AllChannelHandler -> DecodeHandler -> HeaderExchangeHandler -> DubboProtocol的requestHandler
        //下面会调用MultiMessageHandler.received()方法
        handler.received(ch, msg);
    }
    ...
}
```

```
public class MultiMessageHandler extends AbstractChannelHandlerDelegate {
    ...
    public void received(Channel channel, Object message) throws RemotingException {
        ...
        //下面会执行HeartbeatHandler.received()方法
        handler.received(channel, message);
    }
    ...
}

public class HeartbeatHandler extends AbstractChannelHandlerDelegate {
    ...
    public void received(Channel channel, Object message) throws RemotingException {
        //记录最近的读写事件时间戳
        setReadTimestamp(channel);

        if (isHeartbeatRequest(message)) {
            //收到心跳请求
            Request req = (Request) message;
            if (req.isTwoWay()) {
                //返回心跳响应，注意，携带请求的ID
                Response res = new Response(req.getId(), req.getVersion());
                res.setEvent(HEARTBEAT_EVENT);
                channel.send(res);
                ...
            }
            return;
        }
        ...
        //下面会执行AllChannelHandler.received()方法
        handler.received(channel, message);
    }
    ...
}

public class AllChannelHandler extends WrappedChannelHandler {
    ...
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        //通过父类WrappedChannelHandler的方法获取线程池
        ExecutorService executor = getPreferredExecutorService(message);
        try {
            //将消息封装成ChannelEventRunnable任务，提交到线程池中执行
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            //如果线程池满了，请求会被拒绝，这里会根据请求配置决定是否返回一个说明性的响应
            if (message instanceof Request && t instanceof RejectedExecutionException){
                sendFeedback(channel, (Request) message, t);
                return;
            }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }
    ...
}

public class WrappedChannelHandler implements ChannelHandlerDelegate {
    ...
    public ExecutorService getPreferredExecutorService(Object msg) {
        if (msg instanceof Response) {
            Response response = (Response) msg;
            //获取请求关联的DefaultFuture
            DefaultFuture responseFuture = DefaultFuture.getFuture(response.getId());

            if (responseFuture == null) {
                return getSharedExecutorService();
            } else {
                //如果请求关联了线程池，则会获取相关的线程来处理响应
                ExecutorService executor = responseFuture.getExecutor();
                if (executor == null || executor.isShutdown()) {
                    executor = getSharedExecutorService();
                }
                return executor;
            }
        } else {
            //返回共享线程池
            return getSharedExecutorService();
        }
    }

    public ExecutorService getSharedExecutorService() {
        if (url.getApplicationModel() == null || url.getApplicationModel().isDestroyed()) {
            return GlobalResourcesRepository.getGlobalExecutorService();
        }
        ApplicationModel applicationModel = url.getOrDefaultApplicationModel();

        //ExecutorRepository主要是负责来获取线程池的，ExecutorRepository是一个存储线程池的组件
        //下面会通过ExtensionLoader，基于SPI扩展机制，去获取具体扩展实现类对象
        ExecutorRepository executorRepository = applicationModel.getExtensionLoader(ExecutorRepository.class).getDefaultExtension();
        //传入一个url，从url里提取一些参数出来，然后根据url参数来决定会获取到什么样的线程池
        ExecutorService executor = executorRepository.getExecutor(url);
        if (executor == null) {
            //创建和构建线程池
            executor = executorRepository.createExecutorIfAbsent(url);
        }
        return executor;
    }
    ...
}

public class ChannelEventRunnable implements Runnable {
    ...
    public void run() {
        ...
        //接收到了请求
        //比如会调用DecodeHandler.received()方法
        handler.received(channel, message);
        ...
    }
    ...
}

public class DecodeHandler extends AbstractChannelHandlerDelegate {
    public void received(Channel channel, Object message) throws RemotingException {
        if (message instanceof Decodeable) {
            decode(message);
        }
        if (message instanceof Request) {
            decode(((Request) message).getData());
        }
        if (message instanceof Response) {
            decode(((Response) message).getResult());
        }
        //比如会调用HeaderExchangeHandler.received()
        handler.received(channel, message);
    }
    ...
}

public class HeaderExchangeHandler implements ChannelHandlerDelegate {
    ...
    public void received(Channel channel, Object message) throws RemotingException {
        //收到Request请求
        if (message instanceof Request) {
        //handle request.
        Request request = (Request) message;
        if (request.isEvent()) {
            //事件类型的请求
            handlerEvent(channel, request);
        } else {
            //非事件的请求
            if (request.isTwoWay()) {//twoway
                //下面会对请求进行处理
                handleRequest(exchangeChannel, request);
            } else {//oneway
                handler.received(exchangeChannel, request.getData());
            }
        }
        ...
    }

    void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
        Response res = new Response(req.getId(), req.getVersion());
        //请求解码失败
        if (req.isBroken()) {
            Object data = req.getData();
            String msg;
            if (data == null) {
                msg = null;
            } else if (data instanceof Throwable) {
                msg = StringUtils.toString((Throwable) data);
            } else {
                msg = data.toString();
            }
            res.setErrorMessage("Fail to decode request due to: " + msg);
            res.setStatus(Response.BAD_REQUEST);
            //将异常响应返回给对端
            channel.send(res);
            return;
        }

        //find handler by message class.
        Object msg = req.getData();
        try {
            //交给上层实现的ExchangeHandler进行请求处理，比如DubboProtocol的requestHandler
            CompletionStage<Object> future = handler.reply(channel, msg);
            future.whenComplete((appResult, t) -> {
                //处理结束后的回调
                try {
                    if (t == null) {//返回正常响应
                        res.setStatus(Response.OK);
                        res.setResult(appResult);
                    } else {//处理过程发生异常，设置异常信息和错误码
                        res.setStatus(Response.SERVICE_ERROR);
                        res.setErrorMessage(StringUtils.toString(t));
                    }
                    //发送响应
                    channel.send(res);
                } catch (RemotingException e) {
                    logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
                }
            });
        } catch (Throwable e) {
            res.setStatus(Response.SERVICE_ERROR);
            res.setErrorMessage(StringUtils.toString(e));
            channel.send(res);
        }
    }
    ...
}

public class DubboProtocol extends AbstractProtocol {
    ...
    private final ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
        public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {
            ...
            //将客户端发过来的message转换为Invocation对象
            Invocation inv = (Invocation) message;
            //获取此次调用Invoker对象
            Invoker<?> invoker = getInvoker(channel, inv);
            ...
            //Provider服务端会在下面执行真正的本地实现类的调用
            //比如下面会调用到FilterChainBuilder.FilterChainNode.invoke()方法
            Result result = invoker.invoke(inv);
            //返回结果
            return result.thenApply(Function.identity());
        }
        ...
    };
    ...
}

public interface FilterChainBuilder {
    class FilterChainNode<T, TYPE extends Invoker<T>, FILTER extends BaseFilter> implements Invoker<T> {
        ...
        public Result invoke(Invocation invocation) throws RpcException {
            ...
            //比如Consumer端处理时下面会调用ConsumerContextFilter的invoke()方法
            //Provider端处理时下面会最终调用AbstractProxyInvoker.invoke()方法
            asyncResult = filter.invoke(nextNode, invocation);
            ...
        }
    }
    ...
}

public abstract class AbstractProxyInvoker<T> implements Invoker<T> {
    ...
    //当Netty Server接受到了请求后，经过解析就会知道是要调用什么
    //然后会把解析出来的数据放入Invocation中
    //于是可以通过AbstractProxyInvoker的invoke()方法来进行调用
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        //执行doInvoke()方法，调用业务实现
        Object value = doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments());
    }
    ...
}

public class JavassistProxyFactory extends AbstractProxyFactory {
    ...
    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException {
        try {
            //下面会通过Wrapper创建一个包装类对象
            //该对象是动态构建出来的，它属于Wrapper的一个子类，里面会拼接一个关键的方法invokeMethod()，拼接代码由Javassist动态生成
            final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
            //下面会创建一个实现了AbstractProxyInvoker的匿名内部类
            //其doInvoker()方法会直接委托给Wrapper对象的invokeMethod()方法
            return new AbstractProxyInvoker<T>(proxy, type, url) {
                @Override
                protected Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable {
                    //当AbstractProxyInvoker.invoke()方法被调用时，便会执行到这里
                    //这里会通过类似于JDK反射的技术，调用本地实现类如DemoServiceImpl.sayHello()
                    //这个wrapper对象是由javassist技术动态生成的，已经对本地实现类进行包装
                    //这个动态生成的wrapper对象会通过javassist技术自己特有的方法
                    //在invokerMethod()方法被调用时执行本地实现类的目标方法
                    return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
                }
            };
        } catch (Throwable fromJavassist) {
            //使用JDK的反射去调用本地，这时没有动态生成的Wrapper类了
            Invoker<T> invoker = jdkProxyFactory.getInvoker(proxy, type, url);
            return invoker;
        }
    }
    ...
}
```

<br>

**10.Dubbo的分层架构原理**

**(1)架构图说明**

**(2)各层简单介绍**

**(3)各层关系说明**

<br>

**(1)架构图说明**

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/c324ab3e-1d26-427e-97da-1be888432287" />

**说明一：** 左边淡蓝色背景的为服务消费方使用的接口，右边淡绿色背景的为服务提供方使用的接口，位于中轴线上的为双方都用到的接口。

**说明二：** 图中从下至上分为十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系。每一层都可以剥离上层被复用，其中Service和Config层为API，其它各层均为SPI。

**说明三：** 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。

**说明四：** 图中蓝色虚线为初始化过程，即启动时组装链，红色实线为方法调用过程，即运行时调用链。紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。

<br>

**(2)各层简单介绍**

```
服务 -> 配置 -> 代理 -> 注册 -> 
路由 -> 监控 -> 调用 -> 
交换 -> 传输 -> 序列化

服配代注 路监调 交传序
```

一.Service服务接口层

与实际业务逻辑相关的，根据服务提供方和服务消费方的业务，设计对应的接口和实现。

```
//Interface
//Implement
```

二.Config配置层

对外配置接口，以ServiceConfig、ReferenceConfig为中心，可以直接new配置类，也可以通过spring解析配置生成配置类。

```
//ReferenceConfig
//ServiceConfig
```

三.Proxy服务代理层

服务接口透明代理，生成服务的客户端Stub和服务器端Skeleton，以ServiceProxy为中心，扩展接口为ProxyFactory。

```
//Proxy
//ProxyFactory
//Invoker
```

四.Registry注册中心层

封装服务地址的注册与发现，以服务URL为中心，扩展接口为RegistryFactory、Registry、RegistryService。

```
//NotifyListener
//Registry
//RegistryFactory
//RegistryDirectory
//RegistryProtocol
```

五.Cluster路由层

封装多个提供者的路由及负载均衡，并桥接注册中心。以Invoker为中心，扩展接口为Cluster、Directory、Router、LoadBalance。

```
//Invoker
//Directory
//LoadBalance
//Cluster
//Router
//RouterFactory
```

六.Monitor监控层

RPC调用次数和调用时间监控，以Statistics为中心，扩展接口为MonitorFactory、Monitor、MonitorService。

```
//MonitorFilter
//Monitor
//MonitorFactory
```

七.Protocol远程调用层

封装RPC调用，以Invocation、Result为中心，扩展接口为Protocol、Invoker、Exporter。

```
//Filter
//Invoker
//DubboInvoker
//Protocol
//DubboProtocol
//Exporter
//DubboExporter
//DubboHandler
```

八.Exchange信息交换层

封装请求响应模式，同步转异步。以Request、Response为中心，扩展接口为Exchanger、ExchangeChannel、ExchangeClient、ExchangeServer。

```
//ExchangeClient
//Exchanger
//ExchangeServer
//ExchangeHandler
```

九.Transport传输层

抽象Mina和Netty为统一接口，以message为中心，扩展接口为Channel、Transporter、Client、Server、Codec。

```
//Client
//Transporter
//Server
//ChannelHandler
//Codec
//Dispatcher
```

十.Serialize数据序列化层

可复用的一些工具，扩展接口为Serialization、ObjectInput、ObjectOutput、ThreadPool。

```
//ObjectOutput
//Serialization
//ObjectInput
//ThreadPool
```

<br>

**(3)各层关系说明**

**说明一：** 在RPC中，Protocol是核心层。也就是只要有Protocol+Invoker+Exporter就可以完成非透明的RPC调用，然后在Invoker的主过程上Filter拦截点。

**说明二：** 图中的Consumer和Provider是抽象概念，只是为了更直观呈现哪些类属于客户端和服务器端。不用Client和Server的原因是Dubbo使用Provider、Consumer、Registry、Monitor划分逻辑拓扑节点，保持概念统一。

**说明三：** 图中Cluster是外围概念，所以Cluster的目的是将多个Invoker伪装成一个Invoker。这样其他人只要关注Protocol层Invoker即可，Cluster的有无对其他层都不会造成影响，因为只有一个提供者时无需Cluster。

**说明四：** Proxy层封装了所有接口的透明化代理，而在其他层都以Invoker为中心。只有到了暴露给用户使用时，才用Proxy将Invoker转成接口，或将接口实现转成Invoker。也就是去掉Proxy层RPC是可以运行的，只是不那么透明，不那么看起来像调本地服务一样调远程服务。

**说明五：** Remoting实现是Dubbo协议的实现，如果选择RMI协议则整个Remoting都不会用上。Remoting内部再划分为Transport传输层和Exchange信息交换层。Transport层只负责单向消息传输，是对Mina、Netty、Grizzly的抽象，它也可以扩展UDP传输。Exchange层是在传输层之上封装了Request-Response语义。

**说明六：** Registry和Monitor实际上不算一层而是一个独立的节点，只是为了全局概览，用层的方式画在一起。
