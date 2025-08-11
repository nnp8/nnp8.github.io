# Dubbo源码—7.Provider端的主要模块下

**大纲**

**1.Dubbo服务启动源码细节**

**2.Dubbo对外暴露接口的反射处理过程**

**3.Dubbo启动过程的Refresh源码**

**4.剖析SPI自动激活@Activate注解**

**5.Dubbo服务启动的init源码流程**

**6.Dubbo服务本地发布的源码细节**

**7.Javassist技术动态生成类的源码细节**

**8.Protocol SPI自适应机制的源码细节**

**9.Protocol SPI进行本地发布的源码细节**

**10.Dubbo服务远程发布的源码细节**

**11.RegistryProtocol远程发布的源码细节**

**12.基于ZooKeeper的注册中心实现细节**

**13.DubboProtocol发布过程的源码细节**

**14.结合Exchange实例梳理SPI自适应机制原理**

**15.Exchange网络组件的源码细节**

**16.Transporter网络通信组件的源码细节**

**17.NettyServer构建和打开的全流程**

**18.NettyServerHandler的源码细节**

**19.NettyServer如何转发请求给业务线程池**

<br>

**1.Dubbo服务启动源码细节**

**Dubbo服务启动过程中，在执行服务实例的刷新操作时，首先会初始化一个ProviderConfig即provider服务实例。在下面的源码中可以看到：model组件体系对整个Dubbo源码的运行很关键的，可以认为它是SPI机制使用的入口。而ScopeModel是model组件体系的一个基础，ScopeModel类型是可以转换为ModuleModel、ApplicationModel。比如像ModuleServiceRepository，ModelEnvironment，BeanFactory等很多通用的组件都可以通过ScopeModel去获取。**

```
-> ServiceConfig.export()
-> getScopeModel().getDeployer().start()
-> AbstractConfig.refresh()
-> ServiceConfigBase.preProcessRefresh()
-> ServiceConfigBase.convertProviderIdToProvider()
-> AbstractConfig.assignProperties()
-> AbstractConfig.processExtraRefresh()
-> AbstractInterfaceConfig.processExtraRefresh()

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    public void export() {
        ...
        //ensure start module, compatible with old api usage
        //getScopeModel()返回的是一个ModuleModel
        getScopeModel().getDeployer().start();
        
        //2.执行服务实例的刷新操作，也就是ProviderConfig->MethodConfig->ArgumentConfig体系
        if (!this.isRefreshed()) {
            this.refresh();
        }
        ...
    }
    ...
}

public abstract class AbstractConfig implements Serializable {
    ...
    public void refresh() {
        //check and init before do refresh
        //preProcessRefresh()是一个抽象方法
        //AbstractConfig的子类ServiceConfigBase的该方法会初始化一个ProviderConfig即provider服务实例
        preProcessRefresh();

        //model组件体系对整个Dubbo源码的运行很关键的，可以认为它是SPI机制使用的入口
        //ScopeModel是一个基础，ScopeModel类型是可以转换为ModuleModel、ApplicationModel
        //比如像ModuleServiceRepository，ModelEnvironment，BeanFactory等很多通用的组件都可以通过ScopeModel去获取
        Environment environment = getScopeModel().getModelEnvironment();//获取Environment对象
        ...
      	
        //使用反射注入需要的方法
        assignProperties(this, environment, subProperties, subPropsConfiguration);
    	
        //process extra refresh of subclass, e.g. refresh method configs
        //该方法中preferredPrefix是关键，它的值可能是：dubbo.service.org.apache.dubbo.demo.DemoService
        //其中dubbo.service代表dubbo服务名称的一个固定前缀，属于固定拼接的
        //而中间的org.apache.dubbo.demo，则是从服务接口所在包名里截取出来的，并且最后会加上服务接口的接口名
        //所以preferredPrefix会作为当前Dubbo服务的全限定的名字
        //而这段refresh的代码，作用就是处理这个preferredPrefix以及其他相关的配置信息
        processExtraRefresh(preferredPrefix, subPropsConfiguration);
        ...
    }
    ...
}

public abstract class ServiceConfigBase<T> extends AbstractServiceConfig {
    //The provider configuration
    protected ProviderConfig provider;
    //The providerIds
    protected String providerIds;
    ...

    protected void preProcessRefresh() {
        super.preProcessRefresh();
        convertProviderIdToProvider();
        if (provider == null) {
            provider = getModuleConfigManager().getDefaultProvider().orElseThrow(() -> new IllegalStateException("Default provider is not initialized"));
        }
    }
    
    protected void convertProviderIdToProvider() {
        if (provider == null && StringUtils.hasText(providerIds)) {
            provider = getModuleConfigManager().getProvider(providerIds).orElseThrow(() -> new IllegalStateException("Provider config not found: " + providerIds));
        }
    }
    ...
}

//ProviderConfig表示Provider服务实例对应的一些默认的配置
public class ProviderConfig extends AbstractServiceConfig {
    //Provider服务实例所在的地址
    private String host;

    //Provider服务实例暴露出去对外提供服务的端口号
    private Integer port;

    //NettyServer线程模型中的线程数量
    private Integer threads;

    //处理IO的线程数量
    private Integer iothreads;

    //线程池中的队列长度
    private Integer queues;

    //能接收的最大连接数
    private Integer accepts;

    //序列化的字符集编码
    private String charset;

    //最大的负载
    private Integer payload;

    //网络IO的buffer大小
    private Integer buffer;

    //Transporter名字
    private String transporter;

    //对于不同的网络事件，可以采用不同的策略来分发给业务线程池进行处理，dispatcher便代表一种策略
    private String dispatcher;
    ...
}
```

<br>

**2.Dubbo对外暴露接口的反射处理过程**

**将对外暴露的接口里的每个方法都构建一个MethodConfig，每个MethodConfig里也需要一批ArgumentConfig。因为作为对外暴露的接口，后续要被人调用，肯定需要知道方法和参数的情况。总不可能每次都进行反射调用，拿到method和args去进行处理。所以在刚开始启动时就对接口进行解析，拿到所有的method和args来进行处理。**

```
-> ServiceConfig.export()
-> getScopeModel().getDeployer().start()
-> AbstractConfig.refresh()
-> ServiceConfigBase.preProcessRefresh()
-> ServiceConfigBase.convertProviderIdToProvider()
-> AbstractConfig.assignProperties()
-> AbstractConfig.processExtraRefresh()
-> AbstractInterfaceConfig.processExtraRefresh()

public abstract class AbstractConfig implements Serializable {
    ...
    public void refresh() {
        //check and init before do refresh
        //preProcessRefresh()是一个抽象方法
        //AbstractConfig的子类ServiceConfigBase的该方法会初始化一个ProviderConfig即provider服务实例
        preProcessRefresh();

        //model组件体系对整个Dubbo源码的运行很关键的，可以认为它是SPI机制使用的入口
        //ScopeModel是一个基础，ScopeModel类型是可以转换为ModuleModel、ApplicationModel
        //比如像ModuleServiceRepository，ModelEnvironment，BeanFactory等很多通用的组件都可以通过ScopeModel去获取
        Environment environment = getScopeModel().getModelEnvironment();//获取Environment对象
        ...
        
        //使用反射注入需要的方法
        assignProperties(this, environment, subProperties, subPropsConfiguration);
        
        //process extra refresh of subclass, e.g. refresh method configs
        //该方法中preferredPrefix是关键，它的值可能是：dubbo.service.org.apache.dubbo.demo.DemoService
        //其中dubbo.service代表dubbo服务名称的一个固定前缀，属于固定拼接的
        //而中间的org.apache.dubbo.demo，则是从服务接口所在包名里截取出来的，并且最后会加上服务接口的接口名
        //所以preferredPrefix会作为当前Dubbo服务的全限定的名字
        //而这段refresh的代码，作用就是处理这个preferredPrefix以及其他相关的配置信息
        processExtraRefresh(preferredPrefix, subPropsConfiguration);
        ...
    }
    ...
}

public abstract class AbstractInterfaceConfig extends AbstractMethodConfig {
    ...
    //该方法就是通过反射技术，对我们暴露的接口、方法和参数进行反射
    //把方法和参数都进行MethodConfig、ArgumentConfig封装，以及做一些校验处理
    @Override
    protected void processExtraRefresh(String preferredPrefix, InmemoryConfiguration subPropsConfiguration) {
        if (StringUtils.hasText(interfaceName)) {
            //先通过反射技术拿到对外暴露服务的接口
            Class<?> interfaceClass;
            interfaceClass = ClassUtils.forName(interfaceName);
            ...
            
            //Auto create MethodConfig/ArgumentConfig according to config props
            Map<String, String> configProperties = subPropsConfiguration.getProperties();
            //获取到对外暴露的接口的各种方法
            Method[] methods;
            methods = interfaceClass.getMethods();
            ...
            
            //接下来对暴露的服务接口进行处理，通过反射技术拿到接口Class
            //以及接口Class里面的方法，每个方法其实就是当前服务对外暴露的一个可以调用的接口
            for (Method method : methods) {
                if (ConfigurationUtils.hasSubProperties(configProperties, method.getName())) {
                    MethodConfig methodConfig = getMethodByName(method.getName());
                    //在这个过程中，非常关键的一点就是要把接口里的每个方法都创建一个MethodConfig，每个MethodConfig里，也需要一批ArgumentConfig
                    //因为作为对外暴露的接口在后续被人调用时，肯定需要知道方法和参数的情况，总不可能每次都进行反射调用来拿到method和args去进行处理
                    //所以在刚开始启动时就对接口进行解析，拿到所有的method和args来进行处理
                    //Add method config if not found
                    if (methodConfig == null) {
                        //会给对外暴露的服务的每个方法，创建一个对应的MethodConfig
                        methodConfig = new MethodConfig();
                        methodConfig.setName(method.getName());
                        this.addMethod(methodConfig);
                    }
                    
                    //Add argument config
                    //dubbo.service.{interfaceName}.{methodName}.{arg-index}.xxx=xxx
                    java.lang.reflect.Parameter[] arguments = method.getParameters();
                    for (int i = 0; i < arguments.length; i++) {
                        if (getArgumentByIndex(methodConfig, i) == null && hasArgumentConfigProps(configProperties, methodConfig.getName(), i)) {
                            //对方法里的每个args参数都创建一个对应的ArgumentConfig
                            ArgumentConfig argumentConfig = new ArgumentConfig();
                            argumentConfig.setIndex(i);
                            methodConfig.addArgument(argumentConfig);
                        }
                    }
                }
            }
            ...
        }
    }
    ...
}

public class MethodConfig extends AbstractMethodConfig {
    //方法名称
    private String name;
    
    //统计
    private Integer stat;
    
    //方法是否支持重试
    private Boolean retry;
    
    //方法是否具有可靠性
    private Boolean reliable;
    
    //对该方法最多可以同时允许多少线程并发访问
    private Integer executes;
    
    //方法是否过期
    private Boolean deprecated;
    
    //方法是否启用sticky机制
    //sticky表示粘滞连接(粘滞调用)，所谓粘滞连接(粘滞调用)是指Consumer会尽可能的调用同一个Provider节点，除非这个Provider无法提供服务
    //也就是把一个Consumer端和一个Provider端粘在一起
    private Boolean sticky;
    
    //是否需要返回值
    private Boolean isReturn;
    
    //异步调用的回调实例
    private Object oninvoke;
    
    //异步调用的回调方法
    private String oninvokeMethod;
    
    //方法的入参列表
    private List<ArgumentConfig> arguments;
    ...
}

public class ArgumentConfig implements Serializable {
    //该参数在方法中的入参顺序
    private Integer index = -1;
    
    //该参数在方法中的入参类型
    private String type;
    ...
}
```

<br>

**3.Dubbo启动过程的Refresh源码**

```
-> ServiceConfig.export()
-> getScopeModel().getDeployer().start()
-> AbstractConfig.refresh()
-> ServiceConfigBase.preProcessRefresh()
-> ServiceConfigBase.convertProviderIdToProvider()
-> AbstractConfig.assignProperties()
-> AbstractConfig.processExtraRefresh()
-> AbstractInterfaceConfig.processExtraRefresh()
-> ServiceConfig.postProcessRefresh()
-> ServiceConfig.checkAndUpdateSubConfigs()

public abstract class AbstractConfig implements Serializable {
    ...
    public void refresh() {
        //check and init before do refresh
        //preProcessRefresh()是一个抽象方法
        //AbstractConfig的子类ServiceConfigBase的该方法会初始化一个ProviderConfig即provider服务实例
        preProcessRefresh();

        //model组件体系对整个Dubbo源码的运行很关键的，可以认为它是SPI机制使用的入口
        //ScopeModel是一个基础，ScopeModel类型是可以转换为ModuleModel、ApplicationModel
        //比如像ModuleServiceRepository，ModelEnvironment，BeanFactory等很多通用的组件都可以通过ScopeModel去获取
        Environment environment = getScopeModel().getModelEnvironment();//获取Environment对象
        ...
        
        //使用反射注入需要的方法
        assignProperties(this, environment, subProperties, subPropsConfiguration);
        
        //process extra refresh of subclass, e.g. refresh method configs
        //该方法中preferredPrefix是关键，它的值可能是：dubbo.service.org.apache.dubbo.demo.DemoService
        //其中dubbo.service代表dubbo服务名称的一个固定前缀，属于固定拼接的
        //而中间的org.apache.dubbo.demo，则是从服务接口所在包名里截取出来的，并且最后会加上服务接口的接口名
        //所以preferredPrefix会作为当前Dubbo服务的全限定的名字
        //而这段refresh的代码，作用就是处理这个preferredPrefix以及其他相关的配置信息
        processExtraRefresh(preferredPrefix, subPropsConfiguration);
        
        ...
        postProcessRefresh();
        refreshed.set(true);
    }
    ...
}

public abstract class AbstractInterfaceConfig extends AbstractMethodConfig {
    ...
    //该方法就是通过反射技术，对我们暴露的接口、方法和参数进行反射
    //把方法和参数都进行MethodConfig、ArgumentConfig封装，以及做一些校验处理
    @Override
    protected void processExtraRefresh(String preferredPrefix, InmemoryConfiguration subPropsConfiguration) {
        ...
        //refresh MethodConfigs，刚才解析出来的一些MethodConfig
        //这个步骤属于上一个步骤的后置的处理，本质上来说是在延续对MethodConfig的处理
        List<MethodConfig> methodConfigs = this.getMethods();
        if (methodConfigs != null && methodConfigs.size() > 0) {
            //whether ignore invalid method config
            Object ignoreInvalidMethodConfigVal = getEnvironment().getConfiguration()
                .getProperty(ConfigKeys.DUBBO_CONFIG_IGNORE_INVALID_METHOD_CONFIG, "false");
            boolean ignoreInvalidMethodConfig = Boolean.parseBoolean(ignoreInvalidMethodConfigVal.toString());
            Class<?> finalInterfaceClass = interfaceClass;
            List<MethodConfig> validMethodConfigs = methodConfigs.stream().filter(methodConfig -> {
                methodConfig.setParentPrefix(preferredPrefix);
                //在这里都要去关联一下model组件
                methodConfig.setScopeModel(getScopeModel());
                methodConfig.refresh();
                //verify method config
                return verifyMethodConfig(methodConfig, finalInterfaceClass, ignoreInvalidMethodConfig);
            }).collect(Collectors.toList());
            this.setMethods(validMethodConfigs);
        }
    }
    ...
}

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    protected void postProcessRefresh() {
        super.postProcessRefresh();
        //检查并更新各项配置
        checkAndUpdateSubConfigs();
    }
    
    private void checkAndUpdateSubConfigs() {
        ...
        //下面这行代码就利用了SPI机制，使用了Activate自动激活的功能：this.getExtensionLoader(ConfigInitializer.class).getActivateExtension(...)
        List<ConfigInitializer> configInitializers = this.getExtensionLoader(ConfigInitializer.class)
            .getActivateExtension(URL.valueOf("configInitializer://", getScopeModel()), (String[]) null);
        ...
    }
    ...
}
```

<br>

**4.剖析SPI自动激活@Activate注解**

**在ServiceConfig中使用自动激活的示例入口如下：**

```
this.getExtensionLoader(ConfigInitializer.class).getActivateExtension(
    URL.valueOf("configInitializer://", getScopeModel()), (String[]) null
);
```

**这样在下面传入的URL就是"configInitializer://"。**

```
public class ExtensionLoader<T> {
    private final Map<String, Object> cachedActivates = Collections.synchronizedMap(new LinkedHashMap<>());
    private final Map<String, Set<String>> cachedActivateGroups = Collections.synchronizedMap(new LinkedHashMap<>());
    private final Map<String, String[][]> cachedActivateValues = Collections.synchronizedMap(new LinkedHashMap<>());
    ...
    
    public List<T> getActivateExtension(URL url, String[] values) {
        return getActivateExtension(url, values, null);
    }
    
    //根据@Activate注解，对SPI接口可以自动激活多个实现类
    //在ServiceConfig.checkAndUpdateSubConfigs()会通过如下代码被调用：
    //this.getExtensionLoader(ConfigInitializer.class).getActivateExtension(URL.valueOf("configInitializer://", getScopeModel()), (String[]) null);
    public List<T> getActivateExtension(URL url, String[] values, String group) {
        checkDestroyed();
        
        //solve the bug of using @SPI's wrapper method to report a null pointer exception.
        Map<Class<?>, T> activateExtensionsMap = new TreeMap<>(activateComparator);
        
        //values就是扩展名
        List<String> names = values == null ? new ArrayList<>(0) : asList(values);
        Set<String> namesSet = new HashSet<>(names);
        if (!namesSet.contains(REMOVE_VALUE_PREFIX + DEFAULT_KEY)) {
            if (cachedActivateGroups.size() == 0) {
                synchronized (cachedActivateGroups) {
                    //cache all extensions
                    if (cachedActivateGroups.size() == 0) {
                        //首先会基于cachedActivates等缓存字段来加载，缓存取不到再基于配置文件去加载扩展点接口所有的实现类
                        getExtensionClasses();
                        
                        //如果扩展点接口的所有实现类都加了@Activate注解，那么这些注解都会被cachedActivates缓存起来
                        //下面便对所有实现类的@Activate注解进行遍历和处理
                        for (Map.Entry<String, Object> entry : cachedActivates.entrySet()) {
                            //拿到缓存好的名称，也就是扩展名
                            String name = entry.getKey();
                            
                            //拿到自动激活的对象实例(@Activate注解)
                            Object activate = entry.getValue();
                            String[] activateGroup, activateValue;
                            
                            //@Activate注解中的配置
                            if (activate instanceof Activate) {
                                //从@Activate注解里提取出来对应的group和value两个参数
                                activateGroup = ((Activate) activate).group();
                                activateValue = ((Activate) activate).value();
                            } else if (activate instanceof com.alibaba.dubbo.common.extension.Activate) {
                                activateGroup = ((com.alibaba.dubbo.common.extension.Activate) activate).group();
                                activateValue = ((com.alibaba.dubbo.common.extension.Activate) activate).value();
                            } else {
                                continue;
                            }
                            
                            //把从Activate注解提取出来的group参数按扩展名name进行缓存
                            cachedActivateGroups.put(name, new HashSet<>(Arrays.asList(activateGroup)));
                            String[][] keyPairs = new String[activateValue.length][];
                            for (int i = 0; i < activateValue.length; i++) {
                                if (activateValue[i].contains(":")) {
                                    keyPairs[i] = new String[2];
                                    String[] arr = activateValue[i].split(":");
                                    keyPairs[i][0] = arr[0];
                                    keyPairs[i][1] = arr[1];
                                } else {
                                    keyPairs[i] = new String[1];
                                    keyPairs[i][0] = activateValue[i];
                                }
                            }
                            //把从Activate注解提取出来的value参数按扩展名name进行缓存
                            cachedActivateValues.put(name, keyPairs);
                        }
                    }
                }
            }
            
            //traverse all cached extensions
            cachedActivateGroups.forEach((name, activateGroup) -> {
                if (isMatchGroup(group, activateGroup)//匹配group
                    && !namesSet.contains(name)//匹配扩展名
                    && !namesSet.contains(REMOVE_VALUE_PREFIX + name)//如果包含"-"表示不激活该扩展实现
                    && isActive(cachedActivateValues.get(name), url)) {//检测URL中是否出现了指定的Key
                    //getExtensionClass()根据扩展名name去获取每个name对应的ExtensionClass扩展实现类
                    //getExtension()根据扩展名name去获取一个extension扩展实现类实例
                    //所以，凡是打了@Activate注解的实现类，都会在这里被加载和初始化
                    activateExtensionsMap.put(getExtensionClass(name), getExtension(name));//加载扩展实现
                }
            });
        }
        
        if (namesSet.contains(DEFAULT_KEY)) {
            ArrayList<T> extensionsResult = new ArrayList<>(activateExtensionsMap.size() + names.size());
            for (String name : names) {
                //通过"-"开头的配置明确指定不激活的扩展实现，直接就忽略了
                if (name.startsWith(REMOVE_VALUE_PREFIX) || namesSet.contains(REMOVE_VALUE_PREFIX + name)) {
                    continue;
                }
                if (DEFAULT_KEY.equals(name)) {
                    extensionsResult.addAll(activateExtensionsMap.values());
                    continue;
                }
                if (containsExtension(name)) {
                    extensionsResult.add(getExtension(name));
                }
            }
            return extensionsResult;
        } else {
            // add extensions, will be sorted by its order
            for (String name : names) {
                if (name.startsWith(REMOVE_VALUE_PREFIX) || namesSet.contains(REMOVE_VALUE_PREFIX + name)) {
                    continue;
                }
                if (DEFAULT_KEY.equals(name)) {
                    continue;
                }
                if (containsExtension(name)) {
                    activateExtensionsMap.put(getExtensionClass(name), getExtension(name));
                }
            }
            return new ArrayList<>(activateExtensionsMap.values());
        }
    }
    ...
}
```

**SPI的自动激活机制的用处：某SPI扩展接口可能会有很多实现类，在Dubbo运行时，并非只能使用其中一个实现类，也可以使用多个实现类。而@Activate自动激活机制，便可以激活一个SPI扩展接口的很多实现类来一起使用。**

**像下面代码中的postProcessConfig()方法：最后会将获取到的ConfigPostProcessor的多个实现类进行遍历，然后让每个实现类都调用各自的方法执行处理。**

```
public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    protected void postProcessRefresh() {
        super.postProcessRefresh();
        //检查并更新各项配置
        checkAndUpdateSubConfigs();
    }
    
    private void checkAndUpdateSubConfigs() {
        ...
        //下面这行代码就利用了SPI机制，使用了Activate自动激活的功能：this.getExtensionLoader(ConfigInitializer.class).getActivateExtension(...)
        List<ConfigInitializer> configInitializers = this.getExtensionLoader(ConfigInitializer.class)
            .getActivateExtension(URL.valueOf("configInitializer://", getScopeModel()), (String[]) null);
        ...
        postProcessConfig();
    }
    
    private void postProcessConfig() {
        //SPI的自动激活机制的用处：
        //某SPI扩展接口可能会有很多实现类，在Dubbo运行时，并非只能使用其中一个实现类，也可以使用多个实现类
        //而@Activate自动激活机制，便可以激活一个SPI扩展接口的很多实现类
        List<ConfigPostProcessor> configPostProcessors = this.getExtensionLoader(ConfigPostProcessor.class)
            .getActivateExtension(URL.valueOf("configPostProcessor://", getScopeModel()), (String[]) null);
        
        //将获取到的ConfigPostProcessor的多个实现类进行遍历，每个实现类都调用各自的postProcessServiceConfig()方法进行处理
        configPostProcessors.forEach(component -> component.postProcessServiceConfig(this));
    }
    ...
}
```

<br>

**5.Dubbo服务启动的init源码流程**

```
-> ServiceConfig.export()
-> ServiceConfig.init()
-> getExtensionLoader(ServiceListener.class)
-> AbstractConfig.getExtensionLoader()
-> extensionLoader.getSupportedExtensionInstances()
-> ServiceConfig.initServiceMetadata()

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    public void export() {
        ...
        //ensure start module, compatible with old api usage
        //getScopeModel()返回的是一个ModuleModel
        getScopeModel().getDeployer().start();

        //2.执行服务实例的刷新操作，也就是ProviderConfig->MethodConfig->ArgumentConfig体系
        if (!this.isRefreshed()) {
            this.refresh();
        }

        //3.这里会执行服务实例的初始化的工作，metadata
        //也就是会把metadata元数据给准备好，后续可以准备进行元数据上报
        this.init();
        ...
        
        //4.这里可以作为一个服务实例里的服务发布这个功能场景的入口
        doExport();
    }
    
    public void init() {
        //通过SPI机制获取ServiceListener扩展点的所有实现类实例，然后添加到ServiceConfig的serviceListeners字段里
        if (this.initialized.compareAndSet(false, true)) {
            //load ServiceListeners from extension
            ExtensionLoader<ServiceListener> extensionLoader = this.getExtensionLoader(ServiceListener.class);
            this.serviceListeners.addAll(extensionLoader.getSupportedExtensionInstances());
        }
        
        //初始化ServiceMetadata，也就是服务元数据
        //这需要与前面设置的MetadataCenter元数据中心配合起来来看
        //ServiceMetadata作为服务实例的元数据，会对服务实例做一些描述，比如版本号、实现类等
        initServiceMetadata(provider);
        serviceMetadata.setServiceType(getInterfaceClass());
        serviceMetadata.setTarget(getRef());
        serviceMetadata.generateServiceKey();
    }
    ...
}

public abstract class AbstractConfig implements Serializable {
    ...
    protected <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        if (scopeModel == null) {
            throw new IllegalStateException("scopeModel is not initialized");
        }
        return scopeModel.getExtensionLoader(type);
    }
    ...
}

public class ExtensionLoader<T> {
    ...
    public Set<T> getSupportedExtensionInstances() {
        checkDestroyed();
        List<T> instances = new LinkedList<>();
        Set<String> supportedExtensions = getSupportedExtensions();
        if (CollectionUtils.isNotEmpty(supportedExtensions)) {
            for (String name : supportedExtensions) {
                instances.add(getExtension(name));
            }
        }
        
        // sort the Prioritized instances
        instances.sort(Prioritized.COMPARATOR);
        return new LinkedHashSet<>(instances);
    }
    ...
}
```

<br>

**6.Dubbo服务本地发布的源码细节**

```
-> ServiceConfig.export()
-> ServiceConfig.doExport()
-> ServiceConfig.doExportUrls()
-> repository.registerService()
-> repository.registerProvider()
-> ServiceConfig.doExportUrlsFor1Protocol()
-> ServiceConfig.exportUrl()
-> ServiceConfig.exportLocal()
-> ServiceConfig.doExportUrl()
-> JavassistProxyFactory.getInvoker()
-> protocolSPI.export(invoker)

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    //服务本地发布
    private void exportLocal(URL url) {
        //创建新URL
        URL local = URLBuilder.from(url)
            .setProtocol(LOCAL_PROTOCOL)
            .setHost(LOCALHOST_VALUE)
            .setPort(0)
            .build();
        local = local.setScopeModel(getScopeModel()).setServiceModel(providerModel);
        
        //本地发布
        doExportUrl(local, false);
        
        //exportLocal指的是发布到本地，也就是系统自己内部，即在JVM内部做一次export发布
        //具体就是在JVM内部完成组件之间的一些交互关系和发布
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
    }
    
    private void doExportUrl(URL url, boolean withMetaData) {
        ...
        //下面这一行代码是为服务实现类的对象创建相应的Invoker，第三个参数中，会将服务URL作为export参数添加到RegistryURL中
        //这里的PROXY_FACTORY是ProxyFactory接口的适配器，比如下面会调用JavassistProxyFactory.getInvoker()方法
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
        if (withMetaData) {
            //DelegateProviderMetaDataInvoker是个装饰类，将当前ServiceConfig和Invoker关联起来而已，invoke()方法透传给底层Invoker对象
            invoker = new DelegateProviderMetaDataInvoker(invoker, this);
        }
        
        //调用Protocol实现，进行发布，protocolSPI是Protocol接口的适配器
        //本地发布时，使用的是InjvmProtocol+InjvmExporter
        //进行远程发布时，使用了RegistryProtocol，它会对DubboProtocol进行包装和装饰
        //RegistryProtocol会先执行，先去做服务注册的事情，接着再执行DubboProtocol，启动NettyServer作为网络服务器
        Exporter<?> exporter = protocolSPI.export(invoker);
        exporters.add(exporter);
        ...
    }
    ...
}

public class JavassistProxyFactory extends AbstractProxyFactory {
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException {
        //下面会通过Wrapper创建一个包装类对象
        //该对象是动态构建出来的，它属于Wrapper的一个子类，里面会拼接一个关键的方法invokeMethod()，拼接代码由Javassist动态生成
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        
        //下面会创建一个实现了AbstractProxyInvoker的匿名内部类，其doInvoker()方法会直接委托给Wrapper对象的invokeMethod()方法
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable {
                //当AbstractProxyInvoker.invoke()方法被调用时，便会执行到这里
                //这里会通过类似于jdk反射的技术，去调用本地实现类如DemoServiceImpl.sayHello
                //这个wrapper类是由javassist技术动态生成的，已经对本地实现类进行包装
                //这个动态生成的wrapper类，在这里会通过javassist技术自己特有的方法，在invokerMethod调用时会去执行本地实现类的目标方法
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
    ...
}
```

<br>

**7.Javassist技术动态生成类的源码细节**

```
-> JavassistProxyFactory.getInvoker()
-> Wrapper.getWrapper()
-> Wrapper.makeWrapper()
-> ClassGenerator.newInstance()
-> ClassGenerator.toClass()

public abstract class Wrapper {
    ...
    public static Wrapper getWrapper(Class<?> c) {
        ...
        return WRAPPER_MAP.computeIfAbsent(c, Wrapper::makeWrapper);
    }
    
    private static Wrapper makeWrapper(Class<?> c) {
        ...
        //动态拼接wrapperClass的代码
        ClassGenerator cc = ClassGenerator.newInstance(cl);
        ...
        //最终生成的是一个Wrapper的子类，子类的代码全部是动态代码拼接的
        Class<?> wc = cc.toClass(c);
    }
    ...
}

public final class ClassGenerator {
    ...
    public Class<?> toClass(Class<?> neighborClass, ClassLoader loader, ProtectionDomain pd) {
        if (mCtc != null) {
            mCtc.detach();
        }
        
        //在代理类继承父类的时候，会将该id作为后缀编号，防止代理类重名
        long id = CLASS_NAME_COUNTER.getAndIncrement();
        CtClass ctcs = mSuperClass == null ? null : mPool.get(mSuperClass);
        
        //确定代理类的名称
        if (mClassName == null) {
            mClassName = (mSuperClass == null || javassist.Modifier.isPublic(ctcs.getModifiers()) ? ClassGenerator.class.getName() : mSuperClass + "$sc") + id;
        }
        
        //创建CtClass，用来生成代理类
        mCtc = mPool.makeClass(mClassName);
        
        //设置代理类的父类
        if (mSuperClass != null) {
            mCtc.setSuperclass(ctcs);
        }
        
        //设置代理类实现的接口，默认会添加DC这个接口
        mCtc.addInterface(mPool.get(DC.class.getName())); // add dynamic class tag.
        if (mInterfaces != null) {
            for (String cl : mInterfaces) {
                mCtc.addInterface(mPool.get(cl));
            }
        }
        
        //设置代理类的字段
        if (mFields != null) {
            for (String code : mFields) {
                mCtc.addField(CtField.make(code, mCtc));
            }
        }
        
        //生成代理类的方法
        if (mMethods != null) {
            for (String code : mMethods) {
                if (code.charAt(0) == ':') {
                    mCtc.addMethod(CtNewMethod.copy(getCtMethod(mCopyMethods.get(code.substring(1))), code.substring(1, code.indexOf('(')), mCtc, null));
                } else {
                    mCtc.addMethod(CtNewMethod.make(code, mCtc));
                }
            }
        }
        
        //生成默认的构造方法
        if (mDefaultConstructor) {
            mCtc.addConstructor(CtNewConstructor.defaultConstructor(mCtc));
        }
        
        //生成构造方法
        if (mConstructors != null) {
            for (String code : mConstructors) {
                if (code.charAt(0) == ':') {
                    mCtc.addConstructor(CtNewConstructor.copy(getCtConstructor(mCopyConstructors.get(code.substring(1))), mCtc, null));
                } else {
                    String[] sn = mCtc.getSimpleName().split("\$+"); // inner class name include $.
                    mCtc.addConstructor(CtNewConstructor.make(code.replaceFirst(SIMPLE_NAME_TAG, sn[sn.length - 1]), mCtc));
                }
            }
        }
        
        //按照上述配置生成类，也就是将动态拼接得到的类代码字符串，使用Javassist来生成
        try {
            return mPool.toClass(mCtc, neighborClass, loader, pd);
        } catch (Throwable t) {
            if (!(t instanceof CannotCompileException)) {
                return mPool.toClass(mCtc, loader, pd);
            }
            throw t;
        }
    }
    ...
}
```

<br>

**8.Protocol SPI自适应机制的源码细节**

**首先ServiceConfig的protocolSPI会在postProcessAfterScopeModelChanged()方法中被赋值，其中会涉及到ExtensionLoader.getAdaptiveExtension()方法。**

```
-> this.getExtensionLoader(Protocol.class).getAdaptiveExtension()
-> ExtensionLoader.getAdaptiveExtension()
-> ExtensionLoader.createAdaptiveExtension()
-> ExtensionLoader.getAdaptiveExtensionClass()
-> ExtensionLoader.createAdaptiveExtensionClass()
-> ServiceConfig.doExportUrl()

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    @Override
    protected void postProcessAfterScopeModelChanged(ScopeModel oldScopeModel, ScopeModel newScopeModel) {
        super.postProcessAfterScopeModelChanged(oldScopeModel, newScopeModel);
        protocolSPI = this.getExtensionLoader(Protocol.class).getAdaptiveExtension();
        proxyFactory = this.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
    }
    
    private void doExportUrl(URL url, boolean withMetaData) {
        ...
        //下面这一行代码是为服务实现类的对象创建相应的Invoker，第三个参数中，会将服务URL作为export参数添加到RegistryURL中
        //这里的PROXY_FACTORY是ProxyFactory接口的适配器，比如下面会调用JavassistProxyFactory.getInvoker()方法
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
        if (withMetaData) {
            //DelegateProviderMetaDataInvoker是个装饰类，将当前ServiceConfig和Invoker关联起来而已，invoke()方法透传给底层Invoker对象
            invoker = new DelegateProviderMetaDataInvoker(invoker, this);
        }
        
        //调用Protocol实现，进行发布，protocolSPI是Protocol接口的适配器
        //本地发布时，使用的是InjvmProtocol+InjvmExporter
        //进行远程发布时，使用了RegistryProtocol，它会对DubboProtocol进行包装和装饰
        //RegistryProtocol会先执行，先去做服务注册的事情，接着再执行DubboProtocol，启动NettyServer作为网络服务器
        Exporter<?> exporter = protocolSPI.export(invoker);
        exporters.add(exporter);
        ...
    }
    ...
}

public class ExtensionLoader<T> {
    //缓存的一些Adaptive类型的扩展实现类的实例对象
    //Adaptive是自适应的意思，即后续会根据扩展实现类的@Adaptive注解+URL参数，自动去筛选和判断到底用哪个实现类
    private final Holder<Object> cachedAdaptiveInstance = new Holder<>();
    ...
    
    //对于包含@Adaptive注解的SPI扩展类，如果它有多个实现类，那么就可以根据url里的一些参数直接匹配和定位对应的一个实现类
    //createAdaptiveExtension()会动态生成出来一个类，但这个类不是一个直接给我们用的实现类
    //生成出来的这一个类，核心要做的事情，就是根据url里提取出一些参数，动态去匹配真正的实现类
    @SuppressWarnings("unchecked")
    public T getAdaptiveExtension() {
        checkDestroyed();
        //检查cachedAdaptiveInstance字段中是否已缓存了适配器实例，如果已缓存，则直接返回该实例即可
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
            if (createAdaptiveInstanceError != null) {
                throw new IllegalStateException("Failed to create adaptive instance: " + createAdaptiveInstanceError.toString(), createAdaptiveInstanceError);
            }
            
            //cachedAdaptiveInstance是一个Holder，主要是用来对目标对象进行加锁的
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //动态生成一个类
                        instance = createAdaptiveExtension();
                        
                        //将适配器实例缓存到cachedAdaptiveInstance字段，然后返回适配器实例
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        }
        return (T) instance;
    }
    
    private T createAdaptiveExtension() {
        try {
            //创建自适应扩展类对应的实例
            T instance = (T) getAdaptiveExtensionClass().newInstance();
    
            //下面会对该实例对象进行初始化前的处理
            instance = postProcessBeforeInitialization(instance, null);
            
            //调用injectExtension()方法进行自动装配，就能得到一个完整的适配器实例
            injectExtension(instance);
            
            //下面会对该实例对象进行初始化后的处理
            instance = postProcessAfterInitialization(instance, null);
            
            //如果扩展实现类实现了Lifecycle接口，在initExtension()方法中会调用initialize()方法进行初始化；
            initExtension(instance);
            
            return instance;
        } catch (Exception e) {
            throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
    
    private Class<?> getAdaptiveExtensionClass() {
        //调用getExtensionClasses()方法，其中就会触发loadClass()方法，完成cachedAdaptiveClass字段的填充
        getExtensionClasses();
        
        //如果存在@Adaptive注解修饰的扩展实现类，该类就是适配器类，通过newInstance()将其实例化即可
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
        
        //如果不存在@Adaptive注解修饰的扩展实现类
        //就需要通过createAdaptiveExtensionClass()方法扫描扩展接口中方法上的@Adaptive注解，动态生成适配器类，然后实例化
        //所以光加载ExtensionClasses还不够，还需要创建adaptive自适应的扩展类实例
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
    
    private Class<?> createAdaptiveExtensionClass() {
        // Adaptive Classes' ClassLoader should be the same with Real SPI interface classes' ClassLoader
        ClassLoader classLoader = type.getClassLoader();
        if (NativeUtils.isNative()) {
            return classLoader.loadClass(type.getName() + "$Adaptive");
        }
        
        //下面使用了AdaptiveClassCodeGenerator来做类代码的动态生成并进行获取，也就是说自适应扩展类的代码是动态生成的
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
        
        //自适应扩展类的代码经过动态生成后，还需要进行动态编译，编译成Class字节码对象
        org.apache.dubbo.common.compiler.Compiler compiler = extensionDirector.getExtensionLoader(
            org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        
        return compiler.compile(type, code, classLoader);
    }
    ...
}
```

<br>

**9.Protocol SPI进行本地发布的源码细节**

```
-> ServiceConfig.exportLocal()
-> ServiceConfig.doExportUrl()
-> protocolSPI.export()
-> InjvmProtocol.export()

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    //服务本地发布
    private void exportLocal(URL url) {
        //创建新URL
        URL local = URLBuilder.from(url)
            .setProtocol(LOCAL_PROTOCOL)
            .setHost(LOCALHOST_VALUE)
            .setPort(0)
            .build();
        local = local.setScopeModel(getScopeModel()).setServiceModel(providerModel);
        
        //本地发布
        doExportUrl(local, false);
        
        //exportLocal指的是发布到本地，也就是系统自己内部，即在JVM内部做一次export发布
        //具体就是在JVM内部完成组件之间的一些交互关系和发布
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
    }
   
    private void doExportUrl(URL url, boolean withMetaData) {
        ...
        //下面这一行代码是为服务实现类的对象创建相应的Invoker，第三个参数中，会将服务URL作为export参数添加到RegistryURL中
        //这里的PROXY_FACTORY是ProxyFactory接口的适配器，比如下面会调用JavassistProxyFactory.getInvoker()方法
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
        if (withMetaData) {
            //DelegateProviderMetaDataInvoker是个装饰类，将当前ServiceConfig和Invoker关联起来而已，invoke()方法透传给底层Invoker对象
            invoker = new DelegateProviderMetaDataInvoker(invoker, this);
        }
        
        //调用Protocol实现，进行发布，protocolSPI是Protocol接口的适配器
        //本地发布时，使用的是InjvmProtocol+InjvmExporter
        //进行远程发布时，使用了RegistryProtocol，它会对DubboProtocol进行包装和装饰
        //RegistryProtocol会先执行，先去做服务注册的事情，接着再执行DubboProtocol，启动NettyServer作为网络服务器
        Exporter<?> exporter = protocolSPI.export(invoker);
        exporters.add(exporter);
        ...
    }
    ...
}

public class InjvmProtocol extends AbstractProtocol {
    ...
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        //直接封装一个InjvmInvoker，可以用于进行本地的服务调用
        return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
    }
    ...
}
```

<br>

**10.Dubbo服务远程发布的源码细节**

```
-> ServiceConfig.export()
-> ServiceConfig.doExport()
-> ServiceConfig.doExportUrls()
-> ServiceConfig.doExportUrlsFor1Protocol()
-> ServiceConfig.exportUrl()
-> ServiceConfig.exportLocal()
-> ServiceConfig.exportRemote()
-> ServiceConfig.doExportUrl()
-> JavassistProxyFactory.getInvoker()
-> protocolSPI.export(invoker)
-> RegistryProtocol.export()

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    //需要确保要对外暴露的服务可以对外提供访问
    private URL exportRemote(URL url, List<URL> registryURLs) {
        //如果当前配置了至少一个注册中心
        if (CollectionUtils.isNotEmpty(registryURLs)) {
            //URL里有很多的信息，比如协议、各种各样的参数、各种各样的信息，URL可以在后续代码运行时提供配置和信息
            //接下来会向每个注册中心发布服务
            for (URL registryURL : registryURLs) {
                //registryURL.getProtocol()会获取协议
                if (SERVICE_REGISTRY_PROTOCOL.equals(registryURL.getProtocol())) {
                    url = url.addParameterIfAbsent(SERVICE_NAME_MAPPING_KEY, "true");
                }
                
                //if protocol is only injvm ,not register
                //injvm协议只在exportLocal()中有用，不会将服务发布到注册中心，所以这里忽略injvm协议
                if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                    continue;
                }
                
                //设置服务URL的dynamic参数
                url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                
                //创建monitorUrl，并作为monitor参数添加到服务URL中
                URL monitorUrl = ConfigValidationUtils.loadMonitor(this, registryURL);
                if (monitorUrl != null) {
                    url = url.putAttribute(MONITOR_KEY, monitorUrl);
                }
                
                //For providers, this is used to enable custom proxy to generate invoker
                //设置服务URL的proxy参数，即生成动态代理方式(jdk或是javassist)，作为参数添加到RegistryURL中
                String proxy = url.getParameter(PROXY_KEY);
                if (StringUtils.isNotEmpty(proxy)) {
                    registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                }
                ...
                
                doExportUrl(registryURL.putAttribute(EXPORT_KEY, url), true);
            }
        } else {
            //不存在注册中心，仅发布服务，不会将服务信息发布到注册中心
            //Consumer没法在注册中心找到该服务的信息，但是可以直连，具体的发布过程与上面的过程类似
            doExportUrl(url, true);
        }
        return url;
    }
    
    private void doExportUrl(URL url, boolean withMetaData) {
        ...
        //下面这一行代码是为服务实现类的对象创建相应的Invoker，第三个参数中，会将服务URL作为export参数添加到RegistryURL中
        //这里的PROXY_FACTORY是ProxyFactory接口的适配器，比如下面会调用JavassistProxyFactory.getInvoker()方法
        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, url);
        if (withMetaData) {
            //DelegateProviderMetaDataInvoker是个装饰类，将当前ServiceConfig和Invoker关联起来而已，invoke()方法透传给底层Invoker对象
            invoker = new DelegateProviderMetaDataInvoker(invoker, this);
        }
        
        //调用Protocol实现，进行发布，protocolSPI是Protocol接口的适配器
        //本地发布时，使用的是InjvmProtocol+InjvmExporter
        //进行远程发布时，使用了RegistryProtocol，它会对DubboProtocol进行包装和装饰
        //RegistryProtocol会先执行，先去做服务注册的事情，接着再执行DubboProtocol，启动NettyServer作为网络服务器
        Exporter<?> exporter = protocolSPI.export(invoker);
        exporters.add(exporter);
        ...
    }
    
    public class JavassistProxyFactory extends AbstractProxyFactory {
        ...
        public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException {
            try {
                //下面会通过Wrapper创建一个包装类对象
                //该对象是动态构建出来的，它属于Wrapper的一个子类，里面会拼接一个关键的方法invokeMethod()，拼接代码由Javassist动态生成
                final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
                
                //下面会创建一个实现了AbstractProxyInvoker的匿名内部类，其doInvoker()方法会直接委托给Wrapper对象的invokeMethod()方法
                return new AbstractProxyInvoker<T>(proxy, type, url) {
                    @Override
                    protected Object doInvoke(T proxy, String methodName, Class<?>[] parameterTypes, Object[] arguments) throws Throwable {
                        //当AbstractProxyInvoker.invoke()方法被调用时，便会执行到这里
                        //这里会通过类似于JDK反射的技术，去调用本地实现类如DemoServiceImpl.sayHello
                        //这个wrapper类是由javassist技术动态生成的，已经对本地实现类进行包装
                        //这个动态生成的wrapper类，在这里会通过javassist技术自己特有的方法，在invokerMethod调用时会去执行本地实现类的目标方法
                        return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
                    }
                };
            } catch (Throwable fromJavassist) {
                //try fall back to JDK proxy factory
                try {
                    //使用JDK的反射去调用本地，这时没有动态生成的Wrapper类了
                    Invoker<T> invoker = jdkProxyFactory.getInvoker(proxy, type, url);
                    return invoker;
                } catch (Throwable fromJdk) {
                    throw new RpcException(fromJavassist);
                }
            }
        }
        ...
    }
    ...
}
```

<br>

**11.RegistryProtocol远程发布的源码细节**

**RegistryProtocol远程发布的核心逻辑其实涉及两大Protocol：一是DubboProtocol建立网络服务进行监听，二是RegistryProtocol向注册中心注册。**

```
-> RegistryProtocol.export()
-> RegistryProtocol.doLocalExport()
-> RegistryProtocol.getRegistry()
-> RegistryProtocol.register()

public class RegistryProtocol implements Protocol, ScopeModelAware {
    ...
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        //将"registry://"协议(Remote URL)转换成"zookeeper://"协议(Registry URL)
        URL registryUrl = getRegistryUrl(originInvoker);

        //获取export参数，其中存储了一个"dubbo://"协议的Provider URL
        URL providerUrl = getProviderUrl(originInvoker);

        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        Map<URL, NotifyListener> overrideListeners = getProviderConfigurationListener(providerUrl).getOverrideListeners();
        overrideListeners.put(registryUrl, overrideSubscribeListener);
        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);

        //export invoker
        //1.下面进行导出服务，底层会通过会执行DubboProtocol.export()方法，启动对应的Server
        //也就是会涉及到对另外一个protocol组件的调用，远程发布服务时其实就是执行DubboProtocol的export方法
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);
        
        //2.url to registry 完成服务注册的事情
        //下面会根据RegistryURL获取对应的注册中心Registry对象，其中会依赖RegistryFactory
        //远程发布时，下面的registry其实是一个ListenerRegistryWrapper装饰器，装饰着使用了ZookeeperServiceDiscovery的ServiceDiscoveryRegistry
        //在基于注册中心的url地址去构建对应的注册中心组件时，默认是基于zk的
        //而构建一个基于zk的注册中心组件，同时跟zk完成连接的建立，则由curator5框架来实现
        final Registry registry = getRegistry(registryUrl);

        //获取将要发布到注册中心上的Provider URL，其中会删除一些多余的参数信息
        final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);
        
        //根据register参数值决定是否注册服务
        boolean register = providerUrl.getParameter(REGISTER_KEY, true) && registryUrl.getParameter(REGISTER_KEY, true);
        if (register) {
            //调用Registry.register()方法将registeredProviderUrl发布到注册中心
            register(registry, registeredProviderUrl);
        }
        
        //将Provider相关信息记录到的ProviderModel中
        registerStatedUrl(registryUrl, registeredProviderUrl, register);
        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);
        if (!registry.isServiceDiscovery()) {
            //向注册中心进行订阅override数据，主要是监听该服务的configurators节点
            registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        }
        
        //触发RegistryProtocolListener监听器
        notifyExport(exporter);
        return new DestroyableExporter<>(exporter);
    }
    
    private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        String key = getCacheKey(originInvoker);
        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
            Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
            //下面会调用DubboProtocol的export()方法
            return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
        });
    }
    
    protected Registry getRegistry(final URL registryUrl) {
        //通过SPI自适应机制，去拿到对应的extension实例
        RegistryFactory registryFactory = ScopeModelUtil.getExtensionLoader(RegistryFactory.class, registryUrl.getScopeModel()).getAdaptiveExtension();
        return registryFactory.getRegistry(registryUrl);
    }
    
    private void register(Registry registry, URL registeredProviderUrl) {
        //下面会调用ListenerRegistryWrapper.register()方法
        registry.register(registeredProviderUrl);
    }
    ...
}
```

<br>

**12.基于ZooKeeper的注册中心的实现细节**

**(1)Provider端使用ServiceDiscoveryFactory建立与zk的连接**

**(2)Consumer端使用ZookeeperRegistry建立与zk的连接**

<br>

**在RegistryProtocol的getRegistry()过程中，如果发现还没有注册中心实例，就会去进行创建。不过从Debug可以知道不是通过ZookeeperRegistry去构建，而是通过ServiceDiscoveryRegistry去构建，然后通过ZookeeperServiceDiscoveryFactory的createDiscovery()方法去建立与zk的连接。**

<br>

**(1)Provider端使用ServiceDiscoveryFactory建立与zk的连接**

```
-> RegistryProtocol.getRegistry()
-> registryFactory.getRegistry(registryUrl)
-> RegistryFactory$Adaptive.getRegistry()
-> RegistryFactoryWrapper.getRegistry()
-> AbstractRegistryFactory.getRegistry()
-> ServiceDiscoveryRegistryFactory.createRegistry()
-> new ServiceDiscoveryRegistry()
-> ServiceDiscoveryRegistry.createServiceDiscovery()
-> ServiceDiscoveryRegistry.getServiceDiscovery()
-> AbstractServiceDiscoveryFactory.getServiceDiscovery()
-> AbstractServiceDiscoveryFactory.createDiscovery()
-> ZookeeperServiceDiscoveryFactory.createDiscovery()
-> new ZookeeperServiceDiscovery()

public class RegistryProtocol implements Protocol, ScopeModelAware {
    ...
    protected Registry getRegistry(final URL registryUrl) {
        //通过SPI机制，去自适应拿到对应的extension实例
        RegistryFactory registryFactory = ScopeModelUtil.getExtensionLoader(RegistryFactory.class, registryUrl.getScopeModel()).getAdaptiveExtension();
        return registryFactory.getRegistry(registryUrl);
    }
    ...
}

public class RegistryFactoryWrapper implements RegistryFactory {
    ...
    public Registry getRegistry(URL url) {
        //下面会调用AbstractRegistryFactory.getRegistry()
        return new ListenerRegistryWrapper(registryFactory.getRegistry(url),
            Collections.unmodifiableList(url.getOrDefaultApplicationModel().getExtensionLoader(RegistryServiceListener.class).getActivateExtension(url, "registry.listeners")));
    }
    ...
}

public abstract class AbstractRegistryFactory implements RegistryFactory, ScopeModelAware {
    ...
    public Registry getRegistry(URL url) {
        ...
        registry = createRegistry(url);
        ...
        return registry;
    }
    
    protected abstract Registry createRegistry(URL url);
    ...
}

public class ServiceDiscoveryRegistryFactory extends AbstractRegistryFactory {
    @Override
    protected Registry createRegistry(URL url) {
        if (UrlUtils.hasServiceDiscoveryRegistryProtocol(url)) {
            String protocol = url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY);
            url = url.setProtocol(protocol).removeParameter(REGISTRY_KEY);
        }
        return new ServiceDiscoveryRegistry(url, applicationModel);
    }
}

public class ServiceDiscoveryRegistry extends FailbackRegistry {
    ...
    public ServiceDiscoveryRegistry(URL registryURL, ApplicationModel applicationModel) {
        super(registryURL);
        this.serviceDiscovery = createServiceDiscovery(registryURL);
        this.serviceNameMapping = (AbstractServiceNameMapping) ServiceNameMapping.getDefaultExtension(registryURL.getScopeModel());
        super.applicationModel = applicationModel;
    }
    
    protected ServiceDiscovery createServiceDiscovery(URL registryURL) {
        //根据registryURL获取对应的ServiceDiscovery实现
        return getServiceDiscovery(registryURL.addParameter(INTERFACE_KEY, ServiceDiscovery.class.getName())
            .removeParameter(REGISTRY_TYPE_KEY));
    }
    
    private ServiceDiscovery getServiceDiscovery(URL registryURL) {
        ServiceDiscoveryFactory factory = getExtension(registryURL);
        return factory.getServiceDiscovery(registryURL);
    }
    ...
}

public abstract class AbstractServiceDiscoveryFactory implements ServiceDiscoveryFactory, ScopeModelAware {
    ...
    public ServiceDiscovery getServiceDiscovery(URL registryURL) {
        String key = registryURL.toServiceStringWithoutResolving();
        return discoveries.computeIfAbsent(key, k -> createDiscovery(registryURL));
    }
    
    protected abstract ServiceDiscovery createDiscovery(URL registryURL);
}

public class ZookeeperServiceDiscoveryFactory extends AbstractServiceDiscoveryFactory {
    @Override
    protected ServiceDiscovery createDiscovery(URL registryURL) {
        return new ZookeeperServiceDiscovery(applicationModel, registryURL);
    }
}

public class ZookeeperServiceDiscovery extends AbstractServiceDiscovery {
    ...
    public ZookeeperServiceDiscovery(ApplicationModel applicationModel, URL registryURL) {
        super(applicationModel, registryURL);
        try {
            //先去构建和zk连接的一个客户端
            this.curatorFramework = buildCuratorFramework(registryURL, this);
            this.rootPath = getRootPath(registryURL);
            this.serviceDiscovery = buildServiceDiscovery(curatorFramework, rootPath);
            this.serviceDiscovery.start();
        } catch (Exception e) {
            throw new IllegalStateException("Create zookeeper service discovery failed.", e);
        }
    }
    ...
}
```

<br>

**(2)Consumer端使用ZookeeperRegistry建立与zk的连接**

**刚开始构建ZookeeperRegistry，其核心就是去连接zk，与zk建立连接。**

```
-> ZookeeperRegistryFactory.createRegistry()
-> new ZookeeperRegistry()
-> zookeeperTransporter.connect(url)
-> AbstractZookeeperTransporter.connect()
-> createZookeeperClient(url)
-> Curator5ZookeeperTransporter.createZookeeperClient()
-> new Curator5ZookeeperClient()

public class RegistryProtocol implements Protocol, ScopeModelAware {
    ...
    protected Registry getRegistry(final URL registryUrl) {
        //通过SPI机制，去自适应拿到对应的extension实例
        RegistryFactory registryFactory = ScopeModelUtil.getExtensionLoader(RegistryFactory.class, registryUrl.getScopeModel()).getAdaptiveExtension();
        return registryFactory.getRegistry(registryUrl);
    }
    ...
}

public abstract class AbstractRegistryFactory implements RegistryFactory, ScopeModelAware {
    ...
    public Registry getRegistry(URL url) {
        ...
        registry = createRegistry(url);
        ...
        return registry;
    }
    
    protected abstract Registry createRegistry(URL url);
    ...
}
    
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
    //跟zk之间的网络通信的组件
    private ZookeeperTransporter zookeeperTransporter;
    
    public ZookeeperRegistryFactory(ApplicationModel applicationModel) {
        this.applicationModel = applicationModel;
        //通过SPI机制来进行获取
        this.zookeeperTransporter = ZookeeperTransporter.getExtension(applicationModel);
    }
    
    public Registry createRegistry(URL url) {
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }
}

// ZookeeperRegistry
// 基于zookeeper的注册中心组件

// dubbo注册中心的三级类体系结构设计：ZK/Nacos/Redis具体注册中心技术 -> Cacheable缓存层 -> Failback故障重试层
// 通过父类体系的设计，dubbo会把cache和failback两套复杂机制，提取到父类里去，而不是一个父类，是两个父类
// 这样就可以把不同的机制模块进行拆分，cache和failback也就各自会有一套机制模块
// ZooKeeperRegistry、NacosRegistry、KubernetesRegistry、DnsRegistry、ConsulRegistry这些基于具体技术实现的注册中心，
// 它们都可以继承相同的一套cache缓存和failback故障重试的机制，只要继承父类就可以了

// 注册中心的服务发现和订阅中类体系结构一共是5层：tech层、cacheable层、failback层、abstract层、interface层
public class ZookeeperRegistry extends CacheableFailbackRegistry {
    ...
    //刚开始构建ZookeeperRegistry，其核心就是去连接zk，与zk建立连接
    public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        //传入的url是比如就是zk的连接地址zookeeper://localhost:2181/，首先根据该url执行父类的构造函数
        super(url);
        String group = url.getGroup(DEFAULT_ROOT);
        if (!group.startsWith(PATH_SEPARATOR)) {
            group = PATH_SEPARATOR + group;
        }
        this.root = group;
        
        //基于zk的API去构建与zk之间的连接
        this.zkClient = zookeeperTransporter.connect(url);
        this.zkClient.addStateListener((state) -> {
            if (state == StateListener.RECONNECTED) {
                //连接突然断开后，很快就进行了重新连接的操作，reconnected
                logger.warn("...");
                
                //一旦重新连接后，就会尝试重新拉取之前订阅过的provider服务实例的最新集群地址
                //在本机和zk之间的短暂的连接断开时，恰好zk端的provider服务实例地址有了变化，但当时网络连接短暂断开，导致没有办法及时反向推送回本机
                //所以本机作为客户端，如果有短暂断开再重连的情况，必须去重新拉取一下最新的地址
                //在这种情况下，是否有必要去执行本机consumer节点的重新注册呢？provider节点是否有必要在这里进行重新注册呢？其实是没有必要的
                //因为如果之前去进行注册时，注册过程创建的就是一个zk临时节点，所以如果仅仅是网络闪断，zk是不会删除创建的临时节点的
                //而之前发起的subscribe订阅请求，也因为仅仅是临时的网络断开，所以zk也是不会删除该订阅和监听的
                ZookeeperRegistry.this.fetchLatestAddresses();
            } else if (state == StateListener.NEW_SESSION_CREATED) {
                //new_session_created新会话创建，经历了session断开过程，需要重新进行注册和订阅的监听
                logger.warn("...");
                ZookeeperRegistry.this.recover();
            } else if (state == StateListener.SESSION_LOST) {
                //连接断开的时间超过了一定的阈值，超出了sessionExpire过期的时间
                //会话一旦过期，zk服务端就会把客户端之前注册时创建的临时节点删除，同时其施加的订阅监听也会被删除
                //此时客户端就会收到一个状态变更，session_lost，表示断开时间太长了
                logger.warn("...");
            } else if (state == StateListener.SUSPENDED) {
            } else if (state == StateListener.CONNECTED) {
            }
        });
    }
    ...
}

public abstract class AbstractZookeeperTransporter implements ZookeeperTransporter {
    ...
    public ZookeeperClient connect(URL url) {
        ...
        //构建ZookeeperClient
        zookeeperClient = createZookeeperClient(url);
        ...
        return zookeeperClient;
    }
    
    protected abstract ZookeeperClient createZookeeperClient(URL url);
    ...
}

public class Curator5ZookeeperTransporter extends AbstractZookeeperTransporter {
    @Override
    public ZookeeperClient createZookeeperClient(URL url) {
        //创建CuratorZookeeperClient实例
        return new Curator5ZookeeperClient(url);
    }
}

public class Curator5ZookeeperClient extends AbstractZookeeperClient<Curator5ZookeeperClient.NodeCacheListenerImpl, Curator5ZookeeperClient.CuratorWatcherImpl> {
    ...
    public Curator5ZookeeperClient(URL url) {
        super(url);
        try {
            //连接超时时间
            int timeout = url.getParameter(TIMEOUT_KEY, DEFAULT_CONNECTION_TIMEOUT_MS);
            
            //session过期时间
            int sessionExpireMs = url.getParameter(SESSION_KEY, DEFAULT_SESSION_TIMEOUT_MS);
            
            //基于curator框架去构建zk client，curator框架是对zk原生client做了一层包装，正如Redisson和Jedis封装了redis一样，提供了大量高阶功能
            CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
                //zk地址(包括备用地址)
                .connectString(url.getBackupAddress())
                //重试参数
                .retryPolicy(new RetryNTimes(1, 1000))
                //超时时间
                .connectionTimeoutMs(timeout)
                //session过期时间
                .sessionTimeoutMs(sessionExpireMs);
            
            String userInformation = url.getUserInformation();
            if (userInformation != null && userInformation.length() > 0) {
                builder = builder.authorization("digest", userInformation.getBytes());
                builder.aclProvider(new ACLProvider() {
                    @Override
                    public List<ACL> getDefaultAcl() {
                        return ZooDefs.Ids.CREATOR_ALL_ACL;
                    }
                    
                    @Override
                    public List<ACL> getAclForPath(String path) {
                        return ZooDefs.Ids.CREATOR_ALL_ACL;
                    }
                });
            }
            client = builder.build();
            
            //添加连接状态的监听
            client.getConnectionStateListenable().addListener(new CuratorConnectionStateListener(url));
            client.start();
            
            //阻塞等待直到连接Zookeeper集群成功，超时时间为5秒，5秒过后还没连接上则抛异常
            boolean connected = client.blockUntilConnected(timeout, TimeUnit.MILLISECONDS);
        } catch (Exception e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
    }
    ...
}
```

<br>

**13.DubboProtocol发布过程的源码细节**

```
-> RegistryProtocol.export()
-> RegistryProtocol.doLocalExport()
-> Protocol$Adaptive.export()
-> ProtocolSerializationWrapper.export()
-> ProtocolFilterWrapper.export()
-> ProtocolListenerWrapper.export()
-> DubboProtocol.export()
-> DubboProtocol.openServer()
-> DubboProtocol.createServer()
-> Exchangers.bind()
-> Exchangers.getExchanger()
-> HeaderExchanger.bind()
-> DubboProtocol.optimizeSerialization()

public class RegistryProtocol implements Protocol, ScopeModelAware {
    ...
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        ...
        //1.下面进行导出服务，底层会通过会执行DubboProtocol.export()方法，启动对应的Server
        //也就是会涉及到对另外一个protocol组件的调用，远程发布服务时其实就是执行DubboProtocol的export方法
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);
        ...
    }
    
    private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        String key = getCacheKey(originInvoker);
        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
            Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
            //下面会调用DubboProtocol的export()方法
            return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
    	});
    }
    ...
}

@Activate
public class ProtocolSerializationWrapper implements Protocol {
    private Protocol protocol;
    
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        getFrameworkModel(invoker.getUrl().getScopeModel()).getServiceRepository().registerProviderUrl(invoker.getUrl());
        //下面会调用ProtocolFilterWrapper.export()方法
        return protocol.export(invoker);
    }
    ...
}

@Activate(order = 100)
public class ProtocolFilterWrapper implements Protocol {
    private final Protocol protocol;
    
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (UrlUtils.isRegistry(invoker.getUrl())) {
            return protocol.export(invoker);
        }
        FilterChainBuilder builder = getFilterChainBuilder(invoker.getUrl());
        //下面会调用ProtocolListenerWrapper.export()方法
        return protocol.export(builder.buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER));
    }
    ...
}

@Activate(order = 200)
public class ProtocolListenerWrapper implements Protocol {
    private final Protocol protocol;
    
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        if (UrlUtils.isRegistry(invoker.getUrl())) {
            return protocol.export(invoker);
        }
        //下面会调用DubboProtocol.export()
        return new ListenerExporterWrapper<T>(protocol.export(invoker),
            Collections.unmodifiableList(ScopeModelUtil.getExtensionLoader(ExporterListener.class, invoker.getUrl().getScopeModel()).getActivateExtension(invoker.getUrl(), EXPORTER_LISTENER_KEY)));
    }
    ...
}

// dubbo protocol support.
// 负责把Dubbo服务对外进行网络发布
public class DubboProtocol extends AbstractProtocol {
    ...
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        checkDestroyed();
        URL url = invoker.getUrl();

        //创建ServiceKey
        String key = serviceKey(url);
        
        //exporter组件代表了指定的invoker被发布出去了
        //下面代码会将上层传入的Invoker对象封装成DubboExporter对象，然后记录到exporterMap集合中
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        ...
        
        //启动ProtocolServer，这个就是打开对外的网络服务器，可以对外提供网络请求处理
        openServer(url);
        
        //序列化的优化处理
        optimizeSerialization(url);
        
        return exporter;
    }
    
    private void openServer(URL url) {
        checkDestroyed();
        
        //find server.
        //获取host:port这个地址
        String key = url.getAddress();
        
        //client can export a service which only for server to invoke
        boolean isServer = url.getParameter(IS_SERVER_KEY, true);
        if (isServer) {//只有Server端才能启动Server对象
            //serverMap是网络服务器缓存，其中key就是host:port，value就是ProtocolServer，serverMap位于AbstractProtocol中
            ProtocolServer server = serverMap.get(key);
            if (server == null) {//无ProtocolServer监听该地址
                synchronized (this) {//DoubleCheck，防止并发问题
                    server = serverMap.get(key);
                    if (server == null) {
                        //调用createServer()方法创建ProtocolServer对象
                        serverMap.put(key, createServer(url));
                        return;
                    }
                }
            }
            
            //server supports reset, use together with override
            server.reset(url);
        }
    }
    
    private ProtocolServer createServer(URL url) {
        url = URLBuilder.from(url)
            //send readonly event when server closes, it's enabled by default
            //ReadOnly请求是否阻塞等待
            .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
            //enable heartbeat by default
            //心跳间隔
            .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
            .addParameter(CODEC_KEY, DubboCodec.NAME)
            .build();
        
        //SERVER_KEY参数检查
        String transporter = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);
        if (StringUtils.isNotEmpty(transporter) && !url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).hasExtension(transporter)) {
            throw new RpcException("Unsupported server type: " + transporter + ", url: " + url);
        }
        
        ExchangeServer server;
        try {
            //通过Exchangers门面类，创建ExchangeServer对象
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }
        
        //检测CLIENT_KEY参数指定的Transporter扩展实现是否合法
        transporter = url.getParameter(CLIENT_KEY);
        if (StringUtils.isNotEmpty(transporter) && !url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).hasExtension(transporter)) {
            throw new RpcException("Unsupported client type: " + transporter);
        }
        
        //将ExchangeServer封装成DubboProtocolServer返回
        DubboProtocolServer protocolServer = new DubboProtocolServer(server);
        loadServerProperties(protocolServer);
        
        return protocolServer;
    }
    ...
}

public class Exchangers {
    ...
    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        ...
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        //先获取到一个Exchanger组件，再用这个Exchanger组件去进行bind，拿到对应的ExchangeServer
        //getExchanger()会通过SPI机制，获取到HeaderExchanger
        return getExchanger(url).bind(url, handler);
    }
    
    public static Exchanger getExchanger(URL url) {
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        //根据SPI机制，通过model组件体系去拿到对应的SPI扩展实现类实例
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Exchanger.class).getExtension(type);
    }
    ...
}

public class HeaderExchanger implements Exchanger {
    ...
    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        //Exchanger这一层的代码可以理解为是位于上层的代码，它会把一些RpcInvocation调用转为请求/响应的模型，以及进行同步转异步的处理；
        //从Exchanger这层开始，便进入网络模型的范围，引入了请求的概念，并最终会通过底层的网络框架把请求发送出去；
        //因此需要获取到网络框架底层的Server和Client，并将它们封装到Exchanger组件如HeaderExchangeServer/HeaderExchangeClient中；
        //为什么需要不同的Transporter？
        //在Exchanger这一层里其实是可以使用不同的网络技术的，比如Netty、Mina这些网络通信框架；
        //由于Netty、Mina这些不同的框架，它们的用法和API都是不同的，所以在Exchanger这一层，不能把Netty、Mina的API直接提供过来；
        //为了把这些不同的网络框架技术进行统一的封装，需要做一层Transporter，由Transporter来实现抽象统一的底层网络框架的使用标准；
        //所以Exchanger这一层是基于Transporter这一层提供的标准模型来实现请求/响应处理；
        //下面的Transporters.bind()会返回一个NettyServer
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/f69d02e5-3a55-4a0e-906b-2477974a62da" />

<br>

**14.结合Exchange实例梳理SPI自适应机制原理**

```
public class Exchangers {
    ...
    public static Exchanger getExchanger(URL url) {
        String type = url.getParameter(Constants.EXCHANGER_KEY, Constants.DEFAULT_EXCHANGER);
        //根据SPI机制，通过model组件体系去拿到对应的SPI扩展实现类实例
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Exchanger.class).getExtension(type);
    }
    ...
}

@SPI(value = HeaderExchanger.NAME, scope = ExtensionScope.FRAMEWORK)
public interface Exchanger {
    //为什么@Adaptive注解是加在方法上面
    //根据自适应获取SPI extension实例的方法getExtension()代码可知，如果Exchanger接口里的方法加了@Adaptive注解
    //那么针对Exchanger创建代理类时，它里面的方法代码都是动态拼接出来的，都是根据url里具体的参数去获取到指定的一个值，
    //然后根据这个值去动态获取要用的实现类，找到对应的一个实现类后，再通过调用真正的实现类的extension实例来执行对应的bind方法
    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException;
    
    @Adaptive({Constants.EXCHANGER_KEY})
    ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException;
}
```

**不过最直观的例子应该是：Transporters.getTransporter()方法中获取自适应Transporter类：**

```
public class Transporters {
    ...
    public static RemotingServer bind(URL url, ChannelHandler... handlers) throws RemotingException {
        ...
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        return getTransporter(url).bind(url, handler);
    }
    
    public static Transporter getTransporter(URL url) {
        //下面使用了getAdaptiveExtension()自适应机制，针对接口动态生成代码然后创建代理类，
        //代理类的方法，会根据url的参数动态提取对应的实现类的name名称，以及获取真正的需要使用的实现类
        //有了真正的实现类后，就可以去调用实现类的extension实例的方法了
        //比如下面会获取到一个NettyTransporter实例
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
    ...
}

@SPI(value = "netty", scope = ExtensionScope.FRAMEWORK)
public interface Transporter {
    //Bind a server.
    @Adaptive({Constants.SERVER_KEY, Constants.TRANSPORTER_KEY})
    RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException;
    
    //Connect to a server.
    @Adaptive({Constants.CLIENT_KEY, Constants.TRANSPORTER_KEY})
    Client connect(URL url, ChannelHandler handler) throws RemotingException;
}
```

**在运行过程中，getExtensionLoader(Transporter.class).getAdaptiveExtension()动态拼接生成的代理类代码如下：可以看到对比Transporter接口的其中一个实现类NettyTransporter，它们的方法区别在于动态拼接生成的多了URL判断。**

```
package org.apache.dubbo.remoting;
import org.apache.dubbo.rpc.model.ScopeModel;
import org.apache.dubbo.rpc.model.ScopeModelUtil;
public class Transporter$Adaptive implements org.apache.dubbo.remoting.Transporter {
    public org.apache.dubbo.remoting.Client connect(org.apache.dubbo.common.URL arg0, org.apache.dubbo.remoting.ChannelHandler arg1) throws org.apache.dubbo.remoting.RemotingException {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("client", url.getParameter("transporter", "netty"));
        if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.remoting.Transporter) name from url (" + url.toString() + ") use keys([client, transporter])");
        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), org.apache.dubbo.remoting.Transporter.class);
        org.apache.dubbo.remoting.Transporter extension = (org.apache.dubbo.remoting.Transporter)scopeModel.getExtensionLoader(org.apache.dubbo.remoting.Transporter.class).getExtension(extName);
        return extension.connect(arg0, arg1);
    }
    
    public org.apache.dubbo.remoting.RemotingServer bind(org.apache.dubbo.common.URL arg0, org.apache.dubbo.remoting.ChannelHandler arg1) throws org.apache.dubbo.remoting.RemotingException {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("server", url.getParameter("transporter", "netty"));
        if(extName == null) throw new IllegalStateException("Failed to get extension (org.apache.dubbo.remoting.Transporter) name from url (" + url.toString() + ") use keys([server, transporter])");
        ScopeModel scopeModel = ScopeModelUtil.getOrDefault(url.getScopeModel(), org.apache.dubbo.remoting.Transporter.class);
        org.apache.dubbo.remoting.Transporter extension = (org.apache.dubbo.remoting.Transporter)scopeModel.getExtensionLoader(org.apache.dubbo.remoting.Transporter.class).getExtension(extName);
        return extension.bind(arg0, arg1);
    }
}

.....................................................................

public class NettyTransporter implements Transporter {
    public static final String NAME = "netty";
    
    @Override
    public RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException {
        return new NettyServer(url, handler);
    }
    
    @Override
    public Client connect(URL url, ChannelHandler handler) throws RemotingException {
        return new NettyClient(url, handler);
    }
}
```

<br>

**15.Exchange网络组件的源码细节**

```
-> HeaderExchanger.bind()
-> new HeaderExchangeHandler() 网络请求处理器构建
-> new DecodeHandler()
-> Transporters.bind()
-> new HeaderExchangeServer()

public class DubboProtocol extends AbstractProtocol {
    ...
    private final ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
        ...
    }

    private ProtocolServer createServer(URL url) {
        ...
        //通过Exchangers门面类，将requestHandler处理器传入Exchangers.bind()方法中，创建ExchangeServer对象
        server = Exchangers.bind(url, requestHandler);
        ...
    }
    ...
}

public class Exchangers {
    ...
    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        ...
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        //先获取到一个Exchanger组件，再用这个Exchanger组件去进行bind，拿到对应的ExchangeServer
        //getExchanger()会通过SPI机制，获取到HeaderExchanger
        //然后将DubboProtocol的requestHandler传入bind()方法中
        return getExchanger(url).bind(url, handler);
    }
    ...
}

public class HeaderExchanger implements Exchanger {
    ...
    @Override
    public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        //Exchanger这一层的代码可以理解为是位于上层的代码，它会把一些RpcInvocation调用转为请求/响应的模型，以及进行同步转异步的处理；
    	//从Exchanger这层开始，便进入网络模型的范围，引入了请求的概念，并最终会通过底层的网络框架把请求发送出去；
    	//因此需要获取到网络框架底层的Server和Client，并将它们封装到Exchanger组件如HeaderExchangeServer/HeaderExchangeClient中；
    	//为什么需要不同的Transporter？
    	//在Exchanger这一层里其实是可以使用不同的网络技术的，比如Netty、Mina这些网络通信框架；
    	//由于Netty、Mina这些不同的框架，它们的用法和API都是不同的，所以在Exchanger这一层，不能把Netty、Mina的API直接提供过来；
    	//为了把这些不同的网络框架技术进行统一的封装，需要做一层Transporter，由Transporter来实现抽象统一的底层网络框架的使用标准；
    	//所以Exchanger这一层是基于Transporter这一层提供的标准模型来实现请求/响应处理；
    	//下面的Transporters.bind()会返回一个NettyServer
    	return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
    ...
}

public class Transporters {
    ...
    public static RemotingServer bind(URL url, ChannelHandler... handlers) throws RemotingException {
        ...
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        return getTransporter(url).bind(url, handler);
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/8a0e2112-e211-4670-8668-132ee58becd3" />

<br>

**16.Transporter网络通信组件的源码细节**

```
-> Transporters.bind()
-> NettyTransporter.bind()
-> new NettyServer()

public class Transporters {
    ...
    public static RemotingServer bind(URL url, ChannelHandler... handlers) throws RemotingException {
        ...
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
            handler = new ChannelHandlerDispatcher(handlers);
        }
        return getTransporter(url).bind(url, handler);
    }
    
    public static Transporter getTransporter(URL url) {
        //下面使用了getAdaptiveExtension()自适应机制，针对接口动态生成代码然后创建代理类，
        //代理类的方法，会根据url的参数动态提取对应的实现类的name名称，以及获取真正的需要使用的实现类
        //有了真正的实现类后，就可以去调用实现类的extension实例的方法了
        //比如下面会获取到一个NettyTransporter实例
        return url.getOrDefaultFrameworkModel().getExtensionLoader(Transporter.class).getAdaptiveExtension();
    }
    ...
}

// 不同的框架可以有不同的Transporter
// 每个框架对应的Transporter可以创建自己的server和client
public class NettyTransporter implements Transporter {
    ...
    @Override
    public RemotingServer bind(URL url, ChannelHandler handler) throws RemotingException {
        return new NettyServer(url, handler);
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/31811bbf-1f00-4262-b63b-6ebdcfa9cd3f" />

<br>

**17.NettyServer构建和打开的全流程**

**NettyServer构造函数中的入参handler其实就是DubboProtocol中的requestHandler。bossGroup线程数是1，workerGroup线程数是CPU核数+1，但最多不会超过32个线程。**

```
-> Transporters.bind()
-> NettyTransporter.bind()
-> new NettyServer()
-> new AbstractServer()
-> NettyServer.doOpen()
-> DefaultExecutorRepository.createExecutorIfAbsent()
-> DefaultExecutorRepository.createExecutor()

public class NettyServer extends AbstractServer {
    ...
    //NettyServer在构建的过程中，会构建和打开真正的网络服务器
    //这里是基于netty4技术去实现了网络服务器构建和打开的，一旦打开后，Netty Server就开始监听指定的端口号
    //当发现有请求过来就可以去进行处理，也就是通过ProxyInvoker去调用本地实现类的目标方法
    //入参handler其实就是DubboProtocol中的requestHandler
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        super(ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME), ChannelHandlers.wrap(handler, url));
        serverShutdownTimeoutMills = ConfigurationUtils.getServerShutdownTimeout(getUrl().getOrDefaultModuleModel());
    }
    
    protected void doOpen() throws Throwable {
        //对于dubbo这种工业级的中间件而言，其关于netty的用法，可以称为最佳教科书
        bootstrap = new ServerBootstrap();//创建ServerBootstrap
    	
        //EventLoop，也可以理解为网络服务器，它会监听一个本地的端口号
        //外部系统针对本地服务器端口号发起连接、通信、网络事件时，监听的端口号就会不停的产生网络事件
        //EventLoop网络服务器，还会不停轮询监听到的网络事件
        //boss的意思是负责监听端口号是否有外部系统的连接请求，它是一个EventLoopGroup线程池
        bossGroup = createBossGroup();//创建boss EventLoopGroup，线程数是1
    	
        //如果发现了网络事件，就需要进行请求处理，可以通过workerGroup里的多个线程进行并发处理
        workerGroup = createWorkerGroup();//创建worker EventLoopGroup，线程数是CPU核数+1，但最多不会超过32个线程
    	
        //创建NettyServerHandler，它是一个Netty中的ChannelHandler实现，不是Dubbo Remoting层的ChannelHandler接口的实现
        final NettyServerHandler nettyServerHandler = createNettyServerHandler();
    	
        //获取当前NettyServer创建的所有Channel，这里的channels集合中的Channel不是Netty中的Channel对象，而是Dubbo Remoting层的Channel对象
        channels = nettyServerHandler.getChannels();
    	
        //初始化ServerBootstrap，指定boss和worker EventLoopGroup
        initServerBootstrap(nettyServerHandler);
    	
        //绑定指定的地址和端口
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
    	
        //等待bind操作完成
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();
    }
    ...
}

public abstract class AbstractServer extends AbstractEndpoint implements RemotingServer {
    private Set<ExecutorService> executors = new ConcurrentHashSet<>();//业务线程池
    ...
    
    public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
        //调用父类的构造方法
        super(url, handler);
        
        //通过使用SPI机制，从applicationModel组件中根据扩展接口ExecutorRepository去获取ExtensionLoader，然后拿到其默认实现类
        executorRepository = url.getOrDefaultApplicationModel().getExtensionLoader(ExecutorRepository.class).getDefaultExtension();
        
        //根据传入的URL初始化localAddress和bindAddress
        localAddress = getUrl().toInetSocketAddress();
        String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
        int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
        if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
            bindIp = ANYHOST_VALUE;
        }
        bindAddress = new InetSocketAddress(bindIp, bindPort);
        
        //初始化accepts等字段
        this.accepts = url.getParameter(ACCEPTS_KEY, DEFAULT_ACCEPTS);
        
        //调用doOpen()这个抽象方法，启动该Server
        doOpen();
        
        //获取该Server关联的线程池，通过DefaultExecutorRepository创建一个FixedThreadPool线程池出来
        executors.add(executorRepository.createExecutorIfAbsent(url));
    }
    
    protected abstract void doOpen() throws Throwable;
    ...
}

public class DefaultExecutorRepository implements ExecutorRepository, ExtensionAccessorAware {
    ...
    public synchronized ExecutorService createExecutorIfAbsent(URL url) {
        Map<Integer, ExecutorService> executors = data.computeIfAbsent(getExecutorKey(url), k -> new ConcurrentHashMap<>());
        //Consumer's executor is sharing globally, key=Integer.MAX_VALUE. Provider's executor is sharing by protocol.
        //根据URL中的side参数值决定第一层key，根据URL中的port值确定第二层key
        Integer portKey = CONSUMER_SIDE.equalsIgnoreCase(url.getParameter(SIDE_KEY)) ? Integer.MAX_VALUE : url.getPort();
        if (url.getParameter(THREAD_NAME_KEY) == null) {
            url = url.putAttribute(THREAD_NAME_KEY, "Dubbo-protocol-" + portKey);
        }
        URL finalUrl = url;
        ExecutorService executor = executors.computeIfAbsent(portKey, k -> createExecutor(finalUrl));
        
        //If executor has been shut down, create a new one
        //如果缓存中相应的线程池已关闭，则同样需要调用createExecutor()方法
        //创建新的线程池，并替换掉缓存中已关闭的线程持
        if (executor.isShutdown() || executor.isTerminated()) {
            executors.remove(portKey);
            executor = createExecutor(url);
            executors.put(portKey, executor);
        }
        return executor;
    }
    
    private ExecutorService createExecutor(URL url) {
        //通过SPI机制去创建线程池，默认的线程池策略ThreadPool是FixedThreadPool
        return (ExecutorService) extensionAccessor.getExtensionLoader(ThreadPool.class).getAdaptiveExtension().getExecutor(url);
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/ba46ed9a-55c8-4fce-9983-6ba55a7c35de" />

<br>

**18.NettyServerHandler的源码细节**

**NettyServer本身就是一个handler，因为它的父类AbstractPeer就实现过ChannelHandler接口。而且AbstractPeer作为一个ChannelHandler实现，会实现handler的received()方法。NettyServer的createNettyServerHandler()方法会将NettyServer自己作为参数传入NettyServerHandler构造函数中。**

**一.构建NettyServer时，会将DubboProtocol的requestHandler传入NettyServer的构造函数。**

**二.NettyServer的构造函数会将该requestHandler进行ChannelHandlers.wrap()后再交给其父类的构造函数。**

**三.父类的构造函数最终找到AbstractPeer中，将包装的requestHandler赋值给AbstractPeer的handler字段。**

**四.NettyServer在构造函数中会初始化NettyServerHandler，然后作为Netty服务器的ChannelHandler并启动。**

**五.初始化NettyServerHandler时，NettyServerHandler会在构造函数中将NettyServer赋值给它的handler字段。**

**六.所以后续当Netty服务器接到网络请求要处理IO事件时，便会调用NettyServerHandler的channelRead()方法。**

**七.NettyServerHandler的channelRead()方法便会调用其handler的received()方法，也就是NettyServer的received()方法。**

**八.而NettyServer的received()方法，就是AbstractPeer的received()方法，因此会调用AbstractPeer的received()方法。**

**九.在AbstractPeer的received()方法中，会调用其handler的received()方法。也就是经过ChannelHandlers.wrap()包装后的DubboProtocol的requestHandler的received()方法。**

**十.因此，最后会调用MultiMessageHandler的received()方法。**

```
-> NettyServer.doOpen()
-> NettyServer.createNettyServerHandler()
-> new NettyServerHandler()

public class NettyServer extends AbstractServer {
    ...
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        ...
        super(ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME), ChannelHandlers.wrap(handler, url));
        ...
    }
    
    protected void doOpen() throws Throwable {
        ...
    	//创建NettyServerHandler，它是一个Netty中的ChannelHandler实现，不是Dubbo Remoting层的ChannelHandler接口的实现
    	final NettyServerHandler nettyServerHandler = createNettyServerHandler();
    	
    	//获取当前NettyServer创建的所有Channel，这里的channels集合中的Channel不是Netty中的Channel对象，而是Dubbo Remoting层的Channel对象
    	channels = nettyServerHandler.getChannels();
    	
    	//初始化ServerBootstrap，指定boss和worker EventLoopGroup
    	initServerBootstrap(nettyServerHandler);
    	...
    }
    
    //NettyServer本身就是一个handler，它的顶级父类实现了ChannelHandler接口
    //这个方法会将NettyServer自己作为参数传入NettyServerHandler之中
    protected NettyServerHandler createNettyServerHandler() {
    	return new NettyServerHandler(getUrl(), this);
    }
    ...
}

public abstract class AbstractPeer implements Endpoint, ChannelHandler {
    private final ChannelHandler handler;
    ...
    
    public void received(Channel ch, Object msg) throws RemotingException {
        if (closed) {
            return;
        }
        //下面会调用MultiMessageHandler.received()方法
        handler.received(ch, msg);
    }
    ...
}

public class NettyServerHandler extends ChannelDuplexHandler {
    private final ChannelHandler handler;
    ...
   
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        //下面会调用AbstractPeer.received()方法
        handler.received(channel, msg);
    }
    ...
}

public class DubboProtocol extends AbstractProtocol {
    ...
    private final ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {
        @Override
        public void received(Channel channel, Object message) throws RemotingException {
            ...
            reply((ExchangeChannel) channel, message);
            ...
        }
        
        @Override
        public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {
            ...
            //将客户端发过来的message转换为Invocation对象
            Invocation inv = (Invocation) message;
            
            //获取此次调用Invoker对象
            Invoker<?> invoker = getInvoker(channel, inv);
            inv.setServiceModel(invoker.getUrl().getServiceModel());
            
            // switch TCCL
            if (invoker.getUrl().getServiceModel() != null) {
                Thread.currentThread().setContextClassLoader(invoker.getUrl().getServiceModel().getClassLoader());
            }
            
            // need to consider backward-compatibility if it's a callback
            if (Boolean.TRUE.toString().equals(inv.getObjectAttachmentWithoutConvert(IS_CALLBACK_SERVICE_INVOKE))) {
                String methodsStr = invoker.getUrl().getParameters().get("methods");
                boolean hasMethod = false;
                if (methodsStr == null || !methodsStr.contains(",")) {
                    hasMethod = inv.getMethodName().equals(methodsStr);
                } else {
                    String[] methods = methodsStr.split(",");
                    for (String method : methods) {
                        if (inv.getMethodName().equals(method)) {
                            hasMethod = true;
                            break;
                        }
                    }
                }
                ...
            }
            
            //将客户端的地址记录到RpcContext中
            RpcContext.getServiceContext().setRemoteAddress(channel.getRemoteAddress());
            
            //provider服务端会在下面执行真正的本地实现类的调用
            //比如下面会调用到FilterChainBuilder.FilterChainNode.invoke()方法 --> AbstractProxyInvoker.invoke()方法
            Result result = invoker.invoke(inv);
            
            //返回结果
            return result.thenApply(Function.identity());
        }
    };
    ...
    
    Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
        ...
        //根据服务名称获取到DubboExporter服务发布实例
        DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);
        
        ...
        //根据服务发布实例获取目标实现类的代理ProxyInvoker
        return exporter.getInvoker();
    }
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/d3301b14-1891-491f-841b-eca74f68bf23" />

<br>

**19.NettyServer如何转发请求给业务线程池**

**关键在于：将DubboProtocol的ExchangeHandler类型的requestHandler传入时，会进行包装，包装的过程是由ChannelHandlers.wrap()来实现的。**

```
//构造NettyServer时
-> ChannelHandlers.wrap()
-> ChannelHandlers.wrapInternal()

//处理请求时
-> MultiMessageHandler.received()
-> AllChannelHandler.received()
-> executor.execute()

public class NettyServer extends AbstractServer {
    ...
    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        ...
        super(ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME), ChannelHandlers.wrap(handler, url));
        ...
    }
    ...
}

public class ChannelHandlers {
    ...
    public static ChannelHandler wrap(ChannelHandler handler, URL url) {
        return ChannelHandlers.getInstance().wrapInternal(handler, url);
    }
    
    protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
        return new MultiMessageHandler(new HeartbeatHandler(url.getOrDefaultFrameworkModel().getExtensionLoader(Dispatcher.class)
            .getAdaptiveExtension().dispatch(handler, url)));
    }
    ...
}

public class MultiMessageHandler extends AbstractChannelHandlerDelegate {
    ...
    public void received(Channel channel, Object message) throws RemotingException {
        if (message instanceof MultiMessage) {
            MultiMessage list = (MultiMessage) message;
            for (Object obj : list) {
                try {
                    //下面会执行HeartbeatHandler.received()方法
                    handler.received(channel, obj);
                } catch (Throwable t) {
                    logger.error("MultiMessageHandler received fail.", t);
                    try {
                        handler.caught(channel, t);
                    } catch (Throwable t1) {
                        logger.error("MultiMessageHandler caught fail.", t1);
                    }
                }
            }
        } else {
            //下面会执行HeartbeatHandler.received()方法
            handler.received(channel, message);
        }
    }
}

public class HeartbeatHandler extends AbstractChannelHandlerDelegate {
    ...
    public void received(Channel channel, Object message) throws RemotingException {
        //记录最近的读写事件时间戳
        setReadTimestamp(channel);
        if (isHeartbeatRequest(message)) {
            //收到心跳请求
            Request req = (Request) message;
            if (req.isTwoWay()) {
                //返回心跳响应，注意，携带请求的ID
                Response res = new Response(req.getId(), req.getVersion());
                res.setEvent(HEARTBEAT_EVENT);
                channel.send(res);
                if (logger.isDebugEnabled()) {
                    int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                    logger.debug("..."));
                }
            }
            return;
        }
        
        if (isHeartbeatResponse(message)) {
            //收到心跳响应
            if (logger.isDebugEnabled()) {
                logger.debug("Receive heartbeat response in thread " + Thread.currentThread().getName());
            }
            return;
        }
        
        //下面会执行AllChannelHandler.received()方法
        handler.received(channel, message);
    }
    ...
}

//为什么需要不同的分发策略，这与不同的情况是有关系
//如果要执行的代码并没有外部数据库的IO操作，那么可以选择DirectDispatcher
//如果不关注网络连接和断开，只关注请求和响应的处理，那么可以选择MessageOnlyDispatcher或者ExecutionDispatcher
//如果对网络连接和端口特别关注，那么可以选择ConnectionOrderedChannelHandler
public class AllChannelHandler extends WrappedChannelHandler {
    ...
    //如果收到某consumer端一个请求
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        //获取线程池
        ExecutorService executor = getPreferredExecutorService(message);
        try {
            //将消息封装成ChannelEventRunnable任务，提交到线程池中执行
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            //如果线程池满了，请求会被拒绝，这里会根据请求配置决定是否返回一个说明性的响应
            if (message instanceof Request && t instanceof RejectedExecutionException){
                sendFeedback(channel, (Request) message, t);
                return;
            }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
    	}
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/d44fd570-1543-4c32-ba27-0b6383f77e07" />
