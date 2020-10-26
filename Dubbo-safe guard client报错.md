# Dubbo-safe guard client报错

> 最近检测发现mq的消费端「dubbo-consumer」遇到一个周期性的异常「safe guard client , should not be called ,must have a bug.」导致周期性的消息出现重发。

出现问题的原因是在LazyConnectExchangeClient「延迟连接的交换机客户端」。

在发送方法

```java
    @Override
    public CompletableFuture<Object> request(Object request) throws RemotingException {
      	//预警告
        warning();
      	//初始化客户端，建立channel连接
        initClient();
      	//发送请求
        return client.request(request);
    }
```

```java
/**
 * If {@link #REQUEST_WITH_WARNING_KEY} is configured, then warn once every 5000 invocations.
 */
//如果配置了REQUEST_WITH_WARNING_KEY为true
//每5000次请求，会发送一次safe guard client , should not be called ,must have a bug.
//而且是通过logger.warn的方式。
private void warning() {
    if (requestWithWarning) {
        if (warningcount.get() % warning_period == 0) {
            logger.warn(new IllegalStateException("safe guard client , should not be called ,must have a bug."));
        }
        warningcount.incrementAndGet();
    }
}
```

那么问题就来到了「REQUEST_WITH_WARNING_KEY的设置」 或者 「生产日志输出级别」。

## REQUEST_WITH_WARNING_KEY的设置

> LazyConnectExchangeClient的创建有两种情况：
>
> - dubbo-consumer设置了LAZY_CONNECT_KEY「延迟连接参数」
>
> 但是这时初始化的参数里
>
> ```java
> this.requestWithWarning = url.getParameter(REQUEST_WITH_WARNING_KEY, false);
> ```
>
> 不会出现问题。
>
> - provider断开channel连接
>
> 心跳检测发现通道关闭，会调用exchangeClient的close方法。在引用计数这一层ReferenceCountExchangeClient进行LazyConnectExchangeClient的替换。
>
> ```java
> public void close(int timeout) {
>   	//引用计数递减
>     if (referenceCount.decrementAndGet() <= 0) {
>       	//关闭客户端
>         if (timeout == 0) {
>             client.close();
> 
>         } else {
>             client.close(timeout);
>         }
> 				//替换lazyClient
>         replaceWithLazyClient();
>     }
> }
> ```
>
> **这里设置的REQUEST_WITH_WARNING_KEY是true**
>
> ```java
> .addParameter(LazyConnectExchangeClient.REQUEST_WITH_WARNING_KEY, true)
> ```

那么问题就找到了。

> 服务端主动断开连接，导致的exchangeClient替换。
>
> 且
>
> dubbo的日志输出级别是warn以上。

因为考虑到**微服务启动顺序不需要「有序」**，我们选择修改日志输出级别。

```xml
<!--log4j2 设置dubbo输出级别为error-->
<logger name="org.apache.dubbo" level="error" additivity="false">
  <AppenderRef ref="Console"/>
</logger>
```

