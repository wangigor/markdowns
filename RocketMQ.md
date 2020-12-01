# RocketMQ

![RocketMQ0](/Users/wangke/Downloads/RocketMQ0.png)

**整体结构，RocketMQ分为namesrv、broker、client三个部分。**

***

**namesrv**

namesrv是一个非常简单的路由注册中心，纯内存应用。类似于zookeeper注册中心。支持broker的注册和发现。

- 接收broker的注册信息，并保存下来作为路由信息的基础数据。
- 通过心跳检测机制，检测broker是否存活。
- 每一个namesrv都完整保存整个broker集群的完整路由信息。
- 各个namesrv节点不进行互相通信，broker启动需要向所有namesrv节点依次注册。

***

**broker**

负责消息接收、存储、转发到消费队列、集群消息消费进度管理、消息查询、master和slave的数据同步、消息拉取、到达消息通知、事务消息回调、延迟消息管理等功能。

***

**client**

消息生产者「producer」和消息消费者「consumer」都是client。

- producer负责消息发送「同步发送、异步发送、单向发送「oneway」」，将业务系统中的消息，发送到broker。
- consumer负责消息消费「顺序消费、并发消费」，有两种消费模式「pull和push，没有真正意义的push，是通过通知consumer来pull实现的。」
- client与随机一个namesrv保持长连接。
- client与broker保持长连接。

***

**消息贯穿整个RocketMQ。**

***

**message**

mq传输信息的载体，生产和消费数据的最小单位。

- 每一个message都有一个全局唯一messageId。
- 只能属于一个topic「主题」
- 可以携带具有业务标识的key。
- 可以使用不同标签「TAG」进行过滤区分。
- 可以使用用户属性进行过滤「SQL92」

***

**topic**

一类消息的集合，是消息订阅的基本单位。

- 一个topic可根据配置创建多个消费队列「可以位于多个broker中」。

***

**TAG**

用于消息区分，过滤。

可以根据tag实现相同主题，不同「子主题」的不同消费逻辑，使消费具有扩展性。

***

**KEY**

key是具体业务的标记。比如「订单号」。

使得message同类型的消息进入同一消费队列，顺序消费。

***

**消息队列**

消费队列位于broker端，用于消息消费。

- 一个topic对应多个消费队列。
- 一个消费队列，同一时刻，只能被一个consumer的一个线程锁定「否则造成重复消费」。
- 消息发送时，按照topic和消息队列id，发送到指定broker。
- 消息存入commitlog之后，通过消息转发，进入一个消息队列。
- 消费者consumer，通过定期负载均衡，获取当前应该执行的消息队列，锁定，消费。

***

**消费进度**

集群模式的消费进度保存在broker。

广播模式的消费进度保存在各个consumer。

***

## 使用示例

> 我是直接从[github](https://github.com/apache/rocketmq)拉取源码，本地编译执行的。
>
> ```shell
>  mvn -Prelease-all -DskipTests -Dcheckstyle.skip clean install -U
> ```
>
> 需要设置的环境变量
>
> ```properties
> ROCKETMQ_HOME=/Users/wangke/IdeaProjects/git/rocketmq/home
> #工作空间，用于存储配置，日志，应用文件等。
> ```

### namesrv启动

- 使用application方式启动**NamesrvStartup**，也可以
- 使用sh方式运行distribution/target/rocketmq-4.7.1/rocketmq-4.7.1/bin/**mqnamesrv**，也可以。

vm启动参数**可选**「日志记录和配置文件默认读取位置」

```properties
user.home=/Users/wangke/IdeaProjects/git/rocketmq
```

也可以程序启动参数设置配置文件位置

```shell
-c /Users/wangke/IdeaProjects/git/rocketmq/home/namesrv/namesrv.conf &
```

增加logback.xml日志配置「默认读取位置是**${user.home}/conf/logback_namesrv.xml**」

这个是rocketmq默认提供的日志配置「位置在${源码目录}/rocketmq/distribution/target/rocketmq-4.7.1/rocketmq-4.7.1/conf/**logback_namesrv.xml**」

```xml
<configuration>

    <!--也可以手动指定user.home参数-->
    <property name="user.home" value="/Users/wangke/IdeaProjects/git/rocketmq/home"/>

    <appender name="DefaultAppender"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${user.home}/logs/rocketmqlogs/namesrv_default.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${user.home}/logs/rocketmqlogs/otherdays/namesrv_default.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>5</maxIndex>
        </rollingPolicy>
        <triggeringPolicy
                class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <appender name="RocketmqNamesrvAppender_inner"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${user.home}/logs/rocketmqlogs/namesrv.log</file>
        <append>true</append>
        <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
            <fileNamePattern>${user.home}/logs/rocketmqlogs/otherdays/namesrv.%i.log.gz</fileNamePattern>
            <minIndex>1</minIndex>
            <maxIndex>5</maxIndex>
        </rollingPolicy>
        <triggeringPolicy
                class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <maxFileSize>100MB</maxFileSize>
        </triggeringPolicy>
        <encoder>
            <pattern>%d{yyy-MM-dd HH:mm:ss,GMT+8} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>
    <appender name="RocketmqNamesrvAppender" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="RocketmqNamesrvAppender_inner"/>
        <discardingThreshold>0</discardingThreshold>
    </appender>

    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <append>true</append>
        <encoder>
            <pattern>%d{yyy-MM-dd HH\:mm\:ss,SSS} %p %t - %m%n</pattern>
            <charset class="java.nio.charset.Charset">UTF-8</charset>
        </encoder>
    </appender>

    <logger name="RocketmqNamesrv" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqNamesrvAppender"/>
    </logger>

    <logger name="RocketmqCommon" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqNamesrvAppender"/>
    </logger>

    <logger name="RocketmqRemoting" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="RocketmqNamesrvAppender"/>
    </logger>

    <logger name="RocketmqNamesrvConsole" additivity="false">
        <level value="INFO"/>
        <appender-ref ref="STDOUT"/>
    </logger>

    <root>
        <level value="INFO"/>
        <appender-ref ref="DefaultAppender"/>
    </root>
</configuration>
```

可以什么都不配置，使用默认配置启动。

### broker启动

- 使用application方式启动**BrokerStartup**，也可以
- 使用sh方式运行distribution/target/rocketmq-4.7.1/rocketmq-4.7.1/bin/**mqbroker**，也可以。

broker启动没有默认读取配置文件的位置

- 需要手动-c指定配置文件位置

- mqbroker命令行有几个可指定参数

  ```shell
  usage: mqbroker [-c <arg>] [-h] [-m] [-n <arg>] [-p]
   -c,--configFile <arg>       Broker config properties file
   -h,--help                   Print help
   -m,--printImportantConfig   Print important config item
   -n,--namesrvAddr <arg>      Name server address list, eg: 192.168.0.1:9876;192.168.0.2:9876
   -p,--printConfigItem        Print all config item
  #注意：-m 和 -p都是打印配置信息并推出。只能用于参数检查。
  ```

进行简单配置就可以启动broker。

```properties
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
autoCreateTopicEnable=true
enablePropertyFilter=true
namesrvAddr = localhost:9876
storePathRootDir = /Users/wangke/IdeaProjects/git/rocketmq/home/store
storePathCommitLog = /Users/wangke/IdeaProjects/git/rocketmq/home/store/commitlog
storePathConsumeQueue = /Users/wangke/IdeaProjects/git/rocketmq/home/store/consumequeue
storePathIndex = /Users/wangke/IdeaProjects/git/rocketmq/home/store/index
storeCheckPoint = /Users/wangke/IdeaProjects/git/rocketmq/home/store/checkpoint
abortFile = /Users/wangke/IdeaProjects/git/rocketmq/home/store/abort
```

日志输出默认读取位置在**${user.home}/conf/logback_broker.xml**「在distribution/target/rocketmq-4.7.1/rocketmq-4.7.1文件夹下有demo文件」

> 另外**distribution/target/rocketmq-4.7.1/rocketmq-4.7.1/conf文件夹**还提供集群的几种配置方式的简单demo
>
> - 2m-2s-async：两主两从异步同步异步落盘集群
> - 2m-2s-sync：两主两从同步同步异步落盘集群
> - 2m-noslave：两主无从集群
> - dledger：基于raft的自动切换集群

### 添加client依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.7.1</version>
</dependency>
```

如果是在源码project里创建module的方式，直接引用为parent作为module引用就可以。

```xml
<parent>
  <artifactId>rocketmq-all</artifactId>
  <groupId>org.apache.rocketmq</groupId>
  <version>4.7.1</version>
</parent>
```

### 简单并发消费者

```java
@Test
public void customer() throws MQClientException, InterruptedException {
  	//创建consumer
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("simple");
  	//设置namesrv地址
    consumer.setNamesrvAddr("localhost:9876;localhost:9877;localhost:9878");
  	//声明当前consumer订阅的topic和tag过滤
    consumer.subscribe("simple_test", "Tag-1 || Tag-2");
  	//设置ConsumeMessageThread线程数「默认20」
    consumer.setConsumeThreadMax(5);
    consumer.setConsumeThreadMin(5);
  	//设置listener
    consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {
        log.info("reveive new messages :{}|{}", context, msgs);
      	//虽然入参是消息集合，但是默认配置是1「逐条消费consumeMessageBatchMaxSize」
        msgs.forEach(msg -> {
            log.debug(new String(msg.getBody()));
        });
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    });
  	//启动
    consumer.start();
  
  	//阻塞当前线程退出
    new CountDownLatch(1).await();
}
```

### 生产者

```java
@SneakyThrows
private DefaultMQProducer getProducer() {
  //创建producer
  DefaultMQProducer producer = new DefaultMQProducer("simple", true);
  producer.setNamesrvAddr("localhost:9876;localhost:9877;localhost:9878");
  producer.start();
  return producer;
}
```

#### 同步发送

```java
@Test
public void syncProducer() throws InterruptedException {
		
  	//创建producer
    DefaultMQProducer producer = getProducer();

  	//简单消息
    Message message = new Message("simple_test", "Tag-1", ("hello rocketmq " + index).getBytes());
    try {
      //发送并接收发送结果
      SendResult result = producer.send(message);
      log.info(result.toString());
    } catch (MQClientException | RemotingException | MQBrokerException | InterruptedException e) {
      e.printStackTrace();
    }
}
```

#### 异步发送

```java
@Test
public void asyncProducer() throws InterruptedException {
  	//创建producer
    DefaultMQProducer producer = getProducer();

  	//创建一个CountDownLatch，用于100条发送成功后，退出测试。
    CountDownLatch latch = new CountDownLatch(100);

    IntStream.range(0, 100).forEach(index -> {
      	//创建消息
        Message message = new Message("simple_test", "Tag-1", ("hello rocketmq async " + index).getBytes());
        try {
          	//发送消息，并设置发送结果回调callback。
            producer.send(message, new SendCallback() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    log.info("发送成功：{}", sendResult.toString());
                    latch.countDown();
                }

                @Override
                public void onException(Throwable e) {
                    log.error("发送失败：", e);
                    latch.countDown();
                }
            });
        } catch (MQClientException | RemotingException | InterruptedException e) {
            e.printStackTrace();
        }
    });

  	//发送完成退出
    latch.await();
}
```

#### oneway发送

```java
//对可靠性要求不高，如日志记录等
//不等待应答
@Test
public void OnewayProducer() throws InterruptedException {
    DefaultMQProducer producer = getProducer();

    IntStream.range(0, 100).forEach(index -> {
        Message message = new Message("simple_test", "Tag-1", ("hello rocketmq one way " + index).getBytes());
        try {
          	//就是要发，管你成功不成功，我都不在乎。
            producer.sendOneway(message);
        } catch (MQClientException | RemotingException | InterruptedException e) {
            e.printStackTrace();
        }

    });

    new CountDownLatch(1).await();
}
```

### 广播模式

> 广播模式和集群模式，是对同一个消费者组而言的。
>
> 假设一个消费者组有三个消费者：
>
> - 广播模式下三个消费者都会消费同一条消息。
> - 集群模式下一条消息只被一个消费者消费。
>
> 还有一点要注意的是：广播模式的消费进度保存在consumer端，5秒持久化一次到文件，如果还没保存到文件，就重启consumer，会造成消息重复消费。「虽然设置了jvm hook进行持久化操作，但是仍然不能解决kill -9或宕机的重复消费。」

```java
@Test
public void consumer0() throws MQClientException, InterruptedException {
    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("broadcast_test");
    consumer.setNamesrvAddr("localhost:9876");
    consumer.subscribe("test_broadcast", "*");

    consumer.setMessageModel(MessageModel.BROADCASTING);//设置广播模式
    consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {

        for (MessageExt msg : msgs) {
            log.info("received message :{}", new String(msg.getBody()));
        }

        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    });

    consumer.start();
  	//阻塞退出
    new CountDownLatch(1).await();
}
```

### 批量消息

> - 相同的topic
> - 不能是显示消息
> - 总长度不能超过4M限制
> - 相同的waitStoreMsgOK「消息是否落盘后再响应应答。」

```java
@Test
public void producer() throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
    DefaultMQProducer producer = new DefaultMQProducer("barch_test");
    producer.setNamesrvAddr("localhost:9876");
    producer.start();

    //注意这里1w的时候还可以。
    //但是100w的时候，就会报「MQClientException: CODE: 13  DESC: the message body size over max value，MAX: 4194304」
    //最大4M
    int count = 1000000;
    List<Message> messages = new ArrayList<>(count);
    for (int i = 0; i < count; i++) {
        Message message = new Message("test_batch", "tag", ("dbody-" + i).getBytes());
      	//这里需要注意：消息的唯一id在批量消息中需要优先设置「唯一id设置时机不同」
      	//因为这里占用了8+56长度的属性，不会在下面ListSplitter时体现出来，而是在后续发送之前填进去的。
      	//或是需要预留出来
      	MessageClientIDSetter.setUniqID(message);
        messages.add(message);
    }

    //拆分发送
    ListSplitter splitter = new ListSplitter(messages);

    while (splitter.hasNext()) {
        SendResult sendResult = producer.send(splitter.next());
        log.info(sendResult.getSendStatus().toString());
    }
    log.info("发送完毕");

}

/**
 * 官网demo SplitBatchProducer
 */
class ListSplitter implements Iterator<List<Message>> {
  	//限定4M
    private int sizeLimit = 1024 * 1024 * 4;
    private final List<Message> messages;
    private int currIndex;

    public ListSplitter(List<Message> messages) {
        this.messages = messages;
    }

    @Override
    public boolean hasNext() {
        return currIndex < messages.size();
    }

    @Override
    public List<Message> next() {
        int nextIndex = currIndex;
        int totalSize = 0;
        for (; nextIndex < messages.size(); nextIndex++) {
            Message message = messages.get(nextIndex);
            int tmpSize = message.getTopic().length() + message.getBody().length;
            Map<String, String> properties = message.getProperties();
            for (Map.Entry<String, String> entry : properties.entrySet()) {
                tmpSize += entry.getKey().length() + entry.getValue().length();
            }
            tmpSize = tmpSize + 100; //for log overhead
          	//这里官网的demo写的是20不太严谨，因为commitlog也对长度进行了校验，但是字段却增加了不少
          	//参见CommitLog.calMsgLength方法
          	//否则即便通过了client的长度检测，还是会报错CODE: 1  DESC: java.lang.RuntimeException: message size exceeded, org.apache.rocketmq.store.CommitLog$MessageExtBatchEncoder.encode(CommitLog.java:1974)
            if (tmpSize > sizeLimit) {
                if (nextIndex - currIndex == 0) {
                    nextIndex++;
                }
                break;
            }
            if (tmpSize + totalSize > sizeLimit) {
                break;
            } else {
                totalSize += tmpSize;
            }

        }
        List<Message> subList = messages.subList(currIndex, nextIndex);
        currIndex = nextIndex;
        return subList;
    }

    @Override
    public void remove() {
        throw new UnsupportedOperationException("Not allowed to remove");
    }
}
```

### 顺序消费

> 比方说订单的状态修改事件，需要触发不同的后续业务操作，使用mq解耦：
>
> 两个订单，总共触发6个事件「订单1-下单完成」「订单1-付款成功」「订单1-出库完成」「订单2-下单完成」「订单2-付款成功」「订单2-出库完成」，需要保证各自订单的事件有序性，也就是局部有序性。
>
> 要保证局部有序性，需要进行两个地方的改造：
>
> - **同一个订单事件，只进一个队列。**
>
> 之前消息的send流程是，如果不指定消息队列，按照当前topic的所有队列进行轮询放置。
>
> 也提供了带有MessageQueueSelector队列选择器的send方法，只要保证订单id相同的进同一个队列即可。
>
> - **consumer拉取消息后，对消息顺序消费。**
>
> 之前消费默认使用的是**ConsumeMessageOrderlyService**，从一个队列拉取默认32条消息后，按照消费批数「默认1」进行拆分，然后每个拆分的最小单位放入线程池中执行。这不能保证有序性。
>
> 需要使用**ConsumeMessageOrderlyService**，拉取32条消息后，作为一批放入线程池中，单线程对每个消费批数进行处理。
>
> 下面是demo「使用了lambda，不太直观。哈哈哈哈哈哈哈哈哈哈哈哈哈哈哈哈哈」。

producer修改

```java
@Test
public void producer() throws MQClientException, InterruptedException {
    DefaultMQProducer producer = new DefaultMQProducer("order_test");
    producer.setNamesrvAddr("localhost:9876");
    producer.start();

  	//order-1的线程
    new Thread(() -> {
      	//顺序创建10个事件
        for (int i = 0; i < 10; i++) {
            Message message = new Message("test_order", "test", ("order-1-" + i).getBytes());
            try {
              	//传参是
              	//Message msg 消息
              	//MessageQueueSelector selector 队列选择器
              	//Object arg 队列选择器入参 「单号」
                SendResult result = producer.send(message, (mqs, msg, arg) -> {
                  	//最简单的就是按照单号对所有队列取模。
                    int index = (int) arg % mqs.size();
                    return mqs.get(index);
                }, 1/**单号**/);
                log.debug(result.getSendStatus().name());
            } catch (MQClientException | RemotingException | InterruptedException | MQBrokerException e) {
                e.printStackTrace();
            }
        }
    }, "order1").start();

  	//order-2的线程
    new Thread(() -> {
        for (int i = 0; i < 10; i++) {
            Message message = new Message("test_order", "test", ("order-2-" + i).getBytes());
            try {
                SendResult result = producer.send(message, (mqs, msg, arg) -> {
                    int index = (int) arg % mqs.size();
                    return mqs.get(index);
                }, 2/**单号**/);
                log.debug(result.getSendStatus().name());
            } catch (MQClientException | RemotingException | InterruptedException | MQBrokerException e) {
                e.printStackTrace();
            }
        }
    }, "order2").start();

    new CountDownLatch(1).await();

}
```

consumer修改

```java
@Test
public void consumer() throws MQClientException, InterruptedException {

    DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("order_test");
    consumer.setNamesrvAddr("localhost:9876");
    consumer.subscribe("test_order", "*");

    consumer.setConsumeThreadMin(3);//即使使用多线程消费端，也没啥用，

    consumer.registerMessageListener((MessageListenerOrderly) (msgs, context) -> {
        msgs.forEach(msg ->
                log.info("{} received message :{}", Thread.currentThread().getName(), new String(msg.getBody()))
        );
        return ConsumeOrderlyStatus.SUCCESS;
    });
    consumer.start();

    new CountDownLatch(1).await();

}
```

## 源码解析

### NameServer

### Borker

### 消息发送

### 消息存储

### 消息消费

### 消息过滤

### 事务消息/延迟消息

### 主从同步

## 使用问题

## 参数配置



