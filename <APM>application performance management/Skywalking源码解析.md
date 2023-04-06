# Skywalking源码解析

> 源码编译参照[官方-How-to-build](https://github.com/apache/skywalking/blob/master/docs/en/guides/How-to-build.md)。

## 可能遇到的问题

### 本地编译apm-webapp报错

有可能是npm源的问题。如果是```https://registry.npmjs.org```连接报错，修改为淘宝的源。

```xml
                    <!--apm-webapp/pom.xml-->
				            <execution>
                        <id>npm install</id>
                        <goals>
                            <goal>npm</goal>
                        </goals>
                        <configuration>
                          	<!--改为淘宝源-->
                            <arguments>install --registry=https://registry.npm.taobao.org/ --sass_binary_site=https://npm.taobao.org/mirrors/node-sass/</arguments>
                        </configuration>
                    </execution>
```

### 本机编译apm-sniffer时插件缺失

如果在```skywalking```进行全部编译，```skywalking/skywalking-agent```目录结构如下

<img src="https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20210811170452367.png" alt="image-20210811170452367" style="zoom:50%;" />

如果只编译`skywalking/apm-sniffer/apm-agent`是不行的，会缺少`activations、bootstrap-plugins、optional-plugins、plugins`四个文件夹。这就会导致「如果本地改了源码需要debug，增加了启动前编译」的时候，可能重新启动项目插件缺失。就什么也统计不到了。

因为在`skywalking/apm-sniffer/apm-agent`编译，只生成了jar/config/logs。

需要在上级菜单`skywalking/apm-sniffer`进行编译。

## javaagent初始化

> 初始化使用到了javaagent机制和ByteBuddy字节码增强技术。

```premain```启动入口在```apm-sniffer/apm-agent```的pom文件中：```org.apache.skywalking.apm.agent.SkyWalkingAgent```

```xml
    <properties>
        <premain.class>org.apache.skywalking.apm.agent.SkyWalkingAgent</premain.class>
        <can.redefine.classes>true</can.redefine.classes>
        <can.retransform.classes>true</can.retransform.classes>
    </properties>
		<build>
        <finalName>skywalking-agent</finalName>
        <plugins>
            <plugin>
                <artifactId>maven-shade-plugin</artifactId>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <manifestEntries>
                                        <Premain-Class>${premain.class}</Premain-Class>
                                        <Can-Redefine-Classes>${can.redefine.classes}</Can-Redefine-Classes>
                                        <Can-Retransform-Classes>${can.retransform.classes}</Can-Retransform-Classes>
                                    </manifestEntries>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

```java
public class SkyWalkingAgent {
    private static final ILog logger = LogManager.getLogger(SkyWalkingAgent.class);

    //javaagent:premain入口
    public static void premain(String agentArgs, Instrumentation instrumentation) throws PluginException, IOException {
        final PluginFinder pluginFinder;
        try {
          	//加载配置文件
            SnifferConfigInitializer.initialize(agentArgs);
						//加载插件集合
            pluginFinder = new PluginFinder(new PluginBootstrap().loadPlugins());

        } catch{
          //忽略异常
        }

      	//下面是基于ByteBuddy的agent实现
        final ByteBuddy byteBuddy = new ByteBuddy()
            .with(TypeValidation.of(Config.Agent.IS_OPEN_DEBUGGING_CLASS));

        AgentBuilder agentBuilder = new AgentBuilder.Default(byteBuddy)
          	//配置忽略的类的包名前缀
            .ignore(
                nameStartsWith("net.bytebuddy.")
                    .or(nameStartsWith("org.slf4j."))
                    .or(nameStartsWith("org.groovy."))
                    .or(nameContains("javassist"))
                    .or(nameContains(".asm."))
                    .or(nameContains(".reflectasm."))
                    .or(nameStartsWith("sun.reflect"))
                    .or(allSkyWalkingAgentExcludeToolkit())
                    .or(ElementMatchers.<TypeDescription>isSynthetic()));

        JDK9ModuleExporter.EdgeClasses edgeClasses = new JDK9ModuleExporter.EdgeClasses();
        try {
            agentBuilder = BootstrapInstrumentBoost.inject(pluginFinder, instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            logger.error(e, "SkyWalking agent inject bootstrap instrumentation failure. Shutting down.");
            return;
        }

        try {
            agentBuilder = JDK9ModuleExporter.openReadEdge(instrumentation, agentBuilder, edgeClasses);
        } catch (Exception e) {
            logger.error(e, "SkyWalking agent open read edge in JDK 9+ failure. Shutting down.");
            return;
        }

      	//agent加载进instrumentation的transformer中
        agentBuilder
            .type(pluginFinder.buildMatch())
          	//创建一个自定义的Transformer
            .transform(new Transformer(pluginFinder))
            .with(AgentBuilder.RedefinitionStrategy.RETRANSFORMATION)
          	//创建一个类的行为监听器
            .with(new Listener())
            .installOn(instrumentation);

      	//启动
        try {
            ServiceManager.INSTANCE.boot();
        } catch (Exception e) {
            logger.error(e, "Skywalking agent boot failure.");
        }
				
      	//添加jvm shutdownhook
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override public void run() {
                ServiceManager.INSTANCE.shutdown();
            }
        }, "skywalking service shutdown thread"));
    }
}
```

### 加载配置SnifferConfigInitializer
