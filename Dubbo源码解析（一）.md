# Dubbo源码（一）

> dubbo版本：2.7.4.1
>
> 这一个md记录dubbo和springboot和spi、自适应扩展。

## dubbo与springboot

### springboot启动过程

```java
public ConfigurableApplicationContext run(String... args) {
  	//stopwatch是spring提供的程序执行计数器。可以进行多task即时
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
  	//初始化一个事件派发器。进行事件监听。
		SpringApplicationRunListeners listeners = getRunListeners(args);
  	//发布一个ApplicationStartingEvent事件。
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      //初始化整个应用的Environment配置
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
      //打印banner
			Banner printedBanner = printBanner(environment);
      //创建容器
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
      //初始化容器
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
      //发布ApplicationStartedEvent事件
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
      //发布ApplicationReadyEvent事件
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

### spring启动过程

> springboot程序启动的时候会创建容器，执行==refreshContext(context);==容器初始化。
>
> 也就是spring容器的抽象父类==AbstractApplicationContext==的刷新方法。

```java
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			
      // 为刷新容器做准备
			prepareRefresh();
			// 创建beanFactory
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// 初始化beanFactory。设置classLoader。
			prepareBeanFactory(beanFactory);

			try {
				
        //为子类容器提供postProcess扩展。比如web容器
				postProcessBeanFactory(beanFactory);

				// 执行BeanFactoryPostProcessors
        // 这里添加ConfigurationClassPostProcessor对@Configuration进行解析
        // 加载到了主配置@EnableDubbo
				invokeBeanFactoryPostProcessors(beanFactory);

				//注册BeanPostProcessors
				registerBeanPostProcessors(beanFactory);

				//i18n国际化支持
				initMessageSource();

				//初始化容器的时间派发器
				initApplicationEventMulticaster();

				//刷新。为子类容器提供扩展
				onRefresh();

				//注册时间监听器，到事件派发器中
				registerListeners();

				//初始化所有非延迟加载的单例bean
				finishBeanFactoryInitialization(beanFactory);

				//发布容器刷新完成事件
        //dubbo在这里进行服务暴露。
				finishRefresh();
			}
		}
	}
```

> 博客：https://www.cnblogs.com/xuerge/p/jie-hespring-tu-jiedubbo-qi-dong-guo-cheng.html
>
> 有一个很好的流程图![spring-dubbo流程图](/Users/wangke/Documents/markdown/file/dubbo/springboot-dubbo加载流程.jpg)

### dubbo启动过程

> 两种方式。
>
> 第一种是==注解式==注入。配置采用properties内容前缀，注入不同的bean。
>
> 比如有dubbo.application前缀的键值对，则产生ApplicationConfig的bean。
>
> 第二种是==xml==注入。通过@ImportResource增加xml，使用spring自定义标签名称空间，根据配置注入多个dubboConfig的bean。

例如：@EnableDubbo注解和@ImportResources都在同一个XXXConfiguration中

在ConfigurationClassPostProcesser解析到这个XXXConfiguration时，会调用loadBeanDefinitionsForConfigurationClass方法，加载这个class引入的资源。

```java
	private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

		//...
    
		//xml方式入口
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    //注解式入口
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
```



#### 主配置@EnableDubbo

> 适用于dubbo的注解方式。如果是通过xml方式，添加了也是浪费。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
//添加DubboConfigBindingBeanPostProcessor
@EnableDubboConfig
//添加ServiceAnnotationBeanPostProcessor
//添加ReferenceAnnotationBeanPostProcesser
@DubboComponentScan
public @interface EnableDubbo {
		//指定DubboComponentScan的扫描包路径
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};
		//指定DubboComponentScan的扫描类路径
    @AliasFor(annotation = DubboComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};
		//对应EnableDubboConfig的是否多配置
    @AliasFor(annotation = EnableDubboConfig.class, attribute = "multiple")
    boolean multipleConfig() default true;
}
```

##### 注解@EnableDubboConfig

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
//添加DubboConfigBindingBeanPostProcessor
@Import(DubboConfigConfigurationRegistrar.class)
public @interface EnableDubboConfig {

    /**
     * It indicates whether binding to multiple Spring Beans.
     *
     * @return the default value is <code>false</code>
     * @revised 2.5.9
     */
  	//TODO
    boolean multiple() default true;
}
```

###### DubboConfigConfigurationRegistrar类

```java
public class DubboConfigConfigurationRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
				
      	//map组装成AnnotationAttributes
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(
 				//获取EnableDubboConfig注解属性。组装成map
importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName())
        );
				//获取是否多配置属性boolean。默认是true
        boolean multiple = attributes.getBoolean("multiple");

        // Single Config Bindings
      	// 注册单配置bean
        registerBeans(registry, DubboConfigConfiguration.Single.class);

      	// 如果是多配置属性为true，注册多配置bean
        if (multiple) { // Since 2.6.6 https://github.com/apache/dubbo/issues/3193
            registerBeans(registry, DubboConfigConfiguration.Multiple.class);
        }
				//注册dubboConfig的定义信息后置处理器DubboConfigAliasPostProcessor
        registerDubboConfigAliasPostProcessor(registry);
    }
		
  	//注册DubboConfigAliasPostProcessor的定义信息后置处理器
    private void registerDubboConfigAliasPostProcessor(BeanDefinitionRegistry registry) {
        registerInfrastructureBean(registry, DubboConfigAliasPostProcessor.BEAN_NAME, DubboConfigAliasPostProcessor.class);
    }

}
```

###### DubboConfigAliasPostProcessor

> bean后置处理器。给dubbo的AbstractConfig添加id跟bean的别名映射。

```java
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        
      	if (bean instanceof AbstractConfig) {
            String id = ((AbstractConfig) bean).getId();
            if (hasText(id)                                     // id MUST be present in AbstractConfig
                    && !nullSafeEquals(id, beanName)            // id MUST NOT be equal to bean name
                    && !hasAlias(registry, beanName, id)) {     // id MUST NOT be present in AliasRegistry
                registry.registerAlias(beanName, id);
            }
        }
        return bean;
    }
```



###### config的配置类DubboConfigConfiguration

> single和multiple是DubboConfigConfiguration的静态内部类。
>
> 使用@EnableDubboConfigBindings和@EnableDubboConfigBinding注解把DubboConfig注入到spring中。

```java
public class DubboConfigConfiguration {

    /**
     * Single Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.config-center", type = ConfigCenterBean.class),
            @EnableDubboConfigBinding(prefix = "dubbo.metadata-report", type = MetadataReportConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.metrics", type = MetricsConfig.class)
    })
    public static class Single {

    }

    /**
     * Multiple Dubbo {@link AbstractConfig Config} Bean Binding
     */
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.applications", type = ApplicationConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.modules", type = ModuleConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.registries", type = RegistryConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.protocols", type = ProtocolConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.monitors", type = MonitorConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.providers", type = ProviderConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.consumers", type = ConsumerConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.config-centers", type = ConfigCenterBean.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.metadata-reports", type = MetadataReportConfig.class, multiple = true),
            @EnableDubboConfigBinding(prefix = "dubbo.metricses", type = MetricsConfig.class, multiple = true)
    })
    public static class Multiple {

    }
}
```

###### @EnableDubboConfigBindings

> 他是一个EnableDubboConfigBinding标签的集合标签。
>
> 且。通过@Import导入了DubboConfigBinding==s==Registrar类

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboConfigBindingsRegistrar.class)
public @interface EnableDubboConfigBindings {

    EnableDubboConfigBinding[] value();

}

```

###### @EnableDubboConfigBinding

> 通过前缀匹配。解析成AbstractConfig的配置类型。
>
> 也注入了DubboConfigBindingRegistrar类。

```java
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(EnableDubboConfigBindings.class)
//这里的import不会生效。
//单独标注不会生效。
@Import(DubboConfigBindingRegistrar.class)
public @interface EnableDubboConfigBinding {

    String prefix();

    Class<? extends AbstractConfig> type();

    boolean multiple() default false;

}
```

###### DubboConfigBindingsRegistrar

> 实现了ImportBeanDefinitionRegistrar接口。用于注册bean的定义信息。
> 
> 实现了EnvironmentAware接口。给实例注入Environment。

```java
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
				//从DubboConfigConfiguration.Multiple.class类的元数据信息importingClassMetadata获取@EnableDubboConfigBindings的内容
      	AnnotationAttributes attributes = AnnotationAttributes.fromMap(
                importingClassMetadata.getAnnotationAttributes(EnableDubboConfigBindings.class.getName()));
				//获取注解属性的value。也就是@EnableDubboConfigBinding的标签集合。
      	//见下图evaluate
        AnnotationAttributes[] annotationAttributes = attributes.getAnnotationArray("value");
				//创建DubboConfigBindingRegistrar类实例。
      	//也就是单个标签Import的绑定信息注册类。用于注册所有标签中的AbstractConfig子类bean。
        DubboConfigBindingRegistrar registrar = new DubboConfigBindingRegistrar();
        registrar.setEnvironment(environment);

        for (AnnotationAttributes element : annotationAttributes) {
						//遍历单个注册器标签。注册。
            registrar.registerBeanDefinitions(element, registry);

        }
    }
```

attributes：<img src="/Users/wangke/Documents/markdown/file/dubbo/@EnableDubboConfigBindings-value.png" style="zoom:50%;" />

单个Config注册。

> DubboConfigBindingRegistrar类。
>
> 
> AnnotationAttributes是一个LinkedHashMap
> 

```java
    protected void registerBeanDefinitions(AnnotationAttributes attributes, BeanDefinitionRegistry registry) {
				//获取prefix属性的占位符。如dubbo.applications
        String prefix = environment.resolvePlaceholders(attributes.getString("prefix"));
				//获取type表示的bean的class对象
        Class<? extends AbstractConfig> configClass = attributes.getClass("type");
				//获取multiple属性
        boolean multiple = attributes.getBoolean("multiple");
				//注册dubbo的子类AbstractConfigBean
        registerDubboConfigBeans(prefix, configClass, multiple, registry);

    }
		//注册
    private void registerDubboConfigBeans(String prefix,
                                          Class<? extends AbstractConfig> configClass,
                                          boolean multiple,
                                          BeanDefinitionRegistry registry) {
				//获取环境变量中的的以prefix为前缀的所有属性集合。
        Map<String, Object> properties = getPrefixedProperties(environment.getPropertySources(), prefix);
				//没有就跳过
        if (CollectionUtils.isEmpty(properties)) {
            if (log.isDebugEnabled()) {
                log.debug("There is no property for binding to dubbo config class [" + configClass.getName()
                        + "] within prefix [" + prefix + "]");
            }
            return;
        }
				//这里是按beanname进行分组。返回set集合。
      	//单个beanconfig的用Collections.singleton返回set。
      	//e,g. 配置文件dubbo.applications.test1.name=test11 beanName就是test1
      	//e,g. 配置文件dubbo.application.name=test beanName就是org.apache.dubbo.config.ApplicationConfig#0
        Set<String> beanNames = multiple ? resolveMultipleBeanNames(properties) :
      					//单个configBean要生成自己独立的bean
                Collections.singleton(resolveSingleBeanName(properties, configClass, registry));
				//遍历所有的beanName
        for (String beanName : beanNames) {
						//使用beanName作为创建configClass类型bean的名称
            registerDubboConfigBean(beanName, configClass, registry);
						//注册DubboConfigBindingBeanPostProcessor后置处理器
            registerDubboConfigBindingBeanPostProcessor(prefix, beanName, multiple, registry);

        }
				//注册dubboConfigBean的定制器NamePropertyDefaultValueDubboConfigBeanCustomizer
        registerDubboConfigBeanCustomizers(registry);

    }

```

###### registerDubboConfigBean注册dubboConfigBean

> 这里跟xml注入的方式。不共用。

```java
    private void registerDubboConfigBean(String beanName, Class<? extends AbstractConfig> configClass,
                                         BeanDefinitionRegistry registry) {
				
      	//这里使用构造者创建bean定义信息。
      	//BeanDefinitionBuilder builder = new BeanDefinitionBuilder(new RootBeanDefinition());
				//builder.beanDefinition.setBeanClass(beanClass);
        BeanDefinitionBuilder builder = rootBeanDefinition(configClass);

        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();
				//注册进容器
        registry.registerBeanDefinition(beanName, beanDefinition);

        if (log.isInfoEnabled()) {
            log.info("The dubbo config bean definition [name : " + beanName + ", class : " + configClass.getName() +
                    "] has been registered.");
        }

    }
```

###### registerDubboConfigBindingBeanPostProcessor

> 注册DubboConfigBindingBeanPostProcessor。dubboConfigBean的后置处理器。

```java
    private void registerDubboConfigBindingBeanPostProcessor(String prefix, String beanName, boolean multiple,
                                                             BeanDefinitionRegistry registry) {
				//DubboConfigBindingBeanPostProcessor类
        Class<?> processorClass = DubboConfigBindingBeanPostProcessor.class;

        BeanDefinitionBuilder builder = rootBeanDefinition(processorClass);
				//获取真实的前缀
      	//multiple false dubbo.application
      	//multiple true  dubbo.applications.customBeanName
        String actualPrefix = multiple ? buildPrefix(prefix) + beanName : prefix;
				//顺序地添加进ConstructorArgumentValues中。
      	//{0:dubbo.application,1:org.apache.dubbo.config.ApplicationConfig#0}
 				//意思这里是针对beanName的当前config，进行后置处理。
        builder.addConstructorArgValue(actualPrefix).addConstructorArgValue(beanName);

        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();

        beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
				//注册进容器中
        registerWithGeneratedName(beanDefinition, registry);

        if (log.isInfoEnabled()) {
            log.info("The BeanPostProcessor bean definition [" + processorClass.getName()
                    + "] for dubbo config bean [name : " + beanName + "] has been registered.");
        }

    }
```

###### DubboConfigBindingBeanPostProcessor

> ConfigBean的后置处理器。用于绑定参数。
>
> 实现了==BeanPostProcessor==，后置处理器，==在AbstractConfig创建之后==，进行postProcessBeforeInitialization操作
>
> 实现了==InitializingBean==，当前bean创建时，进行afterPropertiesSet()，用于组件初始化。
>
> 后置处理器实例化时机是refresh->registerBeanPostProcessors.

```java
public class DubboConfigBindingBeanPostProcessor implements BeanPostProcessor, ApplicationContextAware, InitializingBean {
    //properties前缀
  	private final String prefix;
		//实例beanName
    private final String beanName;
		//config建造者
  	//可以自定义，实现DubboConfigBinder
  	//默认使用DefaultDubboConfigBinder
    private DubboConfigBinder dubboConfigBinder;

    private ApplicationContext applicationContext;
		//默认忽略未知字段
    private boolean ignoreUnknownFields = true;
		//默认忽略非法字段
    private boolean ignoreInvalidFields = true;
		//如果没有自定义。默认有一个NamePropertyDefaultValueDubboConfigBeanCustomizer
    private List<DubboConfigBeanCustomizer> configBeanCustomizers = Collections.emptyList();

		//后置处理器对AbstractConfig的包装入口
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

      	//每一个AbstractConfig类型的bean都有自己的后置处理器。
      	//类型和beanName检测
        if (this.beanName.equals(beanName) && bean instanceof AbstractConfig) {
						
            AbstractConfig dubboConfig = (AbstractConfig) bean;
						//执行dubboConfigBinder的bind方法
            bind(prefix, dubboConfig);
						//已经排序好的configBeanCustomizers，顺序遍历customize
            customize(beanName, dubboConfig);

        }

        return bean;

    }

		//本实例完成实例化赋值之后.
  	//对DubboConfigBinder和ConfigBeanCustomizers进行初始化。
    @Override
    public void afterPropertiesSet() throws Exception {
				
      	//优先获取容器中的DubboConfigBinder子类。
      	//默认DefaultDubboConfigBinder
      	//设置忽略未知字段和非法字段
        initDubboConfigBinder();

      	//从容器中获取所有的ConfigBeanCustomizers
      	//排序。支持Order接口、@Order、@Primary
        initConfigBeanCustomizers();

    }
		//DefaultDubboConfigBinder创建
    protected DubboConfigBinder createDubboConfigBinder(Environment environment) {
        DefaultDubboConfigBinder defaultDubboConfigBinder = new DefaultDubboConfigBinder();、
        //only set environment
        defaultDubboConfigBinder.setEnvironment(environment);
        return defaultDubboConfigBinder;
    }
  
  
  //....................

}
```

###### DefaultDubboConfigBinder 配置绑定

> 使用spring的DataBinder进行properties属性和dubboConfig的绑定

```java
public class DefaultDubboConfigBinder extends AbstractDubboConfigBinder {

    @Override
    public <C extends AbstractConfig> void bind(String prefix, C dubboConfig) {
      	//传入dubboConfig类
        DataBinder dataBinder = new DataBinder(dubboConfig);
        // Set ignored*设置忽略
        dataBinder.setIgnoreInvalidFields(isIgnoreInvalidFields());
        dataBinder.setIgnoreUnknownFields(isIgnoreUnknownFields());
        // 获取所有prefix的properties属性
        Map<String, Object> properties = getPrefixedProperties(getPropertySources(), prefix);
        // Convert Map to MutablePropertyValues
        MutablePropertyValues propertyValues = new MutablePropertyValues(properties);
        // 绑定
        dataBinder.bind(propertyValues);
    }

}


```

###### NamePropertyDefaultValueDubboConfigBeanCustomizer

> name属性默认值的自定义器
>
> 也就是反射获取dubboConfigBean的getName和setName方法。如果name没有值，就把beanName放进去。

##### 注解@DubboComponentScan

> 基于包路径或者类路径的dubbo组件扫描，针对@Service和@Reference
>
> 注入了一个DubboComponentScanRegistrar组件

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {

    String[] value() default {};

    String[] basePackages() default {};

    Class<?>[] basePackageClasses() default {};

}
```

###### DubboComponentScanRegistrar

> 注册ReferenceAnnotationBeanPostProcessor和ServiceAnnotationBeanPostProcessor的bean定义信息。

```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
				//获取扫包路径集合
      	//可以在@EnableDubbo或者单独@DubboComponentScan指定
      	//如果不指定。就使用注解标签所在的class所在包路径。
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

      	//注册ServiceAnnotationBeanPostProcessor
      	//@Service注解bean的后置处理器
        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);

      	//注册ReferenceAnnotationBeanPostProcessor
      	//@Refefence注解bean的后置处理器
        registerReferenceAnnotationBeanPostProcessor(registry);

    }

  //.............................
}
```

###### ==ServiceAnnotationBeanPostProcessor==

> 是一个BeanDefinitionRegistryPostProcessor。
>
> 执行时机refresh->invokeBeanFactoryPostProcessors->invokeBeanDefinitionRegistryPostProcessors

```java
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {

		//......

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
				//对扫包路径进行替换解析 占位符（如 ${}）替换
        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
          	//扫包。注册ServiceBeans。
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }
		
    //扫包。注册ServiceBeans。
    private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

      	//继承ClassPathBeanDefinitionScanner
      	//功能：把标注的bean定义信息注入到容器中。
        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);
				//beanName生成器。默认是AnnotationBeanNameGenerator。
        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

        scanner.setBeanNameGenerator(beanNameGenerator);
				//对新老版本的dubbo@Service注解进行过滤
        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));
        scanner.addIncludeFilter(new AnnotationTypeFilter(com.alibaba.dubbo.config.annotation.Service.class));
				
      	//多个包路径遍历扫描
        for (String packageToScan : packagesToScan) {

            //注册被dubbo@Service标注的bean的beanDefinitionHolders
          	//注意：已经被@ComponentScan扫入容器的不会在这个方法的父方法返回。
          	//这个scan也只是返回增加个数。
          	//要用下面的方法重新查找，进行容错。
            scanner.scan(packageToScan);

            //不管有没有被@ComponentScan注解扫入容器的。
          	//查找所有被@Service标注的bean
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {
								//循环注册ServiceBean
                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }
								//日志输出 忽略
            }

        }

    }

		//注册ServiceBean
    private void registerServiceBean(BeanDefinitionHolder beanDefinitionHolder, BeanDefinitionRegistry registry,
                                     DubboClassPathBeanDefinitionScanner scanner) {
				//bean的类class。也就是XXXXServiceImpl.class
        Class<?> beanClass = resolveClass(beanDefinitionHolder);
				//获取@Service注解。
        Annotation service = findServiceAnnotation(beanClass);
				//获取@Service注解属性，组成AnnotationAttributes
        AnnotationAttributes serviceAnnotationAttributes = getAnnotationAttributes(service, false, false);

      	//获取bean实现的接口类
      	//没有接口，则使用属性声明的interfaceClass；
      	//获取所有接口，取第一个。
        Class<?> interfaceClass = resolveServiceInterfaceClass(serviceAnnotationAttributes, beanClass);
				
      	//beanName
        String annotatedServiceBeanName = beanDefinitionHolder.getBeanName();
				
      	//生成ServiceBean的BeanDefinition
        AbstractBeanDefinition serviceBeanDefinition =
                buildServiceBeanDefinition(service, serviceAnnotationAttributes, interfaceClass, annotatedServiceBeanName);

        // ServiceBean的BeanName
      	// 使用ServiceBeanNameBuilder建造者 参数：interfaceClass、属性group、属性version
        String beanName = generateServiceBeanName(serviceAnnotationAttributes, interfaceClass);

        if (scanner.checkCandidate(beanName, serviceBeanDefinition)) { // check duplicated candidate bean
            //注册进容器中。
          	registry.registerBeanDefinition(beanName, serviceBeanDefinition);
						// debug日志
        } else {
						// warm日志：多个bean实现同一个接口。版本和分组相同。
        }

    }
		
    //创建ServiceBean的定义信息
    private AbstractBeanDefinition buildServiceBeanDefinition(Annotation serviceAnnotation,
                                                              AnnotationAttributes serviceAnnotationAttributes,
                                                              Class<?> interfaceClass,
                                                              String annotatedServiceBeanName) {
				//建造者模式
        BeanDefinitionBuilder builder = rootBeanDefinition(ServiceBean.class);

        AbstractBeanDefinition beanDefinition = builder.getBeanDefinition();

        MutablePropertyValues propertyValues = beanDefinition.getPropertyValues();

      	//对象转数组 ObjectUtils.of
        String[] ignoreAttributeNames = of("provider", "monitor", "application", "module", "registry", "protocol",
                "interface", "interfaceName", "parameters");
				
      	//获取所有的@Service配置的跟默认属性不同的列表组成PropertyValue集合
      	//设置的值会跟默认值通过ObjectUtils.nullSafeEquals对比。过滤掉相同的。
        propertyValues.addPropertyValues(new AnnotationPropertyValuesAdapter(serviceAnnotation, environment, ignoreAttributeNames));

        // 添加ref属性。指向被注解类的beanName
        addPropertyReference(builder, "ref", annotatedServiceBeanName);
        // 添加interface属性。被注解类实现的接口。
        builder.addPropertyValue("interface", interfaceClass.getName());
        // 把属性列表{key1, value1, key2, value2} 转成 key-value的PropertyValue
        builder.addPropertyValue("parameters", convertParameters(serviceAnnotationAttributes.getStringArray("parameters")));
        // 添加 methods参数
        List<MethodConfig> methodConfigs = convertMethodConfigs(serviceAnnotationAttributes.get("methods"));
        if (!methodConfigs.isEmpty()) {
            builder.addPropertyValue("methods", methodConfigs);
        }

      	//provider、monitor、application、module、registry、protocol
      	//这些属性如果有指定。就添加RuntimeBeanReference。包装进PropertyValue中。

        return builder.getBeanDefinition();

    }

}
```

###### ==ServiceBean==

> 继承ServiceConfig<T>
>
> 实现了InitializingBean。属性完成之后，对父ServiceConfig的dubboConfig相关进行检查赋值。
>
> 实现了DisposableBean。已转到org.apache.dubbo.config.spring.extension.SpringExtensionFactory.ShutdownHookListener
>
> 实现了ApplicationContextAware、BeanNameAware、ApplicationEventPublisherAware
>
> 实现了ApplicationListener<ContextRefreshedEvent> 对容器刷新完成事件进行监听。调用父类暴露服务。
>
> 发布ServiceBeanExportedEvent事件。
>
> 后面服务注册再看。

###### ==ReferenceAnnotationBeanPostProcessor==

> 继承AnnotationInjectedBeanPostProcessor。这是dubbo提供的父抽象类。
>
> ==AnnotationInjectedBeanPostProcessor==继承了InstantiationAwareBeanPostProcessorAdapter。实现了
>MergedBeanDefinitionPostProcessor, PriorityOrdered,BeanFactoryAware, BeanClassLoaderAware, EnvironmentAware, DisposableBean接口。跟==AutowiredAnnotationBeanPostProcessor==一样。只是高一个优先级。

例如

```java
@RestController
public class AuthorityController {

    @Reference
    private IUserAuthorityService userAuthorityService;
    @Reference
    private IAuthorityService authorityService;
}
```

```java
		// 实现MergedBeanDefinitionPostProcessor接口
		// 实例化之后，放入earlySingletonExposure之前，populateBean之前。
		// 调用所有的MergedBeanDefinitionPostProcessor处理
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}
//====================================================================================
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
        //执行MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition方法
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
//====================================================================================
		//父类AnnotationInjectedBeanPostProcessor
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanType != null) {
          	//获取注入用的元数据
            InjectionMetadata metadata = findInjectionMetadata(beanName, beanType, null);
          	//元数据检查
            metadata.checkConfigMembers(beanDefinition);
        }
    }
```

获取注入用的元数据findInjectionMetadata。

> 只获取和组装。不使用。

```java
    private InjectionMetadata findInjectionMetadata(String beanName, Class<?> clazz, PropertyValues pvs) {
       
      	//获得缓存key值 默认使用beanName，没有就使用类全名
        String cacheKey = (StringUtils.hasLength(beanName) ? beanName : clazz.getName());
      	//成员级别的本地缓存。this.injectionMetadataCache是一个初始大小32位的ConCurrentHashMap
        //类似于单例的双端检测。读-锁-读
      	
      	//读
        AnnotationInjectedBeanPostProcessor.AnnotatedInjectionMetadata metadata = this.injectionMetadataCache.get(cacheKey);
        //检查是否需要刷新【重新生成】
      	//metadata为空或metadata跟clazz不匹配
      	if (InjectionMetadata.needsRefresh(metadata, clazz)) {
          	//锁
            synchronized (this.injectionMetadataCache) {
              	//读
                metadata = this.injectionMetadataCache.get(cacheKey);
              	//再检查是否需要刷新
                if (InjectionMetadata.needsRefresh(metadata, clazz)) {
                    if (metadata != null) {
                        metadata.clear(pvs);
                    }
                    try {
                      	//创建注解过得元数据
                        metadata = buildAnnotatedMetadata(clazz);
                      	//放入缓存中
                        this.injectionMetadataCache.put(cacheKey, metadata);
                    } catch (NoClassDefFoundError err) {
                        throw new IllegalStateException("Failed to introspect object class [" + clazz.getName() +
                                "] for annotation metadata: could not find class that it depends on", err);
                    }
                }
            }
        }
        return metadata;
    }
```

构建数据元buildAnnotatedMetadata

```java
    private AnnotationInjectedBeanPostProcessor.AnnotatedInjectionMetadata buildAnnotatedMetadata(final Class<?> beanClass) {
        //获取注解属性数据元
      	//AnnotatedFieldElement是spring的InjectionMetadata.InjectedElement子类。
      	//保存单个属性信息
      	Collection<AnnotationInjectedBeanPostProcessor.AnnotatedFieldElement> fieldElements = findFieldAnnotationMetadata(beanClass);
        //获取注解方法数据元
      	//AnnotatedMethodElement是spring的InjectionMetadata.InjectedElement子类。
      	//保存单个方法信息
      	Collection<AnnotationInjectedBeanPostProcessor.AnnotatedMethodElement> methodElements = findAnnotatedMethodMetadata(beanClass);
        //构建AnnotationInjectedBeanPostProcessor.AnnotatedInjectionMetadata
      	return new AnnotationInjectedBeanPostProcessor.AnnotatedInjectionMetadata(beanClass, fieldElements, methodElements);
    }
```

- findFieldAnnotationMetadata

  ```java
      private List<AnnotationInjectedBeanPostProcessor.AnnotatedFieldElement> findFieldAnnotationMetadata(final Class<?> beanClass) {
  				//新建链表
          final List<AnnotationInjectedBeanPostProcessor.AnnotatedFieldElement> elements = new LinkedList<AnnotationInjectedBeanPostProcessor.AnnotatedFieldElement>();
  				//通过反射工具类。对每一个属性进行操作
          ReflectionUtils.doWithFields(beanClass, field -> {
  						
            	//内层遍历。新老版本的@Reference注解类
              for (Class<? extends Annotation> annotationType : getAnnotationTypes()) {
  								//获取注解的属性集合
                  AnnotationAttributes attributes = getMergedAttributes(field, annotationType, getEnvironment(), true);
  
                  if (attributes != null) {
  										//静态属性报错
                      if (Modifier.isStatic(field.getModifiers())) {
                          if (logger.isWarnEnabled()) {
                              logger.warn("@" + annotationType.getName() + " is not supported on static fields: " + field);
                          }
                          return;
                      }
  										//生成AnnotatedFieldElement对象
                      elements.add(new AnnotatedFieldElement(field, attributes));
                  }
              }
          });
  
          return elements;
  
      }
  ```

  

- findAnnotatedMethodMetadata

  > findBridgedMethod是spring.core.BridgeMethodResolver的获取桥接方法的方法。
  >
  > ==java桥接方法==是1.5引入泛型之后，为了向前兼容。
  >
  > 【例如1.5没有泛型之前，集合操作，任何Object的子类都可以存放，使用需要进行类型判断。加入泛型之后，如果没有桥接方法，会编译期报错。】
  >
  > 也就是子类方法【public String method(String arg)】对于父类泛型方法【public <T> T method(T arg)】的实现。为了兼容1.5之前版本。会生成【public Object method(Object arg)】的桥接方法，方法内把Object类型强转为String类型调用子类实际方法。
  >
  > ==PropertyDescriptor==对类的属性操作器。
  >
  > 一个类中有属性fieldA。那么对fieldA的读操作get/isFieldA和写操作setFieldA是固定的。
  >
  > 且。手写反射调用方法需要aField.setAccessible(true)，属于越权的中危漏洞。
  >
  > 且。一个类的属性的操作器不能每次都new。防止内存泄漏。
  >
  > 所以Spring提供了一个BeanUtils，做了应用级别的缓存。

```java
    private List<AnnotationInjectedBeanPostProcessor.AnnotatedMethodElement> findAnnotatedMethodMetadata(final Class<?> beanClass) {
				//新建链表
        final List<AnnotationInjectedBeanPostProcessor.AnnotatedMethodElement> elements = new LinkedList<AnnotationInjectedBeanPostProcessor.AnnotatedMethodElement>();
				//通过注解类，对所有方法进行操作
        ReflectionUtils.doWithMethods(beanClass, method -> {
						//如果当前方式是桥接方法，就找到他的目标方法
            Method bridgedMethod = findBridgedMethod(method);
						//不可见。不操作。
            if (!isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                return;
            }

						//内层对新老版本的@Reference进行遍历
            for (Class<? extends Annotation> annotationType : getAnnotationTypes()) {
								//获取注解属性
                AnnotationAttributes attributes = getMergedAttributes(bridgedMethod, annotationType, getEnvironment(), true);

                if (attributes != null && method.equals(ClassUtils.getMostSpecificMethod(method, beanClass))) {
                    //方法不能是静态的
                  	if (Modifier.isStatic(method.getModifiers())) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("@" + annotationType.getName() + " annotation is not supported on static methods: " + method);
                        }
                        return;
                    }
                  	//方法需要有参数
                    if (method.getParameterTypes().length == 0) {
                        if (logger.isWarnEnabled()) {
                            logger.warn("@" + annotationType.getName() + " annotation should only be used on methods with parameters: " +
                                    method);
                        }
                    }
                  	//获取方法的属性操作者(getter/setter) PropertyDescriptor
                  	//这里一般是获取不到值的。因为注入的时候不会同时添加getter/setter
                    PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, beanClass);
                    //生成AnnotatedMethodElement对象
                  	elements.add(new AnnotatedMethodElement(method, pd, attributes));
                }
            }
        });

        return elements;

    }
```

这时。就在ReferenceAnnotationBeanPostProcessor缓存中有了元数据。

接着就到了authorityController的属性赋值populateBean

> populateBean中会调用所有的InstantiationAwareBeanPostProcessor接口bean。
>
> 依次调用postProcessAfterInstantiation、postProcessProperties、postProcessPropertyValues方法
>
> ReferenceAnnotationBeanPostProcessor的父类继承了InstantiationAwareBeanPostProcessorAdapter实现了InstantiationAwareBeanPostProcessor接口。postProcessAfterInstantiation为true【false不会执行后面方法】。
>
> 不对properties进行后置处理，postProcessProperties直接返回bean。主要操作在postProcessPropertyValues中。ReferenceAnnotationBeanPostProcessor父类进行了重写。

```java
    
		//在这里调用之前会执行filterPropertyDescriptorsForDependencyCheck
		//默认增加一个GenericTypeAwarePropertyDescriptor[name=class]
		//对Object的class属性进行操作的属性执行器
		@Override
    public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
				//获取注入用的元数据。这跟前面元数据封装的时候相同。
      	//本次不会进行刷新，直接从缓存中获取
        InjectionMetadata metadata = findInjectionMetadata(beanName, bean.getClass(), pvs);
        try {
          	//进行注入
            metadata.inject(bean, beanName, pvs);
        } //异常捕获.throw ex..
        return pvs;
    }
```

Metadata注入

```java
	public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
		//这些元素是数据组装完成后。经过检查。需要放入externallyManagedConfigMembers集合中的元素。
    //TODO 还不知道干嘛用的。
    Collection<InjectedElement> checkedElements = this.checkedElements;
    //没有checkedElements就使用injectedElements
		Collection<InjectedElement> elementsToIterate =
				(checkedElements != null ? checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
			for (InjectedElement element : elementsToIterate) {
				//日志...忽略
        
       	//注入
				element.inject(target, beanName, pvs);
			}
		}
	}
```

element注入

```java
        @Override
        protected void inject(Object bean, String beanName, PropertyValues pvs) throws Throwable {
						//注入属性类型
            Class<?> injectedType = field.getType();
						//获取注入对象
            Object injectedObject = getInjectedObject(attributes, bean, beanName, injectedType, this);
						//如果属性是private。设置makeAccessible。
            ReflectionUtils.makeAccessible(field);
						//设置属性
            field.set(bean, injectedObject);

        }
```

getInjectedObject获取注入对象

```java
    protected Object getInjectedObject(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {
				//组装注入对象的缓存key
      	//类型+名称
      	//e,g. ServiceBean:com.uteam.zen.api.IUserAuthorityService#source=private com.uteam.zen.api.IUserAuthorityService com.uteam.zen.admin.web.controller.AuthorityController.userAuthorityService#attributes={}
        String cacheKey = buildInjectedObjectCacheKey(attributes, bean, beanName, injectedType, injectedElement);
				//优先取缓存
        Object injectedObject = injectedObjectsCache.get(cacheKey);

        if (injectedObject == null) {
          	// 获取目标对象
            injectedObject = doGetInjectedBean(attributes, bean, beanName, injectedType, injectedElement);
            // 放入缓存
            injectedObjectsCache.putIfAbsent(cacheKey, injectedObject);
        }

        return injectedObject;

    }
```

doGetInjectedBean获取注入对象

```java
    @Override
    protected Object doGetInjectedBean(AnnotationAttributes attributes, Object bean, String beanName, Class<?> injectedType,
                                       InjectionMetadata.InjectedElement injectedElement) throws Exception {

        /**
         * The name of bean that annotated Dubbo's {@link Service @Service} in local Spring {@link ApplicationContext}
         */
      	//获取目标对象的bean名称 通过ServiceBeanNameBuilder
      	//e,g. ServiceBean:{injectedType}
        String referencedBeanName = buildReferencedBeanName(attributes, injectedType);

        /**
         * The name of bean that is declared by {@link Reference @Reference} annotation injection
         */
      	//获取被注接属性名称 优先使用注解@Reference的id属性
      	//e,g. @Reference {injectedType}
        String referenceBeanName = getReferenceBeanName(attributes, injectedType);
				//如果不存在则创建对象
        ReferenceBean referenceBean = buildReferenceBeanIfAbsent(referenceBeanName, attributes, injectedType);
				//TODO 
        registerReferenceBean(referencedBeanName, referenceBean, attributes, injectedType);
				//根据类型放入injectedFieldReferenceBeanCache或者injectedMethodReferenceBeanCache缓存
        cacheInjectedReferenceBean(referenceBean, injectedElement);
				//返回代理 TODO
        return getOrCreateProxy(referencedBeanName, referenceBeanName, referenceBean, injectedType);
    }
```

buildReferenceBeanIfAbsent创建ReferenceBean对象

```java
    private ReferenceBean buildReferenceBeanIfAbsent(String referenceBeanName, AnnotationAttributes attributes,
                                                     Class<?> referencedType)
            throws Exception {
				//缓存中获取
        ReferenceBean<?> referenceBean = referenceBeanCache.get(referenceBeanName);

        if (referenceBean == null) {
          	//使用ReferenceBeanBuilder进行创建
          	//组装ReferenceBeanBuilder
            ReferenceBeanBuilder beanBuilder = ReferenceBeanBuilder
                    .create(attributes, applicationContext)
                    .interfaceClass(referencedType);
          	//执行创建
            referenceBean = beanBuilder.build();
          	//放入referenceBeanCache缓存
            referenceBeanCache.put(referenceBeanName, referenceBean);
        } else if (!referencedType.isAssignableFrom(referenceBean.getInterfaceClass())) {
            //bean重名 报错
          	throw new IllegalArgumentException("reference bean name " + referenceBeanName + " has been duplicated, but interfaceClass " +
                    referenceBean.getInterfaceClass().getName() + " cannot be assigned to " + referencedType.getName());
        }
        return referenceBean;
    }
```

ReferenceBeanBuilder创建对象

> 继承了AnnotatedInterfaceConfigBeanBuilder<C extends AbstractInterfaceConfig>是模板方法模式抽象类
>
> ```java
>     public final C build() throws Exception {
> 				//依赖检查 空方法
>         checkDependencies();
> 				//需要子类实现的doBuild方法
>         C configBean = doBuild();
> 				//配置bean
>         configureBean(configBean);
> 				//返回
>         return configBean;
> 
>     }
> ```
>
> 创建了一个ReferenceBean对象

- doBuild();

  ```java
  //子类ReferenceBeanBuilder
  @Override
  protected ReferenceBean doBuild() {
  	return new ReferenceBean<Object>();
  }
  ```

- configureBean(configBean);

  ```java
  protected void configureBean(C configBean) throws Exception {
  		
      preConfigureBean(attributes, configBean);
  
      configureRegistryConfigs(configBean);
  
      configureMonitorConfig(configBean);
  
      configureApplicationConfig(configBean);
  
      configureModuleConfig(configBean);
  
      postConfigureBean(attributes, configBean);
  
  }
  ```

  - preConfigureBean

    ```java
        @Override
        protected void preConfigureBean(AnnotationAttributes attributes, ReferenceBean referenceBean) {
            //检查interface设值
          	Assert.notNull(interfaceClass, "The interface class must set first!");
          	//创建一个target是referenceBean的数据绑定器
            DataBinder dataBinder = new DataBinder(referenceBean);
          	//给dataBinder注册属性的自定义编辑器。
            dataBinder.registerCustomEditor(String.class, "filter", new StringTrimmerEditor(true));
            dataBinder.registerCustomEditor(String.class, "listener", new StringTrimmerEditor(true));
            dataBinder.registerCustomEditor(Map.class, "parameters", new PropertyEditorSupport() {
                @Override
                public void setAsText(String text) throws java.lang.IllegalArgumentException {
                    // Trim all whitespace
                    String content = StringUtils.trimAllWhitespace(text);
                    if (!StringUtils.hasText(content)) { // No content , ignore directly
                        return;
                    }
                    // replace "=" to ","
                    content = StringUtils.replace(content, "=", ",");
                    // replace ":" to ","
                    content = StringUtils.replace(content, ":", ",");
                    // String[] to Map
                    Map<String, String> parameters = CollectionUtils.toStringMap(commaDelimitedListToStringArray(content));
                    setValue(parameters);
                }
            });
            //绑定注解属性
            dataBinder.bind(new AnnotationPropertyValuesAdapter(attributes, applicationContext.getEnvironment(), IGNORE_FIELD_NAMES));
        }
    ```

  - configureRegistryConfigs
    
    ```java
        private void configureRegistryConfigs(C configBean) {
    				
          	//如果属性配置了registry
            String[] registryConfigBeanIds = resolveRegistryConfigBeanNames(attributes);
    				//从容器中获取指定名称的RegistryConfig实例
            List<RegistryConfig> registryConfigs = getBeans(applicationContext, registryConfigBeanIds, RegistryConfig.class);
    				//如果这里是单纯的dubbo消费端。没有RegistryConfig
          	//就是一个Collections.EmptyList
            configBean.setRegistries(registryConfigs);
    
        }
    ```
    
  - configureMonitorConfig
    
    ```java
        private void configureMonitorConfig(C configBean) {
    				//如果属性配置了monitor。获取设值的monitorBeanName
            String monitorBeanName = resolveMonitorConfigBeanName(attributes);
    				//从容器中获取指定beanName的MonitorConfig
            MonitorConfig monitorConfig = getNullableBean(applicationContext, monitorBeanName, MonitorConfig.class);
    				//设值。可以为null。
            configBean.setMonitor(monitorConfig);
    
        }
    ```
    
  - configureApplicationConfig、configureModuleConfig跟前面一样。
    
  - postConfigureBean
  
    ```java
        @Override
        protected void postConfigureBean(AnnotationAttributes attributes, ReferenceBean bean) throws Exception {
    				//set容器
            bean.setApplicationContext(applicationContext);
    				//设置Interface相关
            configureInterface(attributes, bean);
    				//设置ConsumerConfig TODO
            configureConsumerConfig(attributes, bean);
    				//设置MethodConfig TODO
            configureMethodConfig(attributes, bean);
    				//执行ReferenceBean的afterPropertiesSet
          	//这里是对属性进行手动赋值。
            bean.afterPropertiesSet();
    
        }
    ```
  
    
  

###### ==ReferenceBean==

> 是一个工厂bean，实现了FactoryBean接口。
>
> 实现了ApplicationContextAware接口，给组件注入容器属性。
>
> 实现了InitializingBean, DisposableBean接口，初始化和销毁bean方法。
>
> 继承了ReferenceConfig。
>
> 主要看初始化方法==afterPropertiesSet==和工厂bean方法==getObject==
>
> 后面服务发现再看。

#### xml方式注入dubbo配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://dubbo.apache.org/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://dubbo.apache.org/schema/dubbo
       http://dubbo.apache.org/schema/dubbo/dubbo.xsd">
    <!--提供方应用信息，用于计算依赖关系-->
    <dubbo:application name="${application.name}">
        <dubbo:parameter key="qos.enable" value="false"/>
    </dubbo:application>
		<!--配置注册中心地址-->
    <dubbo:registry protocol="zookeeper" address="${dubbo.registry.address}" check="false"/>
  	<!--数据源报告-->
    <dubbo:metadata-report address="${dubbo.registry.address}"/>
		<!--配置dubbo协议-->
    <dubbo:protocol name="${dubbo.protocol}" port="-1"/>

</beans>
```
```java
@ImportResource(locations = {"classpath*:dubbo-provider.xml"})
```

> 使用ApplicationContext获取文件全路径，使用XmlBeanDefinitionReader对文件进行编码，解析document。
>
> 获取 http://dubbo.apache.org/schema/dubbo 名称空间。在dubbo包下/META-INF/spring.handlers。获得名称空间处理器DubboNamespaceHandler。初始化。执行init方法。

##### DubboNamespaceHandler


```java
    @Override
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("config-center", new DubboBeanDefinitionParser(ConfigCenterBean.class, true));
        registerBeanDefinitionParser("metadata-report", new DubboBeanDefinitionParser(MetadataReportConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("metrics", new DubboBeanDefinitionParser(MetricsConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new AnnotationBeanDefinitionParser());
    }
```

###### DubboBeanDefinitionParser

> 【建造者】dubbo的bean定义信息解析器。负责解析生成beanDefinition。

构造器

```java
    public DubboBeanDefinitionParser(Class<?> beanClass, boolean required) {
        this.beanClass = beanClass;
      	//是否需要节点id
      	//如果需要但是没有指定。则按照规则生成
        this.required = required;
    }
```

实现了BeanDefinitionParser接口

```java
public class DubboBeanDefinitionParser implements BeanDefinitionParser
//spring定义的bean定义信息解析器。
//将xml节点element解析成BeanDefinition。
public interface BeanDefinitionParser {
  
	@Nullable
	BeanDefinition parse(Element element, ParserContext parserContext);

}
```

###### registerBeanDefinitionParser

> 继承spring的抽象NamespaceHandlerSupport类

NamespaceHandlerSupport类

```java
public abstract class NamespaceHandlerSupport implements NamespaceHandler {
  //提供一个Map存放节点名称跟对应解析器的关系
  private final Map<String, BeanDefinitionParser> parsers = new HashMap<>();
  //加入map中
  protected final void registerBeanDefinitionParser(String elementName, BeanDefinitionParser parser) {
		this.parsers.put(elementName, parser);
	}
  //对<beans>的一级子节点进行解析
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		return (parser != null ? parser.parse(element, parserContext) : null);
	}

	//根据节点名称从map中获取对应的BeanDefinitionParser
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		String localName = parserContext.getDelegate().getLocalName(element);
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}
}
```

###### 解析dubbo节点配置

```java
    @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
      	//给容器注册一个DubboConfigAliasPostProcessor。给bean添加alias。
        registerDubboConfigAliasPostProcessor(parserContext.getRegistry());

        return parse(element, parserContext, beanClass, required);
    }
```

解析

> RuntimeBeanReference。字面意思运行时bean引用。在bean的populateBean装配属性阶段。会把PropertyValues中的所有拿出来进行赋值。当value是RuntimeBeanReference类型时，会使用BeanDefinitionValueResolver将指定的beanName转化为bean实例。
>
> ```java
> public Object resolveValueIfNecessary(Object argName, @Nullable Object value) {
> 		// We must check each value to see whether it requires a runtime reference
> 		// to another bean to be resolved.
> 		if (value instanceof RuntimeBeanReference) {
> 			RuntimeBeanReference ref = (RuntimeBeanReference) value;
> 			return resolveReference(argName, ref);
> 		}
>   	//......
> }
> ```
>
> 

```java
private static BeanDefinition parse(Element element, ParserContext parserContext, Class<?> beanClass, boolean required) {
        //新建定义信息
  			RootBeanDefinition beanDefinition = new RootBeanDefinition();
  			//beanClass
        beanDefinition.setBeanClass(beanClass);
  			//不是懒加载。容器启动就实例化。
        beanDefinition.setLazyInit(false);
  			//获取当前节点的id属性
        String id = element.getAttribute("id");
  			//如果没有id，id又是需要的情况
        if (StringUtils.isEmpty(id) && required) {
          	//获取节点name属性
            String generatedBeanName = element.getAttribute("name");
            if (StringUtils.isEmpty(generatedBeanName)) {
              	//协议配置bean的默认名称是dubbo
                if (ProtocolConfig.class.equals(beanClass)) {
                    generatedBeanName = "dubbo";
                } else {
                  	//其他配置优先使用interface属性值作为名称
                    generatedBeanName = element.getAttribute("interface");
                }
            }
          	//如果要是没有就使用beanClass的全类名
            if (StringUtils.isEmpty(generatedBeanName)) {
                generatedBeanName = beanClass.getName();
            }
            id = generatedBeanName;
            int counter = 2;
          	//如果当前容器中有重复的bean。后缀增加序号。
            while (parserContext.getRegistry().containsBeanDefinition(id)) {
                id = generatedBeanName + (counter++);
            }
        }
        if (StringUtils.isNotEmpty(id)) {
          	//重复。抛异常。
            if (parserContext.getRegistry().containsBeanDefinition(id)) {
                throw new IllegalStateException("Duplicate spring bean id " + id);
            }
          	//注册进bean容器中。
            parserContext.getRegistry().registerBeanDefinition(id, beanDefinition);
          	//增加bean的属性。
            beanDefinition.getPropertyValues().addPropertyValue("id", id);
        }
  			
        if (ProtocolConfig.class.equals(beanClass)) {
          	//dubbo:protocol
          	//对容器中的所有bean定义信息进行遍历
          	//如果属性里有需要protocol引用的。就把当前的ProtocolConfig的id封装成RuntimeBeanReference设置为属性value。
            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
                if (property != null) {
                    Object value = property.getValue();
                    if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                        definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                    }
                }
            }
        } else if (ServiceBean.class.equals(beanClass)) {
          	//dubbo:service
          	//获取指定的class
            String className = element.getAttribute("class");
            if (StringUtils.isNotEmpty(className)) {
              	//为目标实现类，生成bean的定义信息。
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                parseProperties(element.getChildNodes(), classDefinition);
              	//并生成实现类bean定义信息的持有者BeanDefinitionHolder
              	//放在属性ref中
              	//beanName为id+"Impl"
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        } else if (ProviderConfig.class.equals(beanClass)) {
          	//dubbo:provider
          	//嵌套解析dubbo:service
            parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        } else if (ConsumerConfig.class.equals(beanClass)) {
          	//嵌套解析dubbo:reference
            parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
        }
        Set<String> props = new HashSet<>();
        ManagedMap parameters = null;
  			//获取所有的方法
        for (Method setter : beanClass.getMethods()) {
            String name = setter.getName();
          	//public setXXXX(单参) 方法
            if (name.length() > 3 && name.startsWith("set")
                    && Modifier.isPublic(setter.getModifiers())
                    && setter.getParameterTypes().length == 1) {
                Class<?> type = setter.getParameterTypes()[0];
              	//截取方法名，转为首字母小写。
                String beanProperty = name.substring(3, 4).toLowerCase() + name.substring(4);
              	//驼峰命名转为-连接的字符串
              	//camelToSplitName -> camel-to-split-name
                String property = StringUtils.camelToSplitName(beanProperty, "-");
             		//加入属性表中
                props.add(property);
                // check the setter/getter whether match
              	// 获取getXXXX或者isXXXX方法。
                Method getter = null;
                try {
                    getter = beanClass.getMethod("get" + name.substring(3), new Class<?>[0]);
                } catch (NoSuchMethodException e) {
                    try {
                        getter = beanClass.getMethod("is" + name.substring(3), new Class<?>[0]);
                    } catch (NoSuchMethodException e2) {
                        // ignore, there is no need any log here since some class implement the interface: EnvironmentAware,
                        // ApplicationAware, etc. They only have setter method, otherwise will cause the error log during application start up.
                    }
                }
                if (getter == null
                        || !Modifier.isPublic(getter.getModifiers())
                        || !type.equals(getter.getReturnType())) {
                    continue;
                }
                if ("parameters".equals(property)) {
                  	//如果属性为parameters
                  	//解析子节点的dubbo:parameter集合。组装成ManagedMap TODO
                    parameters = parseParameters(element.getChildNodes(), beanDefinition);
                } else if ("methods".equals(property)) {
                  	//如果属性为methods
                  	//解析子节点的dubbo:method集合。封装成对应bean的BeanDefinitionHolder组成的ManagedList TODO
                    parseMethods(id, element.getChildNodes(), beanDefinition, parserContext);
                } else if ("arguments".equals(property)) {
                  	//如果属性为arguments
                  	//解析子节点dubbo:argument集合。封装成对应bean的BeanDefinitionHolder组成的ManagedList TODO
                    parseArguments(id, element.getChildNodes(), beanDefinition, parserContext);
                } else {
                  	//如果当前节点声明有这个属性。
                    String value = element.getAttribute(property);
                    if (value != null) {
                        value = value.trim();
                        if (value.length() > 0) {
                          	//当前节点设置了registry是N/A。直接new一个RegistryConfig塞到属性中
                            if ("registry".equals(property) && RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(value)) {
                                RegistryConfig registryConfig = new RegistryConfig();
                                registryConfig.setAddress(RegistryConfig.NO_AVAILABLE);
                                beanDefinition.getPropertyValues().addPropertyValue(beanProperty, registryConfig);
                            } else if ("provider".equals(property) || "registry".equals(property) || ("protocol".equals(property) && ServiceBean.class.equals(beanClass))) {
                                /**
                                 * For 'provider' 'protocol' 'registry', keep literal value (should be id/name) and set the value to 'registryIds' 'providerIds' protocolIds'
                                 * The following process should make sure each id refers to the corresponding instance, here's how to find the instance for different use cases:
                                 * 1. Spring, check existing bean by id, see{@link ServiceBean#afterPropertiesSet()}; then try to use id to find configs defined in remote Config Center
                                 * 2. API, directly use id to find configs defined in remote Config Center; if all config instances are defined locally, please use {@link org.apache.dubbo.config.ServiceConfig#setRegistries(List)}
                                 */
                                beanDefinition.getPropertyValues().addPropertyValue(beanProperty + "Ids", value);
                            } else {
                                Object reference;
                                if (isPrimitive(type)) {
                                  	//出于对老版本的兼容。把老版本之前属性的默认值统一修改为null。
                                    if ("async".equals(property) && "false".equals(value)
                                            || "timeout".equals(property) && "0".equals(value)
                                            || "delay".equals(property) && "0".equals(value)
                                            || "version".equals(property) && "0.0.0".equals(value)
                                            || "stat".equals(property) && "-1".equals(value)
                                            || "reliable".equals(property) && "false".equals(value)) {
                                        // backward compatibility for the default value in old version's xsd
                                        value = null;
                                    }
                                    reference = value;
                                } else if (ONRETURN.equals(property) || ONTHROW.equals(property) || ONINVOKE.equals(property)) {
                                  	//设置调用前，调用后，调用异常的回调。
                                    int index = value.lastIndexOf(".");
                                    String ref = value.substring(0, index);
                                    String method = value.substring(index + 1);
                                    reference = new RuntimeBeanReference(ref);
                                    beanDefinition.getPropertyValues().addPropertyValue(property + METHOD, method);
                                } else {
                                  	//如果当前属性是ref，那目标引用一定要是单例。
                                    if ("ref".equals(property) && parserContext.getRegistry().containsBeanDefinition(value)) {
                                        BeanDefinition refBean = parserContext.getRegistry().getBeanDefinition(value);
                                        if (!refBean.isSingleton()) {
                                            throw new IllegalStateException("The exported service ref " + value + " must be singleton! Please set the " + value + " bean scope to singleton, eg: <bean id=\"" + value + "\" scope=\"singleton\" ...>");
                                        }
                                    }
                                  	//添加对目标引用的RuntimeBeanReference
                                    reference = new RuntimeBeanReference(value);
                                }
                              	//设置属性
                                beanDefinition.getPropertyValues().addPropertyValue(beanProperty, reference);
                            }
                        }
                    }
                }
            }
        }
  			//把当前节点声明的其他属性。放入parameters集合中。
        NamedNodeMap attributes = element.getAttributes();
        int len = attributes.getLength();
        for (int i = 0; i < len; i++) {
            Node node = attributes.item(i);
            String name = node.getLocalName();
            if (!props.contains(name)) {
                if (parameters == null) {
                    parameters = new ManagedMap();
                }
                String value = node.getNodeValue();
                parameters.put(name, new TypedStringValue(value, String.class));
            }
        }
        if (parameters != null) {
            beanDefinition.getPropertyValues().addPropertyValue("parameters", parameters);
        }
        return beanDefinition;
    }
```

## 服务注册

> 服务注册之前需要先看Dubbo SPI和自适应扩展

## 服务发现

> 服务发现之前需要先看Dubbo SPI和自适应扩展

## Dubbo SPI

> dubbo官网SPI源码导读：http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html
>
> 官网提供了demo和源码解析。

> SPI跟API的区别在于：
>
> api是客户端对服务端的消费。服务端提供接口的实现。
>
> spi是客户端对服务端的扩展。客户端提供接口的实现。

> java spi解决了传统双亲委派机制的扩展性。如果服务端的ClassLoader比扩展类的ClassLoader级别高，调用方加载不到扩展类。需要使用TCCL线程上下文类加载器，由applicationClassLoader开始加载。

 ### java SPI 示例

> 包路径都是com.test.SPI。

定义一个interface接口 

```java
public interface ITest {
    String execute();
}
```

定义两个实现类

```java
public class TestImpl1 implements ITest {
    @Override
    public String execute() {
        System.out.println("业务处理逻辑1111111");
        return "test11111111业务处理";
    }
}

public class TestImpl2 implements ITest {
    @Override
    public String execute() {
        System.out.println("业务处理逻辑22222");
        return "test22222业务处理";
    }
}
```

在META-INF/services文件加下，创建文件，文件名是ITest接口的全限定名com.test.SPI.ITest。

文件内容是

```properties
com.test.SPI.TestImpl1
com.test.SPI.TestImpl2
```

测试代码

```java
public class SPITest {
    @Test
    public void test(){
        ServiceLoader<ITest> iTests = ServiceLoader.load(ITest.class);

        System.out.println(iTests);
        for (ITest iTest : iTests) {
            System.out.println(iTest.execute());
        }
    }
}
```

执行结果

```tex
/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/bin/java -ea -Didea.test.cyclic.buffer.size=1048576 "-javaagent:/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=58554:/Applications/IntelliJ IDEA.app/Contents/bin" -Dfile.encoding=UTF-8 -classpath "/Applications/IntelliJ IDEA.app/Contents/lib/idea_rt.jar:/Applications/IntelliJ IDEA.app/Contents/plugins/junit/lib/junit5-rt.jar:/Applications/IntelliJ IDEA.app/Contents/plugins/junit/lib/junit-rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/charsets.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/deploy.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/dnsns.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/jaccess.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/nashorn.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/sunec.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/ext/zipfs.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/javaws.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/jce.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/jfxswt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/management-agent.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/plugin.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/lib/dt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/lib/javafx-mx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/lib/jconsole.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/lib/packager.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/lib/sa-jdi.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_231.jdk/Contents/Home/lib/tools.jar:/Users/wangke/IdeaProjects/test-projects/LeetCodeTest/target/classes:/Users/wangke/.m2/repository/junit/junit/4.12/junit-4.12.jar:/Users/wangke/.m2/repository/org/hamcrest/hamcrest-core/1.3/hamcrest-core-1.3.jar:/Users/wangke/.m2/repository/org/projectlombok/lombok/1.18.10/lombok-1.18.10.jar:/Users/wangke/.m2/repository/org/slf4j/slf4j-api/1.7.25/slf4j-api-1.7.25.jar:/Users/wangke/.m2/repository/org/slf4j/slf4j-simple/1.7.29/slf4j-simple-1.7.29.jar" com.intellij.rt.junit.JUnitStarter -ideVersion5 -junit4 com.test.SPI.SPITest,test
java.util.ServiceLoader[com.test.SPI.ITest]
业务处理逻辑1111111
test11111111业务处理
业务处理逻辑22222
test22222业务处理

Process finished with exit code 0
```

### Dubbo SPI

#### 示例

定义一个接口。需要标记@SPI注解

```java
@SPI
public interface ITest {
    String hello();
}
```

两个实现类

```java
package com.test.dubboSpi;

public class JackHelloTest implements ITest {
    @Override
    public String hello() {
        return "hello jack.";
    }
}
public class TomHelloTest implements ITest {
    @Override
    public String hello() {
        return "hello Tom.";
    }
}
```

在META-INF/dubbo下创建文件。

文件名是接口类的全限定名com.test.dubboSpi.ITest。

文件内容为

```properties
jack=com.test.dubboSpi.JackHelloTest
tom=com.test.dubboSpi.TomHelloTest
```

测试类

```java
public class SpiTest {

    @Test
    public void test(){
        ExtensionLoader<ITest> extensionLoader = ExtensionLoader.getExtensionLoader(ITest.class);
        ITest tom = extensionLoader.getExtension("tom");
        System.out.println(tom.hello());
        ITest jack = extensionLoader.getExtension("jack");
        System.out.println(jack.hello());
    }
}
```

执行结果

```java
Connected to the target VM, address: '127.0.0.1:51091', transport: 'socket'
17:41:29.997 [main] INFO  org.apache.dubbo.common.logger.LoggerFactory - using logger: org.apache.dubbo.common.logger.slf4j.Slf4jLoggerAdapter
hello Tom.
hello jack.
Disconnected from the target VM, address: '127.0.0.1:51091', transport: 'socket'

Process finished with exit code 0
```

#### 源码

入口：extensionLoader.getExtension()

```java
    public T getExtension(String name) {
      	//非空判断
        if (StringUtils.isEmpty(name)) {
            throw new IllegalArgumentException("Extension name == null");
        }
      	//获取默认的拓展实现类
        if ("true".equals(name)) {
            return getDefaultExtension();
        }
      	//获取目标对象持有者Holder。
      	//这里Holder是缓存对象value
    		//**cachedInstances.putIfAbsent(name, new Holder<>());
        final Holder<Object> holder = getOrCreateHolder(name);
        Object instance = holder.get();
      	//双端检测DCL
        if (instance == null) {
            synchronized (holder) {
                instance = holder.get();
                if (instance == null) {
                  	//创建拓展实例
                    instance = createExtension(name);
                  	//交给holder。
                    holder.set(instance);
                }
            }
        }
        return (T) instance;
    }
```

创建扩展实例createExtension

```java
    private T createExtension(String name) {
      	//从所有扩展类的集合中。获取name对应的扩展类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
          	//优先从缓存中获取
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
              	//没有就通过反射newInstance创建实例
                EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);//放入缓存
            }
          	//向扩展类实例中注入依赖
            injectExtension(instance);
          	//如果type接口有包装类。使用包装类构造函数生成实例
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (CollectionUtils.isNotEmpty(wrapperClasses)) {
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance (name: " + name + ", class: " +
                    type + ") couldn't be instantiated: " + t.getMessage(), t);
        }
    }
```

##### 加载扩展类s

获取所有的扩展类getExtensionClasses()，入缓存。

```java
    private Map<String, Class<?>> getExtensionClasses() {
      	//cachedClassed是缓存的持有者
      	// Holder<Map<String, Class<?>>> cachedClasses = new Holder<>()
        Map<String, Class<?>> classes = cachedClasses.get();
      	//装载缓存
      	//也是双端检测
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                  	//加载缓存-获取所有type的扩展类
                    classes = loadExtensionClasses();
                  	//设置缓存
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
    }
```

获取所有type的扩展类loadExtensionClasses

```java
    /**
     * synchronized in getExtensionClasses
     * */
    private Map<String, Class<?>> loadExtensionClasses() {
      	//缓存默认扩展类名称,去@SPI注解的value，逗号分隔。只能有一个
        cacheDefaultExtensionName();
				
      	//返回map
        Map<String, Class<?>> extensionClasses = new HashMap<>();
      
      	//每个文件夹都会执行typeName由org.apache替换为com.alibaba的二次查找。
      	//可能是为了开源之后的向前兼容性
      	
      	//加载META-INF/dubbo/internal文件夹下
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        //加载META-INF/dubbo文件夹下
      	loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        //加载META-INF/services文件夹下【java spi目录】
      	loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
        loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
        return extensionClasses;
    }
```

加载文件夹loadDirectory

```java
    private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir, String type) {
        //文件名 文件夹+接口类全限定名
      	String fileName = dir + type;
        try {
            Enumeration<java.net.URL> urls;
            ClassLoader classLoader = findClassLoader();
          	//加载所有的同名文件
            if (classLoader != null) {
                urls = classLoader.getResources(fileName);
            } else {
                urls = ClassLoader.getSystemResources(fileName);
            }
            if (urls != null) {
                while (urls.hasMoreElements()) {
                  	//jar:file:/Users/wangke/.m2/repository/org/apache/dubbo/dubbo/2.7.4.1/dubbo-2.7.4.1.jar!/META-INF/dubbo/internal/org.apache.dubbo.common.extension.ExtensionFactory
                    java.net.URL resourceURL = urls.nextElement();
                  	//加载资源
                    loadResource(extensionClasses, classLoader, resourceURL);
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", description file: " + fileName + ").", t);
        }
    }
```

加载资源loadResource

```java
    private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
        try {
          	//文件流
            try (BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), StandardCharsets.UTF_8))) {
                String line;
              	//按行读取
                while ((line = reader.readLine()) != null) {
                  	//过滤 #后面的注释
                    final int ci = line.indexOf('#');
                    if (ci >= 0) {
                        line = line.substring(0, ci);
                    }
                    line = line.trim();
                    if (line.length() > 0) {
                        try {
                            String name = null;
                          	//按=号截取。name=IMPL.class
                            int i = line.indexOf('=');
                            if (i > 0) {
                                name = line.substring(0, i).trim();
                                line = line.substring(i + 1).trim();
                            }
                            if (line.length() > 0) {
                              	//加载类 并缓存
                                loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                            }
                        } catch (Throwable t) {
                            IllegalStateException e = new IllegalStateException("Failed to load extension class (interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                            exceptions.put(line, e);
                        }
                    }
                }
            }
        } catch (Throwable t) {
            logger.error("Exception occurred when loading extension class (interface: " +
                    type + ", class file: " + resourceURL + ") in " + resourceURL, t);
        }
    }
```

加载类loadClass

> Class.forName(line, true, classLoader)加载类。initialize为true，执行静态代码块和静态变量初始化。

```java
    private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        //扩展实现类不是type接口的子类。报错。
      	if (!type.isAssignableFrom(clazz)) {
            throw new IllegalStateException("Error occurred when loading extension class (interface: " +
                    type + ", class line: " + clazz.getName() + "), class "
                    + clazz.getName() + " is not subtype of interface.");
        }
      	//扩展类上是否有@Adaptive注解 e,g. AdaptiveExtensionFactory
      	//注意。@Adaptive是说当前类是type的自适应类。他只会设置为cacheAdaptiveClass。不会作为type的扩展类生成实例。
        if (clazz.isAnnotationPresent(Adaptive.class)) {
          	//cachedAdaptiveClass赋值
            cacheAdaptiveClass(clazz);
        //扩展类是否包装类 Wrapper 具有type参数的构造器 e,g. ProtocolFilterWrapper
        } else if (isWrapperClass(clazz)) {
          	//添加进cachedWrapperClasses缓存
            cacheWrapperClass(clazz);
       	//普通扩展类
        } else {
          	//检查无参构造器。没有就抛异常
            clazz.getConstructor();
          	//获取name，没有就从扩展类@Extension注解获取，还没有就使用clazz的getSimpleName()【小写类名】
            if (StringUtils.isEmpty(name)) {
                name = findAnnotationName(clazz);
                if (name.length() == 0) {
                    throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
                }
            }
						//按逗号切分。
            String[] names = NAME_SEPARATOR.split(name);
            if (ArrayUtils.isNotEmpty(names)) {
              	//默认使用第一个作为激活扩展类
                cacheActivateClass(clazz, names[0]);
                for (String n : names) {
                  	//缓存类和名称的关系。cachedNames
                    cacheName(clazz, n);
                  	//缓存进extensionClasses 名称和实现类class
                    saveInExtensionClass(extensionClasses, clazz, n);
                }
            }
        }
    }
```

到这里。所有扩展类都已经入jvm缓存中。

##### 注入扩展类

> 注入前，反射newInstance生成实例。

```java
    private T injectExtension(T instance) {
				
      	//初始化objectFactory【ExtensionFactory类】的时候。这里是null
      	//ExtensionFactory类没有属性需要初始化。
      
      	//adaptive=org.apache.dubbo.common.extension.factory.AdaptiveExtensionFactory
				//spi=org.apache.dubbo.common.extension.factory.SpiExtensionFactory
				//spring=org.apache.dubbo.config.spring.extension.SpringExtensionFactory
        if (objectFactory == null) {
            return instance;
        }

        try {
          	//反射，遍历所有方法
            for (Method method : instance.getClass().getMethods()) {
              	//寻找setter方法。
              	//以set开头。且单个入参。且public
                if (!isSetter(method)) {
                    continue;
                }
                /**
                 * Check {@link DisableInject} to see if we need auto injection for this property
                 */
              	//不需要注入的，使用@DisableInject过滤
                if (method.getAnnotation(DisableInject.class) != null) {
                    continue;
                }
              	//pt：入参类型class
                Class<?> pt = method.getParameterTypes()[0];
                if (ReflectUtils.isPrimitives(pt)) {
                    continue;
                }

                try {
                  	//首字母小写的Field名称
                  	//e,g. setName -> name
                    String property = getSetterProperty(method);
                  	//获得class为pt，name为property的扩展类实例
                  	//使用dubbo的spi和spring的分别获取。
                    Object object = objectFactory.getExtension(pt, property);
                    if (object != null) {
                      	//执行set方法。为属性赋值。
                        method.invoke(instance, object);
                    }
                } catch (Exception e) {
                    logger.error("Failed to inject via method " + method.getName()
                            + " of interface " + type.getName() + ": " + e.getMessage(), e);
                }

            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```





## 自适应扩展机制

### 方法级别自适应扩展

入口：ExtensionLoader.getExtensionLoader(T.class).getAdaptiveExtension()

```java
    public T getAdaptiveExtension() {
      	//从缓存中获取自适应扩展。
        Object instance = cachedAdaptiveInstance.get();
        if (instance == null) {
          	//缓存未命中。
          	//有异常抛异常
          	//TODO 为啥要在这里抛
            if (createAdaptiveInstanceError != null) {
                throw new IllegalStateException("Failed to create adaptive instance: " +
                        createAdaptiveInstanceError.toString(),
                        createAdaptiveInstanceError);
            }
						//双端检索DCL
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                      	//创建自适应扩展
                        instance = createAdaptiveExtension();
                      	//放入缓存
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                      	//TODO 就是这个异常
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("Failed to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        }

        return (T) instance;
    }
```

创建自适应扩展createAdaptiveExtension

```java
  //在dubbo SPI源码有讲  
	//不同的是这里是获取自适应扩展类getAdaptiveExtensionClass
	private T createAdaptiveExtension() {
        try {
          	//(T) getAdaptiveExtensionClass()获取自适应对象class
          	//newInstance()实例化
          	//实例注入（dubbo IOC）
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can't create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

一个一个方法看吧。

getAdaptiveExtensionClass获取自适应对象class

> 细节提示。getExtension()获取所有如果当前接口的子类实现里有@Adaptive注解的扩展类。
>
> 这个@Adaptive扩展类就是自适应扩展类class。不需要创建。直接返回。

```java
    private Class<?> getAdaptiveExtensionClass() {
      	//通过dubbo spi 获得所有扩展类【cachedClasses缓存】
        getExtensionClasses();
      	//如果有类级别的扩展类【@Adaptive注解在类上 e,g. AdaptiveExtensionFactory】
      	//直接返回
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
      	//否则创建方法级别自适应扩展 TODO 先这么写着看看
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
    }
```

createAdaptiveExtensionClass创建自适应扩展类class

> 先生成自适应扩展类源码。
>
> 再通过Compiler 【dubbo默认的javassist编译器】编译，得到class实例。

```java
    private Class<?> createAdaptiveExtensionClass() {
      	//构建扩展类源码String
        String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
      	//获取ExtensionLoader的classLoader。
        ClassLoader classLoader = findClassLoader();
      	//获取编译器Compiler实现类
        org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        //编译生成class实例
      	return compiler.compile(code, classLoader);
    }
```

#### 自适应扩展类代码生成

```java
String code = new AdaptiveClassCodeGenerator(type, cachedDefaultName).generate();
```

使用AdaptiveClassCodeGenerator自适应扩展类源码生成器。

构造器传入扩展类类型和默认使用名称

```java
    public AdaptiveClassCodeGenerator(Class<?> type, String defaultExtName) {
        this.type = type;
        this.defaultExtName = defaultExtName;
    }
```

执行generate生成逻辑

```java
    public String generate() {
        //检查所有方法。如果没有@Adaptive注解的，就报错。
      	//return Arrays.stream(type.getMethods()).anyMatch(m -> m.isAnnotationPresent(Adaptive.class));
        if (!hasAdaptiveMethod()) {
            throw new IllegalStateException("No adaptive method exist on extension " + type.getName() + ", refuse to create the adaptive class!");
        }
				
      	//按照java代码顺序，生成源码
      	
        StringBuilder code = new StringBuilder();
      	//包信息
      	//type接口所在包
      	// e,g. package org.apache.dubbo.rpc;\n
        code.append(generatePackageInfo());
      
      	//imports信息
      	//引入ExtensionLoader的class
      	//import org.apache.dubbo.common.extension.ExtensionLoader;\n
        code.append(generateImports());
      	//类声明信息
      	//format: public class %s$Adaptive implements %s {\n
      	//使用type的simpleName和全限定名canonicalName
      	// e,g. public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
        code.append(generateClassDeclaration());
				
      	//循环生成方法
        Method[] methods = type.getMethods();
        for (Method method : methods) {
            code.append(generateMethod(method));
        }
      	//类结尾 }
        code.append("}");

        if (logger.isDebugEnabled()) {
            logger.debug(code.toString());
        }
        return code.toString();
    }
```

方法生成generateMethod

```java
    /**
     * generate method declaration
     */
    private String generateMethod(Method method) {
      	//返回类型的标准名称，也就是类全限定名
        String methodReturnType = method.getReturnType().getCanonicalName();
      	//方法名
        String methodName = method.getName();
      	//方法体
        String methodContent = generateMethodContent(method);
      	//方法参数 按照原方法参数顺序，转换成arg%d
      	//e,g. refer(Class<T> type, URL url)  ->  java.lang.Class arg0, org.apache.dubbo.common.URL arg1
        String methodArgs = generateMethodArguments(method);
      	//方法异常声明
      	//沿用之前的异常声明
      	// e,g. throws RpcException  ->  throws org.apache.dubbo.rpc.RpcException
        String methodThrows = generateMethodThrows(method);
        return String.format(CODE_METHOD_DECLARATION, methodReturnType, methodName, methodArgs, methodThrows, methodContent);
    }
```

##### 生成方法内容generateMethodContent

> 功能：通过传入的url，截取出对应的实现类。然后进行调用。
>
> 所以在适配方法里。需要获取url信息。调用具体对应扩展类。

```java
    private String generateMethodContent(Method method) {
      	//获取方法上的@Adaptive注解
        Adaptive adaptiveAnnotation = method.getAnnotation(Adaptive.class);
        StringBuilder code = new StringBuilder(512);
      	//没有被@Adaptive注解的方法体 直接抛出UnsupportedOperationException异常
      	//throw new UnsupportedOperationException("The method %s of interface %s is not adaptive method!");\n
        //e,g. 
        //public void destroy()  {
				//throw new UnsupportedOperationException("The method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
				//}
      	if (adaptiveAnnotation == null) {
            return generateUnsupported(method);
        } else {
          	//获取url参数
          	//通过遍历参数。获取org.apache.dubbo.common.URL参数的位置。没有返回-1
            int urlTypeIndex = getUrlTypeIndex(method);

            if (urlTypeIndex != -1) {
                //添加url参数value的非空校验。
              	//if (arg%d == null) 
              	//		throw new IllegalArgumentException(\"url == null\");\n
              	// %s url = arg%d;\n
                code.append(generateUrlNullCheck(urlTypeIndex));
            } else {
                //不能直接从参数列表中获取URL参数。
              	//遍历所有参数，获取参数中URL的getter方法。没有就报错。
              	// 1. 方法名以 get 开头，或方法名大于3个字符
                // 2. 方法的访问权限为 public
                // 3. 非静态方法
                // 4. 方法参数数量为0
               	// 5. 方法返回值类型为 URL
              	//同样进行非空判断。
              	//if (arg0 == null) 
              	//		throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
								//if (arg0.getUrl() == null) 
              	//		throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");
								//org.apache.dubbo.common.URL url = arg0.getUrl();
                code.append(generateUrlAssignmentIndirectly(method));
            }
						//获取@Adaptive注解声明参数
          	//Protocol的就是'protocol'
            String[] value = getMethodAdaptiveValue(adaptiveAnnotation);
						//判断是否有org.apache.dubbo.rpc.Invocation类型的参数
            boolean hasInvocation = hasInvocationArgument(method);
						//对Invocation类型的参数进行非空判断
          	//if (arg%d == null) 
          	//		throw new IllegalArgumentException(\"invocation == null\"); 
          	//String methodName = arg%d.getMethodName();
            code.append(generateInvocationArgumentNullCheck(method));

          	//扩展名逻辑 通过url及参数获取扩展类名称，得到extName
            code.append(generateExtNameAssignment(value, hasInvocation));
            // extName非空判断
            code.append(generateExtNameNullCheck(value));
						//获取扩展类 
          	//%s extension = (%<s)%s.getExtensionLoader(%s.class).getExtension(extName);
            code.append(generateExtensionAssignment());

            // return statement
            code.append(generateReturnAndInvocation(method));
        }

        return code.toString();
    }
```

- 生成扩展名逻辑

  > 目的是为了从url中获取扩展名，生成的获取代码会交给getNameCode变量
  >
  > 整段代码，对下面几种情况进行区分：
  >
  > 1）protocol可以从url中，通过getProtocol方式获取；
  >
  > 其他参数则是从请求参数中获取。获取方式不同，需要进行判断。
  >
  > 2）参数中有Invocation参数的，需要先调用invocation的getMethodName方法。
  >
  > 3）有没有默认扩展类名称。
  >
  > 4）最后一个元素。 ==TODO== 不明白。这里要写几个测试用例。
  >
  > 

  

  ```java
      private String generateExtNameAssignment(String[] value, boolean hasInvocation) {
         	
          String getNameCode = null;
        	//@Adaptive注解的values,由后向前遍历
          for (int i = value.length - 1; i >= 0; --i) {
            	//最后一个元素的处理逻辑 【也就是第一个节点】
              if (i == value.length - 1) {
                	// 默认扩展名非空
                  if (null != defaultExtName) {
                    	//非protocol节点
                      if (!"protocol".equals(value[i])) {
                          if (hasInvocation) {
                            	//参数列表中有Invocation参数
                            	//url.getMethodParameter(methodName, value[i], defaultExtName)
                            	//这里会先执行上面获取methodName的方法。对methodName和value[i]进行拼接
                            	//TODO 验证
                            	// 以 LoadBalance 接口的 select 方法为例，最终生成的代码如下：
                              // url.getMethodParameter(methodName, "loadbalance", "random")
                              getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                          } else {
                            	//url.getParameter(value[i], defaultExtName)
                              getNameCode = String.format("url.getParameter(\"%s\", \"%s\")", value[i], defaultExtName);
                          }
                      } else {
                      		//protocol节点
                        	//( url.getProtocol() == null ? defaultExtName : url.getProtocol() )
                          getNameCode = String.format("( url.getProtocol() == null ? \"%s\" : url.getProtocol() )", defaultExtName);
                      }
                  // 默认扩展名为空
                  } else {
                    	//非protocol节点
                      if (!"protocol".equals(value[i])) {
                          if (hasInvocation) {
                              getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                          } else {
                              getNameCode = String.format("url.getParameter(\"%s\")", value[i]);
                          }
                      } else {
                        	//protocol节点
                          getNameCode = "url.getProtocol()";
                      }
                  }
              //不是第一个节点
              } else {
                  if (!"protocol".equals(value[i])) {
                      if (hasInvocation) {
                          getNameCode = String.format("url.getMethodParameter(methodName, \"%s\", \"%s\")", value[i], defaultExtName);
                      } else {
                        	// 以 Transporter 接口的 connect 方法为例，最终生成的代码如下：
  	                    	// url.getParameter("client", url.getParameter("transporter", "netty"))
                          //TODO 验证
                        	getNameCode = String.format("url.getParameter(\"%s\", %s)", value[i], getNameCode);
                      }
                  } else {
                    	// 以 Protocol 接口的 connect 方法为例，最终生成的代码如下：
                      //  url.getProtocol() == null ? "dubbo" : url.getProtocol()
                    	//TODO 验证
                      getNameCode = String.format("url.getProtocol() == null ? (%s) : url.getProtocol()", getNameCode);
                  }
              }
          }
  				
        	//String extName = (getNameCode);
          return String.format(CODE_EXT_NAME_ASSIGNMENT, getNameCode);
      }
  ```

- 扩展类调用和返回

  ```java
      /**
       * generate method invocation statement and return it if necessary
       */
      private String generateReturnAndInvocation(Method method) {
  
        	//返回类型是void就不return
          String returnStatement = method.getReturnType().equals(void.class) ? "" : "return ";
  	
        	//拼装参数
        	//转换成arg0,arg1,arg2的形式
          String args = IntStream.range(0, method.getParameters().length)
                  .mapToObj(i -> String.format(CODE_EXTENSION_METHOD_INVOKE_ARGUMENT, i))
                  .collect(Collectors.joining(", "));
  				
       		//生成调用和返回
        	//extension.function(arg0,arg1,arg2)
        	//return extension.function(arg0,arg1,arg2)
          return returnStatement + String.format("extension.%s(%s);\n", method.getName(), args);
      }
  ```

  

到此。扩展类源码生成完成。

#### 编译

> ```java
> //获取dubbo编译器
> org.apache.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
> //执行编译。返回class
> return compiler.compile(code, classLoader);
> ```
> 获取编译器也是使用自适应扩展机制。
>
> ```properties
> #自适应类
> adaptive=org.apache.dubbo.common.compiler.support.AdaptiveCompiler
> #两个扩展类
> jdk=org.apache.dubbo.common.compiler.support.JdkCompiler
> javassist=org.apache.dubbo.common.compiler.support.JavassistCompiler
> ```
> 默认使用javassist
>
> ```java
> @SPI("javassist")
> public interface Compiler {
>     Class<?> compile(String code, ClassLoader classLoader);
> }
> ```

##### 自适应编译器AdaptiveCompiler

```java
@Adaptive
public class AdaptiveCompiler implements Compiler {

  	//编译器扩展名 默认为空 
  	//是类上的静态volatile属性
    private static volatile String DEFAULT_COMPILER;
		//在dubbo的applicationConfig节点可以设置编译器
  	//    public void setCompiler(String compiler) {
    //    		this.compiler = compiler;
    //    		AdaptiveCompiler.setDefaultCompiler(compiler);
    //		}
    public static void setDefaultCompiler(String compiler) {
        DEFAULT_COMPILER = compiler;
    }

  	//根据设置的编译器扩展名选择编译器扩展类。完成编译
    @Override
    public Class<?> compile(String code, ClassLoader classLoader) {
        Compiler compiler;
        ExtensionLoader<Compiler> loader = ExtensionLoader.getExtensionLoader(Compiler.class);
        String name = DEFAULT_COMPILER; // copy reference
        if (name != null && name.length() > 0) {
            compiler = loader.getExtension(name);
        } else {
            compiler = loader.getDefaultExtension();
        }
        return compiler.compile(code, classLoader);
    }

}
```

父类抽象实现AbstractCompiler

```java
public abstract class AbstractCompiler implements Compiler {
		
  	//包名 正则匹配
    private static final Pattern PACKAGE_PATTERN = Pattern.compile("package\\s+([$_a-zA-Z][$_a-zA-Z0-9\\.]*);");
		//类名 正则匹配
    private static final Pattern CLASS_PATTERN = Pattern.compile("class\\s+([$_a-zA-Z][$_a-zA-Z0-9]*)\\s+");

    @Override
    public Class<?> compile(String code, ClassLoader classLoader) {
        code = code.trim();
        Matcher matcher = PACKAGE_PATTERN.matcher(code);
      	//包名
        String pkg;
        if (matcher.find()) {
            pkg = matcher.group(1);
        } else {
            pkg = "";
        }
        matcher = CLASS_PATTERN.matcher(code);
      	//类名
        String cls;
        if (matcher.find()) {
            cls = matcher.group(1);
        } else {
            throw new IllegalArgumentException("No such class name in " + code);
        }
      	//组装className
        String className = pkg != null && pkg.length() > 0 ? pkg + "." + cls : cls;
        try {
          	//如果类已经加载了直接返回。
            return Class.forName(className, true, org.apache.dubbo.common.utils.ClassUtils.getCallerClassLoader(getClass()));
        } catch (ClassNotFoundException e) {
            if (!code.endsWith("}")) {
                throw new IllegalStateException("The java code not endsWith \"}\", code: \n" + code + "\n");
            }
            try {
              	//构造器找不到类。通过子类进行加载。
                return doCompile(className, code);
            } catch (RuntimeException t) {
                throw t;
            } catch (Throwable t) {
                throw new IllegalStateException("Failed to compile class, cause: " + t.getMessage() + ", class: " + className + ", code: \n" + code + "\n, stack: " + ClassUtils.toString(t));
            }
        }
    }

    protected abstract Class<?> doCompile(String name, String source) throws Throwable;

}
```

##### JavassistCompiler编译器

```java
public class JavassistCompiler extends AbstractCompiler {

  	//解析各个节点的正则表达式
    private static final Pattern IMPORT_PATTERN = Pattern.compile("import\\s+([\\w\\.\\*]+);\n");
    private static final Pattern EXTENDS_PATTERN = Pattern.compile("\\s+extends\\s+([\\w\\.]+)[^\\{]*\\{\n");
    private static final Pattern IMPLEMENTS_PATTERN = Pattern.compile("\\s+implements\\s+([\\w\\.]+)\\s*\\{\n");
    private static final Pattern METHODS_PATTERN = Pattern.compile("\n(private|public|protected)\\s+");
    private static final Pattern FIELD_PATTERN = Pattern.compile("[^\n]+=[^\n]+;");

    @Override
    public Class<?> doCompile(String name, String source) throws Throwable {
      	//使用建造者模式
      	//创建一个CtClassBuilder
        CtClassBuilder builder = new CtClassBuilder();
      	//类全限定名
        builder.setClassName(name);

        //获取所有import的类全限定名
        Matcher matcher = IMPORT_PATTERN.matcher(source);
        while (matcher.find()) {
            builder.addImports(matcher.group(1).trim());
        }

        //获取父类className
        matcher = EXTENDS_PATTERN.matcher(source);
        if (matcher.find()) {
            builder.setSuperClassName(matcher.group(1).trim());
        }

        //获取实现接口
        matcher = IMPLEMENTS_PATTERN.matcher(source);
        if (matcher.find()) {
            String[] ifaces = matcher.group(1).trim().split("\\,");
            Arrays.stream(ifaces).forEach(i -> builder.addInterface(i.trim()));
        }

        //body是类{中的内容}
        String body = source.substring(source.indexOf('{') + 1, source.length() - 1);
      	//获取所有的方法和属性
        String[] methods = METHODS_PATTERN.split(body);
        String className = ClassUtils.getSimpleClassName(name);
        Arrays.stream(methods).map(String::trim).filter(m -> !m.isEmpty()).forEach(method -> {
            //构造器
          	if (method.startsWith(className)) {
                builder.addConstructor("public " + method);
            } else if (FIELD_PATTERN.matcher(method).matches()) {
              	//属性
                builder.addField("private " + method);
            } else {
              	//方法
                builder.addMethod("public " + method);
            }
        });

        //当前类的类加载器
        ClassLoader classLoader = org.apache.dubbo.common.utils.ClassUtils.getCallerClassLoader(getClass());
        //执行创建
      	CtClass cls = builder.build(classLoader);
      	//通过javassist.CtClass完成类加载。生成class对象。
      	//这里还有个Class类的权限检测。TODO
        return cls.toClass(classLoader, JavassistCompiler.class.getProtectionDomain());
    }

}
```

TODO 后面的javassist编译过程和JDKCompiler对比暂且不看。

### 手写自适应扩展

TODO



