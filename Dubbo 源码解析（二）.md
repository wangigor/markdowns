# Dubbo 源码解析（二）

> dubbo版本：2.7.4.1
>

## 服务导出

> 服务导出入口在ServiceBean对容器刷新完成事件ContextRefreshedEvent的事件响应方法。

```java
public void onApplicationEvent(ContextRefreshedEvent event) {
  	//已经注册或者不需要注册，不进入注册逻辑
  	//isExported()是一个当前ServiceBean的volatile属性，使用cas修改。
  	//isUnexported()是@Service注解的当前实现类是不是要注册export default true
    if (!isExported() && !isUnexported()) {
        if (logger.isInfoEnabled()) {
            logger.info("The service ready on spring started. service: " + getInterface());
        }
      	//服务导出
        export();
    }
}
```

```java
		public void export() {
      	//调用父类ServiceConfig的export()
        super.export();
        // 发布当前服务注册导出事件ServiceBeanExportedEvent
        publishExportEvent();
    }
```

```java
    //ServiceConfig
		public synchronized void export() {
      	//配置检查、URL装配
        checkAndUpdateSubConfigs();
				
      	//shouldExport是provider的，跟isUnexported不同
      	//<dubbo:provider export="false" />
      	//dubbo.privider.export=false
        if (!shouldExport()) {
            return;
        }
				//查看provider的延迟时间
        if (shouldDelay()) {
          	//通过时间调度器延时执行
          	//Executors.newSingleThreadScheduledExecutor(new NamedThreadFactory("DubboServiceDelayExporter", true));
            DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
        } else {
          	//直接暴露
            doExport();
        }
    }
```

### 配置检查、URL装配

```java
  //ServiceConfig  
	public void checkAndUpdateSubConfigs() {
        //使用显式声明的配置对application、module、registries、monitor、protocols、configCenter进行设置
        //这里有一个单例ConfigManager配置管理器的配置检查
      	completeCompoundConfigs();
        //启动动态配置中心dubbo.config-center优先启动
      	//这里会在动态配置中心详细配置 TODO
        startConfigCenter();
      	//检查provider，如果没有就新建一个，从环境变量赋值【providerConfig.refresh()】
      	//环境变量有SystemConfig、EnvironmentConfig、AppExternalConfig、ExternalConfig、PropertiesConfig
      	//TODO 后续再深入
        checkDefault();
      	//检查protocol
        checkProtocol();
      	//检查application
        checkApplication();
        // if protocol is not injvm checkRegistry
        if (!isOnlyInJvm()) {
          	//检查registry
            checkRegistry();
        }
      	//刷新
        this.refresh();
      	//检查MetadataReport
        checkMetadataReport();

      	//接口检查
        if (StringUtils.isEmpty(interfaceName)) {
            throw new IllegalStateException("<dubbo:service interface=\"\" /> interface not allow null!");
        }
				
    		//泛化服务类型
        if (ref instanceof GenericService) {
            interfaceClass = GenericService.class;
            if (StringUtils.isEmpty(generic)) {
                generic = Boolean.TRUE.toString();//标记为泛化
            }
        } else {
        		//非泛化【通用类型】
          	//加载interfaceClass
            try {
              	//使用tccl当前线程上下文类加载器
                interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                        .getContextClassLoader());
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
          	//接口及方法检查
            checkInterfaceAndMethods(interfaceClass, methods);
          	//ref检查
            checkRef();
            generic = Boolean.FALSE.toString();//标记为非泛化
        }
    		//获取local的本地存根。使用TCCL加载存根类
        if (local != null) {
            if (Boolean.TRUE.toString().equals(local)) {
                local = interfaceName + "Local";
            }
            Class<?> localClass;
            try {
                localClass = ClassUtils.forNameWithThreadContextClassLoader(local);
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            if (!interfaceClass.isAssignableFrom(localClass)) {
                throw new IllegalStateException("The local implementation class " + localClass.getName() + " not implement interface " + interfaceName);
            }
        }
    		//获取stub的本地存根。使用TCCL加载存根类
        if (stub != null) {
            if (Boolean.TRUE.toString().equals(stub)) {
                stub = interfaceName + "Stub";
            }
            Class<?> stubClass;
            try {
                stubClass = ClassUtils.forNameWithThreadContextClassLoader(stub);
            } catch (ClassNotFoundException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            if (!interfaceClass.isAssignableFrom(stubClass)) {
                throw new IllegalStateException("The stub implementation class " + stubClass.getName() + " not implement interface " + interfaceName);
            }
        }
    		//stub 和 local 的检查
        checkStubAndLocal(interfaceClass);
    		//本地伪装检查
        checkMock(interfaceClass);
    }
```

#### completeCompoundConfigs

todo

#### startConfigCenter

todo

#### checkDefault

todo

#### checkProtocol

todo

#### checkApplication

todo

#### checkRegistry

> 如果没有配置config-center。且。注册中心使用zookeeper。
>
> 在这里会进行注册中心连接。startConfigCenter

todo

#### checkMetadataReport

todo

#### stub & local

todo

#### checkMock

todo

### doExport服务暴露

```java
  //这个synchronized没看懂。
	protected synchronized void doExport() {
      	//是否暴露检查
        if (unexported) {
            throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
        }
        if (exported) {
            return;
        }
      	//直接修改为已暴露
        exported = true;

        if (StringUtils.isEmpty(path)) {
            path = interfaceName;
        }
      	//服务暴露
        doExportUrls();
    }
```

> 提供了多协议多注册中心的支持。

```java
    //ServiceConfig
		private void doExportUrls() {
      	//获取所有注册中心的url【排除N/A】
        List<URL> registryURLs = loadRegistries(true);
      	//协议遍历
        for (ProtocolConfig protocolConfig : protocols) {
          	//这里使用contextPath尝试对接口全限定名path进行封装
          	//TODO 暂时可以理解为rest的请求前缀
          	//添加group、version属性
            String pathKey = URL.buildKey(getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), group, version);
            //组装path,ref,interfaceClass成ProviderModel，放入ApplicationModel的静态map中
          	ProviderModel providerModel = new ProviderModel(pathKey, ref, interfaceClass);
            ApplicationModel.initProviderModel(pathKey, providerModel);
          	//单个协议暴露
            doExportUrlsFor1Protocol(protocolConfig, registryURLs);
        }
    }
```

#### loadRegistries

```java
    //AbstractInterfaceConfig
		protected List<URL> loadRegistries(boolean provider) {
        // check && override if necessary
        List<URL> registryList = new ArrayList<URL>();
        if (CollectionUtils.isNotEmpty(registries)) {
          	//遍历注册中心
            for (RegistryConfig config : registries) {
              	//注册中心地址
                String address = config.getAddress();
                if (StringUtils.isEmpty(address)) {
                  	//address为空，设值0.0.0.0
                    address = ANYHOST_VALUE;
                }
              	//排除注册中心地址为N/A的
                if (!RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
                    Map<String, String> map = new HashMap<String, String>();
                  	//application属性添加进map中
                    appendParameters(map, application);
                  	//registry属性添加进map中
                    appendParameters(map, config);
                  	//添加path属性
                    map.put(PATH_KEY, RegistryService.class.getName());
                  	//添加运行时pid、dubbo版本、时间戳、release属性
                    appendRuntimeParameters(map);
                  	//协议默认dubbo
                    if (!map.containsKey(PROTOCOL_KEY)) {
                        map.put(PROTOCOL_KEY, DUBBO_PROTOCOL);
                    }
                  	//解析地址，得到url列表。【可能包含多个注册中心ip】
                    List<URL> urls = UrlUtils.parseURLs(address, map);

                    for (URL url : urls) {
                        url = URLBuilder.from(url)
                                .addParameter(REGISTRY_KEY, url.getProtocol())
                                .setProtocol(REGISTRY_PROTOCOL)
                                .build();
                        if ((provider && url.getParameter(REGISTER_KEY, true))
                                || (!provider && url.getParameter(SUBSCRIBE_KEY, true))) {
                            registryList.add(url);
                        }
                    }
                }
            }
        }
        return registryList;
    }
```

##### appendParameters

```java
    //AbstractConfig
		protected static void appendParameters(Map<String, String> parameters, Object config, String prefix) {
        if (config == null) {
            return;
        }
      	//source config类方法
        Method[] methods = config.getClass().getMethods();
        for (Method method : methods) {
            try {
                String name = method.getName();
              	//getter方法
                if (MethodUtils.isGetter(method)) {
                  	//@Parameter注解标记了excluded，当前方法返回值不添加入map
                  	//当前方法返回不是Object 且 被@Parameter注解。才添加入map
                    Parameter parameter = method.getAnnotation(Parameter.class);
                    if (method.getReturnType() == Object.class || parameter != null && parameter.excluded()) {
                        continue;
                    }
                  	//定义map的key值
                  	//指定了key，就使用指定key
                    String key;
                    if (parameter != null && parameter.key().length() > 0) {
                        key = parameter.key();
                    } else {
                      	//没有指定key，使用方法名，去get前缀，首字母小写，作为key
                      	//e,g. getProtocol -> protocol
                        key = calculatePropertyFromGetter(name);
                    }
                  	//反射调用获取value
                    Object value = method.invoke(config);
                    String str = String.valueOf(value).trim();
                    if (value != null && str.length() > 0) {
                      	//@Parameter标记了escaped。进行url encode。
                        if (parameter != null && parameter.escaped()) {
                            str = URL.encode(str);
                        }
                      	//value 附加前缀
                        if (parameter != null && parameter.append()) {
                            String pre = parameters.get(DEFAULT_KEY + "." + key);
                            if (pre != null && pre.length() > 0) {
                                str = pre + "," + str;
                            }
                            pre = parameters.get(key);
                            if (pre != null && pre.length() > 0) {
                                str = pre + "," + str;
                            }
                        }
                      	//key 附加前缀
                        if (prefix != null && prefix.length() > 0) {
                            key = prefix + "." + key;
                        }
                      	//放入map
                        parameters.put(key, str);
                    } else if (parameter != null && parameter.required()) {
                      	//非空判断
                        throw new IllegalStateException(config.getClass().getSimpleName() + "." + key + " == null");
                    }
                } else if (isParametersGetter(method)) {
                  	//方法名是getParameters返回map的。把返回的map添加进map。
                    Map<String, String> map = (Map<String, String>) method.invoke(config, new Object[0]);
                    parameters.putAll(convert(map, prefix));
                }
            } catch (Exception e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
        }
    }
```

#### 单协议暴露 doExportUrlsFor1Protocol

> 这里代码太长。按功能分成两部分。

##### 参数设置

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    //当前协议没有名称 默认dubbo
  	String name = protocolConfig.getName();
    if (StringUtils.isEmpty(name)) {
        name = DUBBO;
    }
		
  	//参数列表map
    Map<String, String> map = new HashMap<String, String>();
  	//设置 side -> provider
    map.put(SIDE_KEY, PROVIDER_SIDE);
		//添加运行时pid、dubbo版本、时间戳、release属性
  	//"release" -> "2.7.4.1"
		//"dubbo" -> "2.0.2"
		//"pid" -> "15409"
		//"timestamp" -> "1590563560629"
    appendRuntimeParameters(map);
  	//添加metrics监控中心参数MetricsConfig
    appendParameters(map, metrics);
  	//添加application参数ApplicationConfig
  	//"application" -> "system-service-dubbo"
		//"qos.enable" -> "false"
    appendParameters(map, application);
  	//添加module参数ModuleConfig
    appendParameters(map, module);
    //添加provider参数ProviderConfig
  	//"dynamic" -> "true"
		//"deprecated" -> "false"
    appendParameters(map, provider);
  	//添加protocol参数ProtocolConfig
    appendParameters(map, protocolConfig);
  	//添加service参数ServiceConfig
  	//"interface" -> "com.uteam.zen.api.IAuthorityService"
		//"bean.name" -> "ServiceBean:com.uteam.zen.api.IAuthorityService"
    appendParameters(map, this);
  	//处理<dubbo:method>节点
    if (CollectionUtils.isNotEmpty(methods)) {
      	//遍历 TODO 我这里没有。往后放放。
        for (MethodConfig method : methods) {
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if (Boolean.FALSE.toString().equals(retryValue)) {
                    map.put(method.getName() + ".retries", "0");
                }
            }
            List<ArgumentConfig> arguments = method.getArguments();
            if (CollectionUtils.isNotEmpty(arguments)) {
                for (ArgumentConfig argument : arguments) {
                    // convert argument type
                    if (argument.getType() != null && argument.getType().length() > 0) {
                        Method[] methods = interfaceClass.getMethods();
                        // visit all methods
                        if (methods != null && methods.length > 0) {
                            for (int i = 0; i < methods.length; i++) {
                                String methodName = methods[i].getName();
                                // target the method, and get its signature
                                if (methodName.equals(method.getName())) {
                                    Class<?>[] argtypes = methods[i].getParameterTypes();
                                    // one callback in the method
                                    if (argument.getIndex() != -1) {
                                        if (argtypes[argument.getIndex()].getName().equals(argument.getType())) {
                                            appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                                        } else {
                                            throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                        }
                                    } else {
                                        // multiple callbacks in the method
                                        for (int j = 0; j < argtypes.length; j++) {
                                            Class<?> argclazz = argtypes[j];
                                            if (argclazz.getName().equals(argument.getType())) {
                                                appendParameters(map, argument, method.getName() + "." + j);
                                                if (argument.getIndex() != -1 && argument.getIndex() != j) {
                                                    throw new IllegalArgumentException("Argument config error : the index attribute and type attribute not match :index :" + argument.getIndex() + ", type:" + argument.getType());
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    } else if (argument.getIndex() != -1) {
                        appendParameters(map, argument, method.getName() + "." + argument.getIndex());
                    } else {
                        throw new IllegalArgumentException("Argument config must set index or type attribute.eg: <dubbo:argument index='0' .../> or <dubbo:argument type=xxx .../>");
                    }

                }
            }
        } // end of methods for
    }
		//泛化添加两个值
    if (ProtocolUtils.isGeneric(generic)) {
        map.put(GENERIC_KEY, generic);//generic -> true
        map.put(METHODS_KEY, ANY_VALUE);//methods -> *
    } else {
      	//没有指定version，就是null
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
          	//revision -> 1.0.0
            map.put(REVISION_KEY, revision);
        }
				
      	//这里生成一个接口的包装类。这里只获取methodNames
      	//Wrapper.getWrapper在Invoker生成里有。
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
          	//methods -> *
            logger.warn("No method found in service interface " + interfaceClass.getName());
            map.put(METHODS_KEY, ANY_VALUE);
        } else {
          	//所有方法名join，逗号分隔
          	//methods -> getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority
            map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
  	//token放入map中
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(TOKEN_KEY, token);
        }
    }
  //{export service代码}
}
```

##### export service

```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {				
				//{参数设置代码。组装map}
  			// export service
  
  			//获取本地服务的host和port
        String host = this.findConfigedHosts(protocolConfig, registryURLs, map);
        Integer port = this.findConfigedPorts(protocolConfig, name, map);
  
  			//组装url
  			//e,g.
  			//dubbo://192.168.2.155:20880/com.uteam.zen.api.IAuthorityService?
  			//anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService
  			//&bind.ip=192.168.2.155&bind.port=20880&deprecated=false&dubbo=2.0.2
  			//&dynamic=true&generic=false&interface=com.uteam.zen.api.IAuthorityService
  			//&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority
  			//&pid=15409&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590563560629
  			//*注意 这里的url是org.apache.dubbo.common.URL
        URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);
				
  			//这里是url的配置扩展类ConfiguratorFactory
  			//官方提供的是override、absent两种实现。
  			//进行url封装
        if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .hasExtension(url.getProtocol())) {
          	//获取ConfiguratorFactory扩展类，实例化对应Configurator，然后配置url。
            url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                    .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
        }

        String scope = url.getParameter(SCOPE_KEY);
  
  			//只有配置了scope为none，不做暴露操作
  			//通常这里scope是null
        if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

          	//暴露到本地 TODO 这个后面再看
          	//scope是null的时候。也会暴露到本地。
            if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
                exportLocal(url);
            }
          	//暴露到远程
            if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
                if (CollectionUtils.isNotEmpty(registryURLs)) {
                  	//遍历注册中心url
                    for (URL registryURL : registryURLs) {
                      	//e,g. registry://localhost:2181/org.apache.dubbo.registry.RegistryService?application=system-service-dubbo&check=false&dubbo=2.0.2&pid=15409&qos.enable=false&registry=zookeeper&release=2.7.4.1&timestamp=1590563503671
                        //协议是injvm,过滤掉
                        if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                            continue;
                        }
                      	//添加dynamic参数，通常为null TODO
                        url = url.addParameterIfAbsent(DYNAMIC_KEY, registryURL.getParameter(DYNAMIC_KEY));
                        //如果配置了监视器monitor
                      	//url添加monitor参数
                      	URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(MONITOR_KEY, monitorUrl.toFullString());
                        }
                      	//{日志输出。忽略。}
												
                      	//自定义代理。添加参数
                      	//TODO 后续试一试。暂时是null。
                        String proxy = url.getParameter(PROXY_KEY);
                        if (StringUtils.isNotEmpty(proxy)) {
                            registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                        }
												
                      	//为服务提供类ref 生成Invoker
                        Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                        //DelegateProviderMetaDataInvoker 持有invoker和serviceConfig
                      	DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);
												
                      	//*暴露服务 并生成exporter
                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
                } else {
                  	//没有配置注册中心。仅暴露
                  
                    //{日志。忽略。}
                  
										//一模一样。
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    Exporter<?> exporter = protocol.export(wrapperInvoker);
                    exporters.add(exporter);
                }
                /**
                 * @since 2.7.0
                 * ServiceData Store
                 */
                MetadataReportService metadataReportService = null;
                if ((metadataReportService = getMetadataReportService()) != null) {
                    metadataReportService.publishProvider(url);
                }
            }
        }
        this.urls.add(url);
    }
```

###### 获取本地服务host

TODO

###### 获取本地服务port

TODO

###### ==Invoker生成==

> 《Dubbo官方文档》Invoker 是实体域，它是 Dubbo 的核心模型，其它模型都向它靠扰，或转换成它，它代表一个可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。
>
> Invoker 是由 ProxyFactory 创建而来，Dubbo 默认的 ProxyFactory 实现类是 JavassistProxyFactory。
>
> ```java
> PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
> ```
> JDKProxyFactory使用的是InvokerInvocationHandler作为代理类。

```java
    //proxy是ServiceBean的ref。type是ServiceBean的接口类。url是registryUrl
		public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        //为代理类创建Wrapper
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        //创建匿名Invoker对象。实现doInvoke方法。
      	return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
              	//将请求转发给包装类。包装类会真实请求目标proxy方法。
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
```

==Wrapper.getWrapper==创建目标对象包裹类。

- 获取Wrapper类上静态缓存。没有则创建。

```java
    public static Wrapper getWrapper(Class<?> c) {
      	//不包装动态类。【ClassGenerator.DC的子类】
      	//获取动态的目标类进行包装
        while (ClassGenerator.isDynamicClass(c)) // can not wrapper on dynamic class.
        {
            c = c.getSuperclass();
        }
				//静态初始化的Object类的包装类
        if (c == Object.class) {
            return OBJECT_WRAPPER;
        }
				
      	//先去缓存中获取
        Wrapper ret = WRAPPER_MAP.get(c);
        if (ret == null) {
          	//创建Wrapper
            ret = makeWrapper(c);
          	//放入缓存
            WRAPPER_MAP.put(c, ret);
        }
        return ret;
    }
```

- 创建Wrapper

```java
    private static Wrapper makeWrapper(Class<?> c) {
      	//c是基本数据类型。抛异常。
        if (c.isPrimitive()) {
            throw new IllegalArgumentException("Can not create wrapper for primitive type: " + c);
        }
				//类的全限定名
        String name = c.getName();
      	//类加载器 从TCCL开始尝试获取
        ClassLoader cl = ClassUtils.getClassLoader(c);
				
      	//c1 存储setPropertyValue的代码块源码
        StringBuilder c1 = new StringBuilder("public void setPropertyValue(Object o, String n, Object v){ ");
        //c2 存储getPropertyValue的代码块源码
      	StringBuilder c2 = new StringBuilder("public Object getPropertyValue(Object o, String n){ ");
        //c3 存储invokeMethod的代码块源码
      	StringBuilder c3 = new StringBuilder("public Object invokeMethod(Object o, String n, Class[] p, Object[] v) throws " + InvocationTargetException.class.getName() + "{ ");

      	//声明$1对象类型为c类型,并进行异常捕获。
        c1.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
        c2.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
        c3.append(name).append(" w; try{ w = ((").append(name).append(")$1); }catch(Throwable e){ throw new IllegalArgumentException(e); }");
				
      	//pts 存储成员变量名称和类型
        Map<String, Class<?>> pts = new HashMap<>(); // <property name, property types>
      	//ms 存储方法签名和方法实例
        Map<String, Method> ms = new LinkedHashMap<>(); // <method desc, Method instance>
      	//mns 方法名列表
        List<String> mns = new ArrayList<>(); // method names.
      	//dmns 声明在当前类中的方法名列表
        List<String> dmns = new ArrayList<>(); // declaring method names.

        // -----------------------分隔1--------------------------
      	//遍历所有public成员属性
        for (Field f : c.getFields()) {
            String fn = f.getName();
            Class<?> ft = f.getType();
          	//忽略static或transient修饰的字段
            if (Modifier.isStatic(f.getModifiers()) || Modifier.isTransient(f.getModifiers())) {
                continue;
            }
						
          	//c1增加代码块：通过属性名称给属性赋值
          	//if( $2.equals("name") ){ w.name=(java.lang.String)$3; return; }
            c1.append(" if( $2.equals(\"").append(fn).append("\") ){ w.").append(fn).append("=").append(arg(ft, "$3")).append("; return; }");
            
          	//c2增加代码块：通过属性名称获取属性值
          	//if( $2.equals("name") ){ return ($w)w.name; }
          	c2.append(" if( $2.equals(\"").append(fn).append("\") ){ return ($w)w.").append(fn).append("; }");
            pts.put(fn, ft);//存储 字段名 -> 字段类型 到pts中。
        }
				// -----------------------分隔2--------------------------
      	//获取c类中的所有public方法，包括父类
        Method[] methods = c.getMethods();
        // 是否有本类方法【非Object.class的方法】，这些方法需要*异常捕获*生成InvocationTargetException异常。
        boolean hasMethod = hasMethods(methods);
        if (hasMethod) {
            c3.append(" try{");
            for (Method m : methods) {
                //忽略Object.class声明的方法
                if (m.getDeclaringClass() == Object.class) {
                    continue;
                }
								
              	//try{ if( "addAuthority".equals( $2 )  方法名判断
                String mn = m.getName();
                c3.append(" if( \"").append(mn).append("\".equals( $2 ) ");
              	//&&  $3.length == 1 方法请求参数长度判断
                int len = m.getParameterTypes().length;
                c3.append(" && ").append(" $3.length == ").append(len);
								
              	//override 方法是否被重载【方法名相同，但方法签名不同（方法实例不同）】
                boolean override = false;
                for (Method m2 : methods) {
                    if (m != m2 && m.getName().equals(m2.getName())) {
                        override = true;
                        break;
                    }
                }
              	//有重载方法。增加类型判断。
              	//官网给的样例是：
              	//    1. void sayHello(Integer, String)
        				//    2. void sayHello(Integer, Integer)
              	// 方法名相同，参数列表长度也相同，因此不能仅通过这两项判断两个方法是否相等。
                if (override) {
                    if (len > 0) {
                        for (int l = 0; l < len; l++) {
                          	//生成参数类型检测代码。
                          	// && $3[0].getName().equals("java.lang.Integer") 
                    				// && $3[1].getName().equals("java.lang.String")
                            c3.append(" && ").append(" $3[").append(l).append("].getName().equals(\"")
                                    .append(m.getParameterTypes()[l].getName()).append("\")");
                        }
                    }
                }
								//if判断结束
              	// if ("sayHello".equals($2) 
        				//     && $3.length == 2
        				//     && $3[0].getName().equals("java.lang.Integer") 
        				//     && $3[1].getName().equals("java.lang.String")) {
                c3.append(" ) { ");
								
              	//根据方法返回类型，判断是否需要return；
                if (m.getReturnType() == Void.TYPE) {
                  	//生成void方法调用代码
                  	// w.sayHello((java.lang.Integer)$4[0], (java.lang.String)$4[1]); return null;
                    c3.append(" w.").append(mn).append('(').append(args(m.getParameterTypes(), "$4")).append(");").append(" return null;");
                } else {
                  	//生成需要return的方法调用代码
                  	// return w.sayHello((java.lang.Integer)$4[0], (java.lang.String)$4[1]);
                    c3.append(" return ($w)w.").append(mn).append('(').append(args(m.getParameterTypes(), "$4")).append(");");
                }
								//方法调用完成
              	//官网e,g.
              	// if ("sayHello".equals($2) 
        				//     && $3.length == 2
        				//     && $3[0].getName().equals("java.lang.Integer") 
        				//     && $3[1].getName().equals("java.lang.String")) {
        				//
        				//     w.sayHello((java.lang.Integer)$4[0], (java.lang.String)$4[1]); 
        				//     return null;
        				// }
                c3.append(" }");

                mns.add(mn);//方法名放入mns中
                if (m.getDeclaringClass() == c) {
                    dmns.add(mn);//当前类声明的方法，放入dmns中。
                }
                ms.put(ReflectUtils.getDesc(m), m);//方法完整的方法签名和方法实例，放入ms中。
            }
          	//异常捕获代码块
            c3.append(" } catch(Throwable e) { ");
            c3.append("     throw new java.lang.reflect.InvocationTargetException(e); ");
            c3.append(" }");
        }
				
      	//添加NoSuchMethodException异常捕获块
        c3.append(" throw new " + NoSuchMethodException.class.getName() + "(\"Not found method \\\"\"+$2+\"\\\" in class " + c.getName() + ".\"); }");

        // -----------------------分隔3--------------------------
      	//get/set/is/can/has开头的方法
        Matcher matcher;
        for (Map.Entry<String, Method> entry : ms.entrySet()) {
            String md = entry.getKey();
            Method method = entry.getValue();
          	//属性取值 使用对应的get方法【c2代码】
            if ((matcher = ReflectUtils.GETTER_METHOD_DESC_PATTERN.matcher(md)).matches()) {
                String pn = propertyName(matcher.group(1));
              	//生成属性判断
              	// if( $2.equals("name") ) { return ($w).w.getName(); }
                c2.append(" if( $2.equals(\"").append(pn).append("\") ){ return ($w)w.").append(method.getName()).append("(); }");
                pts.put(pn, method.getReturnType());
            //属性取值 使用对应的is/has/can方法【c2代码】
            } else if ((matcher = ReflectUtils.IS_HAS_CAN_METHOD_DESC_PATTERN.matcher(md)).matches()) {
                String pn = propertyName(matcher.group(1));
              	// if( $2.equals("dream") ) { return ($w).w.hasDream(); }
                c2.append(" if( $2.equals(\"").append(pn).append("\") ){ return ($w)w.").append(method.getName()).append("(); }");
                pts.put(pn, method.getReturnType());
            //属性赋值 使用对应的set方法【c1代码】
            } else if ((matcher = ReflectUtils.SETTER_METHOD_DESC_PATTERN.matcher(md)).matches()) {
                Class<?> pt = method.getParameterTypes()[0];
                String pn = propertyName(matcher.group(1));
              	// if( $2.equals("name") ) { w.setName((java.lang.String)$3); return; }
                c1.append(" if( $2.equals(\"").append(pn).append("\") ){ w.").append(method.getName()).append("(").append(arg(pt, "$3")).append("); return; }");
                pts.put(pn, pt);
            }
        }
      	//c1/c2都添加属性获取失败异常捕获NoSuchPropertyException
        c1.append(" throw new " + NoSuchPropertyException.class.getName() + "(\"Not found property \\\"\"+$2+\"\\\" field or setter method in class " + c.getName() + ".\"); }");
        c2.append(" throw new " + NoSuchPropertyException.class.getName() + "(\"Not found property \\\"\"+$2+\"\\\" field or setter method in class " + c.getName() + ".\"); }");

        // -----------------------分隔4--------------------------
      	//id是一个Wrapper类上的AtomicLong，记录序号。
        long id = WRAPPER_CLASS_COUNTER.getAndIncrement();
      	//使用Dubbo的ClassGenerator类生成器
      	//ClassGenerator的toClass使用javassist的CtClass创建class
        ClassGenerator cc = ClassGenerator.newInstance(cl);
      	//设置类名org.apache.dubbo.common.bytecode.Wrapper1
        cc.setClassName((Modifier.isPublic(c.getModifiers()) ? Wrapper.class.getName() : c.getName() + "$sw") + id);
        //设置父类
      	cc.setSuperClass(Wrapper.class);
				
      	//添加默认构造器
        cc.addDefaultConstructor();
      	//添加字段
        cc.addField("public static String[] pns;"); // property name array.
        cc.addField("public static " + Map.class.getName() + " pts;"); // property type map.
        cc.addField("public static String[] mns;"); // all method name array.
        cc.addField("public static String[] dmns;"); // declared method name array.
        for (int i = 0, len = ms.size(); i < len; i++) {
            cc.addField("public static Class[] mts" + i + ";");
        }

      	//添加方法
        cc.addMethod("public String[] getPropertyNames(){ return pns; }");
        cc.addMethod("public boolean hasProperty(String n){ return pts.containsKey($1); }");
        cc.addMethod("public Class getPropertyType(String n){ return (Class)pts.get($1); }");
        cc.addMethod("public String[] getMethodNames(){ return mns; }");
        cc.addMethod("public String[] getDeclaredMethodNames(){ return dmns; }");
        cc.addMethod(c1.toString());
        cc.addMethod(c2.toString());
        cc.addMethod(c3.toString());
      	
      	// -----------------------分隔5--------------------------

        try {
          	//生成class   org.apache.dubbo.common.bytecode.Wrapper1
            Class<?> wc = cc.toClass();
            // 初始化静态变量
            wc.getField("pts").set(null, pts);
            wc.getField("pns").set(null, pts.keySet().toArray(new String[0]));
            wc.getField("mns").set(null, mns.toArray(new String[0]));
            wc.getField("dmns").set(null, dmns.toArray(new String[0]));
            int ix = 0;
            for (Method m : ms.values()) {
                wc.getField("mts" + ix++).set(null, m.getParameterTypes());
            }
          	//返回class实例
            return (Wrapper) wc.newInstance();
        } catch (RuntimeException e) {
            throw e;
        } catch (Throwable e) {
            throw new RuntimeException(e.getMessage(), e);
        } finally {
            cc.release();
            ms.clear();
            mns.clear();
            dmns.clear();
        }
    }
```

###### 本地服务暴露exportLocal



```java
    //总是暴露到本地
    private void exportLocal(URL url) {
      	//url ---> dubbo://192.168.59.1:20880/com.uteam.zen.api.IAuthorityService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService&bind.ip=192.168.59.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IAuthorityService&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority&pid=27411&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590630972860
        //local -> injvm://127.0.0.1/com.uteam.zen.api.IAuthorityService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService&bind.ip=192.168.59.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IAuthorityService&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority&pid=27411&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590630972860
      	URL local = URLBuilder.from(url)
                .setProtocol(LOCAL_PROTOCOL) //设置协议头 injvm
                .setHost(LOCALHOST_VALUE)	//设置本地host
                .setPort(0)	//取消端口
                .build();
      	//protocol是javassist生成的protocol$Adapter
      	//使用InjvmProtocol进行服务暴露
        Exporter<?> exporter = protocol.export(
          			//依旧是对ref进行Invoker封装
                PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local));
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
    }
```

```java
		@Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
      	//创建InjvmExporter返回
        return new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
    }
```



###### ==服务暴露到远程==

> 为了保持连贯性。这里还是把前置条件再重复一遍。
>
> ***
>
> 举例：registryUrl是
>
> ==registry==://localhost:2181/org.apache.dubbo.registry.RegistryService?application=system-service-dubbo&check=false&dubbo=2.0.2&pid=30477&qos.enable=false&==registry=zookeeper==&release=2.7.4.1&timestamp=1590651015149
>
> 通过arthas反编译Protocol$Adaptive【或者在控制台可以查看Protocol$Adaptive源码】的export方法
>
> ```java
> public Exporter export(Invoker invoker) throws RpcException {
>         String string;
>         if (invoker == null) {
>             throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
>         }
>         if (invoker.getUrl() == null) {
>             throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
>         }
>         URL uRL = invoker.getUrl();
>         String string2 = string = uRL.getProtocol() == null ? "dubbo" : uRL.getProtocol();
>         if (string == null) {
>             throw new IllegalStateException(new StringBuffer().append("Failed to get extension (org.apache.dubbo.rpc.Protocol) name from url (").append(uRL.toString()).append(") use keys([protocol])").toString());
>         }
>         Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getExtension(string);
>         return protocol.export(invoker);
>     }
> ```
>
> 使用自适应扩展，使用registryUrl的protocol【registry】做为extName。
>
> 注意：Protocol接口有三个Wrapper类
>
> ```properties
> #过滤包裹
> filter=org.apache.dubbo.rpc.protocol.ProtocolFilterWrapper 
> #监听包裹
> listener=org.apache.dubbo.rpc.protocol.ProtocolListenerWrapper
> #qos【服务质量】包裹
> qos=org.apache.dubbo.qos.protocol.QosProtocolWrapper
> ```
>
> 创建Protocol子类实例的时候，会对实例进行这三层包裹。
>
> TODO 这三层先不看。
>
> ***
>
> 入口是RegistryProtocol的export方法。

```java
    //RegistryProtocol
		@Override
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
      	// 注册中心地址
      	// zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService?application=system-service-dubbo&check=false&dubbo=2.0.2&export=dubbo%3A%2F%2F192.168.2.155%3A20880%2Fcom.uteam.zen.api.IAuthorityService%3Fanyhost%3Dtrue%26application%3Dsystem-service-dubbo%26bean.name%3DServiceBean%3Acom.uteam.zen.api.IAuthorityService%26bind.ip%3D192.168.2.155%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dcom.uteam.zen.api.IAuthorityService%26methods%3DgetAuthorityCodesByUserId%2CupdateAuthority%2CloadAuthority%2CdelAuthority%2CaddAuthority%2CqueryAuthority%26pid%3D30584%26qos.enable%3Dfalse%26release%3D2.7.4.1%26side%3Dprovider%26timestamp%3D1590651900177&pid=30584&qos.enable=false&release=2.7.4.1&timestamp=1590651899427
        URL registryUrl = getRegistryUrl(originInvoker);
        // 服务提供者url
      	// dubbo://192.168.2.155:20880/com.uteam.zen.api.IAuthorityService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService&bind.ip=192.168.2.155&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IAuthorityService&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority&pid=30584&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590651900177
        URL providerUrl = getProviderUrl(originInvoker);

        // 订阅URL TODO 是啥
      	// provider://192.168.2.155:20880/com.uteam.zen.api.IAuthorityService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService&bind.ip=192.168.2.155&bind.port=20880&category=configurators&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IAuthorityService&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority&pid=30584&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590651900177
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
      	//创建监听器 OverrideListener TODO
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
				//这里会向注册中心注册订阅url TODO
        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
      
        //***服务导出***
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // 根据注册中心地址的protocol获取Registry实例【如ZookeeperRegistry】
        final Registry registry = getRegistry(originInvoker);
        final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
      	//生成provider的调用包装器
        ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);
        //是否需要注册【延迟注册】
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) {
          	//***服务注册***
            register(registryUrl, registeredProviderUrl);
            providerInvokerWrapper.setReg(true);
        }

        // Deprecated! Subscribe to override rules in 2.6.x or before.
      	//订阅override信息 TODO 
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);
        //返回一个DestroyableExporter
        return new DestroyableExporter<>(exporter);
    }
```

- **服务导出==doLocalExport==(originInvoker, providerUrl);**

```java
    private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, URL providerUrl) {
        //缓存key【也就是提供者url】
      	//dubbo://192.168.2.155:20880/com.uteam.zen.api.IAuthorityService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService&bind.ip=192.168.2.155&bind.port=20880&deprecated=false&dubbo=2.0.2&generic=false&interface=com.uteam.zen.api.IAuthorityService&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority&pid=30584&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590651900177
      	String key = getCacheKey(originInvoker);

      	//访问缓存
      	// bounds是providerurl <--> exporter的ConcurrentHashMap【RegistryProtocol是单例的，所以缓存也是单例的。不需要static。】
        return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
          	//新建
            Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
          	//调用DubboProtocol进行导出
            return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), originInvoker);
        });
    }
```



```java
    //DubboProtocol
		@Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
      	//dubbo://192.168.2.155:20880/com.uteam.zen.api.IAuthorityService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService&bind.ip=192.168.2.155&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IAuthorityService&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority&pid=30584&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590651900177
        URL url = invoker.getUrl();

        // 服务标识 【服务坐标】：服务组名、服务名、版本、端口号
      	//【服务组名：null】com.uteam.zen.api.IAuthorityService【版本：null】:20880
        String key = serviceKey(url);
      	//*创建DubboExporter
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
      	//放入exporterMap中
        exporterMap.put(key, exporter);

        //本地存根相关 TODO
        Boolean isStubSupportEvent = url.getParameter(STUB_EVENT_KEY, DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }

            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

      	//*启动服务
        openServer(url);
      	//优化序列化
        optimizeSerialization(url);

        return exporter;
    }
```

启动服务openServer(url)	

> 在同一台机器上（单网卡），同一个端口上仅允许启动一个服务器实例。若某个端口上已有服务器实例，此时则调用 reset 方法重置服务器的一些配置。

```java
private void openServer(URL url) {
    // 获取host:port 服务器实例的标记
    String key = url.getAddress();
    
    boolean isServer = url.getParameter(IS_SERVER_KEY, true);
    if (isServer) {
      	//先从服务器实例缓存获取server实例
      	//DCL双端检测
        ExchangeServer server = serverMap.get(key);
        if (server == null) {
            synchronized (this) {
                server = serverMap.get(key);
                if (server == null) {
                    serverMap.put(key, createServer(url));
                }
            }
        } else {
            // 刷新server实例 TODO 后面再看
            server.reset(url);
        }
    }
}
```

创建服务器实例createServer

```java
    private ExchangeServer createServer(URL url) {
      	//dubbo://192.168.2.155:20880/com.uteam.zen.api.IAuthorityService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IAuthorityService&bind.ip=192.168.2.155&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IAuthorityService&methods=getAuthorityCodesByUserId,updateAuthority,loadAuthority,delAuthority,addAuthority,queryAuthority&pid=30584&qos.enable=false&release=2.7.4.1&side=provider&timestamp=1590651900177
        url = URLBuilder.from(url)
                // 添加参数channel.readonly.sent=true TODO 不知道干啥的。
                .addParameterIfAbsent(CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
                // 添加默认心跳检测参数 heartbeat=60*1000
                .addParameterIfAbsent(HEARTBEAT_KEY, String.valueOf(DEFAULT_HEARTBEAT))
          			//添加编码解码器参数 codec=dubbo
                .addParameter(CODEC_KEY, DubboCodec.NAME)
                .build();
      	//获取server参数。默认netty。
        String str = url.getParameter(SERVER_KEY, DEFAULT_REMOTING_SERVER);

				//通过SPI对server进行Transporter扩展检测。
        if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
            throw new RpcException("Unsupported server type: " + str + ", url: " + url);
        }

      	//创建ExchangeServer
        ExchangeServer server;
        try {
            server = Exchangers.bind(url, requestHandler);
        } catch (RemotingException e) {
            throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
        }
	
      
      	//获取client参数 TODO 不知道是啥。
        str = url.getParameter(CLIENT_KEY);
        if (str != null && str.length() > 0) {
          	//如果指定了client。查看是否有对应的Transporter扩展类
            Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class).getSupportedExtensions();
            if (!supportedTypes.contains(str)) {
                throw new RpcException("Unsupported client type: " + str);
            }
        }

        return server;
    }
```

创建Exchangers.bind(url, requestHandler)

```java
    public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        //参数判断
      	if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
      	//如果没有指定编码解码器参数，添加codec=exchange
      	//上一步指定了codec=dubbo.这里无操作。
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
      
      	//获取Exchanger。默认为HeaderExchanger。【SPI】
        return getExchanger(url).bind(url, handler);
    }
```

```java
@Override
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
  	//创建HeaderExchangeServer实例 心跳检测
  	//new HeaderExchangeHandler(handler) TODO 
  	//new DecodeHandler(new HeaderExchangeHandler(handler)) TODO
  	//c.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))) TODO
    return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
}
```

> new HeaderExchangeHandler(handler)是HeaderExchanger实例的handler
>
> DecodeHandler是解码器

```java
    //Transporters
		public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
        //{参数校验代码}
        ChannelHandler handler;
        if (handlers.length == 1) {
            handler = handlers[0];
        } else {
          	//如果handlers数量大于1，则创建返回ChannelHandler派发器ChannelHandlerDispatcher。
            handler = new ChannelHandlerDispatcher(handlers);
        }
      	//获取自适应Transporter,执行bind方法。
        return getTransporter().bind(url, handler);
    }
```

> getTransporter()是自适应扩展生成的Adapter类。根据url的transporter参数进行分发。
>
> 默认netty。
>
> ```java
> public static Transporter getTransporter() {
>     return ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension();
> }
> ```

```java
    //NettyTransporter
		public Server bind(URL url, ChannelHandler listener) throws RemotingException {
        return new NettyServer(url, listener);
    }
```

```java
    //NettyServer 基于Netty4
		public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
      	//调用父类构造方法。
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, "DubboServerHandler")));
    
      	//NettyServer创建过程
        protected void doOpen() throws Throwable {
        //服务启动类
        bootstrap = new ServerBootstrap();

        //创建 boss 和 worker 线程
        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        //设置Handler 重写了ChannelDuplexHandler
        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
          			//设置Channel的初始化器
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        // FIXME: should we use getTimeout()?
                        int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                      	//添加pipeline
                      	ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
                                .addLast("handler", nettyServerHandler);
                    }
                });
        // 设置ip：post绑定，
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        // 开启端口监听
        channelFuture.syncUninterruptibly();
        
        //这是netty的boss线程【可以理解为selector】
        channel = channelFuture.channel();

    		}
    }
		//父类AbstractServer
    public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
      	//父类构造方法 TODO 这里稍后再看
        super(url, handler);
      	//本机socket地址 ip:port 【192.168.59.1:20880】
        localAddress = getUrl().toInetSocketAddress();
				
      	//bind.ip 默认使用url的host 【192.168.59.1】
        String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
      	//bindPort 默认使用url的port 【20880】
        int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
        if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
          	//ip设置为0.0.0.0
            bindIp = ANYHOST_VALUE;
        }
        bindAddress = new InetSocketAddress(bindIp, bindPort);
      	//设置最大可接受连接数 0 【无限制】
        this.accepts = url.getParameter(ACCEPTS_KEY, DEFAULT_ACCEPTS);
      	//设置idle超时时间 默认60*1000
        this.idleTimeout = url.getParameter(IDLE_TIMEOUT_KEY, DEFAULT_IDLE_TIMEOUT);
        try {
          	//本类抽象方法 -> 子类实现
            doOpen();
            if (logger.isInfoEnabled()) {
                logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() + ", export " + getLocalAddress());
            }
        } catch (Throwable t) {
            throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " + getClass().getSimpleName()
                    + " on " + getLocalAddress() + ", cause: " + t.getMessage(), t);
        }
        //这里是获取端口对应线程池 WrappedChannelHandler中set的 TODO 
        DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class).getDefaultExtension();
        executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, Integer.toString(url.getPort()));
    }
```



- **==register==(registryUrl, registeredProviderUrl);**

> 注册到注册中心
>
> 以Zookeeper为例。zookeeper://localhost:2181

```java
    public void register(URL registryUrl, URL registeredProviderUrl) {
      	//获取Registry注册中心实例
        Registry registry = registryFactory.getRegistry(registryUrl);
      	//注册
        registry.register(registeredProviderUrl);
    }
```

==获取注册中心实例registryFactory.getRegistry(registryUrl)==

```java
    

		//AbstractRegistryFactory implements RegistryFactory
		//this 是 ZookeeperRegistryFactory
		//这是一个父类的模板方法 
		public Registry getRegistry(URL url) {
      	//zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService?application=system-service-dubbo&check=false&dubbo=2.0.2&export=dubbo%3A%2F%2F192.168.59.1%3A20880%2Fcom.uteam.zen.api.IAuthorityService%3Fanyhost%3Dtrue%26application%3Dsystem-service-dubbo%26bean.name%3DServiceBean%3Acom.uteam.zen.api.IAuthorityService%26bind.ip%3D192.168.59.1%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dcom.uteam.zen.api.IAuthorityService%26methods%3DgetAuthorityCodesByUserId%2CupdateAuthority%2CloadAuthority%2CdelAuthority%2CaddAuthority%2CqueryAuthority%26pid%3D3254%26qos.enable%3Dfalse%26release%3D2.7.4.1%26side%3Dprovider%26timestamp%3D1591340473205&pid=3254&qos.enable=false&release=2.7.4.1&timestamp=1591340468307
        url = URLBuilder.from(url)
                .setPath(RegistryService.class.getName()) //org.apache.dubbo.registry.RegistryService
                .addParameter(INTERFACE_KEY, RegistryService.class.getName()) //设置参数interface -> org.apache.dubbo.registry.RegistryService
                .removeParameters(EXPORT_KEY, REFER_KEY) //删除export（dubbo）和refer参数
                .build();
      	//url只剩注册中心相关信息
      	//zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService?application=system-service-dubbo&check=false&dubbo=2.0.2&interface=org.apache.dubbo.registry.RegistryService&pid=3254&qos.enable=false&release=2.7.4.1&timestamp=1591340468307
        String key = url.toServiceStringWithoutResolving();
      	//key - >zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService
      
        //使用ReentrantLock的非公平锁
        LOCK.lock();
        try {
          	//先取缓存
            Registry registry = REGISTRIES.get(key);
            if (registry != null) {
                return registry;
            }
            //交由子类实现的Registry创建
            registry = createRegistry(url);
            if (registry == null) {
                throw new IllegalStateException("Can not create registry " + url);
            }
            REGISTRIES.put(key, registry);
            return registry;
        } finally {
            // 释放锁
            LOCK.unlock();
        }
    }
```

子类ZookeeperRegistryFactory逻辑

```java
/**
 * ZookeeperRegistryFactory.
 *
 */
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
		
  	//这个是RegistoryFactory$Adapter生成ZookeeperRegistryFactory实例的时候
  	//使用spi注入的ZookeeperTransporter
  	//实例为ZookeeperTransporter$Adapter
    private ZookeeperTransporter zookeeperTransporter;

    /**
     * 供spi使用的setter方法
     * Invisible injection of zookeeper client via IOC/SPI
     * @param zookeeperTransporter
     */
    public void setZookeeperTransporter(ZookeeperTransporter zookeeperTransporter) {
        this.zookeeperTransporter = zookeeperTransporter;
    }

    @Override
    public Registry createRegistry(URL url) {
      	//就这一行生成方法
        return new ZookeeperRegistry(url, zookeeperTransporter);
    }

}
```

ZookeeperRegistry构造方法

> ZookeeperRegistry extends FailbackRegistry
> FailbackRegistry extends AbstractRegistry

```java
//1.AbstractRegistry
public AbstractRegistry(URL url) {
  	//设置注册中心url
    setUrl(url);
    //配置中心的url是否配置了“同步保存文件”属性，默认为false
    syncSaveFile = url.getParameter(REGISTRY_FILESAVE_SYNC_KEY, false);
  	//配置信息本地缓存的文件名
    String filename = url.getParameter(FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(APPLICATION_KEY) + "-" + url.getAddress() + ".cache");
    //初次使用，逐级创建文件路径
  	File file = null;
    if (ConfigUtils.isNotEmpty(filename)) {
        file = new File(filename);
        if (!file.exists() && file.getParentFile() != null && !file.getParentFile().exists()) {
            if (!file.getParentFile().mkdirs()) {
                throw new IllegalArgumentException("Invalid registry cache file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
            }
        }
    }
    this.file = file;
    //如果现有配置缓存，则从缓存文件中加载属性
    loadProperties();
    notify(url.getBackupUrls());
}
```

```java
//2.FailbackRegistry 注册失败时的自动恢复
public FailbackRegistry(URL url) {
    super(url);
  	//注册中心重试周期 默认5s
    this.retryPeriod = url.getParameter(REGISTRY_RETRY_PERIOD_KEY, DEFAULT_REGISTRY_RETRY_PERIOD);

    // 设定时间轮定时器 作为注册失败的处理
  	// 默认128个刻度。
    retryTimer = new HashedWheelTimer(new NamedThreadFactory("DubboRegistryRetryTimer", true), retryPeriod, TimeUnit.MILLISECONDS, 128);
}
```


```java
    //3.ZookeeperRegistry extends FailbackRegistry
		public ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter) {
        super(url);
      	//url的host不能是0.0.0.0
        if (url.isAnyHost()) {
            throw new IllegalStateException("registry address == null");
        }
      	//设置组名 默认是 /dubbo
        String group = url.getParameter(GROUP_KEY, DEFAULT_ROOT);
        if (!group.startsWith(PATH_SEPARATOR)) {
            group = PATH_SEPARATOR + group;
        }
        this.root = group;
      	
      	//使用zookeeperTransporter$adapter生成zkclient
      	//默认 CuratorZookeeperTransporter 【也是一个模板创建】
        zkClient = zookeeperTransporter.connect(url);
      	//添加zkClient状态监听器
        zkClient.addStateListener((state) -> {
            if (state == StateListener.RECONNECTED) {
              	//重连
                logger.warn("Trying to fetch the latest urls, in case there're provider changes during connection loss.\n" +
                        " Since ephemeral ZNode will not get deleted for a connection lose, " +
                        "there's no need to re-register url of this instance.");
                ZookeeperRegistry.this.fetchLatestAddresses();
            } else if (state == StateListener.NEW_SESSION_CREATED) {
              	//首次连接
                logger.warn("Trying to re-register urls and re-subscribe listeners of this instance to registry...");
                try {
                    ZookeeperRegistry.this.recover();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
              	//连接丢失
            } else if (state == StateListener.SESSION_LOST) {
                logger.warn("Url of this instance will be deleted from registry soon. " +
                        "Dubbo client will try to re-register once a new session is created.");
            } else if (state == StateListener.SUSPENDED) {

            } else if (state == StateListener.CONNECTED) {

            }
        });
    }
```

- zkClient创建

```java
//子类CuratorZookeeperTransporter
public class CuratorZookeeperTransporter extends AbstractZookeeperTransporter {
    @Override
    public ZookeeperClient createZookeeperClient(URL url) {
      	//创建ZookeeperClient
        return new CuratorZookeeperClient(url);
    }
}
//父类AbstractZookeeperTransporter
public abstract class AbstractZookeeperTransporter implements ZookeeperTransporter {
  	
 		//本地缓存
    private final Map<String, ZookeeperClient> zookeeperClientMap = new ConcurrentHashMap<>();

    @Override
    public ZookeeperClient connect(URL url) {
        ZookeeperClient zookeeperClient;
      	//获取地址 localhost:2181
        List<String> addressList = getURLBackupAddress(url);
      
      
      	//还是对缓存的双端检索
        if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null && zookeeperClient.isConnected()) {
            return zookeeperClient;
        }
        synchronized (zookeeperClientMap) {
            if ((zookeeperClient = fetchAndUpdateZookeeperClientCache(addressList)) != null && zookeeperClient.isConnected()) {
                return zookeeperClient;
            }
			
          	//子类实现ZookeeperClient创建
            zookeeperClient = createZookeeperClient(url);
          	//放入缓存中
            writeToClientMap(addressList, zookeeperClient);
        }
      
      
        return zookeeperClient;
    }
}
```

- 创建CuratorZookeeperClient

```java
public CuratorZookeeperClient(URL url) {
    super(url);
    try {
      	//连接超时时间 【默认5s】
        int timeout = url.getParameter(TIMEOUT_KEY, DEFAULT_CONNECTION_TIMEOUT_MS);
      	//zookeeper会话到期时间 【默认60s】
        int sessionExpireMs = url.getParameter(ZK_SESSION_EXPIRE_KEY, DEFAULT_SESSION_TIMEOUT_MS);
        
      	//使用CuratorFrameworkFactory创建者【Curator Framework 提供了对zookeeper的高级api封装】
      	CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
                .connectString(url.getBackupAddress()) //链接地址
                .retryPolicy(new RetryNTimes(1, 1000))	//重试策略
                .connectionTimeoutMs(timeout) //连接超时时间
                .sessionTimeoutMs(sessionExpireMs); //会话到期时间
      
      	//权限
        String authority = url.getAuthority();
        if (authority != null && authority.length() > 0) {
            builder = builder.authorization("digest", authority.getBytes());
        }
      	//构建CuratorFramework实例
        client = builder.build();
      	//添加连接状态监听器
        client.getConnectionStateListenable().addListener(new CuratorConnectionStateListener(url));
        //启动客户端
      	client.start();
      	//阻塞直到连接完成
        boolean connected = client.blockUntilConnected(timeout, TimeUnit.MILLISECONDS);
        if (!connected) {
          	//连接失败常见错误zookeeper not connected
            throw new IllegalStateException("zookeeper not connected");
        }
    } catch (Exception e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```

到此。就获取到了Registry实例。下面是==registry.register(registeredProviderUrl)进行服务注册。==

> ZookeeperRegistry 子类实现doRegister
> ​	extends FailbackRegistry 重写方法register -->调用doRegister
> 		extends AbstractRegistry 模板方法register -->提供已注册标记

```java
    //FailbackRegistry
		@Override
    public void register(URL url) {
        super.register(url);
        removeFailedRegistered(url);
        removeFailedUnregistered(url);
        try {
            // 子类实现的注册方法
            doRegister(url);
        } catch (Exception e) {
          
          	//注册失败
            Throwable t = e;

            // 启动检测 check=true ，会直接抛抛异常
            boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
                    && url.getParameter(Constants.CHECK_KEY, true)
                    && !CONSUMER_PROTOCOL.equals(url.getProtocol());
            boolean skipFailback = t instanceof SkipFailbackWrapperException;
            if (check || skipFailback) {
                if (skipFailback) {
                    t = t.getCause();
                }
                throw new IllegalStateException("Failed to register " + url + " to registry " + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
            } else {
                logger.error("Failed to register " + url + ", waiting for retry, cause: " + t.getMessage(), t);
            }

            // 记录注册失败的连接
            addFailedRegistered(url);
        }
    }
```

重点看子类的doRegister(url)实现。用于zookeeper的节点生成。

```java
//ZookeeperRegistry
@Override
public void doRegister(URL url) {
    try {
      	//通过zookeeper客户端创建节点
      	//    /${group}/${serviceInterfaceClass}/providers/${url}
      	//toUrlPath(url) ->
      	//    /dubbo/com.uteam.zen.api.IAuthorityService/providers/dubbo%3A%2F%2F192.168.8.100%3A20880%2Fcom.uteam.zen.api.IAuthorityService%3Fanyhost%3Dtrue%26application%3Dsystem-service-dubbo%26bean.name%3DServiceBean%3Acom.uteam.zen.api.IAuthorityService%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dcom.uteam.zen.api.IAuthorityService%26methods%3DgetAuthorityCodesByUserId%2CupdateAuthority%2CloadAuthority%2CdelAuthority%2CaddAuthority%2CqueryAuthority%26pid%3D4966%26release%3D2.7.4.1%26side%3Dprovider%26timestamp%3D1591599719623
        zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
    } catch (Throwable e) {
        throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
    }
}
```

注册完成！

## 服务引入

> ReferenceBean实现了FactoryBean和InitializingBean类。无论优先级。
>
> 服务引入时机：
>
> - ReferenceBean实例初始化，由容器刷新调用InitializingBean的afterPropertiesSet()方法时。
> - ReferenceBean实例被注入到其他类的引用时。
>
> dubbo默认是懒汉式的注入方式。如果需要饿汉式，需要配置reference的init属性为true。
>
> 引入到其他类中时，是通过ReferenceAnnotationBeanPostProcessor直接调用的referenceBean的get()方法。
>
> 那就从getObject()方法开始。

```java
		@Override
		public Object getObject() {
    		return get();
		}

    public synchronized T get() {
      	//配置检查、url装配 【跟ServiceBean差不多，先不看了。】
        checkAndUpdateSubConfigs();

      	//destroyed检查
        if (destroyed) {
            throw new IllegalStateException("The invoker of ReferenceConfig(" + url + ") has already destroyed!");
        }
      	
      	//ref为空，就通过init进行初始化，生成代理类
        if (ref == null) {
            init();
        }
        return ref;
    }
```



### ReferenceConfig初始化

```java
    private void init() {
      	// 避免重复创建
        if (initialized) {
            return;
        }
      	//本地存根、本地伪装检查 【跟ServiceBean一样。是父类AbstractInterfaceConfig的公共方法。】
        checkStubAndLocal(interfaceClass);
        checkMock(interfaceClass);
        Map<String, String> map = new HashMap<String, String>();

      	//添加消费侧参数 side -> consumer
        map.put(SIDE_KEY, CONSUMER_SIDE);

      	//添加运行时参数
        appendRuntimeParameters(map);
      	//检查不是泛化接口
        if (!ProtocolUtils.isGeneric(getGeneric())) {
          	//版本号信息
            String revision = Version.getVersion(interfaceClass, version);
            if (revision != null && revision.length() > 0) {
                map.put(REVISION_KEY, revision);
            }
						//通过interfaceClass的目标对象包裹类，获取所有方法名
            String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
          	//添加方法参数
            if (methods.length == 0) {
                logger.warn("No method found in service interface " + interfaceClass.getName());
                map.put(METHODS_KEY, ANY_VALUE);
            } else {
                map.put(METHODS_KEY, StringUtils.join(new HashSet<String>(Arrays.asList(methods)), COMMA_SEPARATOR));
            }
        }
      	//添加接口参数
        map.put(INTERFACE_KEY, interfaceName);
      	//添加metrics参数
        appendParameters(map, metrics);
      	//添加application参数
        appendParameters(map, application);
      	//添加module参数
        appendParameters(map, module);
       	//添加consumer参数
        appendParameters(map, consumer);
      	//添加reference参数
        appendParameters(map, this);
        Map<String, Object> attributes = null;
      	//添加方法重试参数
        if (CollectionUtils.isNotEmpty(methods)) {
            attributes = new HashMap<String, Object>();
            for (MethodConfig methodConfig : methods) {
                appendParameters(map, methodConfig, methodConfig.getName());
                String retryKey = methodConfig.getName() + ".retry";
                if (map.containsKey(retryKey)) {
                    String retryValue = map.remove(retryKey);
                    if ("false".equals(retryValue)) {
                        map.put(methodConfig.getName() + ".retries", "0");
                    }
                }
              	//methodConfig的属性参数，比如onreturn、onthrow、oninvoke 等
                attributes.put(methodConfig.getName(), convertMethodConfig2AsyncInfo(methodConfig));
            }
        }
				//消费者ip地址
      	//可以通过系统环境变量DUBBO_IP_TO_REGISTRY指定
        String hostToRegistry = ConfigUtils.getSystemProperty(DUBBO_IP_TO_REGISTRY);
        if (StringUtils.isEmpty(hostToRegistry)) {
            hostToRegistry = NetUtils.getLocalHost();
        } else if (isInvalidLocalHost(hostToRegistry)) {
            throw new IllegalArgumentException("Specified invalid registry ip from property:" + DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
        }
        map.put(REGISTER_IP_KEY, hostToRegistry);

      	//创建代理
        ref = createProxy(map);

        String serviceKey = URL.buildKey(interfaceName, group, version);
        ApplicationModel.initConsumerModel(serviceKey, buildConsumerModel(serviceKey, attributes));
        //首尾呼应...【synchronized加在了get()方法里】
      	initialized = true;
    }
```

### 创建代理createProxy

> map的样例
>
> "side" -> "consumer"
> "application" -> "zen.admin.web"
> "register.ip" -> "192.168.8.100"
> "release" -> "2.7.4.1"
> "methods" -> "getUserById,loadUser,delUser,addUser,getUserByOpenId,updateUser,getUserByName,queryUser"
> "lazy" -> "false"
> "sticky" -> "false"
> "dubbo" -> "2.0.2"
> "pid" -> "19048"
> "check" -> "false"
> "interface" -> "com.uteam.zen.api.IUserService"
> "timestamp" -> "1591690601210"
>
> 调用方式有三种
>
> - 本地调用 injvm
> - 点对点直连远程调用
> - 通过注册中心远程调用





```java
private T createProxy(Map<String, String> map) {
  			//本地调用 injvm
        if (shouldJvmRefer(map)) {
          	//本地调用的协议是injvm TODO
            URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            //使用对应的protocol生成InjvmInvoker实例
          	invoker = REF_PROTOCOL.refer(interfaceClass, url);
            //{日志}
        } else {
            urls.clear(); // reference retry init will add url to urls, lead to OOM
          	//指定了url。进行点对点调用 TODO
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                String[] us = SEMICOLON_SPLIT_PATTERN.split(url);//多个url使用分号分隔。
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                      	//指定注册中心。
                        if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // assemble URL from register center's configuration
                // if protocols not injvm checkRegistry
                if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())){
                    checkRegistry();
                  	//加载注册中心url
                    List<URL> us = loadRegistries(false);
                    if (CollectionUtils.isNotEmpty(us)) {
                        for (URL u : us) {
                            URL monitorUrl = loadMonitor(u);
                            if (monitorUrl != null) {
                                map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                            }
                          	//注册中心地址 【参数添加后】
                          	//e,g.  registry://localhost:2181/org.apache.dubbo.registry.RegistryService?application=zen.admin.web&dubbo=2.0.2&pid=19048&refer=application%3Dzen.admin.web%26check%3Dfalse%26dubbo%3D2.0.2%26interface%3Dcom.uteam.zen.api.IUserService%26lazy%3Dfalse%26methods%3DgetUserById%2CloadUser%2CdelUser%2CaddUser%2CgetUserByOpenId%2CupdateUser%2CgetUserByName%2CqueryUser%26pid%3D19048%26register.ip%3D192.168.8.100%26release%3D2.7.4.1%26side%3Dconsumer%26sticky%3Dfalse%26timestamp%3D1591690601210&registry=zookeeper&release=2.7.4.1&timestamp=1591695255820
                            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        }
                    }
                    if (urls.isEmpty()) {
                        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                    }
                }
            }
						//单个注册中心或单个服务直连
            if (urls.size() == 1) {
              	//构造invoker
                invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            } else {
              	//TODO 这种模式还没试过
              	//多个注册中心或多个服务直连，或混合
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
              	//每个都构造invoker
                for (URL url : urls) {
                    invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                    if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // use last registry url
                    }
                }
              	//如果有注册中心的url
                if (registryURL != null) { // registry url is available
                  	//使用AvailableCluster
                    // use RegistryAwareCluster only when register's CLUSTER is available
                    URL u = registryURL.addParameter(CLUSTER_KEY, RegistryAwareCluster.NAME);
                    // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                    //创建 StaticDirectory 实例，并由 Cluster 对多个 Invoker 进行合并
                  	invoker = CLUSTER.join(new StaticDirectory(u, invokers));
                } else { // not a registry url, must be direct invoke.
                    invoker = CLUSTER.join(new StaticDirectory(invokers));
                }
            }
        }
	
  			//invoker可用性检查
        if (shouldCheck() && !invoker.isAvailable()) {
            throw new IllegalStateException("Failed to check the status of the service " + interfaceName + ". No provider available for the service " + (group == null ? "" : group + "/") + interfaceName + (version == null ? "" : ":" + version) + " from the url " + invoker.getUrl() + " to the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion());
        }
        if (logger.isInfoEnabled()) {
            logger.info("Refer dubbo service " + interfaceClass.getName() + " from url " + invoker.getUrl());
        }
        /**
         * @since 2.7.0
         * ServiceData Store
         */
        MetadataReportService metadataReportService = null;
        if ((metadataReportService = getMetadataReportService()) != null) {
            URL consumerURL = new URL(CONSUMER_PROTOCOL, map.remove(REGISTER_IP_KEY), 0, map.get(INTERFACE_KEY), map);
            metadataReportService.publishConsumer(consumerURL);
        }
        // 创建代理类
        return (T) PROXY_FACTORY.getProxy(invoker);
    }
```

#### 创建invoker

##### RegistryProtocol

> REF_PROTOCOL.refer(interfaceClass, urls.get(0))
>
> url为注册中心url【==registry==://localhost:2181/org.apache.dubbo.registry.RegistryService?application=zen.admin.web&dubbo=2.0.2&pid=19048&refer=application%3Dzen.admin.web%26check%3Dfalse%26dubbo%3D2.0.2%26interface%3Dcom.uteam.zen.api.IUserService%26lazy%3Dfalse%26methods%3DgetUserById%2CloadUser%2CdelUser%2CaddUser%2CgetUserByOpenId%2CupdateUser%2CgetUserByName%2CqueryUser%26pid%3D19048%26register.ip%3D192.168.8.100%26release%3D2.7.4.1%26side%3Dconsumer%26sticky%3Dfalse%26timestamp%3D1591690601210&registry=zookeeper&release=2.7.4.1&timestamp=1591695255820】
>
> REF_PROTOCOL是Protocol的自适应扩展。【经过ProtocolFilterWrapper、ProtocolListenerWrapper、QosProtocolWrapper三个wrapper就到了**==RegistryProtocol==**】

```java
    //RegistryProtocol
		public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
      	//把url里的registry注册中心参数【默认dubbo。注册中心我已经指定了zookeeper，所以是zookeeper】
      	//放到协议位置，并删掉仓储
        url = URLBuilder.from(url)
                .setProtocol(url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY))
                .removeParameter(REGISTRY_KEY)
                .build();
      	//zookeeper://localhost:2181/org.apache.dubbo.registry.RegistryService?application=zen.admin.web&dubbo=2.0.2&pid=19048&refer=application%3Dzen.admin.web%26check%3Dfalse%26dubbo%3D2.0.2%26interface%3Dcom.uteam.zen.api.IUserService%26lazy%3Dfalse%26methods%3DgetUserById%2CloadUser%2CdelUser%2CaddUser%2CgetUserByOpenId%2CupdateUser%2CgetUserByName%2CqueryUser%26pid%3D19048%26register.ip%3D192.168.8.100%26release%3D2.7.4.1%26side%3Dconsumer%26sticky%3Dfalse%26timestamp%3D1591690601210&release=2.7.4.1&timestamp=1591695255820
        
      	//获取注册中心实例
      	Registry registry = registryFactory.getRegistry(url);
				
      	//TODO
      	if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }

        // url字符串转map
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        //获取group配置
      	String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
              	// 通过 SPI 加载 MergeableCluster 实例，并调用 doRefer 继续执行服务引用逻辑
              	//TODO 这里先不看
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
      	//调用 doRefer 继续执行服务引用逻辑
        return doRefer(cluster, registry, type, url);
    }
```

doRefer

> 上一段代码refer的doRefer调用，第一个参数不同。
>
> 有group分组的时候，cluster为MergeableCluster
>
> 否则，是Cluster$Adapter自适应扩展类，从url参数中获取cluster，默认failover。

```java
    //RegistryProtocol
		private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
      
      	//创建RegistryDirectory服务字典实例
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
      	//设置注册中心
        directory.setRegistry(registry);
      	//设置协议
        directory.setProtocol(protocol);
        // url所有参数
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
      	//生成消费端url
      	//consumer://192.168.8.211/com.uteam.zen.api.IUserService?application=zen.admin.web&check=false&dubbo=2.0.2&interface=com.uteam.zen.api.IUserService&lazy=false&methods=getUserById,loadUser,delUser,addUser,getUserByOpenId,getUserByName,updateUser,queryUser&pid=25627&release=2.7.4.1&side=consumer&sticky=false&timestamp=1591771384197
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        //serviceInterface不是* 且 registry不是false
      	if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
          	//consumer://192.168.8.211/com.uteam.zen.api.IUserService?application=zen.admin.web&【设置类别为消费侧】category=consumers&check=false&dubbo=2.0.2&interface=com.uteam.zen.api.IUserService&lazy=false&methods=getUserById,loadUser,delUser,addUser,getUserByOpenId,getUserByName,updateUser,queryUser&pid=25627&release=2.7.4.1&side=consumer&sticky=false&timestamp=1591771384197
            directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
            //向注册中心consumers目录下注册当前消费者
          	//registry是ZookeeperRegistry实例。跟注册providers节点一样。
          	//ZookeeperRegistry的doRegister方法中toUrlPath(url)生成了customers节点信息
          	//    格式/${group}/${serviceInterfaceClass}/customers/${url}
          	///dubbo/com.uteam.zen.api.IUserService/consumers/consumer%3A%2F%2F192.168.8.211%2Fcom.uteam.zen.api.IUserService%3Fapplication%3Dzen.admin.web%26category%3Dconsumers%26check%3Dfalse%26dubbo%3D2.0.2%26interface%3Dcom.uteam.zen.api.IUserService%26lazy%3Dfalse%26methods%3DgetUserById%2CloadUser%2CdelUser%2CaddUser%2CgetUserByOpenId%2CgetUserByName%2CupdateUser%2CqueryUser%26pid%3D25627%26release%3D2.7.4.1%26side%3Dconsumer%26sticky%3Dfalse%26timestamp%3D1591771384197
          	registry.register(directory.getRegisteredConsumerUrl());
        }
        
      	//添加已经激活的路由工厂
      	// service = org.apache.dubbo.rpc.cluster.router.condition.config.ServiceRouterFactory
				// app = org.apache.dubbo.rpc.cluster.router.condition.config.AppRouterFactory
				// tag = org.apache.dubbo.rpc.cluster.router.tag.TagRouterFactory
				// mock = org.apache.dubbo.rpc.cluster.router.mock.MockRouterFactory
      	//分别生成对应的路由MockInvokerSelector、AppRouter、TagRouter、MockRouter实例。
        directory.buildRouterChain(subscribeUrl);
      	//订阅providers、configurators、routers等节点信息。
        directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
                PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));

      	//集群合并。生成Invoker。
      	//默认生成FailoverClusterInvoker.被MockClusterWrapper包裹。
        Invoker invoker = cluster.join(directory);
      	//对invoker进行CustomerInvokerWrapper包裹，放入ProviderConsumerRegTable的缓存中。
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```



##### DubboProtocol

> Dubbo协议的Invoker创建比较简单
>
> refer是使用父类抽象模板AbstractProtocol方法refer，子类提供protocolBindingRefer抽象方法实现。

```java
		//AbstractProtocol
		@Override
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        return new AsyncToSyncInvoker<>(protocolBindingRefer(type, url));
    }


		//DubboProtocol
		@Override
    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);

        // 创建 DubboInvoker
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);

        return invoker;
    }
```

> 这里的重点是getClients。TODO



#### 生成代理(T) PROXY_FACTORY.getProxy(invoker)

![image-20200610161102745](https://gitee.com/wangigor/typora-images/raw/master/dubbo-ProxyFactory-hierarchy.png)

> 默认是JavassistProxyFactory。

```java
    //AbstractProxyFactory父类单参方法
		@Override
    public <T> T getProxy(Invoker<T> invoker) throws RpcException {
      	//调用重载方法
        return getProxy(invoker, false);
    }
		//AbstractProxyFactory父类重载方法
    @Override
    public <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException {
        Class<?>[] interfaces = null;
      	//TODO 不知道干嘛的 directry的url的interfaces参数
        String config = invoker.getUrl().getParameter(INTERFACES);
        if (config != null && config.length() > 0) {
          	//逗号分隔
            String[] types = COMMA_SPLIT_PATTERN.split(config);
            if (types != null && types.length > 0) {
                interfaces = new Class<?>[types.length + 2];
                interfaces[0] = invoker.getInterface();
                interfaces[1] = EchoService.class;
                for (int i = 0; i < types.length; i++) {
                    // TODO can we load successfully for a different classloader?.
                    interfaces[i + 2] = ReflectUtils.forName(types[i]);
                }
            }
        }
        if (interfaces == null) {
          	//目标接口class和EchoService接口class
            interfaces = new Class<?>[]{invoker.getInterface(), EchoService.class};
        }

      	//泛化调用支持 增加GenericService接口class
        if (!GenericService.class.isAssignableFrom(invoker.getInterface()) && generic) {
            int len = interfaces.length;
            Class<?>[] temp = interfaces;
            interfaces = new Class<?>[len + 1];
            System.arraycopy(temp, 0, interfaces, 0, len);
            interfaces[len] = com.alibaba.dubbo.rpc.service.GenericService.class;
        }
				//生成代理。子类重载。
        return getProxy(invoker, interfaces);
    }
```

JavassistProxyFactory

```java
    @Override
    @SuppressWarnings("unchecked")
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
      	//生成Proxy代理子类。
				//InvokerInvocationHandler是InvocationHandler，实例交给proxy代理
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
```

```java
		//abstract Proxy
		public static Proxy getProxy(Class<?>... ics) {
      	//调用重载方法。
        return getProxy(ClassUtils.getClassLoader(Proxy.class), ics);
    }

    public static Proxy getProxy(ClassLoader cl, Class<?>... ics) {
      	//不能超过65535个代理接口
        if (ics.length > MAX_PROXY_COUNT) {
            throw new IllegalArgumentException("interface limit exceeded");
        }

        StringBuilder sb = new StringBuilder();
      	//遍历所有接口
        for (int i = 0; i < ics.length; i++) {
            String itf = ics[i].getName();
          	//接口检查
            if (!ics[i].isInterface()) {
                throw new RuntimeException(itf + " is not a interface.");
            }
						
          	//重新加载接口
            Class<?> tmp = null;
            try {
                tmp = Class.forName(itf, false, cl);
            } catch (ClassNotFoundException e) {
            }
						//当前classloader不可见。抛异常
            if (tmp != ics[i]) {
                throw new IllegalArgumentException(ics[i] + " is not visible from class loader");
            }
						//拼接接口类的全限定名。分号分隔。
            sb.append(itf).append(';');
        }

        // e,g. com.uteam.zen.api.IUserService;com.alibaba.dubbo.rpc.service.EchoService;
        String key = sb.toString();

        // 缓存获取 类的静态成员new WeakHashMap<ClassLoader, Map<String, Object>>();
        final Map<String, Object> cache;
        synchronized (PROXY_CACHE_MAP) {
            cache = PROXY_CACHE_MAP.computeIfAbsent(cl, k -> new HashMap<>());
        }

        Proxy proxy = null;
      	//并发时，第一个线程进来把key对应的value改成PENDING_GENERATION_MARKER。
      	//后续线程都等待创建完成。
        synchronized (cache) {
            do {
                Object value = cache.get(key);
                if (value instanceof Reference<?>) {
                    proxy = (Proxy) ((Reference<?>) value).get();
                    if (proxy != null) {
                        return proxy;
                    }
                }

                if (value == PENDING_GENERATION_MARKER) {
                  	//创建过程中，其他线程都阻塞在这里。
                    try {
                        cache.wait();
                    } catch (InterruptedException e) {
                    }
                } else {
                  	//只有第一个创建线程走到这里。跳出循环。
                    cache.put(key, PENDING_GENERATION_MARKER);
                    break;
                }
            }
            while (true);
        }

      	//创建开始
      
      	//id。使用AtomicLong计数
        long id = PROXY_CLASS_COUNTER.getAndIncrement();
        String pkg = null;
        ClassGenerator ccp = null, ccm = null;
        try {
          	//生成ClassGenerator对象。
          	//TODO ClassGenerator是dubbo对javassist的封装
            ccp = ClassGenerator.newInstance(cl);

            Set<String> worked = new HashSet<>();
            List<Method> methods = new ArrayList<>();
	
          	//遍历接口class
            for (int i = 0; i < ics.length; i++) {
              	//非public的类检查
                if (!Modifier.isPublic(ics[i].getModifiers())) {
                  	//获取包名
                    String npkg = ics[i].getPackage().getName();
                    if (pkg == null) {
                        pkg = npkg;
                    } else {
                      	//非public级别的接口，必须在同一个包下。否则抛出异常
                        if (!pkg.equals(npkg)) {
                            throw new IllegalArgumentException("non-public interfaces from different packages");
                        }
                    }
                }
              	//ClassGenerator添加接口类
                ccp.addInterface(ics[i]);
								
              	//遍历接口的方法
                for (Method method : ics[i].getMethods()) {
                  	//获取方法描述【方法签名】
                    String desc = ReflectUtils.getDesc(method);
                  	//worked中保存了所有已经实现的方法签名。
                  	//假设A、B接口有相同的方法。只操作一次。
                    if (worked.contains(desc)) {
                        continue;
                    }
                  	//跳过接口中的静态方法
                    if (ics[i].isInterface() && Modifier.isStatic(method.getModifiers())) {
                        continue;
                    }
                    worked.add(desc);

                    int ix = methods.size();
                  	//方法返回
                    Class<?> rt = method.getReturnType();
                  	//方法参数
                    Class<?>[] pts = method.getParameterTypes();
										//生成代码 Object[] args = new Object[方法参数长度];
                    StringBuilder code = new StringBuilder("Object[] args = new Object[").append(pts.length).append("];");
                    //循环生成赋值代码 args[j]=($w)$j; 
                  	for (int j = 0; j < pts.length; j++) {
                        code.append(" args[").append(j).append("] = ($w)$").append(j + 1).append(";");
                    }
                  	//生成调用InvokerHandler的代码
                  	//Object ret=handler.invoke(this,methds[ix],args);
                    code.append(" Object ret = handler.invoke(this, methods[").append(ix).append("], args);");
                    //如果返回类型不是void，添加带类型的返回语句。
                  	//return (返回类型) ret;
                  	if (!Void.TYPE.equals(rt)) {
                        code.append(" return ").append(asArgument(rt, "ret")).append(";");
                    }

                    methods.add(method);
                  	//添加方法名，方法访问控制，返回类型，参数类型，异常类型，源码到ClassGenerator中。
                    ccp.addMethod(method.getName(), method.getModifiers(), rt, pts, method.getExceptionTypes(), code.toString());
                }
            }
						
          	//有非public的接口，就使用他的包名。不然调用不到。
          	//否则都是用org.apache.dubbo.common.bytecode.Proxy的包名
            if (pkg == null) {
                pkg = PACKAGE_NAME;
            }

            // 代理类名称org.apache.dubbo.common.bytecode.proxy0
            String pcn = pkg + ".proxy" + id;
            ccp.setClassName(pcn);
          	//添加methods属性
            ccp.addField("public static java.lang.reflect.Method[] methods;");
          	//添加handler属性
          	//private java.lang.reflect.InvocationHandler handler;
            ccp.addField("private " + InvocationHandler.class.getName() + " handler;");
						// 为接口代理类添加带有 InvocationHandler 参数的构造方法，比如：
        		// porxy0(java.lang.reflect.InvocationHandler arg0) {
        		//     handler=$1;
    				// }
            ccp.addConstructor(Modifier.PUBLIC, new Class<?>[]{InvocationHandler.class}, new Class<?>[0], "handler=$1;");
            // 为接口代理类添加默认构造方法
          	ccp.addDefaultConstructor();
          	// 生成接口代理类class
            Class<?> clazz = ccp.toClass();
          	// methods属性初始化。 
            clazz.getField("methods").set(null, methods.toArray(new Method[0]));

          	
            // 创建Proxy子类，org.apache.dubbo.common.bytecode.Proxy0 
          	// proxy0和Proxy0的差别
          	// 用于实例化接口代理类。可以理解为工厂类。
            String fcn = Proxy.class.getName() + id;
            ccm = ClassGenerator.newInstance(cl);
            ccm.setClassName(fcn);
            ccm.addDefaultConstructor();
          	//设置父类为Proxy.class
            ccm.setSuperClass(Proxy.class);
          	//添加方法
          	//public Object newInstance(java.lang.reflect.InvocationHandler h){
          	//    return new org.apache.dubbo.common.bytecode.proxy0($1);
        		//}
            ccm.addMethod("public Object newInstance(" + InvocationHandler.class.getName() + " h){ return new " + pcn + "($1); }");
            Class<?> pc = ccm.toClass();
          	//返回代理接口的工厂实例。
            proxy = (Proxy) pc.newInstance();
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException(e.getMessage(), e);
        } finally {
            // release ClassGenerator
            if (ccp != null) {
                ccp.release();
            }
            if (ccm != null) {
                ccm.release();
            }
            synchronized (cache) {
                if (proxy == null) {
                    cache.remove(key);
                } else {
                    cache.put(key, new WeakReference<Proxy>(proxy));
                }
                cache.notifyAll();
            }
        }
        return proxy;
    }
```

代理class源码
> 使用arthas查看源码

```java
//ccm生成的工厂类
public class Proxy0
extends Proxy
implements ClassGenerator.DC {
    @Override
    public Object newInstance(InvocationHandler invocationHandler) {
        return new proxy0(invocationHandler);
    }
}
//ccp生成的IUserService代理
public class proxy0
implements ClassGenerator.DC,
EchoService,
IUserService {
    public static Method[] methods;
    private InvocationHandler handler;

    public UserDto getUserById(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[0], arrobject);
        return (UserDto)object;
    }

    public void delUser(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[1], arrobject);
    }

    public UserDto loadUser(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[2], arrobject);
        return (UserDto)object;
    }

    public void addUser(UserDto userDto) {
        Object[] arrobject = new Object[]{userDto};
        Object object = this.handler.invoke(this, methods[3], arrobject);
    }

    public void updateUser(UserDto userDto) {
        Object[] arrobject = new Object[]{userDto};
        Object object = this.handler.invoke(this, methods[4], arrobject);
    }

    public List queryUser(UserDto userDto) {
        Object[] arrobject = new Object[]{userDto};
        Object object = this.handler.invoke(this, methods[5], arrobject);
        return (List)object;
    }
    public UserDto getUserByOpenId(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[6], arrobject);
        return (UserDto)object;
    }

    public UserDto getUserByName(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[7], arrobject);
        return (UserDto)object;
    }
		
    @Override
    public Object $echo(Object object) {
        Object[] arrobject = new Object[]{object};
        Object object2 = this.handler.invoke(this, methods[8], arrobject);
        return object2;
    }

    public proxy0() {
    }

    public proxy0(InvocationHandler invocationHandler) {
        this.handler = invocationHandler;
    }
}
```







#### 创建RegistryDirectory【==服务字典==】

![image-20200615151013951](https://gitee.com/wangigor/typora-images/raw/master/RegistryDirectoryHierarchy.png)

> 【dubbo官方文档】
>
> 服务字典存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用。在一个服务集群中，服务提供者数量并不是一成不变的，如果集群中新增了一台机器，相应地在服务目录中就要新增一条服务提供者记录。或者，如果服务提供者的配置修改了，服务目录中的记录也要做相应的更新。
>
> 实际上服务字典在获取注册中心的服务配置信息后，会为每条配置信息生成一个 Invoker 对象，并把这个 Invoker 对象存储起来，这个 Invoker 才是服务字典最终持有的对象。Invoker 有什么用呢？看名字就知道了，这是一个具有远程调用功能的对象。讲到这大家应该知道了什么是服务字典了，它可以看做是 Invoker 集合，且这个集合中的元素会随注册中心的变化而进行动态调整。

```java
    //父类AbstractDirectory 中的list方法是个模板方法
		@Override
    public List<Invoker<T>> list(Invocation invocation) throws RpcException {
        if (destroyed) {
            throw new RpcException("Directory already destroyed .url: " + getUrl());
        }
				//交由子类实现
      	//有两个子类实现，动态/静态。
        return doList(invocation);
    }
```

##### 动态服务字典

先看一下动态服务字典的属性

```java
//RegistryDirectory 属性
public class RegistryDirectory<T> extends AbstractDirectory<T> implements NotifyListener {

    private static final Logger logger = LoggerFactory.getLogger(RegistryDirectory.class);

  	//集群自适应扩展的adapter 默认failover
    private static final Cluster CLUSTER = ExtensionLoader.getExtensionLoader(Cluster.class).getAdaptiveExtension();
		//路由工厂的自适应扩展adapter
    private static final RouterFactory ROUTER_FACTORY = ExtensionLoader.getExtensionLoader(RouterFactory.class)
            .getAdaptiveExtension();

  	//服务key，默认为服务接口名。com.alibaba.dubbo.registry.RegistryService.
    private final String serviceKey; 
  	//服务提供者接口类。
    private final Class<T> serviceType; 
  	//服务消费者URL中的所有属性。
    private final Map<String, String> queryMap; 
  	//注册中心URL 
    private final URL directoryUrl; 
  	//是否引用多个服务组 TODO
    private final boolean multiGroup;
  	//协议
    private Protocol protocol;
  	//注册中心实例
    private Registry registry; 
    private volatile boolean forbidden = false;

    private volatile URL overrideDirectoryUrl; // Initialization at construction time, assertion not null, and always assign non null value

    private volatile URL registeredConsumerUrl;

    /**
     * override rules
     * Priority: override>-D>consumer>provider
     * Rule one: for a certain provider <ip:port,timeout=100>
     * Rule two: for all providers <* ,timeout=5000>
     */
    private volatile List<Configurator> configurators; // The initial value is null and the midway may be assigned to null, please use the local variable reference

    // 服务端provider的url对应invoker。
    private volatile Map<String, Invoker<T>> urlInvokerMap;
  	//invoker列表
    private volatile List<Invoker<T>> invokers;

    // Set<invokerUrls> cache invokeUrls to invokers mapping.
    private volatile Set<URL> cachedInvokerUrls; // The initial value is null and the midway may be assigned to null, please use the local variable reference

    private static final ConsumerConfigurationListener CONSUMER_CONFIGURATION_LISTENER = new ConsumerConfigurationListener();
    private ReferenceConfigurationListener serviceConfigurationListener;
}
```

doList

```java
    @Override
    public List<Invoker<T>> doList(Invocation invocation) {
      	//服务提供者关闭或者禁用了服务，抛出FORBIDDEN_EXCEPTION异常。
        if (forbidden) {
            // 1. No service provider 2. Service providers are disabled
            throw new RpcException(RpcException.FORBIDDEN_EXCEPTION, "No provider available from registry " +
                    getUrl().getAddress() + " for service " + getConsumerUrl().getServiceKey() + " on consumer " +
                    NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() +
                    ", please check status of providers(disabled, not registered or in blacklist).");
        }
				//如果引用多个服务组，就直接返回invokers
        if (multiGroup) {
            return this.invokers == null ? Collections.emptyList() : this.invokers;
        }

        List<Invoker<T>> invokers = null;
        try {
            //根据运行时参数动态进行服务路由.
            invokers = routerChain.route(getConsumerUrl(), invocation);
        } catch (Throwable t) {
            logger.error("Failed to execute router: " + getUrl() + ", cause: " + t.getMessage(), t);
        }
      
        return invokers == null ? Collections.emptyList() : invokers;
    }
```

那么现在关键就在于这个invokers的生成。

查看invokers的usage，入口在notify通知方法。也就是RegistryDirectory是NotifyListener接口，可以获取到注册中心的配置修改通知。

notify方法：

```java
    //RegistryDirectory
		@Override
    public synchronized void notify(List<URL> urls) {
      	//从配置中心取到的providers的urls
      	//遍历。过滤。按configurators、routers、providers分组
        Map<String, List<URL>> categoryUrls = urls.stream()
                .filter(Objects::nonNull)
                .filter(this::isValidCategory)
                .filter(this::isNotCompatibleFor26x)
                .collect(Collectors.groupingBy(url -> {
                    if (UrlUtils.isConfigurator(url)) {
                        return CONFIGURATORS_CATEGORY;
                    } else if (UrlUtils.isRoute(url)) {
                        return ROUTERS_CATEGORY;
                    } else if (UrlUtils.isProvider(url)) {
                        return PROVIDERS_CATEGORY;
                    }
                    return "";
                }));
				//map中获取configurators TODO
        List<URL> configuratorURLs = categoryUrls.getOrDefault(CONFIGURATORS_CATEGORY, Collections.emptyList());
        this.configurators = Configurator.toConfigurators(configuratorURLs).orElse(this.configurators);

      	//map中获取routers TODO
        List<URL> routerURLs = categoryUrls.getOrDefault(ROUTERS_CATEGORY, Collections.emptyList());
        toRouters(routerURLs).ifPresent(this::addRouters);

        // map中获取providers 重新刷新invokers
        List<URL> providerURLs = categoryUrls.getOrDefault(PROVIDERS_CATEGORY, Collections.emptyList());
        refreshOverrideAndInvoker(providerURLs);
    }
```

```java
    //RegistryDirectory
		private void refreshOverrideAndInvoker(List<URL> urls) {
        // mock zookeeper://xxx?mock=return null
      	// TODO
        overrideDirectoryUrl();
      	//刷新invokers
        refreshInvoker(urls);
    }
```

```java
    //RegistryDirectory
		private void refreshInvoker(List<URL> invokerUrls) {
        Assert.notNull(invokerUrls, "invokerUrls should not be null");
				
      	//传过来一个url，empty://协议的，清空所有invoker
        if (invokerUrls.size() == 1
                && invokerUrls.get(0) != null
                && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
          	//设置forbidden为true
            this.forbidden = true; 
          	//清空invokers
            this.invokers = Collections.emptyList();
            routerChain.setInvokers(this.invokers);
          	//销毁所有invokers
            destroyAllInvokers(); 
        } else {
            this.forbidden = false; // Allow to access
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
          	//urls为空
            if (invokerUrls == Collections.<URL>emptyList()) {
                invokerUrls = new ArrayList<>();
            }
          	//urls为空，但是缓存中有，从缓存中取。
            if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
              	//设置缓存
                this.cachedInvokerUrls = new HashSet<>();
                this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
            }
            if (invokerUrls.isEmpty()) {
                return;
            }
          	//urls 转Invokers
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

            //空就打印错误日志返回。这里不是报错。
            if (CollectionUtils.isEmptyMap(newUrlInvokerMap)) {
                logger.error(new IllegalStateException("urls to invokers error .invokerUrls.size :" + invokerUrls.size() + ", invoker.size :0. urls :" + invokerUrls
                        .toString()));
                return;
            }
						
          	//设置为不可修改集合。放入invokers属性。
            List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
            // pre-route and build cache, notice that route cache should build on original Invoker list.
            // toMergeMethodInvokerMap() will wrap some invokers having different groups, those wrapped invokers not should be routed.
            routerChain.setInvokers(newInvokers);
            this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
            this.urlInvokerMap = newUrlInvokerMap;

          	//销毁无用的Invoker
            try {
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }
```

###### urls转Invokers

```java
		//RegistryDirectory
    private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
        Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<>();
        if (urls == null || urls.isEmpty()) {
            return newUrlInvokerMap;
        }
        Set<String> keys = new HashSet<>();
      	//获取服务消费端接收的服务协议
        String queryProtocols = this.queryMap.get(PROTOCOL_KEY);
        for (URL providerUrl : urls) {
            // 如果消费端配置了服务协议。则两边的服务协议要相同。
            if (queryProtocols != null && queryProtocols.length() > 0) {
                boolean accept = false;
                String[] acceptProtocols = queryProtocols.split(",");
                for (String acceptProtocol : acceptProtocols) {
                    if (providerUrl.getProtocol().equals(acceptProtocol)) {
                        accept = true;
                        break;
                    }
                }
                if (!accept) {
                    continue;
                }
            }
          	//provider的url不能是empty协议的
            if (EMPTY_PROTOCOL.equals(providerUrl.getProtocol())) {
                continue;
            }
          	//spi检测 消费端需要有对应的协议扩展 不然报错
            if (!ExtensionLoader.getExtensionLoader(Protocol.class).hasExtension(providerUrl.getProtocol())) {
                logger.error(new IllegalStateException("Unsupported protocol " + providerUrl.getProtocol() +
                        " in notified url: " + providerUrl + " from registry " + getUrl().getAddress() +
                        " to consumer " + NetUtils.getLocalHost() + ", supported protocol: " +
                        ExtensionLoader.getExtensionLoader(Protocol.class).getSupportedExtensions()));
                continue;
            }
          	//合并url
          	//增加了一些消费侧参数
          	//dubbo://127.0.0.1:20880/com.uteam.zen.api.IUserService?anyhost=true&application=system-service-dubbo&bean.name=ServiceBean:com.uteam.zen.api.IUserService&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IUserService&methods=loadUser,getUserById,delUser,addUser,getUserByOpenId,updateUser,getUserByName,queryUser&pid=95746&release=2.7.4.1&side=provider&timestamp=1592206611344
            //dubbo://127.0.0.1:20880/com.uteam.zen.api.IUserService?anyhost=true&application=zen.admin.web&bean.name=ServiceBean:com.uteam.zen.api.IUserService&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.uteam.zen.api.IUserService&lazy=false&methods=loadUser,getUserById,delUser,addUser,getUserByOpenId,updateUser,getUserByName,queryUser&pid=99739&register.ip=127.0.0.1&release=2.7.4.1&remote.application=system-service-dubbo&side=consumer&sticky=false&timestamp=1592206611344
          	URL url = mergeUrl(providerUrl);

            String key = url.toFullString(); 
          	//去重
            if (keys.contains(key)) {
                continue;
            }
            keys.add(key);
            // Cache key is url that does not merge with consumer side parameters, regardless of how the consumer combines parameters, if the server url changes, then refer again
            //本地的Invoker交给localUrlInvokerMap
          	Map<String, Invoker<T>> localUrlInvokerMap = this.urlInvokerMap; // local reference
          	//从本地获取。
            Invoker<T> invoker = localUrlInvokerMap == null ? null : localUrlInvokerMap.get(key);
            if (invoker == null) { 
                try {
                  	//有disabled就获取disabled
                  	//否则获取enabled参数 默认为enabled -> true
                    boolean enabled = true;
                    if (url.hasParameter(DISABLED_KEY)) {
                        enabled = !url.getParameter(DISABLED_KEY, false);
                    } else {
                        enabled = url.getParameter(ENABLED_KEY, true);
                    }
                    if (enabled) {
                      	//调用protocol.refer获取Invoker
                      	//生成对应的Invoker。如DubboInvoker【详见DubboInvoker】
                      	//InvokerDelegate是一个InvokerWrapper。封装调用。
                        invoker = new InvokerDelegate<>(protocol.refer(serviceType, url), url, providerUrl);
                    }
                } catch (Throwable t) {
                    logger.error("Failed to refer invoker for interface:" + serviceType + ",url:(" + url + ")" + t.getMessage(), t);
                }
              	//放入本地缓存
                if (invoker != null) { // Put new invoker in cache
                    newUrlInvokerMap.put(key, invoker);
                }
            } else {
              	//如果有就使用本地缓存
                newUrlInvokerMap.put(key, invoker);
            }
        }
        keys.clear();
        return newUrlInvokerMap;
    }
```

###### destroyUnusedInvokers删除无用Invoker

```java
    private void destroyUnusedInvokers(Map<String, Invoker<T>> oldUrlInvokerMap, Map<String, Invoker<T>> newUrlInvokerMap) {
        //新map为空，就清除所有
      	if (newUrlInvokerMap == null || newUrlInvokerMap.size() == 0) {
            destroyAllInvokers();
            return;
        }
        //对比新老，获取需要删除的invoker的url集合
        List<String> deleted = null;
        if (oldUrlInvokerMap != null) {
            Collection<Invoker<T>> newInvokers = newUrlInvokerMap.values();
            for (Map.Entry<String, Invoker<T>> entry : oldUrlInvokerMap.entrySet()) {
                if (!newInvokers.contains(entry.getValue())) {
                    if (deleted == null) {
                        deleted = new ArrayList<>();
                    }
                    deleted.add(entry.getKey());
                }
            }
        }

        if (deleted != null) {
            for (String url : deleted) {
                if (url != null) {
                  	//先从oldmap中删除
                    Invoker<T> invoker = oldUrlInvokerMap.remove(url);
                    if (invoker != null) {
                        try {
                          	//执行invoker的destroy方法
                            invoker.destroy();
                            if (logger.isDebugEnabled()) {
                                logger.debug("destroy invoker[" + invoker.getUrl() + "] success. ");
                            }
                        } catch (Exception e) {
                            logger.warn("destroy invoker[" + invoker.getUrl() + "] failed. " + e.getMessage(), e);
                        }
                    }
                }
            }
        }
    }
```









##### 静态服务字典

TODO





#### Cluster==服务集群==

![img](https://gitee.com/wangigor/typora-images/raw/master/dubbo-cluster.jpg)

> dubbo的集群容错，包含 Cluster、Cluster Invoker、Directory、Router 和 LoadBalance 组件
>
> 主要提供如下几种容错
>
> ```properties
> # mock是一个包裹类
> mock=org.apache.dubbo.rpc.cluster.support.wrapper.MockClusterWrapper
> failover=org.apache.dubbo.rpc.cluster.support.FailoverCluster
> failfast=org.apache.dubbo.rpc.cluster.support.FailfastCluster
> failsafe=org.apache.dubbo.rpc.cluster.support.FailsafeCluster
> failback=org.apache.dubbo.rpc.cluster.support.FailbackCluster
> forking=org.apache.dubbo.rpc.cluster.support.ForkingCluster
> available=org.apache.dubbo.rpc.cluster.support.AvailableCluster
> mergeable=org.apache.dubbo.rpc.cluster.support.MergeableCluster
> broadcast=org.apache.dubbo.rpc.cluster.support.BroadcastCluster
> registryaware=org.apache.dubbo.rpc.cluster.support.RegistryAwareCluster
> ```
>
> Cluster提供Cluster Invoker实例的生成【工厂类】。一个接口一个ClusterInvoker。
>
> ClusterInvoker里的Directory记录的接口的服务字典，且进行配置监听，为每一个provider的url创建Invoker【默认DubboInvoker】。
>
> 动态服务字典RegistryDirectory，使用路由Router，对url进行过滤。
>
> ClusterInvoker获取Directory的Invoker集合，通过LoadBalance组件选择Invoker。进行远程调用。

Cluster是ClusterInvoker的工厂类。

```java
//MockClusterWrapper
public class MockClusterWrapper implements Cluster {

    private Cluster cluster;

    public MockClusterWrapper(Cluster cluster) {
        this.cluster = cluster;
    }

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
      	//生成MockClusterInvoker
        return new MockClusterInvoker<T>(directory,
                this.cluster.join(directory));
    }

}
//FailoverCluster
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
      	//生成FailoverClusterInvoker
        return new FailoverClusterInvoker<T>(directory);
    }

}
//其他一样。
//不同点是MockClusterWrapper是包裹类，所以要传入Cluster
```

##### 父类AbstractClusterInvoker

> AbstractClusterInvoker是父类的抽象模板。提供了几个重要方法。

###### invoke调用入口

```java
		//父类抽象模板AbstractClusterInvoker
    @Override
    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();

        // binding attachments into invocation.
        Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addAttachments(contextAttachments);
        }
				
      	//列出所有的Invoker 
      	//这里list方法调用的directory的list方法。
        List<Invoker<T>> invokers = list(invocation);
      	//初始化负载均衡
      	//通过spi 获取invoker的url参数上的负载均衡参数。默认使用random
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
      
      	//TODO
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
      	
      	//进行集群调用【子类实现】
        return doInvoke(invocation, invokers, loadbalance);
    }
```

###### list所有Invoker

```java
		//list方法
		//获取服务字典中的Invoker集合
		protected List<Invoker<T>> list(Invocation invocation) throws RpcException {
        return directory.list(invocation);
    }
```

###### initLoadBalance初始化负载均衡策略

```java
		//初始化负载均衡策略
		protected LoadBalance initLoadBalance(List<Invoker<T>> invokers, Invocation invocation) {
        if (CollectionUtils.isNotEmpty(invokers)) {
            return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(invokers.get(0).getUrl()
                    .getMethodParameter(RpcUtils.getMethodName(invocation), LOADBALANCE_KEY, DEFAULT_LOADBALANCE));
        } else {
            return ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(DEFAULT_LOADBALANCE);
        }
    }
```

###### select选择invoker（粘滞）

```java
    protected Invoker<T> select(LoadBalance loadbalance, Invocation invocation,
                                List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
				//invokers为空，返回null
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
      	//远程调用方法名
        String methodName = invocation == null ? StringUtils.EMPTY : invocation.getMethodName();

      	//获取sticky配置【粘滞配置：消费者尽量调用同一个服务提供者，除非提供者挂掉】
      	//默认为false
        boolean sticky = invokers.get(0).getUrl()
                .getMethodParameter(methodName, CLUSTER_STICKY_KEY, DEFAULT_CLUSTER_STICKY);

        //如果存在stickyInvoker，但不在invokers列表中。
      	//说明粘滞的提供者已经挂了，清空
        if (stickyInvoker != null && !invokers.contains(stickyInvoker)) {
            stickyInvoker = null;
        }
        //在 sticky 为 true，且 stickyInvoker != null 的情况下。如果 selected 包含 
        // stickyInvoker，表明 stickyInvoker 对应的服务提供者可能因网络原因未能成功提供服务。
        // 但是该提供者并没挂，此时 invokers 列表中仍存在该服务提供者对应的 Invoker。
        if (sticky && stickyInvoker != null && (selected == null || !selected.contains(stickyInvoker))) {
          	//stickInvoker进行可用性检查
            if (availablecheck && stickyInvoker.isAvailable()) {
                return stickyInvoker;
            }
        }
				// 如果线程走到当前代码处，说明前面的 stickyInvoker 为空，或者不可用。
    		// 此时继续调用 doSelect 选择 Invoker
        Invoker<T> invoker = doSelect(loadbalance, invocation, invokers, selected);

      	// 如果 sticky 为 true，则将负载均衡组件选出的 Invoker 赋值给 stickyInvoker
        if (sticky) {
            stickyInvoker = invoker;
        }
        return invoker;
    }
```

###### doSelect选择invoker（负载均衡）

```java
    private Invoker<T> doSelect(LoadBalance loadbalance, Invocation invocation,
                                List<Invoker<T>> invokers, List<Invoker<T>> selected) throws RpcException {
				
      	//空
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
      	//只有一个
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
      	//调用负载均衡策略进行选择
        Invoker<T> invoker = loadbalance.select(invokers, getUrl(), invocation);

        //selected表示invoked调用过，但是失败了。
      	//如果选出的invoker在selected列表中，或者选出的invoker在开启可用性检查的情况下，不可用。重新选择。
        if ((selected != null && selected.contains(invoker))
                || (!invoker.isAvailable() && getUrl() != null && availablecheck)) {
            try {
              	//重新选择
                Invoker<T> rInvoker = reselect(loadbalance, invocation, invokers, selected, availablecheck);
                if (rInvoker != null) {
                  	//赋值给invoker
                    invoker = rInvoker;
                } else {
                  	//重新选择为空
                    int index = invokers.indexOf(invoker);
                    try {
                        //获取当前invoker的index+1的invoker
                      	//【取余是为了防止数组下标越界】
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
```

###### reselect 重新选择

```java
    private Invoker<T> reselect(LoadBalance loadbalance, Invocation invocation,
                                List<Invoker<T>> invokers, List<Invoker<T>> selected, boolean availablecheck) throws RpcException {

        //建议选择列表
        List<Invoker<T>> reselectInvokers = new ArrayList<>(
                invokers.size() > 1 ? (invokers.size() - 1) : invokers.size());

        // 把所有不在selected列表中的，且如果开启了可用性检查，通过了检查的。
      	//放入建议选择列表中
        for (Invoker<T> invoker : invokers) {
            if (availablecheck && !invoker.isAvailable()) {
                continue;
            }

            if (selected == null || !selected.contains(invoker)) {
                reselectInvokers.add(invoker);
            }
        }
				
      	//建议选择列表不为空，交给负载均衡器选择
        if (!reselectInvokers.isEmpty()) {
            return loadbalance.select(reselectInvokers, getUrl(), invocation);
        }

        // 把selected列表中的，重新进行可用性检查。
      	//可用的添加进建议选择列表中
        if (selected != null) {
            for (Invoker<T> invoker : selected) {
                if ((invoker.isAvailable()) // available first
                        && !reselectInvokers.contains(invoker)) {
                    reselectInvokers.add(invoker);
                }
            }
        }
      	//重新使用负载均衡器选择
        if (!reselectInvokers.isEmpty()) {
            return loadbalance.select(reselectInvokers, getUrl(), invocation);
        }

        return null;
    }
```

##### FailoverClusterInvoker 

> 【dubbo默认策略】调用失败，自动重选重发

```java
public class FailoverClusterInvoker<T> extends AbstractClusterInvoker<T> {

    private static final Logger logger = LoggerFactory.getLogger(FailoverClusterInvoker.class);

    public FailoverClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        //拷贝一份放在虚拟机栈中
      	List<Invoker<T>> copyInvokers = invokers;
      	//invokers为空抛异常
        checkInvokers(copyInvokers, invocation);
      	//获取远程调用方法名
        String methodName = RpcUtils.getMethodName(invocation);
      	
      	//获取重试次数
        int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
      
      
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
      	//循环调用，失败重试
        for (int i = 0; i < len; i++) {
            //不是第一次调用的情况
            if (i > 0) {
              	//重试之前，重新列举invokers
              	//如果某个服务挂了，可以重新获取到新的列表
                checkWhetherDestroyed();
                copyInvokers = list(invocation);
                // check again
                checkInvokers(copyInvokers, invocation);
            }
          	//选择一个invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
          	//invoker放到已调用列表中
            invoked.add(invoker);
          	//已调用列表放到rpc上下文中
            RpcContext.getContext().setInvokers((List) invoked);
            try {
              	//执行调用
                Result result = invoker.invoke(invocation);
                if (le != null && logger.isWarnEnabled()) {
                    logger.warn("Although retry the method " + methodName
                            + " in the service " + getInterface().getName()
                            + " was successful by the provider " + invoker.getUrl().getAddress()
                            + ", but there have been failed providers " + providers
                            + " (" + providers.size() + "/" + copyInvokers.size()
                            + ") from the registry " + directory.getUrl().getAddress()
                            + " on the consumer " + NetUtils.getLocalHost()
                            + " using the dubbo version " + Version.getVersion() + ". Last error is: "
                            + le.getMessage(), le);
                }
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
      	//所有重试都失败。抛异常。
        throw new RpcException(le.getCode(), "Failed to invoke the method "
                + methodName + " in the service " + getInterface().getName()
                + ". Tried " + len + " times of the providers " + providers
                + " (" + providers.size() + "/" + copyInvokers.size()
                + ") from the registry " + directory.getUrl().getAddress()
                + " on the consumer " + NetUtils.getLocalHost() + " using the dubbo version "
                + Version.getVersion() + ". Last error is: "
                + le.getMessage(), le.getCause() != null ? le.getCause() : le);
    }

}
```

##### FailbackClusterInvoker

> 调用失败，返回给客户端一个空result。把当前调用添加进失败队列，重发。

```java
public class FailbackClusterInvoker<T> extends AbstractClusterInvoker<T> {

		//{忽略}

  	//添加失败计划
    private void addFailed(LoadBalance loadbalance, Invocation invocation, List<Invoker<T>> invokers, Invoker<T> lastInvoker) {
        //初始化HashedWheelTimer
      	if (failTimer == null) {
            synchronized (this) {
                if (failTimer == null) {
                    failTimer = new HashedWheelTimer(
                            new NamedThreadFactory("failback-cluster-timer", true),
                            1,
                            TimeUnit.SECONDS, 32, failbackTasks);
                }
            }
        }
      	//创建重试任务
        RetryTimerTask retryTimerTask = new RetryTimerTask(loadbalance, invocation, invokers, lastInvoker, retries, RETRY_FAILED_PERIOD);
        try {
          	//放入timer延迟执行
            failTimer.newTimeout(retryTimerTask, RETRY_FAILED_PERIOD, TimeUnit.SECONDS);
        } catch (Throwable e) {
            logger.error("Failback background works error,invocation->" + invocation + ", exception: " + e.getMessage());
        }
    }
		
  	//调用入口
    @Override
    protected Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        Invoker<T> invoker = null;
        try {
          	//检查invokers非空
            checkInvokers(invokers, invocation);
          	//负载均衡选择一个invoker
            invoker = select(loadbalance, invocation, invokers, null);
          	//调用
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failback to invoke method " + invocation.getMethodName() + ", wait for retry in background. Ignored exception: "
                    + e.getMessage() + ", ", e);
          	//调用失败。添加失败计划
            addFailed(loadbalance, invocation, invokers, invoker);
          	//返回异步rpc结果
            return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
        }
    }


 		//重试任务
    private class RetryTimerTask implements TimerTask {
				//{忽略}

        @Override
        public void run(Timeout timeout) {
            try {
              	//选择invoker
                Invoker<T> retryInvoker = select(loadbalance, invocation, invokers, Collections.singletonList(lastInvoker));
                lastInvoker = retryInvoker;
              	//执行
                retryInvoker.invoke(invocation);
            } catch (Throwable e) {
                logger.error("Failed retry to invoke method " + invocation.getMethodName() + ", waiting again.", e);
                //超过重试次数。取消执行
              	if ((++retryTimes) >= retries) {
                    logger.error("Failed retry times exceed threshold (" + retries + "), We have to abandon, invocation->" + invocation);
                } else {
                  	//重新放入timer中执行。
                    rePut(timeout);
                }
            }
        }

        private void rePut(Timeout timeout) {
            if (timeout == null) {
                return;
            }

            Timer timer = timeout.timer();
            if (timer.isStop() || timeout.isCancelled()) {
                return;
            }

            timer.newTimeout(timeout.task(), tick, TimeUnit.SECONDS);
        }
    }
}
```



##### FailfastClusterInvoker

> 只调用一次。失败立刻抛出异常。

```java
public class FailfastClusterInvoker<T> extends AbstractClusterInvoker<T> {

    public FailfastClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
        try {
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            if (e instanceof RpcException && ((RpcException) e).isBiz()) { // biz exception.
                throw (RpcException) e;
            }
            throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0,
                    "Failfast invoke providers " + invoker.getUrl() + " " + loadbalance.getClass().getSimpleName()
                            + " select from all providers " + invokers + " for service " + getInterface().getName()
                            + " method " + invocation.getMethodName() + " on consumer " + NetUtils.getLocalHost()
                            + " use dubbo version " + Version.getVersion()
                            + ", but no luck to perform the invocation. Last error is: " + e.getMessage(),
                    e.getCause() != null ? e.getCause() : e);
        }
    }
}
```

##### FailsafeClusterInvoker

> 只调用一次。失败返回空result。

```java
public class FailsafeClusterInvoker<T> extends AbstractClusterInvoker<T> {
    private static final Logger logger = LoggerFactory.getLogger(FailsafeClusterInvoker.class);

    public FailsafeClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    public Result doInvoke(Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            Invoker<T> invoker = select(loadbalance, invocation, invokers, null);
            return invoker.invoke(invocation);
        } catch (Throwable e) {
            logger.error("Failsafe ignore exception: " + e.getMessage(), e);
            return AsyncRpcResult.newDefaultAsyncResult(null, null, invocation); // ignore
        }
    }
}
```

##### ForkingClusterInvoker

> 通过线程池，并发调用多个服务者。
>
> 有其中一个返回，就终止调用。

```java
public class ForkingClusterInvoker<T> extends AbstractClusterInvoker<T> {

    /**
     * Use {@link NamedInternalThreadFactory} to produce {@link org.apache.dubbo.common.threadlocal.InternalThread}
     * which with the use of {@link org.apache.dubbo.common.threadlocal.InternalThreadLocal} in {@link RpcContext}.
     */
    private final ExecutorService executor = Executors.newCachedThreadPool(
            new NamedInternalThreadFactory("forking-cluster-timer", true));

    public ForkingClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        try {
            checkInvokers(invokers, invocation);
            final List<Invoker<T>> selected;
          	//获取forks参数【多少个分支调用】
            final int forks = getUrl().getParameter(FORKS_KEY, DEFAULT_FORKS);
          	//获取超时时间
            final int timeout = getUrl().getParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
          	//如果forks配置不合理，就使用全部的invokers
            if (forks <= 0 || forks >= invokers.size()) {
                selected = invokers;
            } else {
              	//进行forks次选择。
                selected = new ArrayList<>();
                for (int i = 0; i < forks; i++) {
                    Invoker<T> invoker = select(loadbalance, invocation, invokers, selected);
                    if (!selected.contains(invoker)) {
                        //Avoid add the same invoker several times.
                        selected.add(invoker);
                    }
                }
            }
            RpcContext.getContext().setInvokers((List) selected);
            final AtomicInteger count = new AtomicInteger();
          	//新建阻塞队列
            final BlockingQueue<Object> ref = new LinkedBlockingQueue<>();
            for (final Invoker<T> invoker : selected) {
                executor.execute(() -> {
                    try {
                      	//每个invoker都执行远程调用
                        Result result = invoker.invoke(invocation);
                      	//调用结果放入队列中
                        ref.offer(result);
                    } catch (Throwable e) {
                      	// 仅在 value 大于等于 selected.size() 时，才将异常对象放入阻塞队列中
                       	// 为了让正常返回，优先进队列
                        int value = count.incrementAndGet();
                        if (value >= selected.size()) {
                            ref.offer(e);
                        }
                    }
                });
            }
            try {
              	//队列获取第一个
                Object ret = ref.poll(timeout, TimeUnit.MILLISECONDS);
              	//如果是异常，就抛出异常
                if (ret instanceof Throwable) {
                    Throwable e = (Throwable) ret;
                    throw new RpcException(e instanceof RpcException ? ((RpcException) e).getCode() : 0, "Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e.getCause() != null ? e.getCause() : e);
                }
              	//否则返回正常结果。
                return (Result) ret;
            } catch (InterruptedException e) {
                throw new RpcException("Failed to forking invoke provider " + selected + ", but no luck to perform the invocation. Last error is: " + e.getMessage(), e);
            }
        } finally {
            // clear attachments which is binding to current thread.
            RpcContext.getContext().clearAttachments();
        }
    }
}
```

##### BroadcastClusterInvoker

> 广播所有provider。有一台异常。抛出异常。

```java
public class BroadcastClusterInvoker<T> extends AbstractClusterInvoker<T> {

    private static final Logger logger = LoggerFactory.getLogger(BroadcastClusterInvoker.class);

    public BroadcastClusterInvoker(Directory<T> directory) {
        super(directory);
    }

    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public Result doInvoke(final Invocation invocation, List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        checkInvokers(invokers, invocation);
        RpcContext.getContext().setInvokers((List) invokers);
        RpcException exception = null;
        Result result = null;
      	//调用所有的invoker
        for (Invoker<T> invoker : invokers) {
            try {
                result = invoker.invoke(invocation);
            } catch (RpcException e) {
                exception = e;
                logger.warn(e.getMessage(), e);
            } catch (Throwable e) {
                exception = new RpcException(e.getMessage(), e);
                logger.warn(e.getMessage(), e);
            }
        }
      	//有异常抛异常
        if (exception != null) {
            throw exception;
        }
        return result;
    }

}
```





## 服务调用

> 在服务引入的时候，通过ClassGenerator，生成了IUserService的代理proxy0。
>
> 源码在上文中有。
>
> 例如 ，调用getUserByName方法。

```java
//1.调用proxy0
public class proxy0 implements ClassGenerator.DC,EchoService,IUserService{
    public UserDto getUserByName(String string) {
        Object[] arrobject = new Object[]{string};
        Object object = this.handler.invoke(this, methods[7], arrobject);
        return (UserDto)object;
    }
}
//2.调用InvokerInvocationHandler的invoke
public class InvokerInvocationHandler implements InvocationHandler {
  	//{忽略其他}
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
				//2.1 进行rpc调用封装 【RpcInvocation】
      	//2.2 调用invoke
      	//2.3 获取返回recreate
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
}
```



> 前面提过：这个InvokerInvocationHandler默认是FailoverClusterInvoker，并由MockClusterWrapper包裹。
>
> 所以，真实调用的是MockClusterInvoker类。这里封装了mock和服务降级逻辑。

```java
    // public class MockClusterInvoker<T> implements Invoker<T>
		@Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;

      	//获取mock 的配置参数
        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), MOCK_KEY, Boolean.FALSE.toString()).trim();
       	
      	if (value.length() == 0 || "false".equalsIgnoreCase(value)) {
            //直接远程调用 不mock.
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
            if (logger.isWarnEnabled()) {
                logger.warn("force-mock: " + invocation.getMethodName() + " force-mock enabled , url : " + directory.getUrl());
            }
            //force:direct mock
          	//直接mock。不调用。
            result = doMockInvoke(invocation, null);
        } else {
            //fail-mock
          	//远程失败再mock。
            try {
                result = this.invoker.invoke(invocation);

                //fix:#4585
                if(result.getException() != null && result.getException() instanceof RpcException){
                    RpcException rpcException= (RpcException)result.getException();
                    if(rpcException.isBiz()){
                        throw  rpcException;
                    }else {
                        result = doMockInvoke(invocation, rpcException);
                    }
                }

            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                }

                if (logger.isWarnEnabled()) {
                    logger.warn("fail-mock: " + invocation.getMethodName() + " fail-mock enabled , url : " + directory.getUrl(), e);
                }
                result = doMockInvoke(invocation, e);
            }
        }
        return result;
    }
```

### 远程调用

> result = this.invoker.invoke(invocation);
>
> MockClusterInvoker是对FailoverClusterInvoker的包裹。然后经过负载均衡，调用真实的DubboInvoker。
>
> ![image-20200702154031102](https://gitee.com/wangigor/typora-images/raw/master/dubbinvoker-rpcInvocation-debug.png)
>
> 从调用方法开始。经过了dubbo默认的javassist生成的代理，拦截方法->InvokerInvocationHandler执行经过mork，经过Failover服务集群，经过负载均衡器的选择，最终实现invoker的调用。DubboInvoker也是经过了各种包裹和代理。这个稍后再看，直接看调用。
>
> RpcInvocation记录了当前这一次调用的所有信息。包括方法名，service类型，请求参数，选择的invoker，返回类型，调用模式等。

```java
    
		//DubboInvoker extends AbstractInvoker
		protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
      	//调用目标方法名
        final String methodName = RpcUtils.getMethodName(invocation);
      	//设置目标类的全限定名
        inv.setAttachment(PATH_KEY, getUrl().getPath());
      	//设置版本号 默认0.0.0
        inv.setAttachment(VERSION_KEY, version);

      	//获取网络通信客户端
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
      	
        try {
          	//isOneway为true。表示 单向 通信。异步无返回值
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
          	//获取超时时间
            int timeout = getUrl().getMethodPositiveParameter(methodName, TIMEOUT_KEY, DEFAULT_TIMEOUT);
          	//异步无返回值
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
              	//发送请求
                currentClient.send(inv, isSent);
              	//返回一个空的RpcResult
                return AsyncRpcResult.newDefaultAsyncResult(invocation);
            } else {
              	//有返回值
                AsyncRpcResult asyncRpcResult = new AsyncRpcResult(inv);
              	//发送请求。返回一个CompletableFuture
                CompletableFuture<Object> responseFuture = currentClient.request(inv, timeout);
              	//rpc结果对CompletableFuture进行订阅。
                asyncRpcResult.subscribeTo(responseFuture);
                // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
                //放到请求上下文中
              	FutureContext.getContext().setCompatibleFuture(responseFuture);
                return asyncRpcResult;
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

#### 网络通信

> 发送请求currentClient.request(inv, timeout);
>
> ![image-20200702170722504](https://gitee.com/wangigor/typora-images/raw/master/dubbo-invoker-trace.png)
>
> 调用栈从DubboInvoker开始，经过了ReferenceCountExchangeClient、HeaderExchangeClient、HeaderExchangeChannel、NettyChannel，猜到了netty的消息发送。

网络通信采用C/S模式的传统架构，有几个概念需要先理清楚。

##### ==几个重要组件==

###### EndPoint 端

> Endpoint是个抽象的概念，大概是一个**能够发生网络通信的节点**。端 是通道（Channel）、客户端（Client）、服务端（Server）三者的抽象。
>
> - 通过url进行身份区分，通道和客户端中的url是同一个引用。
> - 记录了可以进行网络通信的本机真实物理网络地址     InetSocketAddress 。
> - 能够主动发起消息传递操作。
> - 具有感知通信事件的能力 ChannelHandler。

```java
public interface Endpoint {

    URL getUrl();

    ChannelHandler getChannelHandler();

    InetSocketAddress getLocalAddress();

    void send(Object message) throws RemotingException;

    void send(Object message, boolean sent) throws RemotingException;

    void close();

    void close(int timeout);

    void startClose();

    boolean isClosed();

}
```

###### Channel 通道

> Channel通道扩展了Endpoint端接口。
>
> - 可以向Channel写入或者从中读取上下文本地属性
>
> - 从Channel的一侧能够获取到另一侧的物理网络地址
>
> - 能够检测当前Channel的双端是否还处于连接状态

```java
public interface Channel extends Endpoint {

    InetSocketAddress getRemoteAddress();

    boolean isConnected();

    boolean hasAttribute(String key);

    Object getAttribute(String key);

    void setAttribute(String key, Object value);

    void removeAttribute(String key);
}
```



###### Client客户端和Server服务端

> 客户端Client继承了Channel，服务端Server成员属性里存了Channel集合。
>
> 都实现了Resetable和IdleSensible接口。
>
> - Resetable接口用于重置本地上下文中的参数
>
> ```java
> public interface Resetable {
> 
>     void reset(URL url);
> }
> ```
>
> - IdleSensible接口标记了是否对空闲连接进行操作，具体的操作要各自实现，嵌入到自己的逻辑中。
>
> client发送心跳，server关闭连接。
>
> ```java
> public interface IdleSensible {
>     default boolean canHandleIdle() {
>         return false;
>     }
> }
> ```
>
> 目前这个接口的实现都是返回true。



```java
//客户端
public interface Client extends Endpoint, Channel, Resetable, IdleSensible {

    void reconnect() throws RemotingException;
}
```

```java
//服务端
public interface Server extends Endpoint, Resetable, IdleSensible {

    boolean isBound();

    Collection<Channel> getChannels();

    Channel getChannel(InetSocketAddress remoteAddress);

}
```

###### ChannelHandler通道事件处理器

> Channel是Client和Server的传输通道，通讯过程中会发生**连接、断连、发送数据、接收数据、异常捕获**5中事件。

```java
public interface ChannelHandler {

    void connected(Channel channel) throws RemotingException;

    void disconnected(Channel channel) throws RemotingException;

    void sent(Channel channel, Object message) throws RemotingException;

    void received(Channel channel, Object message) throws RemotingException;

    void caught(Channel channel, Throwable exception) throws RemotingException;

}
```





##### ReferenceCountExchangeClient 引用计数器

> 维护了当前网络通信客户端的引用计数器。有几个invoker引用就是几。全部invoker销毁【调用close】，exchangeClient销毁。

```java
final class ReferenceCountExchangeClient implements ExchangeClient {

  	//invoker的url
    private final URL url;
  	//原子计数
    private final AtomicInteger referenceCount = new AtomicInteger(0);

  	//HeaderExchangeClient
    private ExchangeClient client;

    public ReferenceCountExchangeClient(ExchangeClient client) {
        this.client = client;
      	//引用计数器递增
        referenceCount.incrementAndGet();
        this.url = client.getUrl();
    }

  	//{省略}

  	//发送请求
    @Override
    public CompletableFuture<Object> request(Object request, int timeout) throws RemotingException {
        return client.request(request, timeout);
    }


    //立即关闭
    @Override
    public void close() {
        close(0);
    }
		//延迟关闭
    @Override
    public void close(int timeout) {
      	//引用计数递减
        if (referenceCount.decrementAndGet() <= 0) {
          	//到0，执行销毁
            if (timeout == 0) {
                client.close();

            } else {
                client.close(timeout);
            }

            replaceWithLazyClient();
        }
    }

    //当client被关闭时，会把client设置成LazyConnectExchangeClient，等待新的调用者来唤醒。
    private void replaceWithLazyClient() {
        // this is a defensive operation to avoid client is closed by accident, the initial state of the client is false
        URL lazyUrl = URLBuilder.from(url)
                .addParameter(LAZY_CONNECT_INITIAL_STATE_KEY, Boolean.FALSE)
                .addParameter(RECONNECT_KEY, Boolean.FALSE)
                .addParameter(SEND_RECONNECT_KEY, Boolean.TRUE.toString())
                .addParameter("warning", Boolean.TRUE.toString())
                .addParameter(LazyConnectExchangeClient.REQUEST_WITH_WARNING_KEY, true)
                .addParameter("_client_memo", "referencecounthandler.replacewithlazyclient")
                .build();

        /**
         * the order of judgment in the if statement cannot be changed.
         */
        if (!(client instanceof LazyConnectExchangeClient) || client.isClosed()) {
            client = new LazyConnectExchangeClient(lazyUrl, client.getExchangeHandler());
        }
    }


    //供外部调用的引用计数自增方法
    public void incrementAndGetCount() {
        referenceCount.incrementAndGet();
    }
}
```

##### HeaderExchangeClient心跳检测

> TODO 这里先不看

```java
public class HeaderExchangeClient implements ExchangeClient {

  	//nettyClient
    private final Client client;
    private final ExchangeChannel channel;
		
    private static final HashedWheelTimer IDLE_CHECK_TIMER = new HashedWheelTimer(
            new NamedThreadFactory("dubbo-client-idleCheck", true), 1, TimeUnit.SECONDS, TICKS_PER_WHEEL);
    private HeartbeatTimerTask heartBeatTimerTask;
    private ReconnectTimerTask reconnectTimerTask;

    public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);

        if (startTimer) {
            URL url = client.getUrl();
            startReconnectTask(url);
            startHeartBeatTask(url);
        }
    }

  	//调用channel发送请求
    @Override
    public CompletableFuture<Object> request(Object request) throws RemotingException {
        return channel.request(request);
    }

   
		//开始心跳检测任务
    private void startHeartBeatTask(URL url) {
        if (!client.canHandleIdle()) {
            AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
            int heartbeat = getHeartbeat(url);
            long heartbeatTick = calculateLeastDuration(heartbeat);
            this.heartBeatTimerTask = new HeartbeatTimerTask(cp, heartbeatTick, heartbeat);
            IDLE_CHECK_TIMER.newTimeout(heartBeatTimerTask, heartbeatTick, TimeUnit.MILLISECONDS);
        }
    }

  	//开始重连任务。
    private void startReconnectTask(URL url) {
        if (shouldReconnect(url)) {
            AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
            int idleTimeout = getIdleTimeout(url);
            long heartbeatTimeoutTick = calculateLeastDuration(idleTimeout);
            this.reconnectTimerTask = new ReconnectTimerTask(cp, heartbeatTimeoutTick, idleTimeout);
            IDLE_CHECK_TIMER.newTimeout(reconnectTimerTask, heartbeatTimeoutTick, TimeUnit.MILLISECONDS);
        }
    }
  
  	//关闭
  	private void doClose() {
        if (heartBeatTimerTask != null) {
            heartBeatTimerTask.cancel();
        }

        if (reconnectTimerTask != null) {
            reconnectTimerTask.cancel();
        }
    }

    //计算周期。不能小于1s
    private long calculateLeastDuration(int time) {
        if (time / HEARTBEAT_CHECK_TICK <= 0) {
            return LEAST_HEARTBEAT_DURATION;
        } else {
            return time / HEARTBEAT_CHECK_TICK;
        }
    }

}
```

##### HeaderExchangeChannel 

> HeaderExchangeClient只负责心跳检测，实际的请求发送交给HeaderExchangeChannel处理。
>
> HeaderExchangeChannel负责：
>
> - RpcInvocation转Request
> - 同步转异步，返回创建DefaultFuture
> - 使用channel【默认NettyClient】发送请求

```java
final class HeaderExchangeChannel implements ExchangeChannel {

    private static final String CHANNEL_KEY = HeaderExchangeChannel.class.getName() + ".CHANNEL";
		
  	//NettyClient
    private final Channel channel;

    private volatile boolean closed = false;

  	//发送请求-重写超时
    @Override
    public CompletableFuture<Object> request(Object request) throws RemotingException {
        return request(request, channel.getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT));
    }
		//发送请求
    @Override
    public CompletableFuture<Object> request(Object request, int timeout) throws RemotingException {
      	//检查channel是否关闭
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        //封装请求
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);//双向通信 为 true
        req.setData(request);//data 是整个RpcInvocation
      	//创建DefaultFuture返回
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout);
        try {
            channel.send(req);//调用NettyClient的send发送请求
        } catch (RemotingException e) {
            future.cancel();//异常 调用future的取消
            throw e;
        }
        return future;
    }
		
  	//忽略...

}
```

###### DefaultFuture

> DefaultFuture 继承了java的CompletableFuture
>
> 没多少东西
>
> - 类上有静态的id->channel的映射集合、id->future的映射集合、超时计时器。
>
> - 实例上记录了超时任务，channel，请求信息，开始结束时间。

```java
public class DefaultFuture extends CompletableFuture<Object> {
		
  	//请求id -> channel 映射
    private static final Map<Long, Channel> CHANNELS = new ConcurrentHashMap<>();
		//请求id -> DefaultFuture 映射
    private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();

    public static final Timer TIME_OUT_TIMER = new HashedWheelTimer(
            new NamedThreadFactory("dubbo-future-timeout", true),
            30,
            TimeUnit.MILLISECONDS);

    // 请求id
    private final Long id;
    private final Channel channel;
    private final Request request;
    private final int timeout;//超时时间
    private final long start = System.currentTimeMillis();//开始时间
    private volatile long sent;//接收到回复的时间
    private Timeout timeoutCheckTask;


    public static DefaultFuture newFuture(Channel channel, Request request, int timeout) {
        final DefaultFuture future = new DefaultFuture(channel, request, timeout);
        // timeout check
        timeoutCheck(future);
        return future;
    }

		//收到回复
    public static void sent(Channel channel, Request request) {
        DefaultFuture future = FUTURES.get(request.getId());
        if (future != null) {
            future.doSent();
        }
    }

   	//channel关闭时，需要把所有的channel对应的request对应的Future，置为通道不活跃的错误回复
    public static void closeChannel(Channel channel) {
        for (Map.Entry<Long, Channel> entry : CHANNELS.entrySet()) {
            if (channel.equals(entry.getValue())) {
                DefaultFuture future = getFuture(entry.getKey());
                if (future != null && !future.isDone()) {
                    Response disconnectResponse = new Response(future.getId());
                    disconnectResponse.setStatus(Response.CHANNEL_INACTIVE);
                    disconnectResponse.setErrorMessage("Channel " +
                            channel +
                            " is inactive. Directly return the unFinished request : " +
                            future.getRequest());
                    DefaultFuture.received(channel, disconnectResponse);
                }
            }
        }
    }
		
  	//接收方法 重载
    public static void received(Channel channel, Response response) {
        received(channel, response, false);
    }

  	//接收方法
    public static void received(Channel channel, Response response, boolean timeout) {
        try {
            DefaultFuture future = FUTURES.remove(response.getId());
            if (future != null) {
                Timeout t = future.timeoutCheckTask;
                if (!timeout) {
                    // decrease Time
                    t.cancel();
                }
                future.doReceived(response);
            } else {
                logger.warn("The timeout response finally returned at "
                        + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                        + ", response " + response
                        + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                        + " -> " + channel.getRemoteAddress()));
            }
        } finally {
            CHANNELS.remove(response.getId());
        }
    }

  	//future的取消 
    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {
        Response errorResult = new Response(id);
        errorResult.setStatus(Response.CLIENT_ERROR);
        errorResult.setErrorMessage("request future has been canceled.");
        this.doReceived(errorResult);
        FUTURES.remove(id);
        CHANNELS.remove(id);
        return true;
    }


    private void doReceived(Response res) {
        if (res == null) {
            throw new IllegalStateException("response cannot be null");
        }
        if (res.getStatus() == Response.OK) {
            this.complete(res.getResult());
        } else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
            this.completeExceptionally(new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage()));
        } else {
            this.completeExceptionally(new RemotingException(channel, res.getErrorMessage()));
        }
    }
		//future可以get到返回时，记录返回时间
    private void doSent() {
        sent = System.currentTimeMillis();
    }
		//返回超时信息
    private String getTimeoutMessage(boolean scan) {
        long nowTimestamp = System.currentTimeMillis();
        return (sent > 0 ? "Waiting server-side response timeout" : "Sending request timeout in client-side")
                + (scan ? " by scan timer" : "") + ". start time: "
                + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date(start))) + ", end time: "
                + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date())) + ","
                + (sent > 0 ? " client elapsed: " + (sent - start)
                + " ms, server elapsed: " + (nowTimestamp - sent)
                : " elapsed: " + (nowTimestamp - start)) + " ms, timeout: "
                + timeout + " ms, request: " + (logger.isDebugEnabled() ? request : getRequestWithoutData()) + ", channel: " + channel.getLocalAddress()
                + " -> " + channel.getRemoteAddress();
    }

		//超时检测任务 内部类
    private static class TimeoutCheckTask implements TimerTask {

        private final Long requestID;

        TimeoutCheckTask(Long requestID) {
            this.requestID = requestID;
        }

      	//到了超时时间，如果future还没有结束。就封装超时返回
        public void run(Timeout timeout) {
            DefaultFuture future = DefaultFuture.getFuture(requestID);
            if (future == null || future.isDone()) {
                return;
            }
            // create exception response.
            Response timeoutResponse = new Response(future.getId());
            // set timeout status.
            timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
            timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
            // handle response.
            DefaultFuture.received(future.getChannel(), timeoutResponse, true);

        }
    }
```

##### NettyClient

> dubbo封装的netty4客户端
>
> ![image-20200703141638657](https://gitee.com/wangigor/typora-images/raw/master/dubbo-netty-client-hierarchy.png)
>
> 

```java
public abstract class AbstractPeer implements Endpoint, ChannelHandler {

  	//通道事件处理Handler
    private final ChannelHandler handler;
		//服务提供者url
    private volatile URL url;
		
  	//调用子类的发送
    @Override
    public void send(Object message) throws RemotingException {
        send(message, url.getParameter(Constants.SENT_KEY, false));
    }
  	//忽略...
}
```

```java
public abstract class AbstractClient extends AbstractEndpoint implements Client {
		
  	//用于连接用的锁
    private final Lock connectLock = new ReentrantLock();
    private final boolean needReconnect;
  	//这个线程池 除了当前实例的close方法用，其他没地方用。
    protected volatile ExecutorService executor;

    @Override
    public void send(Object message, boolean sent) throws RemotingException {
      	//需要重新连接
        if (needReconnect && !isConnected()) {
            connect();
        }
      	//获取连接
        Channel channel = getChannel();
        
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
        }
      	//用NettyChannel发送
        channel.send(message, sent);
    }
}
```

###### 获取channel通道

> 使用Netty长连接，只有当前通道关闭，才会重连。
>
> 就看getChannel()方法获取连接。

```java
public class NettyClient extends AbstractClient {

  	//Netty客户端的EventLoopGroup 默认min(核心线程数+1,32)
    private static final NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup(Constants.DEFAULT_IO_THREADS, new DefaultThreadFactory("NettyClientWorker", true));

  	//socks代理相关的属性 从系统变量里获取
    private static final String SOCKS_PROXY_HOST = "socksProxyHost";
    private static final String SOCKS_PROXY_PORT = "socksProxyPort";
    private static final String DEFAULT_SOCKS_PROXY_PORT = "1080";

    private Bootstrap bootstrap;

   	//当前连接的channel
    private volatile Channel channel;

    //NettyClient构造器 父模板会调用doOpen、doConnent方法
    public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
    	// the handler will be warped: MultiMessageHandler->HeartbeatHandler->handler
    	super(url, wrapChannelHandler(url, handler));
    }

    //初始化Netty的Bootstrap
    @Override
    protected void doOpen() throws Throwable {
        final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
        bootstrap = new Bootstrap();
        bootstrap.group(nioEventLoopGroup)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                .channel(NioSocketChannel.class);

        bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, Math.max(3000, getConnectTimeout()));
        bootstrap.handler(new ChannelInitializer() {

            @Override
            protected void initChannel(Channel ch) throws Exception {
                int heartbeatInterval = UrlUtils.getHeartbeat(getUrl());
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        .addLast("decoder", adapter.getDecoder())
                        .addLast("encoder", adapter.getEncoder())
                        .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
                        .addLast("handler", nettyClientHandler);
                String socksProxyHost = ConfigUtils.getProperty(SOCKS_PROXY_HOST);
                if(socksProxyHost != null) {
                    int socksProxyPort = Integer.parseInt(ConfigUtils.getProperty(SOCKS_PROXY_PORT, DEFAULT_SOCKS_PROXY_PORT));
                    Socks5ProxyHandler socks5ProxyHandler = new Socks5ProxyHandler(new InetSocketAddress(socksProxyHost, socksProxyPort));
                    ch.pipeline().addFirst(socks5ProxyHandler);
                }
            }
        });
    }

  	//获取nio通道
    @Override
    protected void doConnect() throws Throwable {
        long start = System.currentTimeMillis();
        ChannelFuture future = bootstrap.connect(getConnectAddress());
        try {
            boolean ret = future.awaitUninterruptibly(getConnectTimeout(), MILLISECONDS);

            if (ret && future.isSuccess()) {
                Channel newChannel = future.channel();
                try {
                    // Close old channel
                    // copy reference
                    Channel oldChannel = NettyClient.this.channel;
                    if (oldChannel != null) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close old netty channel " + oldChannel + " on create new netty channel " + newChannel);
                            }
                            oldChannel.close();
                        } finally {
                            NettyChannel.removeChannelIfDisconnected(oldChannel);
                        }
                    }
                } finally {
                    if (NettyClient.this.isClosed()) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close new netty channel " + newChannel + ", because the client closed.");
                            }
                            newChannel.close();
                        } finally {
                            NettyClient.this.channel = null;
                            NettyChannel.removeChannelIfDisconnected(newChannel);
                        }
                    } else {
                        NettyClient.this.channel = newChannel;
                    }
                }
            } else if (future.cause() != null) {
                throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                        + getRemoteAddress() + ", error message is:" + future.cause().getMessage(), future.cause());
            } else {
                throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                        + getRemoteAddress() + " client-side timeout "
                        + getConnectTimeout() + "ms (elapsed: " + (System.currentTimeMillis() - start) + "ms) from netty client "
                        + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion());
            }
        } finally {
            // just add new valid channel to NettyChannel's cache
            if (!isConnected()) {
                //future.cancel(true);
            }
        }
    }

    @Override
    protected void doDisConnect() throws Throwable {
        try {
            NettyChannel.removeChannelIfDisconnected(channel);
        } catch (Throwable t) {
            logger.warn(t.getMessage());
        }
    }

  	//通过NettyChannel.getOrAddChannel获取Dubbo封装的Channel
    @Override
    protected org.apache.dubbo.remoting.Channel getChannel() {
        Channel c = channel;
        if (c == null || !c.isActive()) {
            return null;
        }
        return NettyChannel.getOrAddChannel(c, getUrl(), this);
    }
}
```

> NettyChannel是dubbo对netty的channel的封装，**装饰器模式**。
>
> 在类的静态成员变量里提供了jvm缓存
>
> ```java
> ConcurrentMap<Channel, NettyChannel> CHANNEL_MAP = new ConcurrentHashMap<Channel, NettyChannel>();
> ```
>
> 记录了Netty的Channel 和 dubbo的Channel的对应关系。
>
> NettyChannel是对Netty的Channel的装饰器。封装了channel元数据、url、通道事件处理器。



###### 发送请求

> 之前获取到了NettyChannel。就是通过他进行请求发送。

```java
//NettyChannel的发送方法
public void send(Object message, boolean sent) throws RemotingException {
    // 父类检查通道是否关闭
    super.send(message, sent);

    boolean success = true;
    int timeout = 0;
    try {
      	//调用Netty的通道发送数据
        ChannelFuture future = channel.writeAndFlush(message);
      	
      	// sent 的值源于 <dubbo:method sent="true/false" /> 中 sent 的配置值，有两种配置值：
        //   1. true: 等待消息发出，消息发送失败将抛出异常
        //   2. false: 不等待消息发出，将消息放入 IO 队列，即刻返回
        // 默认情况下 sent = false；
        if (sent) {
            timeout = getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
            success = future.await(timeout);
        }
        Throwable cause = future.cause();
        if (cause != null) {
            throw cause;
        }
    } catch (Throwable e) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
    }
    if (!success) {
        throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                + "in timeout(" + timeout + "ms) limit");
    }
}
```

Netty的channel发送稍微提一提。

writeAndFlush会进行pipeline的调用

```java
ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        .addLast("decoder", adapter.getDecoder())
                        .addLast("encoder", adapter.getEncoder())
                        .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
                        .addLast("handler", nettyClientHandler);
```



##### NettyServer

> dubbo封装的netty4服务端
>
> ![image-20200713100239178](https://gitee.com/wangigor/typora-images/raw/master/dubbo-nettyServer-hierarchy.png)



```java
/**
 * NettyServer.
 */
public class NettyServer extends AbstractServer implements Server {

    /**
     * 工作状态的通道worker channel的缓存
     * <ip:port, dubbo channel>
     */
    private Map<String, Channel> channels;
    private ServerBootstrap bootstrap;
    //boss channel 用于接收连接请求。分发到work channel
		private io.netty.channel.Channel channel;

  	//netty 的boss和worker时间轮询器
    private EventLoopGroup bossGroup;
    private EventLoopGroup workerGroup;

    public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        // you can customize name and type of client thread pool by THREAD_NAME_KEY and THREADPOOL_KEY in CommonConstants.
        // the handler will be warped: MultiMessageHandler->HeartbeatHandler->handler
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }

    //初始化Netty的bootstrap并启动
    @Override
    protected void doOpen() throws Throwable {
        bootstrap = new ServerBootstrap();

        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        // FIXME: should we use getTimeout()?
                        int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
                                .addLast("handler", nettyServerHandler);
                    }
                });
        // bind
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();

    }
//忽略...
}
```

> netty接收到请求后，交给nettyServerHandler
>
> 调用栈:
>
> NettyServerHandler channelRead(ChannelHandlerContext ctx, Object msg)
>
> ​	-> AbstractPeer received(Channel ch, Object msg)
>
> ​		->MultiMessageHandler received(Channel channel, Object message)
>
> ​			->HeartbeatHandler received(Channel channel, Object message)
>
> ​				->AllChannelHandler received(Channel channel, Object message)
>
> AllChannelHandler是all类型的线程派发器Dispatcher

###### 线程派发器

> ![img](https://gitee.com/wangigor/typora-images/raw/master/dispatcher-location.jpg)
>
> Dubbo 将底层通信框架中接收请求的线程称为 IO 线程。如果一些事件处理逻辑可以很快执行完，比如只在内存打一个标记，此时直接在 IO 线程上执行该段逻辑即可。但如果事件的处理逻辑比较耗时，比如该段逻辑会发起数据库查询或者 HTTP 请求。此时我们就不应该让事件处理逻辑在 IO 线程上执行，而是应该派发到线程池中去执行。原因也很简单，IO 线程主要用于接收请求，如果 IO 线程被占满，将导致它不能接收新的请求。
>
> | 策略       | 用途                                                         |
> | ---------- | ------------------------------------------------------------ |
> | all        | 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件等 |
> | direct     | 所有消息都不派发到线程池，全部在 IO 线程上直接执行           |
> | message    | 只有**请求**和**响应**消息派发到线程池，其它消息均在 IO 线程上执行 |
> | execution  | 只有**请求**消息派发到线程池，不含响应。其它消息均在 IO 线程上执行 |
> | connection | 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池 |
> Dubbo 使用 all 派发策略，即将所有的消息都派发到线程池中。

```java
@SPI(AllDispatcher.NAME)
public interface Dispatcher {

    /**
     * dispatch the message to threadpool.
     *
     * @param handler
     * @param url
     * @return channel handler
     */
    @Adaptive({Constants.DISPATCHER_KEY, "dispather", "channel.handler"})
    // The last two parameters are reserved for compatibility with the old configuration
    ChannelHandler dispatch(ChannelHandler handler, URL url);

}
```

```java
public class AllChannelHandler extends WrappedChannelHandler {

    public AllChannelHandler(ChannelHandler handler, URL url) {
        super(handler, url);
    }

  	//处理连接事件
    @Override
    public void connected(Channel channel) throws RemotingException {
      	//获取线程池
        ExecutorService executor = getExecutorService();
        try {
          	//将连接事件派发到线程池中处理
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("connect event", channel, getClass() + " error when process connected event .", t);
        }
    }

  	//处理断开连接事件
    @Override
    public void disconnected(Channel channel) throws RemotingException {
      	//获取连接池
        ExecutorService executor = getExecutorService();
        try {
          	//将断开连接事件派发到线程池中处理
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.DISCONNECTED));
        } catch (Throwable t) {
            throw new ExecutionException("disconnect event", channel, getClass() + " error when process disconnected event .", t);
        }
    }

  	//接收请求和响应信息
    @Override
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService executor = getExecutorService();
        try {
          	// 将请求和响应消息派发到线程池中处理
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
           	// 如果通信方式为双向通信，此时将 Server side ... threadpool is exhausted 
            // 错误信息封装到 Response 中，并返回给服务消费方。
          	if(message instanceof Request && t instanceof RejectedExecutionException){
              Request request = (Request)message;
              if(request.isTwoWay()){
                 String msg = "Server side(" + url.getIp() + "," + url.getPort() + ") threadpool is exhausted ,detail msg:" + t.getMessage();
                 Response response = new Response(request.getId(), request.getVersion());
                 response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
                 response.setErrorMessage(msg);
                 // 返回包含错误信息的 Response 对象
                 channel.send(response);
                 return;
              }
           }
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }

  	//处理异常信息
    @Override
    public void caught(Channel channel, Throwable exception) throws RemotingException {
        ExecutorService executor = getExecutorService();
        try {
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.CAUGHT, exception));
        } catch (Throwable t) {
            throw new ExecutionException("caught event", channel, getClass() + " error when process caught event .", t);
        }
    }
}
```

请求对象会被封装 ChannelEventRunnable 中，ChannelEventRunnable 将会是服务调用过程的新起点。

###### 调用服务



> 大多数请求是请求和响应【RECEIVED】,这里把RECEIVED单独拎出来做判断。
>
> 因为有5种类型判断，用switch效率更优。switch好比二叉查找树，if判断相当于线性查找。



```java
public class ChannelEventRunnable implements Runnable {

  	//通道处理器。这里是DecodeHandler
    private final ChannelHandler handler;
    private final Channel channel;
    private final ChannelState state;
    private final Throwable exception;
  	//解码之后就是Request对象
    private final Object message;

    @Override
    public void run() {
        if (state == ChannelState.RECEIVED) {
            try {
                handler.received(channel, message);
            } catch (Exception e) {
                logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                        + ", message is " + message, e);
            }
        } else {
            switch (state) {
            case CONNECTED:
                //...
                break;
            case DISCONNECTED:
                //...
                break;
            case SENT:
                //...
                break;
            case CAUGHT:
                //...
                break;
            default:
                logger.warn("unknown state: " + state + ", message is " + message);
            }
        }

    }

}
```

ChannelEventRunnable不负责具体的调用逻辑，相当于中转站，把请求交给DecodeHandler继续处理。

```java
		//DecodeHandler
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
				//默认的，在IO线程中已经完成了解码操作，继续交由HeaderExchangeHandler处理
        handler.received(channel, message);
    }
```



```java
    //HeaderExchangeHandler
		public void received(Channel channel, Object message) throws RemotingException {
      	//通道设置心跳的时间戳
        channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
        final ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        try {
            if (message instanceof Request) {
                // Request请求信息
                Request request = (Request) message;
                if (request.isEvent()) {
                    handlerEvent(channel, request);
                } else {
                    if (request.isTwoWay()) {
                      	//双向请求，交由handleRequest方法处理
                        handleRequest(exchangeChannel, request);
                    } else {
                      	//单向请求，向后调用具体服务，不处理结果
                        handler.received(exchangeChannel, request.getData());
                    }
                }
            // 处理响应对象，服务消费方会执行此处逻辑，后面分析
            } else if (message instanceof Response) {
                handleResponse(channel, (Response) message);
            // 处理telnet请求
            } else if (message instanceof String) {
                if (isClientSide(channel)) {
                    Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                    logger.error(e.getMessage(), e);
                } else {
                    String echo = handler.telnet(channel, (String) message);
                    if (echo != null && echo.length() > 0) {
                        channel.send(echo);
                    }
                }
            } else {
                handler.received(exchangeChannel, message);
            }
        } finally {
            HeaderExchangeChannel.removeChannelIfDisconnected(channel);
        }
    }
```

```java
//HeaderExchangeHandler
void handleRequest(final ExchangeChannel channel, Request req) throws RemotingException {
  	//创建Response
    Response res = new Response(req.getId(), req.getVersion());
  	//检查请求是否合法，不合法返回BAD_REQUEST
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

        channel.send(res);
        return;
    }
    // 获取Request的data，也就是RpcInvocation对象
    Object msg = req.getData();
    try {
      	//继续向下调用 DubboProtocol的匿名内部类ExchangeHandlerAdapter
        CompletionStage<Object> future = handler.reply(channel, msg);
      	//获取结果
        future.whenComplete((appResult, t) -> {
            try {
                if (t == null) {
                  	//设置OK状态码
                    res.setStatus(Response.OK);
                    res.setResult(appResult);
                } else {
                  	//设置服务端异常SERVICE_ERROR
                    res.setStatus(Response.SERVICE_ERROR);
                    res.setErrorMessage(StringUtils.toString(t));
                }
              
             		//由通道返回结果
                channel.send(res);
            } catch (RemotingException e) {
                logger.warn("Send result to consumer failed, channel is " + channel + ", msg is " + e);
            } finally {
                // HeaderExchangeChannel.removeChannelIfDisconnected(channel);
            }
        });
    } catch (Throwable e) {
      	//设置服务端异常SERVICE_ERROR
        res.setStatus(Response.SERVICE_ERROR);
        res.setErrorMessage(StringUtils.toString(e));
        channel.send(res);
    }
}
```



```java
//DubboProtocol 的匿名内部类
private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

    @Override
    public CompletableFuture<Object> reply(ExchangeChannel channel, Object message) throws RemotingException {

        Invocation inv = (Invocation) message;
      	//获取Invoker实例
        Invoker<?> invoker = getInvoker(channel, inv);
      	//回调相关 忽略
        if (Boolean.TRUE.toString().equals(inv.getAttachments().get(IS_CALLBACK_SERVICE_INVOKE))) {
            //...
        }
        RpcContext.getContext().setRemoteAddress(channel.getRemoteAddress());
      	//通过Invoker调用具体服务
        Result result = invoker.invoke(inv);
        return result.completionFuture().thenApply(Function.identity());
    }

};
```



获取Invoker

```java
    Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
        boolean isCallBackServiceInvoke = false;
        boolean isStubServiceInvoke = false;
        int port = channel.getLocalAddress().getPort();
      	//path 是接口类全限定名 除非回调和存根会去修改。
        String path = inv.getAttachments().get(PATH_KEY);

        // 本地存根逻辑
        isStubServiceInvoke = Boolean.TRUE.toString().equals(inv.getAttachments().get(STUB_EVENT_KEY));
        if (isStubServiceInvoke) {
            port = channel.getRemoteAddress().getPort();
        }

        //回调逻辑 
        isCallBackServiceInvoke = isClientSide(channel) && !isStubServiceInvoke;
        if (isCallBackServiceInvoke) {
            path += "." + inv.getAttachments().get(CALLBACK_SERVICE_KEY);
            inv.getAttachments().put(IS_CALLBACK_SERVICE_INVOKE, Boolean.TRUE.toString());
        }

      	//计算service key
      	//格式为 groupName/serviceName:serviceVersion:port
      	//e,g. org.example.dubbo.api.TestApi:20880
        String serviceKey = serviceKey(port, path, inv.getAttachments().get(VERSION_KEY), inv.getAttachments().get(GROUP_KEY));
        
      	// 从 exporterMap 查找与 serviceKey 相对应的 DubboExporter 对象，
        // 服务导出过程中会将 <serviceKey, DubboExporter> 映射关系存储到 exporterMap 集合中
      	DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);
        if (exporter == null) {
            throw new RemotingException(channel, "Not found exported service: " + serviceKey + " in " + exporterMap.keySet() + ", may be version or group mismatch " +
                    ", channel: consumer: " + channel.getRemoteAddress() + " --> provider: " + channel.getLocalAddress() + ", message:" + inv);
        }
				//返回Invoker对象
        return exporter.getInvoker();
    }
```

invoker执行

> 看一下线程的调用栈
>
> ![image-20200713155914794](https://gitee.com/wangigor/typora-images/raw/master/dubbo-received-msg-trace.png)
>
> ChannelEventRunnable#run()  
> ​	-> DecodeHandler#received(Channel, Object)    
> ​		-> HeaderExchangeHandler#received(Channel, Object)     
> ​			-> HeaderExchangeHandler#handleRequest(ExchangeChannel, Request)        
> ​				-> DubboProtocol.requestHandler#reply(ExchangeChannel, Object)          
> ​					-> Filter#invoke(Invoker, Invocation)            
> ​						-> AbstractProxyInvoker#invoke(Invocation)              
> ​							-> Wrapper0#invokeMethod(Object, String, Class[], Object[])
>
> 就到了provider的业务代码。

在AbstractProxyInvoker调用时，异步调用，获取结果组装成同步RpcResult

```java
    public Result invoke(Invocation invocation) throws RpcException {
        try {
          	//向下调用服务
            Object value = doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments());
          	CompletableFuture<Object> future = wrapWithFuture(value, invocation);
          	//组装同步调用结果并返回
            AsyncRpcResult asyncRpcResult = new AsyncRpcResult(invocation);
            future.whenComplete((obj, t) -> {
                AppResponse result = new AppResponse();
                if (t != null) {
                    if (t instanceof CompletionException) {
                        result.setException(t.getCause());
                    } else {
                        result.setException(t);
                    }
                } else {
                    result.setValue(obj);
                }
                asyncRpcResult.complete(result);
            });
            return asyncRpcResult;
        } catch (InvocationTargetException e) {
            if (RpcContext.getContext().isAsyncStarted() && !RpcContext.getContext().stopAsync()) {
                logger.error("Provider async started, but got an exception from the original method, cannot write the exception back to consumer because an async result may have returned the new thread.", e);
            }
            return AsyncRpcResult.newDefaultAsyncResult(null, e.getTargetException(), invocation);
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

Invoker实例是通过**JavassistProxyFactory**创建的代理。

```java
public class JavassistProxyFactory extends AbstractProxyFactory {

    @Override
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        // 创建匿名类对象
      	return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
              	// 调用 invokeMethod 方法进行后续的调用
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }

}
```

Wrapper 是一个抽象类，其中 invokeMethod 是一个抽象方法。Dubbo 会在运行时通过 Javassist 框架为 Wrapper 生成实现类，并实现 invokeMethod 方法，该方法最终会根据调用信息调用具体的服务。以 TestService为例，Javassist 为其生成的代理类如下。

> 可以使用arthas的jad查看TestService的Wrapper1包装类。

```java
/*
 * Decompiled with CFR.
 *
 * Could not load the following classes:
 *  org.example.dubbo.DTO.TestDTO
 *  org.example.dubbo.provider.service.TestService
 */
package org.apache.dubbo.common.bytecode;

import java.lang.reflect.InvocationTargetException;
import java.util.Map;
import org.apache.dubbo.common.bytecode.ClassGenerator;
import org.apache.dubbo.common.bytecode.NoSuchMethodException;
import org.apache.dubbo.common.bytecode.NoSuchPropertyException;
import org.apache.dubbo.common.bytecode.Wrapper;
import org.example.dubbo.DTO.TestDTO;
import org.example.dubbo.provider.service.TestService;

public class Wrapper1 extends Wrapper implements ClassGenerator.DC {
    public static String[] pns;
    public static Map pts;
    public static String[] mns;
    public static String[] dmns;
    public static Class[] mts0;

		//忽略其他代码
  
  	//方法调用
    public Object invokeMethod(Object object, String string, Class[] arrclass, Object[] arrobject) throws InvocationTargetException {
        TestService testService;
        try {
            testService = (TestService)object;
        }
        catch (Throwable throwable) {
            throw new IllegalArgumentException(throwable);
        }
        try {
            if ("test".equals(string) && arrclass.length == 1) {
                return testService.test((TestDTO)arrobject[0]);
            }
        }
        catch (Throwable throwable) {
            throw new InvocationTargetException(throwable);
        }
        throw new NoSuchMethodException(new StringBuffer().append("Not found method \"").append(string).append("\" in class org.example.dubbo.provider.service.TestService.").toString());
    }
}
```

Response返回跟Request差不多。就不看了。

下面处理最后一个问题，向调用方的用户线程通知结果。

###### 向consumer的用户调用线程通知结果

> 响应数据解码完成后，Dubbo 会将响应对象派发到线程池上。要注意的是，线程池中的线程并非用户的调用线程，所以要想办法将响应对象从线程池线程传递到用户线程上。前面分析过用户线程在发送完请求后的动作，即调用 DefaultFuture 的 get 方法等待响应对象的到来。当响应对象到来后，用户线程会被唤醒，并通过**调用编号**获取属于自己的响应对象。

```java
    //HeaderExchangeHandler
		//接收响应 
		static void handleResponse(Channel channel, Response response) throws RemotingException {
        if (response != null && !response.isHeartbeat()) {
            DefaultFuture.received(channel, response);
        }
    }
```

通过请求id，获取调用方用户线程等待的future

```java
public static void received(Channel channel, Response response, boolean timeout) {
    try {
      	//通过请求id获取future
        DefaultFuture future = FUTURES.remove(response.getId());
        if (future != null) {
            Timeout t = future.timeoutCheckTask;
            if (!timeout) {
                // 如果调用方已经超时了。就丢弃Response
                t.cancel();
            }
            future.doReceived(response);
        } else {
            logger.warn("The timeout response finally returned at "
                    + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                    + ", response " + response
                    + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                    + " -> " + channel.getRemoteAddress()));
        }
    } finally {
        CHANNELS.remove(response.getId());
    }
}
```

```java
    private void doReceived(Response res) {
        if (res == null) {
            throw new IllegalStateException("response cannot be null");
        }
        if (res.getStatus() == Response.OK) {
          	//设置OK的请求结果
            this.complete(res.getResult());
          
        //处理异常
        } else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
            this.completeExceptionally(new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage()));
        } else {
            this.completeExceptionally(new RemotingException(channel, res.getErrorMessage()));
        }
    }
```

以上逻辑是将响应对象保存到相应的 DefaultFuture 实例中，然后再唤醒用户线程，随后用户线程即可从 DefaultFuture 实例中获取到相应结果。

一般情况下，服务消费方会并发调用多个服务，每个用户线程发送请求后，会调用不同 DefaultFuture 对象的 get 方法进行等待。 一段时间后，服务消费方的线程池会收到多个响应对象。这个时候要考虑一个问题，如何将每个响应对象传递给相应的 DefaultFuture 对象，且不出错。答案是通过调用编号。DefaultFuture 被创建时，会要求传入一个 Request 对象。此时 DefaultFuture 可从 Request 对象中获取调用编号，并将 <调用编号, DefaultFuture 对象> 映射关系存入到静态 Map 中，即 FUTURES。线程池中的线程在收到 Response 对象后，会根据 Response 对象中的调用编号到 FUTURES 集合中取出相应的 DefaultFuture 对象，然后再将 Response 对象设置到 DefaultFuture 对象中。最后再唤醒用户线程，这样用户线程即可从 DefaultFuture 对象中获取调用结果了。

![img](https://gitee.com/wangigor/typora-images/raw/master/request-id-application.jpg)

#### 请求编码

下图是Dubbo官网提供的数据包结构。

![img](https://gitee.com/wangigor/typora-images/raw/master/dubbo-data-format.jpg)

请求头供128个bit

| 偏移量(Bit) | 字段         | 取值                                                         |
| ----------- | ------------ | ------------------------------------------------------------ |
| 0 ~ 7       | 魔数高位     | 0xda00                                                       |
| 8 ~ 15      | 魔数低位     | 0xbb                                                         |
| 16          | 数据包类型   | 0 - Response, 1 - Request                                    |
| 17          | 调用方式     | 仅在第16位被设为1的情况下有效<br />0 - 单向调用，1 - 双向调用 |
| 18          | 事件标识     | 0 - 当前数据包是请求或响应包，<br />1 - 当前数据包是心跳包   |
| 19 ~ 23     | 序列化器编号 | 2 - Hessian2Serialization <br />3 - JavaSerialization <br />4 - CompactedJavaSerialization <br />6 - FastJsonSerialization <br />7 - NativeJavaSerialization <br />8 - KryoSerialization <br />9 - FstSerialization |
| 24 ~ 31     | 状态         | 20 - OK <br />30 - CLIENT_TIMEOUT <br />31 - SERVER_TIMEOUT <br />40 - BAD_REQUEST <br />50 - BAD_RESPONSE ...... |
| 32 ~ 95     | 请求编号     | 共8字节，运行时生成                                          |
| 96 ~ 127    | 消息体长度   | 运行时计算                                                   |

> 从NettyClient/NettyServer的NettyBootstrap初始化方法，开始看。
>
> ```java
> NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
> ch.pipeline()
> .addLast("decoder", adapter.getDecoder())
> .addLast("encoder", adapter.getEncoder())
> ```
> NettyCodecAdapter是对编码解码器的统一封装。
>
> ```java
> final public class NettyCodecAdapter {
> 		
>   	//内部编码器 调用codec进行编码 把netty的ByteBuf转为ChannelBuffer
>     private final ChannelHandler encoder = new InternalEncoder();
> 		//内部解码器 调用codec进行解码 把netty的ByteBuf转为ChannelBuffer
>     private final ChannelHandler decoder = new InternalDecoder();
> 
>     private final Codec2 codec;//编码解码器
> 
>     private final URL url;
> 
>     private final org.apache.dubbo.remoting.ChannelHandler handler;
>   	//其他忽略...
> }
> ```
>
> getCodec()是获取url指定的编码解码器，默认为telnet，dubbo协议默认使用dubbo。通过dubbo spi获得。
>
> ```java
> protected static Codec2 getChannelCodec(URL url) {
> 	String codecName = url.getParameter(Constants.CODEC_KEY, "telnet");
> 	if (ExtensionLoader.getExtensionLoader(Codec2.class).hasExtension(codecName)) {
>   	return ExtensionLoader.getExtensionLoader(Codec2.class).getExtension(codecName);
> 	} else {
>   	return new CodecAdapter(ExtensionLoader.getExtensionLoader(Codec.class)
>           .getExtension(codecName));
> 	}
> }
> ```
> ```properties
> dubbo=org.apache.dubbo.rpc.protocol.dubbo.DubboCountCodec
> ```
> DubboCountCodec封装了对解码的半包请求处理。编码没做操作。内部是用成员的DubboCodec进行编码解码。
>
> ![image-20200709152306123](https://gitee.com/wangigor/typora-images/raw/master/DubboCodec-hirarchy.png)

##### 请求编码

> DubboCodec父类ExchangeCodec提供了encode的入口。

```java
//ExchangeCodec
public void encode(Channel channel, ChannelBuffer buffer, Object msg) throws IOException {
    if (msg instanceof Request) {
      	//对Request编码
        encodeRequest(channel, buffer, (Request) msg);
    } else if (msg instanceof Response) {
      	//对Response编码
        encodeResponse(channel, buffer, (Response) msg);
    } else {
        super.encode(channel, buffer, msg);
    }
}
```



```java
//ExchangeCodec
protected void encodeRequest(Channel channel, ChannelBuffer buffer, Request req) throws IOException {
  	//获取url的序列化方式，默认为hessian2
    Serialization serialization = getSerialization(channel);
    // 数据头 16 byte
    byte[] header = new byte[HEADER_LENGTH];
    // 设置魔数 (short) 0xdabb 分成两个byte，分别写。
    Bytes.short2bytes(MAGIC, header);

    // 第三个 byte 先写入第一个bit 1【请求】和后五bit【序列化方式编号】
    header[2] = (byte) (FLAG_REQUEST | serialization.getContentTypeId());

  	//设置第三个byte 的第二位，是否为双向通信
    if (req.isTwoWay()) {
        header[2] |= FLAG_TWOWAY;
    }
  	//设置第三个byte 的第三位 事件标识
    if (req.isEvent()) {
        header[2] |= FLAG_EVENT;
    }

    // 设置请求id，从第四个byte开始设置，共占8个byte
    Bytes.long2bytes(req.getId(), header, 4);

    // 获取buffer的当前写入位置 pos=0
    int savedWriteIndex = buffer.writerIndex();
  	//buffer预留的请求头16字节位置
    buffer.writerIndex(savedWriteIndex + HEADER_LENGTH);
  	//ChannelBufferOutputStream是对ChannelBuffer的write的封装
    ChannelBufferOutputStream bos = new ChannelBufferOutputStream(buffer);
  	//创建返回带序列化器Hessian2ObjectOutput的对象输出
    ObjectOutput out = serialization.serialize(channel.getUrl(), bos);
  	
    if (req.isEvent()) {
      	//对事件数据进行序列化操作
        encodeEventData(channel, out, req.getData());
    } else {
      	//对请求数据进行序列化操作
        encodeRequestData(channel, out, req.getData(), req.getVersion());
    }
    out.flushBuffer();
    if (out instanceof Cleanable) {
        ((Cleanable) out).cleanup();
    }
    bos.flush();
    bos.close();
  	//获取报文体长度
    int len = bos.writtenBytes();
    checkPayload(channel, len);
  	//报文体长度写入header中 从第12个byte开始写 共占用4个byte
    Bytes.int2bytes(len, header, 12);

    // 设置header的写入位置
    buffer.writerIndex(savedWriteIndex);
  	// 写入header
    buffer.writeBytes(header); 
  	// 设置当前写入位置为 savedWriteIndex + HEADER_LENGTH + len
    buffer.writerIndex(savedWriteIndex + HEADER_LENGTH + len);
}
```

DubboCodec重写了encodeRequestData方法，调用它写入请求体。

```java
    @Override
    protected void encodeRequestData(Channel channel, ObjectOutput out, Object data, String version) throws IOException {
        RpcInvocation inv = (RpcInvocation) data;
				//序列化Dubbo version、接口全限定名、接口版本
        out.writeUTF(version);
        out.writeUTF(inv.getAttachment(PATH_KEY));
        out.writeUTF(inv.getAttachment(VERSION_KEY));
				//序列化方法名
        out.writeUTF(inv.getMethodName());
      	//设置方法的参数类型 Class->String
        out.writeUTF(ReflectUtils.getDesc(inv.getParameterTypes()));
      	//对方法入参进行运行时解析。然后再序列化
        Object[] args = inv.getArguments();
        if (args != null) {
            for (int i = 0; i < args.length; i++) {
                out.writeObject(encodeInvocationArgument(channel, inv, i));
            }
        }
      	//设置Attachments附属属性
        out.writeObject(inv.getAttachments());
    }
```

到此。对请求编码就完成了。

##### 请求解码

> 请求解码也是从父类ExchangeCodec开始。

```java
    //ExchangeCodec
		@Override
    public Object decode(Channel channel, ChannelBuffer buffer) throws IOException {
      	//读取buffer中可读数据的总长度
        int readable = buffer.readableBytes();
      	//创建请求头header的byte数组
        byte[] header = new byte[Math.min(readable, HEADER_LENGTH)];
      	//读取请求头数据
        buffer.readBytes(header);
      	//调用重载方法进行后续解码操作
        return decode(channel, buffer, readable, header);
    }

    @Override
    protected Object decode(Channel channel, ChannelBuffer buffer, int readable, byte[] header) throws IOException {
        // 检查魔数
        if (readable > 0 && header[0] != MAGIC_HIGH
                || readable > 1 && header[1] != MAGIC_LOW) {
            int length = header.length;
            if (header.length < readable) {
                header = Bytes.copyOf(header, readable);
                buffer.readBytes(header, length, readable - length);
            }
            for (int i = 1; i < header.length - 1; i++) {
                if (header[i] == MAGIC_HIGH && header[i + 1] == MAGIC_LOW) {
                    buffer.readerIndex(buffer.readerIndex() - header.length + i);
                    header = Bytes.copyOf(header, i);
                    break;
                }
            }
          	// 通过 telnet 命令行发送的数据包不包含消息头，所以这里
            // 调用 TelnetCodec 的 decode 方法对数据包进行解码
            return super.decode(channel, buffer, readable, header);
        }
        // 如果可读数据长度小于header定长16.直接返回NEED_MORE_INPUT
        if (readable < HEADER_LENGTH) {
            return DecodeResult.NEED_MORE_INPUT;
        }

        // 获取header中记录的请求体数据长度
        int len = Bytes.bytes2int(header, 12);
        checkPayload(channel, len);
				
      	//如果可读数据长度小于header长度+请求体数据长度。直接返回NEED_MORE_INPUT
        int tt = len + HEADER_LENGTH;
        if (readable < tt) {
            return DecodeResult.NEED_MORE_INPUT;
        }

        // limit input stream.
        ChannelBufferInputStream is = new ChannelBufferInputStream(buffer, len);

        try {
          	//解析请求体
            return decodeBody(channel, is, header);
        } finally {
            if (is.available() > 0) {
                try {
                    if (logger.isWarnEnabled()) {
                        logger.warn("Skip input stream " + is.available());
                    }
                    StreamUtils.skipUnusedStream(is);
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
        }
    }
```

DubboCodec重写了decodeBody

```java
    //DubboCodec
		@Override
    protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
      	//获取请求头的第三个byte 通过逻辑与得出序列化编号
        byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
        // 从第四个byte开始，读取一个long的请求id
        long id = Bytes.bytes2long(header, 4);
      	//逻辑与计算是否请求
        if ((flag & FLAG_REQUEST) == 0) {
            // 解码响应response
            Response res = new Response(id);
            if ((flag & FLAG_EVENT) != 0) {
                res.setEvent(true);
            }
            // 获取响应状态码
            byte status = header[3];
            res.setStatus(status);
            try {
                if (status == Response.OK) {
                    Object data;
                    if (res.isHeartbeat()) {
                        ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
                        data = decodeHeartbeatData(channel, in);
                    } else if (res.isEvent()) {
                        ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
                        data = decodeEventData(channel, in);
                    } else {
                        DecodeableRpcResult result;
                        if (channel.getUrl().getParameter(DECODE_IN_IO_THREAD_KEY, DEFAULT_DECODE_IN_IO_THREAD)) {
                            result = new DecodeableRpcResult(channel, res, is,
                                    (Invocation) getRequestData(id), proto);
                            result.decode();
                        } else {
                            result = new DecodeableRpcResult(channel, res,
                                    new UnsafeByteArrayInputStream(readMessageData(is)),
                                    (Invocation) getRequestData(id), proto);
                        }
                        data = result;
                    }
                    res.setResult(data);
                } else {
                    ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
                    res.setErrorMessage(in.readUTF());
                }
            } catch (Throwable t) {
                if (log.isWarnEnabled()) {
                    log.warn("Decode response failed: " + t.getMessage(), t);
                }
                res.setStatus(Response.CLIENT_ERROR);
                res.setErrorMessage(StringUtils.toString(t));
            }
            return res;
        } else {
            // 解码请求
            Request req = new Request(id);
          	// 设置dubbo version
            req.setVersion(Version.getProtocolVersion());
          	// 设置双向通信标识
            req.setTwoWay((flag & FLAG_TWOWAY) != 0);
          	// 设置是否为事件
            if ((flag & FLAG_EVENT) != 0) {
                req.setEvent(true);
            }
            try {
                Object data;
              	//处理心跳请求。
                if (req.isHeartbeat()) {
                    ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
                    data = decodeHeartbeatData(channel, in);
                //处理事件请求。
                } else if (req.isEvent()) {
                    ObjectInput in = CodecSupport.deserialize(channel.getUrl(), is, proto);
                    data = decodeEventData(channel, in);
                } else {
                //处理正常请求。
                    DecodeableRpcInvocation inv;
                  	//根据url参数判断是否在当前IO线程 对消息体解码
                    if (channel.getUrl().getParameter(DECODE_IN_IO_THREAD_KEY, DEFAULT_DECODE_IN_IO_THREAD)) {
                        //默认true
                      	inv = new DecodeableRpcInvocation(channel, req, is, proto);
                      	// 在当前线程，也就是 IO 线程上进行后续的解码工作。此工作完成后，可将
                        // 调用方法名、attachment、以及调用参数解析出来
                        inv.decode();
                    } else {
                      	// 仅创建 DecodeableRpcInvocation 对象，但不在当前线程上执行解码逻辑
                        inv = new DecodeableRpcInvocation(channel, req,
                                new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                    }
                    data = inv;
                }
              	// 设置 data 到 Request 对象中
                req.setData(data);
            } catch (Throwable t) {
                if (log.isWarnEnabled()) {
                    log.warn("Decode request failed: " + t.getMessage(), t);
                }
                // 若解码过程中出现异常，则将 broken 字段设为 true，
                // 并将异常对象设置到 Reqeust 对象中
                req.setBroken(true);
                req.setData(t);
            }

            return req;
        }
    }
```

把请求体数据封装成**DecodeableRpcInvocation**。进行解码。

> 解码就是DecodeableRpcInvocation组装父类RpcInvocation的过程

```JAVA
public class DecodeableRpcInvocation extends RpcInvocation implements Codec, Decodeable {

    private static final Logger log = LoggerFactory.getLogger(DecodeableRpcInvocation.class);

    private Channel channel;

    private byte serializationType;

    private InputStream inputStream;

    private Request request;
		
  	//是否已解码标识
    private volatile boolean hasDecoded;


  	//解码入口
    @Override
    public void decode() throws Exception {
      	//容错。防止重复解码。
        if (!hasDecoded && channel != null && inputStream != null) {
            try {
              	//如果没有解码。就在这里调用重载方法解码
                decode(channel, inputStream);
            } catch (Throwable e) {
                if (log.isWarnEnabled()) {
                    log.warn("Decode rpc invocation failed: " + e.getMessage(), e);
                }
              	//解码异常。封装异常
                request.setBroken(true);
                request.setData(e);
            } finally {
              	//标记为解码完成
                hasDecoded = true;
            }
        }
    }
		
  	//解码
    @Override
    public Object decode(Channel channel, InputStream input) throws IOException {
      	//设置反序列化器
        ObjectInput in = CodecSupport.getSerialization(channel.getUrl(), serializationType)
                .deserialize(channel.getUrl(), input);

      	//读取dubbo version
        String dubboVersion = in.readUTF();
      	//dubbo version 设置到request中
        request.setVersion(dubboVersion);
      	//dubbo version、接口全限定名、接口版本设置到attachments【附件属性】中
        setAttachment(DUBBO_VERSION_KEY, dubboVersion);
        setAttachment(PATH_KEY, in.readUTF());
        setAttachment(VERSION_KEY, in.readUTF());
				//设置方法名
        setMethodName(in.readUTF());
        try {
            Object[] args;
            Class<?>[] pts;
            String desc = in.readUTF();
          	//设置参数类型集合
            if (desc.length() == 0) {
              	//空参。设置参数类型集合和参数集合 空集合
                pts = DubboCodec.EMPTY_CLASS_ARRAY;
                args = DubboCodec.EMPTY_OBJECT_ARRAY;
            } else {
              	//将desc解析为参数类型集合
                pts = ReflectUtils.desc2classArray(desc);
                args = new Object[pts.length];
              	//循环获取参数，绑定参数
                for (int i = 0; i < args.length; i++) {
                    try {
                        args[i] = in.readObject(pts[i]);
                    } catch (Exception e) {
                        if (log.isWarnEnabled()) {
                            log.warn("Decode argument failed: " + e.getMessage(), e);
                        }
                    }
                }
            }
          	//设置参数类型集合
            setParameterTypes(pts);

          	//设置附件参数列表。
            Map<String, String> map = (Map<String, String>) in.readObject(Map.class);
            if (map != null && map.size() > 0) {
                Map<String, String> attachment = getAttachments();
                if (attachment == null) {
                    attachment = new HashMap<String, String>();
                }
                attachment.putAll(map);
                setAttachments(attachment);
            }
          
            //对callback的参数进行处理 TODO
            for (int i = 0; i < args.length; i++) {
                args[i] = decodeInvocationArgument(channel, this, pts, i, args[i]);
            }
						
          	//设置参数
            setArguments(args);

        } catch (ClassNotFoundException e) {
            throw new IOException(StringUtils.toString("Read invocation data failed.", e));
        } finally {
            if (in instanceof Cleanable) {
                ((Cleanable) in).cleanup();
            }
        }
        return this;
    }

}
```



### MOCK调用

TODO



# 负载均衡

> 负载均衡是通过dubbo spi，无侵入。单独分析
>
> 目前支持这四种负载均衡算法。
>
> ```properties
> random=org.apache.dubbo.rpc.cluster.loadbalance.RandomLoadBalance
> roundrobin=org.apache.dubbo.rpc.cluster.loadbalance.RoundRobinLoadBalance
> leastactive=org.apache.dubbo.rpc.cluster.loadbalance.LeastActiveLoadBalance
> consistenthash=org.apache.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
> ```

![](https://gitee.com/wangigor/typora-images/raw/master/dubbo-loadbalance-hierarchy.png)

## 父类AbstractLoadBalance

> 父类提供两块功能：
>
> - 负载均衡LoadBalance接口的模板方法select
> - 权重计算的通用逻辑getWeight

```java
    //select方法入口。这里只做参数校验，具体实现交给子类
		@Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) {
      	//invokers为空，返回null
        if (CollectionUtils.isEmpty(invokers)) {
            return null;
        }
      	//invokers只有一个，就直接返回。
        if (invokers.size() == 1) {
            return invokers.get(0);
        }
      	//调用子类选择方法
        return doSelect(invokers, url, invocation);
    }
		//子类实现
    protected abstract <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation);
```

***

```java
    //获取权重
		int getWeight(Invoker<?> invoker, Invocation invocation) {
      	//获取提供者url上的权重参数weight 默认100
        int weight = invoker.getUrl().getMethodParameter(invocation.getMethodName(), WEIGHT_KEY, DEFAULT_WEIGHT);
        if (weight > 0) {
          	//获取服务提供者启动时间戳
            long timestamp = invoker.getUrl().getParameter(TIMESTAMP_KEY, 0L);
            if (timestamp > 0L) {
              	//计算服务提供者运行时长
                long uptime = System.currentTimeMillis() - timestamp;
                if (uptime < 0) {
                    return 1;
                }
              	//获取提供者url上的预热时间 默认十分钟【10 * 60 * 1000】
                int warmup = invoker.getUrl().getParameter(WARMUP_KEY, DEFAULT_WARMUP);
              	//如果运行时长小于预热时间，进行降权计算
                if (uptime > 0 && uptime < warmup) {
                    weight = calculateWarmupWeight((int)uptime, warmup, weight);
                }
            }
        }
        return Math.max(weight, 0);
    }

		//计算预热权重
		//（uptime/warmup）*weight
		// 不超过weight
		static int calculateWarmupWeight(int uptime, int warmup, int weight) {
        int ww = (int) ( uptime / ((float) warmup / weight));
        return ww < 1 ? 1 : (Math.min(ww, weight));
    }
```

上面是权重的计算过程，该过程主要用于保证当服务运行时长小于服务预热时间时，对服务进行降权，避免让服务在启动之初就处于高负载状态。==服务预热==是一个优化手段，与此类似的还有 JVM 预热。主要目的是让服务启动后“低功率”运行一段时间，使其效率慢慢提升至最佳状态。

## 【默认】加权随机RandomLoadBalance

> 假设有三台服务提供者【A，B，C】。权重分别是【5，3，2】。
>
> 三个权重相加，总权重是10.
>
> 获得0~10的随机数，假设8：
>
> 去【5，3，2】的集合中逐个相减，结果小于等于0时，获取下标。

```java
   protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // invokers数量
        int length = invokers.size();
        // 权重相同标记
        boolean sameWeight = true;
        // invokers的权重集合
        int[] weights = new int[length];
     
     		
        // 获取第一个invoker权重【为了计算权重相同标记】
        int firstWeight = getWeight(invokers.get(0), invocation);
        weights[0] = firstWeight;
        // 总权重
        int totalWeight = firstWeight;
     		
     		//循环所有的invokers
     		//看所有的权重是否相等
     		//计算总权重
     		//放入权重集合中
        for (int i = 1; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // save for later use
            weights[i] = weight;
            // Sum
            totalWeight += weight;
            if (sameWeight && weight != firstWeight) {
                sameWeight = false;
            }
        }
     		//按照总权重取随机值 
     		//看随机值落在哪个区间
        if (totalWeight > 0 && !sameWeight) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on totalWeight.
            int offset = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < length; i++) {
                offset -= weights[i];
                if (offset < 0) {
                    return invokers.get(i);
                }
            }
        }
        //如果所有的权重相同。随机获取一个。
        return invokers.get(ThreadLocalRandom.current().nextInt(length));
    }
```



## 最小活跃数LeastActiveLoadBalance

> 最小活跃数是invoker对应的活跃数。当前服务，启动两个节点，则分别计数。
>
> 通过RpcStatus的静态常量缓存进行计数统计。



```java
		protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        // invokers数量
        int length = invokers.size();
        // 最小的活跃数
        int leastActive = -1;
        // 同时拥有最小活跃数的invoker数量
        int leastCount = 0;
        // 同时拥有最小活跃数的invoker的下标集合
        int[] leastIndexes = new int[length];
        // invokers权重集合
        int[] weights = new int[length];
        // 同时拥有最小活跃数的invokers的总权重
        int totalWeight = 0;
        // 同时拥有最小活跃数的invokers的第一个的权重【用于计算sameWeight】
        int firstWeight = 0;
        // 是否权重相同标记
        boolean sameWeight = true;


        // 遍历出所有拥有最小活跃数的invoker
        for (int i = 0; i < length; i++) {
            Invoker<T> invoker = invokers.get(i);
            // 获取invoker对应的活跃数
            int active = RpcStatus.getStatus(invoker.getUrl(), invocation.getMethodName()).getActive();
            // 获取invoker权重【预热后的权重】
            int afterWarmup = getWeight(invoker, invocation);
            // 放入权重数组中
            weights[i] = afterWarmup;
            // 逻辑1：发现更小的，重新开始
            if (leastActive == -1 || active < leastActive) {
                // 重置最小活跃数、最小活跃计数、最小活跃数invoker第一个、总权重、第一个的权重、sameWeight标识。
                leastActive = active;
                leastCount = 1;
                leastIndexes[0] = i;
                totalWeight = afterWarmup;
                firstWeight = afterWarmup;
                sameWeight = true;
            // 逻辑2：发现相等的，追加
            } else if (active == leastActive) {
                // 追加集合、追加总权重、
                leastIndexes[leastCount++] = i;
                // Accumulate the total weight of the least active invoker
                totalWeight += afterWarmup;
                // 计算sameWeight
                if (sameWeight && i > 0
                        && afterWarmup != firstWeight) {
                    sameWeight = false;
                }
            }
        }
        // 如果只有一个最小的，就直接返回。
        if (leastCount == 1) {
            return invokers.get(leastIndexes[0]);
        }
      	//权重不同，进行加权随机。
        if (!sameWeight && totalWeight > 0) {
            // If (not every invoker has the same weight & at least one invoker's weight>0), select randomly based on 
            // totalWeight.
            int offsetWeight = ThreadLocalRandom.current().nextInt(totalWeight);
            // Return a invoker based on the random value.
            for (int i = 0; i < leastCount; i++) {
                int leastIndex = leastIndexes[i];
                offsetWeight -= weights[leastIndex];
                if (offsetWeight < 0) {
                    return invokers.get(leastIndex);
                }
            }
        }
        // 权重相同，随机取一个返回。
        return invokers.get(leastIndexes[ThreadLocalRandom.current().nextInt(leastCount)]);
    }
```





## 一致性哈希ConsistentHashLoadBalance



> 背景知识：一致性哈希算法，是hash算法的进化版。dubbo官网画的图真好看。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/consistent-hash-1.jpg)
>
> - 第一层进化是从指定节点数量到hash环。
>
>   传统操作：3个节点，请求来时，hashcode对3取余。
>
>   缺点：节点数据迁移、缓存雪崩。
>
>   0~2^32-1个节点的哈希环，如上图，cache3节点宕机，存量数据去cache4节点查询
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/consistent-hash-data-incline.jpg)
>
> - 第二层进化是分布不均匀。
>
>   分布不均匀导致invoker1的数据访问量一直大于invoker2
>
>   引入虚拟节点
>   
>   ![img](https://gitee.com/wangigor/typora-images/raw/master/consistent-hash-invoker.jpg)
>   
>   默认情况下。每个invoker节点在换上生成160个离散节点。使得请求尽可能均匀分布。
>   
>   

```java
public class ConsistentHashLoadBalance extends AbstractLoadBalance {
		//{省略}
		//服务+方法 对应 一致性哈希选择器 的本地缓存 
    private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors = new ConcurrentHashMap<String, ConsistentHashSelector<?>>();

    @SuppressWarnings("unchecked")
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
      	//获取方法名
        String methodName = RpcUtils.getMethodName(invocation);
      	//服务api的全限定名+方法名 作为key
        String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;
      	//invokers集合的hashcode 【集合引用对象的内存地址hashcode】
        int identityHashCode = System.identityHashCode(invokers);
      	//从本地缓存中获取一致性哈希选择器
        ConsistentHashSelector<T> selector = (ConsistentHashSelector<T>) selectors.get(key);
        if (selector == null || selector.identityHashCode != identityHashCode) {
          	//创建选择器
            selectors.put(key, new ConsistentHashSelector<T>(invokers, methodName, identityHashCode));
            selector = (ConsistentHashSelector<T>) selectors.get(key);
        }
        return selector.select(invocation);
    }
		
  
  	//一致性哈希选择器内部类
    private static final class ConsistentHashSelector<T> {

        private final TreeMap<Long, Invoker<T>> virtualInvokers;
				
      	//虚拟节点数量
        private final int replicaNumber;
				
      	//invokers的引用对象的对象头hashcode
        private final int identityHashCode;
				
      	//需要进行hash计算的调用参数下标集合
        private final int[] argumentIndex;

        ConsistentHashSelector(List<Invoker<T>> invokers, String methodName, int identityHashCode) {
          	//创建TreeMap保存节点，包括虚拟节点
            this.virtualInvokers = new TreeMap<Long, Invoker<T>>();
            this.identityHashCode = identityHashCode;
          	//获取一个url
            URL url = invokers.get(0).getUrl();
          	//获取url参数上配置的虚拟节点数量 默认160
          	//<dubbo:parameter key="hash.nodes" value="320" />
            this.replicaNumber = url.getMethodParameter(methodName, HASH_NODES, 160);
          	//获取需要进行哈希计算的调用参数下标集 默认只计算第一个参数
          	//<dubbo:parameter key="hash.arguments" value="0,1" />
          	//根据方法调用参数计算hash，看落在环状hash的哪个位置，选择顺时针第一节点。
            String[] index = COMMA_SPLIT_PATTERN.split(url.getMethodParameter(methodName, HASH_ARGUMENTS, "0"));
            argumentIndex = new int[index.length];
            for (int i = 0; i < index.length; i++) {
                argumentIndex[i] = Integer.parseInt(index[i]);
            }
          	//遍历invokers，生成虚拟节点
            for (Invoker<T> invoker : invokers) {
              	//host+port作为key
                String address = invoker.getUrl().getAddress();
              	//生成replicaNumber / 4个组。每组生成四个消息摘要。
                for (int i = 0; i < replicaNumber / 4; i++) {
                  	//对 host+port+i进行md5，得到16个离散byte
                    byte[] digest = md5(address + i);
                  	//16个byte分成4份，0~3，4~7，8~11，12~16
                    for (int h = 0; h < 4; h++) {
                      	//对这四个byte进行位运算，得到hash环坐标
                        long m = hash(digest, h);
                      	//坐标 和 invokers 放入treemap中。
                        virtualInvokers.put(m, invoker);
                    }
                }
            }
        }
				//节点选择
        public Invoker<T> select(Invocation invocation) {
          	//请求参数组装成key
            String key = toKey(invocation.getArguments());
          	//进行md5离散
            byte[] digest = md5(key);
          	//计算hash进行节点选择
            return selectForKey(hash(digest, 0));
        }
				//根据参数下标，获取参数toString，拼接。
        private String toKey(Object[] args) {
            StringBuilder buf = new StringBuilder();
            for (int i : argumentIndex) {
                if (i >= 0 && i < args.length) {
                    buf.append(args[i]);
                }
            }
            return buf.toString();
        }

        private Invoker<T> selectForKey(long hash) {
          	//获取大于等于当前hash的节点entry
            Map.Entry<Long, Invoker<T>> entry = virtualInvokers.ceilingEntry(hash);
            if (entry == null) {
              	//如果没有，就取第一个
                entry = virtualInvokers.firstEntry();
            }
          	//返回invoker
            return entry.getValue();
        }
      
				//四个byte计算hash的位运算
        private long hash(byte[] digest, int number) {
          	//就是拼接。
            return (((long) (digest[3 + number * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + number * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + number * 4] & 0xFF) << 8)
                    | (digest[number * 4] & 0xFF))
                    & 0xFFFFFFFFL;
        }
      
				//java自带的消息摘要
        private byte[] md5(String value) {
            MessageDigest md5;
            try {
                md5 = MessageDigest.getInstance("MD5");
            } catch (NoSuchAlgorithmException e) {
                throw new IllegalStateException(e.getMessage(), e);
            }
            md5.reset();
            byte[] bytes = value.getBytes(StandardCharsets.UTF_8);
            md5.update(bytes);
            return md5.digest();
        }

    }

}
```





## 加权轮询RoundRobinLoadBalance

> 加权轮询采用nginx平滑加权轮询的方式，采用weight+currentWeight的方式。
>
> 假设A、B、C三台提供者权重是【5，1，1】。总权重是7。初始currentWeight权重也是【0，0，0】。每次选择权重(weight+currentWeight)最大的一个，选择完就把它的当前权重减去总权重。下面模拟7（总权重）次请求。
>
> | 请求编号 | weight+currentWeight数组 | 选择结果 | 选择后weight+currentWeight数组 |
> | :------: | :----------------------: | :------: | :----------------------------: |
> |    1     |     【**5**，1，1】      |    A     |        【**-2**，1，1】        |
> |    2     |     【**3**，2，2】      |    A     |        【**-4**，2，2】        |
> |    3     |     【1，**3**，3】      |    B     |        【1，**-4**，3】        |
> |    4     |     【**6**，-3，4】     |    A     |       【**-1**，-3，4】        |
> |    5     |     【4，-2，**5**】     |    C     |       【4，-2，**-2**】        |
> |    6     |    【**9**，-1，-1】     |    A     |       【**2**，-1，-1】        |
> |    7     |     【**7**，0，0】      |    A     |        **【0，0，0】**         |
>
> 就走完了一轮。

```java
public class RoundRobinLoadBalance extends AbstractLoadBalance {
    public static final String NAME = "roundrobin";
    
    private static final int RECYCLE_PERIOD = 60000;
    
  	//内部类。用来封装weight+currentWeight。
  	//增加了一个最后更新时间lastUpdate TODO
    protected static class WeightedRoundRobin {
        private int weight;
        private AtomicLong current = new AtomicLong(0);//current默认为0
        private long lastUpdate;
        //getter/setter
    }
  
		// 嵌套 Map 结构，存储的数据结构示例如下：
    // {
    //     "DemoService.query": {
    //         "invoker1.url": WeightedRoundRobin@123, 
    //         "invoker2.url": WeightedRoundRobin@456, 
    //     },
    //     "DemoService.update": {
    //         "invoker1.url": WeightedRoundRobin@123, 
    //         "invoker2.url": WeightedRoundRobin@456,
    //     }
    // }
  	// service的每个方法单独记录weight+currentWeight
    private ConcurrentMap<String, ConcurrentMap<String, WeightedRoundRobin>> methodWeightMap = new ConcurrentHashMap<String, ConcurrentMap<String, WeightedRoundRobin>>();
    //原子更新锁
  	private AtomicBoolean updateLock = new AtomicBoolean();
    
		//获取所有的invoker的url集合
    protected <T> Collection<String> getInvokerAddrList(List<Invoker<T>> invokers, Invocation invocation) {
        //api全限定名+method方法名作为key
      	String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
        Map<String, WeightedRoundRobin> map = methodWeightMap.get(key);
        if (map != null) {
          	//【 invoker1.url , invoker2.url 】
            return map.keySet();
        }
        return null;
    }
    
    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
      	// api全限定名+method方法名作为key
        String key = invokers.get(0).getUrl().getServiceKey() + "." + invocation.getMethodName();
      	// 获取当前方法的 weight+currentWeight集合信息 。没有就初始化一个空map
        ConcurrentMap<String, WeightedRoundRobin> map = methodWeightMap.get(key);
        if (map == null) {
            methodWeightMap.putIfAbsent(key, new ConcurrentHashMap<String, WeightedRoundRobin>());
            map = methodWeightMap.get(key);
        }
        int totalWeight = 0; // 总权重
        long maxCurrent = Long.MIN_VALUE; // 当前weight+currentWeight中的最大值
        long now = System.currentTimeMillis(); // 当前时间
        Invoker<T> selectedInvoker = null; // 选出来的invoker节点
        WeightedRoundRobin selectedWRR = null; // 选出来的invoker对应的WeightedRoundRobin
        for (Invoker<T> invoker : invokers) {
          	// invoker.url
            String identifyString = invoker.getUrl().toIdentityString();
          	// 获取当前方法，当前invoker.url对应的WeightedRoundRobin
            WeightedRoundRobin weightedRoundRobin = map.get(identifyString);
          	//获取当前invoker的权重
            int weight = getWeight(invoker, invocation);
						//如果WeightedRoundRobin没有初始化。就初始化一个
            if (weightedRoundRobin == null) {
                weightedRoundRobin = new WeightedRoundRobin();
                weightedRoundRobin.setWeight(weight);
                map.putIfAbsent(identifyString, weightedRoundRobin);
            }
          	// 如果消费者端权重发生变化，更新权重
            if (weight != weightedRoundRobin.getWeight()) {
                //weight changed
                weightedRoundRobin.setWeight(weight);
            }
          	//current = weight+currentWeight
            long cur = weightedRoundRobin.increaseCurrent();
            weightedRoundRobin.setLastUpdate(now);
          	//如果当前current大于之前记录的weight+currentWeight的最大值。
          	//替换maxCurrent、selectedInvoker、selectedWRR
            if (cur > maxCurrent) {
                maxCurrent = cur;
                selectedInvoker = invoker;
                selectedWRR = weightedRoundRobin;
            }
          	//计算总权重
            totalWeight += weight;
        }
      
      	//如果invokers集合发生了变化，或者超过轮询周期【十分钟】.重新放置map
        if (!updateLock.get() && invokers.size() != map.size()) {
          	//先获取原子更新锁
            if (updateLock.compareAndSet(false, true)) {
                try {
                    // copy -> modify -> update reference
                    ConcurrentMap<String, WeightedRoundRobin> newMap = new ConcurrentHashMap<>(map);
                    newMap.entrySet().removeIf(item -> now - item.getValue().getLastUpdate() > RECYCLE_PERIOD);
                    methodWeightMap.put(key, newMap);
                } finally {
                  	//释放锁
                    updateLock.set(false);
                }
            }
        }
      	//当前选择的锁的WeightedRoundRobin的current减去总权重。
        if (selectedInvoker != null) {
            selectedWRR.sel(totalWeight);
            return selectedInvoker;
        }
        // 方法不会走到这里
        return invokers.get(0);
    }

}
```

# 动态配置中心

todo

# 泛化调用

todo

# 本地伪装MOCK

todo

# 本地存根

todo

# token

todo



# 参数回调

todo











