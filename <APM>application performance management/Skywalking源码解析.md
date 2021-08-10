# Skywalking源码解析

## javaagent初始化

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

