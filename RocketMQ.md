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
- 消费者consumer，通过定期负载均衡器，获取当前应该执行的消息队列，锁定，消费。

***

**消费进度**

集群模式的消费进度保存在broker。

广播模式的消费进度保存在各个consumer。

***

## 使用案例

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

