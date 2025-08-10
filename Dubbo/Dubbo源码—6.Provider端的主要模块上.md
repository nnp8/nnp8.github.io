# Dubbo源码—6.Provider端的主要模块上

**大纲**

**1.Provider启动时的源码细节**

**2.分析Dubbo的Model组件体系**

**3.ScopeModel抽象父类的源码细节**

**4.ScopeModel和SPI机制及Bean容器的关系**

**5.ModuleModel组件的源码细节**

**6.ModuleModel的服务数据管理机制**

**7.ApplicationModel组件的源码细节**

**8.FrameworkModel顶层组件的源码细节**

**9.Dubbo源码如何使用SPI机制**

**10.ExtensionLoader构造函数的实现细节**

**11.Dubbo的SPI扩展实例的构建细节**

**12.Extension扩展实例的构建过程**

**13.Dubbo自定义Bean容器的源码细节**

**14.ModuleDeployer组件的源码细节**

<br>

**Dubbo的Provider端主要包括如下几大块：**

一.model组件体系(与SPI机制的使用强相关)

二.服务注册

三.网络发布

<br>

**1.Provider启动时的源码细节**

**(1)Provider启动时会调用getScopeModel()方法来进行初始化**

**(2)Provider启动过程中scopeModel的初始化**

<br>

**(1)Provider启动时会调用getScopeModel()方法来进行初始化**

```
-> ServiceConfig.export()
-> getScopeModel().getDeployer().start();
-> AbstractMethodConfig.getScopeModel()

public class Application {
    public static void main(String[] args) throws Exception {
        //ServiceConfig是什么?
        //Service又是什么，Service可以理解为一个服务，每个服务可以包含多个接口
        //ServiceConfig便是针对这个服务的一些配置信息，下面传入的泛型DemoServiceImpl便是服务接口的实现
        ServiceConfig<DemoServiceImpl> service = new ServiceConfig<>();
        
        //设置服务暴露出去的接口
        service.setInterface(DemoService.class);
        
        //设置暴露出去的接口的实现类
        service.setRef(new DemoServiceImpl());
        
        //服务名称，可以在服务框架里进行定位
        service.setApplication(new ApplicationConfig("dubbo-demo-api-provider"));
        
        //所有的RPC框架，必须要和注册中心配合使用，服务启动后必须向注册中心进行注册
        //注册中心可以知道每个服务有几个实例，每个实例在哪台服务器上
        //进行服务调用时，要先找注册中心咨询要调用的服务有几个实例，分别都在什么机器上
        //下面便是设置zookeeper作为注册中心的地址
        service.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
        
        //设置元数据上报的地方，dubbo服务实例启动后，会有自己的元数据，需要上报到一个地方进行管理，比如zookeeper
        service.setMetadataReportConfig(new MetadataReportConfig("zookeeper://127.0.0.1:2181"));
        
        //配置完毕后，调用export()方法：
        //启动网络监听程序、当接收到调用请求时要建立网络连接以便进行网络通信，接收按照协议封装的请求数据，执行RPC调用，以及把自己作为一个服务实例注册到zk里
        service.export();
        System.out.println("dubbo service started");
        new CountDownLatch(1).await();
    }
}

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    public ServiceConfig() {
    }
    
    public void export() {
        ...
        //getScopeModel()返回的是一个ModuleModel
        getScopeModel().getDeployer().start();
        ...
    }
    ...
}

public abstract class AbstractMethodConfig extends AbstractConfig {
    ...
    public ModuleModel getScopeModel() {
        return (ModuleModel) scopeModel;
    }
    ...
}

public abstract class AbstractConfig implements Serializable {
    ...
    //Dubbo里有一个model组件体系，其中ScopeModel是基础，ScopeModel类型可以转换为ModuleModel、ApplicationModel
    protected ScopeModel scopeModel;
    ...
}
```

**需要注意的是：Dubbo3里有一个model组件体系，其中ScopeModel是基础。ScopeModel类型可以转换为ModuleModel、ApplicationModel。**

<br>

**(2)Provider启动过程中scopeModel的初始化**

```
-> new ServiceConfig()
-> new ServiceConfigBase()
-> new AbstractServiceConfig()
-> new AbstractInterfaceConfig()
-> new AbstractMethodConfig()
-> ApplicationModel.defaultModel()
-> new AbstractConfig(scopeModel)

public class Application {
    public static void main(String[] args) throws Exception {
        ServiceConfig<DemoServiceImpl> service = new ServiceConfig<>();
        ...
    }
}

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    public ServiceConfig() {
        //接着会执行new ServiceConfigBase()
    }
    ...
}

public abstract class ServiceConfigBase<T> extends AbstractServiceConfig {
    public ServiceConfigBase() {
        serviceMetadata = new ServiceMetadata();
        serviceMetadata.addAttribute("ORIGIN_CONFIG", this);
        //接着会执行new AbstractServiceConfig()
    }
    ...
}

public abstract class AbstractServiceConfig extends AbstractInterfaceConfig {
    public AbstractServiceConfig() {
        //接着会执行new AbstractInterfaceConfig()
    }
    ...
}

public abstract class AbstractInterfaceConfig extends AbstractMethodConfig {
    public AbstractInterfaceConfig() {
        //接着会执行new AbstractMethodConfig()
    }
    ...
}

public abstract class AbstractMethodConfig extends AbstractConfig {
    ...
    public AbstractMethodConfig() {
        //接着会执行new AbstractConfig(scopeModel)
        super(ApplicationModel.defaultModel().getDefaultModule());
    }
    
    public ModuleModel getScopeModel() {
        return (ModuleModel) scopeModel;
    }
    ...
}

public class ApplicationModel extends ScopeModel {
    ...
    public static ApplicationModel defaultModel() {
        return FrameworkModel.defaultModel().defaultApplication();
    }
    ...
}

public abstract class AbstractConfig implements Serializable {
    //Dubbo里有一个model组件体系，其中ScopeModel是基础，ScopeModel类型可以转换为ModuleModel、ApplicationModel
    protected ScopeModel scopeModel;
    ...
    
    public AbstractConfig(ScopeModel scopeModel) {
        this.setScopeModel(scopeModel);
    }
}
```

<br>

**2.分析Dubbo的Model组件体系**

```
//model组件体系，其实就是封装了Dubbo内部所有的公共的组件体系
//model组件设计思想类似于门面模式，model本身是没有意义的，model的作用就类似于门面，它封装了很多的组件如SPI、service数据、配置、Repository组件、BeanFactory
//所以model就成为了一个门面，在整个Dubbo框架里，如果要用到一些公共组件，就直接找model去获取就可以了
```

**ScopeModel是model组件体系里最顶层的抽象父类，ScopeModel实现了一个关键接口ExtensionAccessor。ExtensionAccessor就是extension(SPI扩展实现类)的访问器，其中Accessor有访问器之意，所以ExtensionAccessor可以理解为是一个获取extension的组件。如果一个类实现了ExtensionAccessor接口，则随时可以获取指定接口的扩展实现类。**

**ScopeModel有一个类型是ScopeModel的parent属性。一个model组件体系会基于ScopeModel的parent属性来构建其组件间的关系结构，该结构与树结构类似。**

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/3fa887f2-711a-4324-84ee-81eb15c20834" />

**model组件体系对整个Dubbo源码的运行很关键的，可以认为它是SPI机制使用的入口。而ScopeModel是model组件体系的一个基础，ScopeModel类型是可以转换为ModuleModel、ApplicationModel。**

**比如ModuleServiceRepository、ModelEnvironment、BeanFactory等很多通用的组件都可以通过ScopeModel去获取。**

<br>

**3.ScopeModel抽象父类的源码细节**

```
//model组件体系，其实就是封装了Dubbo内部所有的公共的组件体系
//model组件设计思想类似于门面模式，model本身是没有意义的，model的作用就类似于门面，它封装了很多的组件如SPI、service数据、配置、Repository组件、BeanFactory
//所以model就成为了一个门面，在整个Dubbo框架里，如果要用到一些公共组件，就直接找model去获取就可以了

//ScopeModel是model组件体系里最顶层的抽象父类
//ScopeModel实现了一个关键接口ExtensionAccessor
//ExtensionAccessor，就是extension(SPI扩展实现类)的访问器，其中Accessor有访问器之意
//所以ExtensionAccessor可以理解为是一个获取extension的组件
//如果一个类实现了ExtensionAccessor接口，则代表该类具备了使用Dubbo的SPI机制，也就是随时可以获取指定接口的扩展实现类
//ScopeModel中比较关键的，其实就是与SPI机制相关的ExtensionScope，ExtensionDirector，BeanFactory
public abstract class ScopeModel implements ExtensionAccessor {
    //内部ID，根据不同的内部ID代表着不同的model组件体系
    //比如1代表的是：FrameworkModel
    //比如1.2代表的是：FrameworkModel -> ApplicationModel
    //比如1.2.0、1.2.1代表的是：FrameworkModel -> ApplicationModel -> ModuleModel
    private String internalId;

    private String modelName;
    private String desc;

    //要使用的类加载器集合
    private Set<ClassLoader> classLoaders;

    //parent的类型是ScopeModel，即一个model组件体系会基于parent属性来构建其组件间的关系结构，该结构与树结构类似
    private final ScopeModel parent;

    //scope属性这与SPI机制的使用有关，scope是一个枚举类型，代表了在这里使用SPI机制的范围
    //可能会有很多model组件，其不同的ExtensionScope范围就决定了：
    //最后创建出的SPI扩展实现类实例，可以在哪一个model组件里使用，或者可以跟其他model组件进行共享使用
    private final ExtensionScope scope;

    //ExtensionDirector本质就是一个ExtensionLoader的管理器(管理组件)
    //如果要对某个接口加载它对应的SPI扩展实现类实例，就必须先通过ExtensionDirector来获取该接口对应的ExtensionLoader组件
    //然后再通过ExtensionLoader组件去获取对应的SPI扩展实现类实例
    private ExtensionDirector extensionDirector;

    //Dubbo内部的bean容器工厂
    private ScopeBeanFactory beanFactory;

    //每个model组件都会有自己的生命周期，比如创建、使用、销毁
    //如果一个model组件有销毁行为，则可以在销毁前先回调其销毁事件的监听器ScopeModelDestroyListener
    private List<ScopeModelDestroyListener> destroyListeners;

    //model组件关联的一些属性数据
    private Map<String, Object> attributes;

    //JDK并发包里提供的一个Atomic变量，destroyed表示model组件是否被销毁
    private final AtomicBoolean destroyed = new AtomicBoolean(false);
    private final boolean internalScope;
    ...
}

//在某个model组件里，如果通过SPI机制获取了一些extension扩展实现类的实例，这些extension实例使用范围会是什么呢？
//究竟是可以在FrameworkModel这个范围里可以用，还是可以在ApplicationModel或ModuleModel这些层级里使用
public enum ExtensionScope {
    FRAMEWORK,
    //一个Application可以有多个Modules，不同的Application创建出来的extension扩展实现类实例是不同的
    //在一个ApplicationModel里创建的extension扩展实现类实例，除了可以给自己ApplicationModel使用外，还可以给自己的子ModuleModel来使用
    APPLICATION,
    MODULE,
    SELF
}

public class ExtensionDirector implements ExtensionAccessor {
    //key是接口类型的class对象，value是ExtensionLoader，每个class都会对应一个ExtensionLoader
    private final ConcurrentMap<Class<?>, ExtensionLoader<?>> extensionLoadersMap = new ConcurrentHashMap<>(64);
    private final ConcurrentMap<Class<?>, ExtensionScope> extensionScopeMap = new ConcurrentHashMap<>(64);

    //ExtensionDirector也有一个父级组件，从而形成一个树形关系
    private final ExtensionDirector parent;

    //表示该ExtensionDirector拿到的ExtensionLoader，然后再通过该ExtensionLoader所获取到的扩展实现类实例的使用范围
    private final ExtensionScope scope;

    //extension扩展实现类实例的后处理器，extension扩展实现类实例初始化后需要进行的后处理
    private final List<ExtensionPostProcessor> extensionPostProcessors = new ArrayList<>();

    //每个model组件都会关联一个ExtensionDirector组件，反过来，每个ExtensionDirector也会关联一个model组件
    private final ScopeModel scopeModel;
    ...
}
```

<br>

**4.ScopeModel和SPI机制及Bean容器的关系**

```
public abstract class ScopeModel implements ExtensionAccessor {
    ...
    //ScopeModel构造函数里，需要传入对应的parent，通过parent变量把model体系组装成一个树
    //需要传入一个ExtensionScope，表示在这个model组件里创建的SPI扩展实现类实例的使用范围是什么
    public ScopeModel(ScopeModel parent, ExtensionScope scope, boolean isInternal) {
    		this.parent = parent;
    		this.scope = scope;
    		this.internalScope = isInternal;
    }

    protected void initialize() {
    		//初始化ExtensionDirector
    		this.extensionDirector = new ExtensionDirector(parent != null ? parent.getExtensionDirector() : null, scope, this);

     		//通过SPI机制创建和获取扩展实现类实例时，在对实例初始化时，有前处理和后处理的过程
    		this.extensionDirector.addExtensionPostProcessor(new ScopeModelAwareExtensionProcessor(this));

     		//构建出一个Dubbo内部的bean容器，小型的bean容器
    		this.beanFactory = new ScopeBeanFactory(parent != null ? parent.getBeanFactory() : null, extensionDirector);

     		this.destroyListeners = new LinkedList<>();
    		this.attributes = new ConcurrentHashMap<>();
    		this.classLoaders = new ConcurrentHashSet<>();

    		//Add Framework's ClassLoader by default
    		ClassLoader dubboClassLoader = ScopeModel.class.getClassLoader();
    		if (dubboClassLoader != null) {
    	    	this.addClassLoader(dubboClassLoader);
    		}
    }
    ...
}

//ScopeBeanFactory是Dubbo框架内部实现的一个bean工厂，bean工厂管理的这些bean可以在Dubbo框架内部进行共享
public class ScopeBeanFactory {
    //ScopeBeanFactory也可以通过parent组成一颗树
    private final ScopeBeanFactory parent;

    //extension扩展实现类实例的获取组件，有了它就可以使用SPI机制了
    private ExtensionAccessor extensionAccessor;

    //extension扩展实现类的实例在初始化后的处理器
    private List<ExtensionPostProcessor> extensionPostProcessors;

    //每一个class都有一个AtomicInteger作为一个计数器
    private Map<Class, AtomicInteger> beanNameIdCounterMap = new ConcurrentHashMap<>();

    //注册过的bean实例信息
    private List<BeanInfo> registeredBeanInfos = new CopyOnWriteArrayList<>();

    //初始化策略逻辑，bean实例的初始化
    private InstantiationStrategy instantiationStrategy;
    ...
}
```

<br>

**5.ModuleModel组件的源码细节**

```
// Model of a service module
// 这是一个ServiceModule的model，也就是一个服务模块的model组件模型
public class ModuleModel extends ScopeModel {
    //ApplicationModel内部封装了其他很多组件，在这里是一个引用关系，通过构造函数传入进来
    private final ApplicationModel applicationModel;
    //包含了ServiceModule环境相关的数据，里面封装的都是各种各样的配置信息
    private ModuleEnvironment moduleEnvironment;
    //serviceRepository是一个服务仓储组件，存储了一些服务相关的数据
    private ModuleServiceRepository serviceRepository;
    //这是module配置管理器，用于存放一些服务相关的配置数据
    private ModuleConfigManager moduleConfigManager;
    //这是ModuleDeployer组件，用于管理其他的一些组件和模块的生命周期
    private ModuleDeployer deployer;
    ...
}

// Service repository for module
// 服务相关的一些数据，都放在了这个repository仓储组件里
public class ModuleServiceRepository {
    //services，代表服务相关的数据
    private final ConcurrentMap<String, List<ServiceDescriptor>> services = new ConcurrentHashMap<>();
    //consumers (key - group/interface:version value - consumerModel list)，代表服务的调用方(consumer即消费方/调用方）
    private final ConcurrentMap<String, List<ConsumerModel>> consumers = new ConcurrentHashMap<>();
    //providers，代表服务提供方
    private final ConcurrentMap<String, ProviderModel> providers = new ConcurrentHashMap<>();
    //FrameworkServiceRepository存储的也是一些服务相关的数据
    private final FrameworkServiceRepository frameworkServiceRepository;
    ...
}

public class ModuleConfigManager extends AbstractConfigManager implements ModuleExt {
    ...
    //缓存了各个服务的配置信息
    private Map<String, AbstractInterfaceConfig> serviceConfigCache = new ConcurrentHashMap<>();
    ...
}

//可以通过ModuleDeployer对其他一些组件和模块的生命周期进行管理，比如初始化、启动、停止、预销毁、销毁后处理等
public interface ModuleDeployer extends Deployer<ModuleModel> {
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/78ae967e-cb19-413e-ab8c-53bd6f488825" />

<br>

**6.ModuleModel的服务数据管理机制**

在构造ModuleModel时，会传递进来一个ApplicationModel，并且设置其创建的扩展实现类的使用范围是MODULE。ModuleModel会把ApplicationModel设置为自己的parent父组件，一个ApplicationModel里可以加入多个ModuleModel。

```
// Model of a service module
// 这是一个ServiceModule的model，也就是一个服务模块的model组件模型
public class ModuleModel extends ScopeModel {
    ...
    //在构造ModuleModel时，会传递进来一个ApplicationModel，并且设置其创建的扩展实现类的使用范围是MODULE
    //ModuleModel会把ApplicationModel设置为自己的parent父组件
    public ModuleModel(ApplicationModel applicationModel, boolean isInternal) {
    		super(applicationModel, ExtensionScope.MODULE, isInternal);
    		Assert.notNull(applicationModel, "ApplicationModel can not be null");
    		this.applicationModel = applicationModel;

    		//由如下一行代码可知，一个ApplicationModel里可以加入多个ModuleModel
    		applicationModel.addModule(this, isInternal);
    		if (LOGGER.isInfoEnabled()) {
    	    	LOGGER.info(getDesc() + " is created");
    		}

    		initialize();
    		Assert.notNull(getServiceRepository(), "ModuleServiceRepository can not be null");
    		Assert.notNull(getConfigManager(), "ModuleConfigManager can not be null");
    		Assert.assertTrue(getConfigManager().isInitialized(), "ModuleConfigManager can not be initialized");

    		// notify application check state
    		ApplicationDeployer applicationDeployer = applicationModel.getDeployer();
    		if (applicationDeployer != null) {
    	    	applicationDeployer.notifyModuleChanged(this, DeployState.PENDING);
    		}
    }

    @Override
    protected void initialize() {
        super.initialize();
        this.serviceRepository = new ModuleServiceRepository(this);
        initModuleExt();

        //通过SPI机制，先获取到ScopeModelInitializer接口的ExtensionLoader
        //由此可见，Dubbo基于SPI机制把扩展性做的特别好，因为它几乎把所有的核心组件都实现了可以基于SPI机制进行扩展和自定义
        ExtensionLoader<ScopeModelInitializer> initializerExtensionLoader = this.getExtensionLoader(ScopeModelInitializer.class);

        //再通过ExtensionLoader去获取接口对应的extension扩展实现类的实例
        Set<ScopeModelInitializer> initializers = initializerExtensionLoader.getSupportedExtensionInstances();

        //下面会做一个遍历，直接回调initializer的initializeModuleModel()方法
        //所以如果要在这里进行扩展，可以自定义ScopeModelInitializer接口的扩展实现，进行SPI配置
        //那么在Dubbo初始化的过程中，就会去回调该自定义的SPI扩展
        for (ScopeModelInitializer initializer : initializers) {
            initializer.initializeModuleModel(this);
        }
    }
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/90f32edb-e5e9-4146-90bb-8f6052e956ba" />

<br>

**7.ApplicationModel组件的源码细节**

```
//ApplicationModel类会被设计成单例或者静态的
public class ApplicationModel extends ScopeModel {
    protected static final Logger LOGGER = LoggerFactory.getLogger(ApplicationModel.class);
    public static final String NAME = "ApplicationModel";

    //包含了多个ModuleModel
    private final List<ModuleModel> moduleModels = new CopyOnWriteArrayList<>();
    private final List<ModuleModel> pubModuleModels = new CopyOnWriteArrayList<>();

    //环境变量、配置信息
    private Environment environment;

    //服务配置相关的一些信息
    private ConfigManager configManager;

    //服务数据相关的一些存储
    private ServiceRepository serviceRepository;

    //属于application层级的一些组件的生命周期管理
    private ApplicationDeployer deployer;

    //父级组件
    private final FrameworkModel frameworkModel;

    //内部的一个ModuleModel组件
    private ModuleModel internalModule;

    //默认的一个ModuleModel组件
    private volatile ModuleModel defaultModule;

    //internal module index is 0, default module index is 1
    private AtomicInteger moduleIndex = new AtomicInteger(0);

    //是一个锁
    private Object moduleLock = new Object();
    ...

    public ApplicationModel(FrameworkModel frameworkModel, boolean isInternal) {
    		//父级组件就是frameworkModel
    		super(frameworkModel, ExtensionScope.APPLICATION, isInternal);
    		Assert.notNull(frameworkModel, "FrameworkModel can not be null");
    		this.frameworkModel = frameworkModel;
    		frameworkModel.addApplication(this);
    		if (LOGGER.isInfoEnabled()) {
    	    	LOGGER.info(getDesc() + " is created");
    		}

    		initialize();
    		Assert.notNull(getApplicationServiceRepository(), "ApplicationServiceRepository can not be null");
    		Assert.notNull(getApplicationConfigManager(), "ApplicationConfigManager can not be null");
    		Assert.assertTrue(getApplicationConfigManager().isInitialized(), "ApplicationConfigManager can not be initialized");
    }

    protected void initialize() {
        super.initialize();
        internalModule = new ModuleModel(this, true);
        this.serviceRepository = new ServiceRepository(this);
        ExtensionLoader<ApplicationInitListener> extensionLoader = this.getExtensionLoader(ApplicationInitListener.class);
        Set<String> listenerNames = extensionLoader.getSupportedExtensions();
        for (String listenerName : listenerNames) {
    	    	extensionLoader.getExtension(listenerName).init();
        }

        initApplicationExts();
        ExtensionLoader<ScopeModelInitializer> initializerExtensionLoader = this.getExtensionLoader(ScopeModelInitializer.class);
        Set<ScopeModelInitializer> initializers = initializerExtensionLoader.getSupportedExtensionInstances();
        for (ScopeModelInitializer initializer : initializers) {
            initializer.initializeApplicationModel(this);
        }
    }
    ...

    public static ApplicationModel defaultModel() {
    		//should get from default FrameworkModel, avoid out of sync
    		return FrameworkModel.defaultModel().defaultApplication();
    }
    ...
}

//在某个model组件里，如果通过SPI机制获取了一些extension扩展实现类的实例，这些extension实例使用范围会是什么呢？
//究竟是可以在FrameworkModel这个范围里可以用，还是可以在ApplicationModel或ModuleModel这些层级里使用
public enum ExtensionScope {
    ...
    //一个Application可以有多个Modules，不同的Application创建出来的extension扩展实现类实例是不同的
    //在一个ApplicationModel里创建的extension扩展实现类实例，除了可以给自己ApplicationModel使用外，还可以给自己的子ModuleModel来使用
    APPLICATION,
    ...
}
```

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/22b574c6-dbbf-43ed-95cf-182f0b7d28f5" />

<br>

**8.FrameworkModel顶层组件的源码细节**

```
public class FrameworkModel extends ScopeModel {
    ...
    //默认单例
    private volatile static FrameworkModel defaultInstance;
    private volatile ApplicationModel defaultAppModel;

    //它没有父层级了，所以只能通过static静态变量，类级别去引用自己的FrameworkModel集合
    private static List<FrameworkModel> allInstances = new CopyOnWriteArrayList<>();

    //包含了多个ApplicationModel
    private List<ApplicationModel> applicationModels = new CopyOnWriteArrayList<>();
    private List<ApplicationModel> pubApplicationModels = new CopyOnWriteArrayList<>();

    //通过Framework、Application、Module各个层级都可以获取到service相关的配置和数据
    private FrameworkServiceRepository serviceRepository;
    ...

    public FrameworkModel() {
    		//FrameworkModel没有了parent
    		super(null, ExtensionScope.FRAMEWORK, false);
    		this.setInternalId(String.valueOf(index.getAndIncrement()));

    		// register FrameworkModel instance early
    		synchronized (globalLock) {
    	    	allInstances.add(this);
    	    	resetDefaultFrameworkModel();
    		}

    		if (LOGGER.isInfoEnabled()) {
    	    	LOGGER.info(getDesc() + " is created");
    		}
    		initialize();
    }
    ...
}
```

<br>

**9.Dubbo源码如何使用SPI机制**

ScopeModel里有一个ExtensionDirector属性，而ExtensionDirector代表着一个ExtensionLoader管理组件。通过ExtensionDirector可以拿到各种接口的ExtensionLoader，从而通过ExtensionLoader去拿到SPI接口的扩展实现类。

下面展示一下通过ExtensionLoader来获取扩展实现类的实例：

```
public class ExtensionLoaderTest {
    ...
    private <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    		return ApplicationModel.defaultModel().getExtensionDirector().getExtensionLoader(type);
    }

    @Test
    public void test_getLoadedExtension() {
    		SimpleExt simpleExt = getExtensionLoader(SimpleExt.class).getExtension("impl1");
    		assertThat(simpleExt, instanceOf(SimpleExtImpl1.class));
    }
    ...
}

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    protected void postProcessAfterScopeModelChanged(ScopeModel oldScopeModel, ScopeModel newScopeModel) {
    		super.postProcessAfterScopeModelChanged(oldScopeModel, newScopeModel);
    		protocolSPI = this.getExtensionLoader(Protocol.class).getAdaptiveExtension();
    		proxyFactory = this.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
    }

    private URL buildUrl(ProtocolConfig protocolConfig, Map<String, String> params) {
        ...
        url = this.getExtensionLoader(ConfiguratorFactory.class)
            .getExtension(url.getProtocol())
            .getConfigurator(url)
            .configure(url);
        ...
    }
    ...
}

//ScopeModel是model组件体系里最顶层的抽象父类
//ScopeModel实现了一个关键接口ExtensionAccessor
//ExtensionAccessor，就是extension(SPI扩展实现类)的访问器，其中Accessor有访问器之意
//所以ExtensionAccessor可以理解为是一个获取extension的组件
//如果一个类实现了ExtensionAccessor接口，则代表该类具备了使用Dubbo的SPI机制，也就是随时可以获取指定接口的扩展实现类
//ScopeModel中比较关键的，其实就是与SPI机制相关的ExtensionScope，ExtensionDirector，BeanFactory
public abstract class ScopeModel implements ExtensionAccessor {
    ...
    //ExtensionDirector本质就是一个ExtensionLoader的管理器(管理组件)
    //如果要对某个接口加载它对应的SPI扩展实现类实例，就必须先通过ExtensionDirector来获取该接口对应的ExtensionLoader组件
    //然后再通过ExtensionLoader组件去获取对应的SPI扩展实现类实例
    private ExtensionDirector extensionDirector;

    @Override
    public ExtensionDirector getExtensionDirector() {
    		return extensionDirector;
    }
    ...
}

public class ExtensionDirector implements ExtensionAccessor {
    //key是接口类型的class对象，value是ExtensionLoader，每个class都会对应一个ExtensionLoader
    private final ConcurrentMap<Class<?>, ExtensionLoader<?>> extensionLoadersMap = new ConcurrentHashMap<>(64);
    private final ConcurrentMap<Class<?>, ExtensionScope> extensionScopeMap = new ConcurrentHashMap<>(64);

    //ExtensionDirector也有一个父级组件，从而形成一个树形关系
    private final ExtensionDirector parent;

    //表示该ExtensionDirector拿到的ExtensionLoader，然后再通过该ExtensionLoader所获取到的扩展实现类实例的使用范围
    private final ExtensionScope scope;

    //extension扩展实现类实例的后处理器，extension扩展实现类实例初始化后需要进行的后处理
    private final List<ExtensionPostProcessor> extensionPostProcessors = new ArrayList<>();

    //每个model组件都会关联一个ExtensionDirector组件，反过来，每个ExtensionDirector也会关联一个model组件
    private final ScopeModel scopeModel;
    ...

    //由于ExtensionDirector是一个ExtensionLoader管理组件
    //所以通过ExtensionDirector可以拿到各种接口的ExtensionLoader，从而通过ExtensionLoader去拿到SPI接口的扩展实现类
    //通常会将一个加了@SPI注解的接口的class传递进getExtensionLoader()方法，来获取一个ExtensionLoader
    @Override
    public <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
        ...
        //1.find in local cache
        //按照type接口的class类型去获取ExtensionLoader缓存
        ExtensionLoader<T> loader = (ExtensionLoader<T>) extensionLoadersMap.get(type);
        ExtensionScope scope = extensionScopeMap.get(type);
        if (scope == null) {
            //从type接口class里面拿到@SPI注解
            SPI annotation = type.getAnnotation(SPI.class);
            //拿到这个SPI注解后，就会获取这个注解里的scope范围ExtensionScope
            scope = annotation.scope();
            extensionScopeMap.put(type, scope);
        }
        
        //获取出来的ExtensionLoader是空的 + 而且scope是SELF范围
        if (loader == null && scope == ExtensionScope.SELF) {
            //create an instance in self scope
            loader = createExtensionLoader0(type);
        }
        
        //2.find in parent
        //如果ExtensionLoader没有拿到，同时scope范围不是self
        if (loader == null) {
            //在创建ExtensionLoader的过程中，会有父组件依赖和搜寻的过程
            if (this.parent != null) {
                loader = this.parent.getExtensionLoader(type);
            }
        }
        
        //3.create it
        if (loader == null) {
            loader = createExtensionLoader(type);
        }
        
        //总结：
        //第一步，先去缓存搜索；
        //第二步，scope=self，尝试直接自己创建；
        //第三步，parent里搜索；
        //最后一步，直接尝试自己创建ExtensionLoader；
        return loader;
    }
    ...
}
```

<br>

**10.ExtensionLoader构造函数的实现细节**

```
-> ExtensionDirector.getExtensionLoader()
-> ExtensionDirector.createExtensionLoader0()
-> new ExtensionLoader<T>(type, this, scopeModel)

public class ExtensionLoader<T> {
    ...
    //这个构造函数的入参包括：
    //所要加载的扩展实现类的接口class，负责管理这个要加载的扩展实现类的ExtensionDirector，以及和该ExtensionDirector绑定的scope
    ExtensionLoader(Class<?> type, ExtensionDirector extensionDirector, ScopeModel scopeModel) {
        this.type = type;
        this.extensionDirector = extensionDirector;
        this.extensionPostProcessors = extensionDirector.getExtensionPostProcessors();
        
        //初始化extension扩展实现类的实例构建的策略逻辑
        initInstantiationStrategy();
        
        //下面就用SPI机制，也就是通过"extensionDirector.getExtensionLoader(ExtensionInjector.class)"的方式，来获取ExtensionInjector
        //injector注入器，类似于依赖注入，extension对象实例构建的过程中会有一个依赖注入的过程
        this.injector = (type == ExtensionInjector.class ? null : extensionDirector.getExtensionLoader(ExtensionInjector.class).getAdaptiveExtension());
        this.activateComparator = new ActivateComparator(extensionDirector);
        this.scopeModel = scopeModel;
    }
   
    private void initInstantiationStrategy() {
        instantiationStrategy = extensionPostProcessors.stream()
            .filter(extensionPostProcessor -> extensionPostProcessor instanceof ScopeModelAccessor)
            .map(extensionPostProcessor -> new InstantiationStrategy((ScopeModelAccessor) extensionPostProcessor))
            .findFirst()
            .orElse(new InstantiationStrategy());
    }
    ...
}
```

<br>

**11.Dubbo的SPI扩展实例的构建细节**

Dubbo源码里会大量使用"extensionDirector.getExtensionLoader(Class).getExtension()"这种代码来获取扩展实现类的实例。

注意：ExtensionLoader会通过Holder来存放构建好的实例，这其实利用了类似于分段加锁这种细粒度加锁的方法。这样设计只会锁当前实例，而不会锁其他实例。

```
-> ExtensionLoader.getExtension()
-> ExtensionLoader.getOrCreateHolder()
-> ExtensionLoader.createExtension()

public class ExtensionLoader<T> {
    //缓存了扩展实现类和其实例的映射关系
    private final ConcurrentMap<Class<?>, Object> extensionInstances = new ConcurrentHashMap<>(64);

    //当前ExtensionLoader实例负责加载的扩展接口
    private final Class<?> type;

    //扩展实现类的实例注入器
    private final ExtensionInjector injector;

    //缓存了该ExtensionLoader加载的：扩展实现类与扩展名之间的映射关系
    private final ConcurrentMap<Class<?>, String> cachedNames = new ConcurrentHashMap<>();

    //缓存了该ExtensionLoader加载的：扩展名与扩展实现类之间的映射关系，cachedNames集合的反向关系缓存
    private final Holder<Map<String, Class<?>>> cachedClasses = new Holder<>();

    //缓存了该ExtensionLoader加载的：扩展名与扩展实现对象之间的映射关系，也就是被缓存起来的实例集合
    //实现name -> extension实例对象的Holder容器，这个Holder其实利用了类似于分段加锁这种细粒度加锁的方法
    private final ConcurrentMap<String, Holder<Object>> cachedInstances = new ConcurrentHashMap<>();

    //缓存的一些Adaptive类型的extension实例对象
    //Adaptive是自适应的意思，即后续会根据扩展实现类的@Adaptive注解+URL参数，自动去筛选和判断到底用哪个实现类
    private final Holder<Object> cachedAdaptiveInstance = new Holder<>();

    ...

    //Dubbo源码里会大量使用"extensionDirector.getExtensionLoader(Class).getExtension()"这种代码来获取扩展实现类的实例
    //getExtension()方法中的name参数一般是从扩展点的SPI注解里提取出来的一个name
    public T getExtension(String name) {
        T extension = getExtension(name, true);//wrap参数默认是true
        if (extension == null) {
            throw new IllegalArgumentException("Not find extension: " + name);
        }
        return extension;
    }
    
    public T getExtension(String name, boolean wrap) {
        checkDestroyed();
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
        
        //这是一个缓存key
        String cacheKey = name;
        if (!wrap) {
            cacheKey += "_origin";
        }
        
        //基于缓存key去构建和创建一个Holder容器，getOrCreateHolder()方法中封装了查找cachedInstances缓存的逻辑
        final Holder<Object> holder = getOrCreateHolder(cacheKey);
        
        //对holder进行get()拿到里面封装的具体的实现类对象
        //这个Holder其实利用了类似于分段加锁这种细粒度加锁的方法，这样设计只会锁当前实例，而不会锁其他实例
        Object instance = holder.get();
        
        //如果是实现类对象是空，则去构建该实现类的对象，并使用多线程的double-check方法去构建
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    //创建一个实现类的对象，根据扩展名从SPI配置文件中查找对应的扩展实现类
                    instance = createExtension(name, wrap);
                    //把创建出来的实现类对象，设置到holder里面
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }

    private Holder<Object> getOrCreateHolder(String name) {
        //根据name名称，先去获取Holder
        Holder<Object> holder = cachedInstances.get(name);
        if (holder == null) {
            //构建一个空的Holder，放入到缓存cachedInstances里
            cachedInstances.putIfAbsent(name, new Holder<>());
            holder = cachedInstances.get(name);
        }
        return holder;
    }
    ...
}
```

<br>

**12.Extension扩展实例的构建过程**

```
-> ExtensionLoader.getExtension()
-> ExtensionLoader.createExtension()
-> ExtensionLoader.getExtensionClasses()
-> ExtensionLoader.loadExtensionClasses()
-> ExtensionLoader.cacheDefaultExtensionName()
-> ExtensionLoader.loadDirectory()
-> ExtensionLoader.loadDirectoryInternal()
-> ExtensionLoader.loadResource()
-> ExtensionLoader.loadFromClass()
-> ExtensionLoader.createExtensionInstance()
-> InstantiationStrategy.instantiate()
-> ExtensionLoader.postProcessBeforeInitialization()
-> ExtensionLoader.injectExtension()
-> ExtensionLoader.postProcessAfterInitialization()
-> ExtensionLoader.initExtension()

public class ExtensionLoader<T> {
    ...
    //Dubbo源码里会大量使用"extensionDirector.getExtensionLoader(Class).getExtension()"这种代码来获取扩展实现类的实例
    //getExtension()方法中的name参数一般是从扩展点的SPI注解里提取出来的一个name
    public T getExtension(String name) {
    		T extension = getExtension(name, true);//wrap参数默认是true
    		...
    		return extension;
    }

    public T getExtension(String name, boolean wrap) {
        ...
        //如果是实现类对象是空，则去构建该实现类的对象，并使用多线程的double-check方法去构建
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                    //创建一个实现类的对象，根据扩展名从SPI配置文件中查找对应的扩展实现类
                    instance = createExtension(name, wrap);
                    //把创建出来的实现类对象，设置到holder里面
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
    
    private T createExtension(String name, boolean wrap) {
        //注意：各个扩展实现类都会有自己的Class类型和对应的name名称
        //getExtensionClasses()会获取cachedClasses缓存，下面这行代码会根据扩展实现类的名称从cachedClasses缓存中获取扩展实现类；
        //其中如果cachedClasses未初始化，则会扫描三个SPI目录获取相应的SPI配置文件，然后加载其中配置的扩展实现类，
        //最后将扩展名和扩展实现类的映射关系记录到cachedClasses缓存中，这部分逻辑在loadExtensionClasses()和loadDirectory()方法中；
        //也就是说会通过对指定的配置文件进行读取和解析，先拿到扩展实现类，然后通过扩展实现类的name名称，去获取到对应的class对象并进行缓存
        Class<?> clazz = getExtensionClasses().get(name);
        ...
        
        //根据扩展实现类从extensionInstances缓存中查找相应的实例；
        //如果查找失败，会通过反射创建扩展实现类的对象；
        T instance = (T) extensionInstances.get(clazz);
        if (instance == null) {
            //首先通过createExtensionInstance()使用反射创建扩展类对应的实例，然后放到extensionInstances缓存里
            extensionInstances.putIfAbsent(clazz, createExtensionInstance(clazz));
            instance = (T) extensionInstances.get(clazz);
            
            //下面会对该实例对象进行初始化前的处理
            instance = postProcessBeforeInitialization(instance, name);
              
            //自动装配扩展实现对象中的属性(即调用其setter)，这里涉及到ExtensionFactory以及自动装配的相关内容；
            //其实就是依赖注入，Dubbo SPI机制扮演了类似Spring容器的机制，它会托管扩展实现类的对象
            injectExtension(instance);
            
            //下面会对该实例对象进行初始化后的处理
            instance = postProcessAfterInitialization(instance, name);
        }
        ...
        
        //如果扩展实现类实现了Lifecycle接口，在initExtension()方法中会调用initialize()方法进行初始化；
        initExtension(instance);
        return instance;
    }
    
    private Map<String, Class<?>> getExtensionClasses() {
        //扩展实现类的缓存集合cachedClasses，通过double-check进行创建，一个扩展点接口可能会有很多实现类
        //cachedClasses是一个缓存集合：缓存了"实现类的名称 -> 实现类的Class"的映射关系，该缓存关系会被放到一个Holder里去
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                    //做具体的load，去寻找对应的ExtensionClasses
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
    
    private Map<String, Class<?>> loadExtensionClasses() throws InterruptedException {
        checkDestroyed();
        
        //缓存默认的扩展名称
        cacheDefaultExtensionName();
        
        //存放扩展实现类的集合，"name -> 实现类Class"的映射关系，各个扩展实现类都会有自己的Class类型和对应的name名称
        Map<String, Class<?>> extensionClasses = new HashMap<>();
        
        //遍历不同的loading策略，去loadDirectory()
        for (LoadingStrategy strategy : strategies) {
            //读取指定目录下的接口的配置文件，并解析配置文件读取出每个实现类的name名称和Class类型
            loadDirectory(extensionClasses, strategy, type.getName());
            //compatible with old ExtensionFactory
            if (this.type == ExtensionInjector.class) {
                loadDirectory(extensionClasses, strategy, ExtensionFactory.class.getName());
            }
        }
        return extensionClasses;
    }
    
    private void loadDirectoryInternal(Map<String, Class<?>> extensionClasses, LoadingStrategy loadingStrategy, String type) throws InterruptedException {
        //目录+type接口的名字，与Dubbo SPI配置的规则匹配上了
        String fileName = loadingStrategy.directory() + type;
        ...
        
        //使用了ClassLoader加载资源的方法，基于文件名拿出了资源集合
        Enumeration<java.net.URL> resources = ClassLoader.getSystemResources(fileName);
        
        //对资源进行遍历执行涉及IO操作的loadResource()方法
        while (resources.hasMoreElements()) {
            loadResource(extensionClasses, null, resources.nextElement(), 
                loadingStrategy.overridden(),
                loadingStrategy.includedPackages(),
                loadingStrategy.excludedPackages(),
                loadingStrategy.onlyExtensionClassLoaderPackages()
            );
        }
        ...
        
        //基于SPI规则去读取各种配置文件，拿到对应的实现类，进行资源(-> ExtensionClass)的加载
        Map<ClassLoader, Set<java.net.URL>> resources = ClassLoaderResourceLoader.loadResources(fileName, classLoadersToLoad);
        resources.forEach(((classLoader, urls) -> {
            loadFromClass(extensionClasses, loadingStrategy.overridden(), urls, classLoader,
                loadingStrategy.includedPackages(),
                loadingStrategy.excludedPackages(),
                loadingStrategy.onlyExtensionClassLoaderPackages()
            );
        }));
        ...
    }
    
    private Object createExtensionInstance(Class<?> type) throws ReflectiveOperationException {
        //下面会用ExtensionLoader在构造函数实例化的策略逻辑，来对这个扩展实现类进行实例化
        return instantiationStrategy.instantiate(type);
    }
    
    private void initExtension(T instance) {
        if (instance instanceof Lifecycle) {
            //扩展实现类一般可以实现一个Lifecycle接口
            Lifecycle lifecycle = (Lifecycle) instance;
            //这个方法是留给我们自己去实现的，里面一般是初始化中涉及的相关操作
            lifecycle.initialize();
        }
    }
    ...
}
```

<br>

**13.Dubbo自定义Bean容器的源码细节**

```
-> ScopeBeanFactory.registerBean(Class<T> bean)
-> ScopeBeanFactory.getOrRegisterBean()
-> ScopeBeanFactory.getBean()
-> ScopeBeanFactory.getBeanInternal()
-> 第一次注册getBean()必然返回null
-> ScopeBeanFactory.createAndRegisterBean()
-> InstantiationStrategy.instantiate()
-> ScopeBeanFactory.registerBean(String name, Object bean)
-> ScopeBeanFactory.initializeBean()
-> registeredBeanInfos.add(new BeanInfo(name, bean)); 完成注册

//ScopeBeanFactory是Dubbo框架内部实现的一个bean工厂，bean工厂管理的这些bean可以在Dubbo框架内部进行共享
public class ScopeBeanFactory {
    //ScopeBeanFactory也可以通过parent组成一颗树
    private final ScopeBeanFactory parent;

    //extension扩展实现类实例的获取组件，有了它就可以使用SPI机制了
    private ExtensionAccessor extensionAccessor;

    //extension扩展实现类的实例在初始化后的处理器
    private List<ExtensionPostProcessor> extensionPostProcessors;

    //每一个class都有一个AtomicInteger作为一个计数器
    private Map<Class, AtomicInteger> beanNameIdCounterMap = new ConcurrentHashMap<>();

    //注册过的bean实例信息
    private List<BeanInfo> registeredBeanInfos = new CopyOnWriteArrayList<>();

    //初始化策略逻辑，用于bean实例的初始化
    private InstantiationStrategy instantiationStrategy;
    ...

    //ScopeBeanFactory是bean管理的容器，需要进行注册
    public <T> T registerBean(Class<T> bean) throws ScopeBeanException {
        //name可以是null
        return this.getOrRegisterBean(null, bean);
    }

    //注册bean的时候，name可以自定义
    public <T> T registerBean(String name, Class<T> clazz) throws ScopeBeanException {
        return getOrRegisterBean(name, clazz);
    }

    public <T> T getOrRegisterBean(String name, Class<T> type) {
        //第一次注册，getBean()必然返回空
        T bean = getBean(name, type);
        if (bean == null) {
            // lock by type
            synchronized (type) {
                bean = getBean(name, type);
                if (bean == null) {
                    bean = createAndRegisterBean(name, type);
                }
            }
        }
        return bean;
    }
    
    public <T> T getBean(String name, Class<T> type) {
        T bean = getBeanInternal(name, type);
        if (bean == null && parent != null) {
            return parent.getBean(name, type);
        }
        return bean;
    }
    
    private <T> T createAndRegisterBean(String name, Class<T> clazz) {
        checkDestroyed();
        T instance = getBean(name, clazz);
        try {
            //直接用扩展实现类的class来初始化一个实例对象
            instance = instantiationStrategy.instantiate(clazz);
        } catch (Throwable e) {
            throw new ScopeBeanException("create bean instance failed, type=" + clazz.getName(), e);
        }
        registerBean(name, instance);
        return instance;
    }
    
    public void registerBean(String name, Object bean) {
        checkDestroyed();
        // avoid duplicated register same bean
        if (containsBean(name, bean)) {
            return;
        }
        Class<?> beanClass = bean.getClass();
        if (name == null) {
            name = beanClass.getName() + "#" + getNextId(beanClass);
        }
        initializeBean(name, bean);
        //完成注册
        registeredBeanInfos.add(new BeanInfo(name, bean));
    }
    ...
}
```

<br>

**14.ModuleDeployer组件源码细节**

回到ServiceConfig的export()方法：对比Dubbo 2.6.x和2.7.x源码，Dubbo 3.0的变动还是有点大的，比如这里使用了ModuleDeployer组件。

Dubbo服务实例内部会有很多的代码组件，通过ModuleDeployer便可以避免零零散散的去调用和初始化。ModuleDeployer会对服务实例的启动做很多的初始化准备工作，比如会对MetadataReport组件进行构建和初始化以及启动(建立跟zk的连接)。

ServiceConfig.export()方法里调用的getScopeModel()返回的是一个ModuleModel。

<img width="100%" height="100%" alt="image" src="https://github.com/user-attachments/assets/ac4f3fac-cc67-473e-bda7-7bca3ce06978" />

```
-> ServiceConfig.export()
-> ServiceConfig.getScopeModel().getDeployer().start()
-> DefaultModuleDeployer.start()
-> DefaultApplicationDeployer.initialize()
-> DefaultApplicationDeployer.startConfigCenter()
-> DefaultApplicationDeployer.startMetadataCenter()
-> DefaultModuleDeployer.startSync()

public class ServiceConfig<T> extends ServiceConfigBase<T> {
    ...
    public void export() {
        //对比Dubbo 2.6.x和2.7.x源码，Dubbo 3.0的变动还是有点大的，比如这里使用了ModuleDeployer组件
        //Dubbo服务实例内部会有很多的代码组件，通过ModuleDeployer便可以避免零零散散的去调用和初始化
        //1.ModuleDeployer会对服务实例的启动做很多的初始化准备工作
        //比如会对MetadataReport组件进行构建和初始化，以及启动(建立跟zk的连接)
        //getScopeModel()返回的是一个ModuleModel
        getScopeModel().getDeployer().start();
        ...
    }
    ...
}

public class DefaultModuleDeployer extends AbstractDeployer<ModuleModel> implements ModuleDeployer {
    //已经完成exported发布的服务实例集合
    private List<ServiceConfigBase<?>> exportedServices = new ArrayList<>();
    
    //下面这4个组件，本身都是跟model组件体系强关联的
    private ModuleModel moduleModel;
    private FrameworkExecutorRepository frameworkExecutorRepository;
    private ExecutorRepository executorRepository;
    private final ModuleConfigManager configManager;
   
    //父级ApplicationDeployer组件
    private ApplicationDeployer applicationDeployer;
    ...
    
    public void prepare() {
        //Module层级是Application层级的下层，Application层级是framework层级的下层
        applicationDeployer.initialize();
        this.initialize();
    }
    
    public Future start() throws IllegalStateException {
        //Module层级是Application层级的下层，Application层级是framework层级的下层
        //initialize，maybe deadlock applicationDeployer lock & moduleDeployer lock
        applicationDeployer.initialize();
        return startSync();
    }
    
    private synchronized Future startSync() throws IllegalStateException {
        ...
        onModuleStarting();
        initialize();
        exportServices();
        if (moduleModel != moduleModel.getApplicationModel().getInternalModule()) {
            applicationDeployer.prepareInternalModule();
        }
        referServices();
        ...
    }
    ...
}

public class DefaultApplicationDeployer extends AbstractDeployer<ApplicationModel> implements ApplicationDeployer {
    ...
    public void initialize() {
        ...
        synchronized (startLock) {
            //注册退出时需要进行资源销毁的ShutdownHook
            registerShutdownHook();
            
            //启动ConfigCenter配置中心
            startConfigCenter();
            
            //加载应用配置
            loadApplicationConfigs();
            
            //初始化ModuleDeployer
            initModuleDeployers();
            
            //@since 2.7.8
            //启动元数据中心
            startMetadataCenter();
            initialized = true;
        }
        ...
    }
    
    //启动配置中心，连接apollo、nacos、zk以实现DynamicConfiguration动态化配置(数据变化能收到通知)
    //会先看是否使用注册中心作为自己的配置中心
    private void startConfigCenter() {
        ...
        //如果有必要的话，直接使用注册中心作为自己的配置中心
        useRegistryAsConfigCenterIfNecessary();
        ...
        environment.setDynamicConfiguration(compositeDynamicConfiguration);
    }
    
    private void initModuleDeployers() {
        // make sure created default module
        applicationModel.getDefaultModule();
        // copy modules and initialize avoid ConcurrentModificationException if add new module
        List<ModuleModel> moduleModels = new ArrayList<>(applicationModel.getModuleModels());
        for (ModuleModel moduleModel : moduleModels) {
            moduleModel.getDeployer().initialize();
        }
    }
    
    //启动元数据中心，会先看是否要用注册中心当做元数据中心
    private void startMetadataCenter() {
        //第一步，先分析一下元数据中心，看是否要用注册中心当做元数据中心
        useRegistryAsMetadataCenterIfNecessary();
        ...
        
        //applicationModel拿到一个bean容器，从bean容器里拿到一个MetadataReportInstance，也就是元数据上报组件的实例
        MetadataReportInstance metadataReportInstance = applicationModel.getBeanFactory().getBean(MetadataReportInstance.class);
        ...
        
        //对于唯一的一个MetadataReport，在这里会进行初始化，它会把我们配置的metadataReport地址和config传进去进行init初始化
        //而所谓的metadataReport启动，其实就是根据我们的配置去拿到对应的factory，然后通过factory创建出对应的metadataReport
        metadataReportInstance.init(validMetadataReportConfigs);
    }
    ...
}
```
