# Netty笔记

> Netty 是一个基于NIO的客户、服务器端的编程框架，使用Netty 可以确保你快速和简单的开发出一个网络应用，例如实现了某种协议的客户、服务端应用。Netty相当于简化和流线化了网络应用的编程开发过程，例如：基于TCP和UDP的socket服务开发。

## NIO

### 零拷贝

#### 传统IO

```java
		/**
     * IO复制文件 1.jpg复制到 2.jpg
     */
    @Test
    public void test00() {
        @Cleanup FileOutputStream fileOutputStream = null;
        @Cleanup FileInputStream fileInputStream = null;
        try {
            fileOutputStream = new FileOutputStream("D:/2.jpg");
            fileInputStream = new FileInputStream("D:/1.jpg");

            byte[] bytes = new byte[1024];

            int length;
            while ((length = fileInputStream.read(bytes)) != -1) {
                fileOutputStream.write(bytes, 0, length);
            }
            
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```




> ![img](https://gitee.com/wangigor/typora-images/raw/master/传统io.png)
>
> - JVM发起**read()** native调用，操作系统由**用户态空间切换到内核态空间**。「第一次上下文切换」
> - 通过DMA引擎将数据从**磁盘拷贝到内核空间缓冲区**内。「第一次拷贝」
> - 将**数据从内核缓冲区复制到用户缓冲区**「第二次拷贝」，**由内核态切换到用户态**「第二次上下文切换」。read函数返回。
> - JVM内业务逻辑代码执行。发起**write()** native调用。
> - **用户态切换到内核态**「第三次上下文切换」，**数据由用户缓冲区拷贝到内核缓冲区**「第三次拷贝」。
> - **write()调用返回**，**由内核态切换到用户态**「第四次上下文切换」。DMA将数据从内核缓冲区拷贝协议引擎。
>
> 用户空间仅仅起到一个数据中转媒介的作用，完全没有必要。

#### mmap

> ![img](https://gitee.com/wangigor/typora-images/raw/master/mmap.png)
>
> [认真分析mmap](https://www.cnblogs.com/huxiao-tee/p/4660352.html)
>
> mmap是一种内存映射文件的方法，即将**一个文件或者其它对象映射到进程的地址空间，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映关系。**实现这样的映射关系后，进程**就可以采用指针的方式读写操作这一段内存**，而系统会自动回写脏页面到对应的文件磁盘上，即完成了**对文件的操作而不必再调用read,write等系统调用函数**。相反，内核空间对这段区域的修改也直接反映用户空间，从而可以**实现不同进程间的文件共享**。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/文件映射.png)

```c
tmp_buf = mmap(file, len);
write(socket, tmp_buf, len);
```

> ![img](https://gitee.com/wangigor/typora-images/raw/master/5377351-d8e3c35347959291.jpg)
>
> - mmap系统调用导致**DMA引擎将文件内容复制到内核缓冲区中**。然后**与用户进程共享缓冲区**「减少一次复制」，而不在内核和用户存储器空间之间执行任何复制。
> - write系统调用导致**内核将数据从原始内核缓冲区复制到与套接字相关联的内核缓冲区**中。
> - DMA引擎将数据从内核套接字缓冲区传递到协议引擎。「第三次复制」

```java
    @Test
    public void test02() throws IOException {
        @Cleanup FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);
        @Cleanup FileChannel outChannel = FileChannel.open(Paths.get("2.jpg"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);

        MappedByteBuffer inByteBuffer = inChannel.map(FileChannel.MapMode.READ_ONLY, 0, inChannel.size());
        MappedByteBuffer outByteBuffer = outChannel.map(FileChannel.MapMode.READ_WRITE, 0, inChannel.size());

        byte[] bytes = new byte[inByteBuffer.limit()];

        inByteBuffer.get(bytes);
        outByteBuffer.put(bytes);

    }
```



#### sendfile

> ![img](https://gitee.com/wangigor/typora-images/raw/master/sendfile.png)
>
> 没有用户态和内核态的数据复制。减少上下文切换次数。
>
> - 发起sendfile()系统调用，操作系统由**用户态空间切换到内核态空间**「第一次上下文切换」
> - 通过DMA引擎将数据从磁盘拷贝到内核态空间缓冲区中「第一次拷贝」
> - 将数据从内核空间拷贝到与之关联的socket缓冲区「第二次拷贝」
> - 将socket缓冲区的数据拷贝到协议引擎中『第三次拷贝』
> - sendfile()系统调用结束，操作系统由用户态空间切换到内核态空间『第二次上下文切换』
>
> 根据以上过程，一共有2次的上下文切换，3次的I/O拷贝。我们看到从用户空间到内核空间并没有出现数据拷贝，从操作系统角度来看，这个就是**零拷贝**。内核空间出现了复制的原因: 通常的硬件在通过DMA访问时期望的是连续的内存空间。

![img](https://gitee.com/wangigor/typora-images/raw/master/scatter-gather-sendfile.png)

> 这次相比sendfile()数据零拷贝，减少了一次从内核空间到与之相关的socket缓冲区的数据拷贝。
>
> - 发起sendfile()系统调用，操作系统由用户态空间切换到内核态空间「第一次上下文切换」
> - 通过DMA引擎建数据从磁盘拷贝到内核态空间缓冲区中『第一次拷贝』
> - **将描述符信息会拷贝到相应的socket缓冲区当中**，该描述符包含了两方面的信息：kernel buffer的内存地址；kernel buffer的偏移量。
> - **DMA gather copy**根据socket缓冲区中描述符提供的位置和偏移量信息直接将内核空间缓冲区中的数据拷贝到协议引擎上『第二次拷贝』，这样就避免了最后一次I/O数据拷贝。
> - sendfile()系统调用结束，操作系统由用户态空间切换到内核态空间『第二次上下文切换』

```java
    @Test
    public void test03() throws IOException {
        @Cleanup FileChannel inChannel = FileChannel.open(Paths.get("D:/1.jpg"), StandardOpenOption.READ);
        @Cleanup FileChannel outChannel = FileChannel.open(Paths.get("D:/2.jpg"), StandardOpenOption.WRITE, StandardOpenOption.READ, StandardOpenOption.CREATE);

        inChannel.transferTo(0, inChannel.size(), outChannel);
//        outChannel.transferFrom(inChannel,0,inChannel.size());
    }
```



### 堆内/外空间

> 之前传统io的例子里，是采用内存数组的方式，进行「运输」
>
> ```java
> byte[] bytes = new byte[1024];
> ```
> 每次运输1k。
>
> 也可以采用java NIO 提供的ByteBuffer进行「传输空间」的开辟。

- **堆内空间**

> ```java
> ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
> ```
>
> 创建的是一个HeapByteBuffer的堆内空间，可由gc回收。
>
> ```java
> 		public static ByteBuffer allocate(int capacity) {
>         if (capacity < 0)
>             throw new IllegalArgumentException();
>         return new HeapByteBuffer(capacity, capacity);
>     }
> ```
>
> 本质跟new byte[1024]一样。HeapByteBuffer是前者的封装类。

- 堆外空间

> ```java
> ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024);
> ```
> 创建的是一个DirectByteBuffer的堆外空间，需要「手动」清理，「不会」被gc操作。
>
> ```java
>     public static ByteBuffer allocateDirect(int capacity) {
>         return new DirectByteBuffer(capacity);
>     }
> ```
>
> 本质是虚引用实现。
>
> 简单看一下代码
>
> ```java
> DirectByteBuffer(int cap) {                   // package-private
> 
>         super(-1, 0, cap, cap);
>         boolean pa = VM.isDirectMemoryPageAligned();
>         int ps = Bits.pageSize();
>         long size = Math.max(1L, (long)cap + (pa ? ps : 0));
>         Bits.reserveMemory(size, cap);
> 
>   			//使用unsafe开辟内存空间
>         long base = 0;
>         try {
>             base = unsafe.allocateMemory(size);
>         } catch (OutOfMemoryError x) {
>             Bits.unreserveMemory(size, cap);
>             throw x;
>         }
>         unsafe.setMemory(base, size, (byte) 0);
>         if (pa && (base % ps != 0)) {
>             // Round up to page boundary
>             address = base + ps - (base & (ps - 1));
>         } else {
>             address = base;
>         }
>   			//创建一个虚引用的子类实现Cleaner
>   			//并将内存地址，空间大小，传递进来。
>         cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
>         att = null;
> 
>     }
> ```
>
> ```java
> public class Cleaner extends PhantomReference<Object> {
>   	//放入ReferenceQueue引用队列
>     private static final ReferenceQueue<Object> dummyQueue = new ReferenceQueue();
> }
> ```
>
> 引用被gc时，进行堆外内存空间的释放清理。

### 缓冲区

> | capacity     | 缓冲区数组的总长度                                          |
> | ------------ | ----------------------------------------------------------- |
> | **position** | **下一个要操作的数据元素的位置**                            |
> | **limit**    | **缓冲区数组中不可操作的下一个元素的位置：limit<=capacity** |

```shell
# 开辟一个10的空间
+-------------------+    postion  0
| | | | | | | | | | |    limit    10
+-------------------+    capacity 10

# 写5个数据
+-------------------+    postion  5
|+|+|+|+|+| | | | | |    limit    10
+-------------------+    capacity 10

# 交给读取方之前要先进行flip
+-------------------+    postion  0
|+|+|+|+|+| | | | | |    limit    5
+-------------------+    capacity 10

# 读完清空 clear
+-------------------+    postion  0
| | | | | | | | | | |    limit    10
+-------------------+    capacity 10
```

```java
    @Test
    public void test01() {
        @Cleanup FileOutputStream outputStream = null;
        @Cleanup FileInputStream inputStream = null;
        @Cleanup FileChannel outChannel = null;
        @Cleanup FileChannel inChannel = null;
        try {
            outputStream = new FileOutputStream("2.jpg");
            inputStream = new FileInputStream("1.jpg");
            outChannel = outputStream.getChannel();
            inChannel = inputStream.getChannel();
          	//这里使用的堆外空间
            ByteBuffer byteBuffer = ByteBuffer.allocateDirect(1024);
          	//读
            while (inChannel.read(byteBuffer) != -1) {
              	//flip
                byteBuffer.flip();
              	//写
                outChannel.write(byteBuffer);
              	//clear
                byteBuffer.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } 
    }
```

### socket对比

#### 传统io socket

> 简单的echo程序。

- 服务端

> 增加了多线程。

```java
		@SneakyThrows
    @Test
    public void ServerIO() {
      	//创建ServerSocket,监听端口3333
        ServerSocket serverSocket = new ServerSocket(3333);
      	//持续接收外部socket连接
        while (true) {
            Socket socket = serverSocket.accept();
          	//放到一个线程中读取
            new ServerThread(socket).start();
        }
    }

		//socket工作线程
    class ServerThread extends Thread {
        Socket socket;

        public ServerThread(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            try {
              	//输出流
                @Cleanup PrintWriter output = new PrintWriter(socket.getOutputStream());
              	//输入流
                @Cleanup BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                
              	//持续读
              	while (true) {
                    String line = null;

                    line = input.readLine();
										//读到，就打印，回复
                    if (line != null) {
                        log.info("接收到 : " + line);
                        output.println("接收到数据：" + line);
                        output.flush();
                    }
                }
//            socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```

- 客户端

> 单线程写，单线程读。

```java
    @SneakyThrows
    @Test
    public void ClientIO() {
      	
      	//创建socket及输入输出流
        Socket socket = new Socket("localhost", 3333);
        BufferedReader input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        PrintWriter output = new PrintWriter(socket.getOutputStream());

      	//发送线程
      	//发送10个hello
        new Thread(() -> {
            IntStream.range(0, 10).forEach(index -> {
                String sendString = "hello" + index;
                log.info("发送：" + sendString);
                output.println(sendString);
                output.flush();
            });
        }, "发送线程").start();

      	//读取线程，读取打印
        new Thread(() -> {
            while (true) {
                String line = null;
                try {
                    line = input.readLine();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                if (line != null) {
                    log.info("服务器返回：" + line);
                }

            }
        }, "读取线程").start();

        new CountDownLatch(1).await();
    }
```

#### nio socket

- 服务端

```java
    private ByteBuffer readBuffer_server = ByteBuffer.allocate(1024);
    private ByteBuffer writeBuffer_server = ByteBuffer.allocate(1024);


    @Test
    public void server() throws IOException {
				
      	//设置本地端口的ServerSocketChannel
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        serverSocketChannel.bind(new InetSocketAddress(8000));

      	//注册选择器。监听通道的连接请求。
        Selector selector = Selector.open();
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

        while (selector.select() > 0) {
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();

                if (!key.isValid()) {
                    continue;
                }
								//连接
                if (key.isAcceptable()) {
                    accept(key, selector);
                //读
                } else if (key.isReadable()) {
                    read(key, selector);
                //写
                } else if (key.isWritable()) {
                    write(key, selector);
                }
            }
        }

    }


    /**
     * 连接
     *
     * @param key
     * @param selector
     * @throws IOException
     */
    private void accept(SelectionKey key, Selector selector) throws IOException {
        ServerSocketChannel serverSocketChannel = (ServerSocketChannel) key.channel();

        SocketChannel channel = serverSocketChannel.accept();
        channel.configureBlocking(false);

        channel.register(selector, SelectionKey.OP_READ);
    }


    /**
     * 读
     *
     * @param key
     * @param selector
     * @throws IOException
     */
    private void read(SelectionKey key, Selector selector) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();

        readBuffer_server.clear();


        //TODO 暂不处理拆包问题。
        int length = channel.read(readBuffer_server);
        if (length > 0) {
            String request = new String(readBuffer_server.array(), 0, length);
            log.info("request:{}", request);

            //业务处理

            String response = "response:" + request;

            //TODO 暂不处理多请求，覆盖问题。
            writeBuffer_server.clear();
            writeBuffer_server.put(response.getBytes());
            writeBuffer_server.flip();
            channel.register(selector, SelectionKey.OP_WRITE);
        }
    }

    /**
     * 写
     *
     * @param key
     * @param selector
     * @throws IOException
     */
    private void write(SelectionKey key, Selector selector) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();
        channel.write(writeBuffer_server);
        channel.register(selector, SelectionKey.OP_READ);
    }
```

- 客户端

```java
    private ByteBuffer readBuffer_client = ByteBuffer.allocate(1024);
    private ByteBuffer writeBuffer_client = ByteBuffer.allocate(1024);

    @Test
    public void client() throws IOException {
        SocketChannel channel = SocketChannel.open();
        channel.configureBlocking(false);
        channel.connect(new InetSocketAddress("localhost", 8000));

      	//注册select，监听连接
        Selector selector = Selector.open();
        channel.register(selector, SelectionKey.OP_CONNECT);
				//console输入
        Scanner scanner = new Scanner(System.in);

        while (selector.select() > 0) {
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();

                if (key.isConnectable()) {
                    if (channel.finishConnect()) {
                        channel.register(selector, SelectionKey.OP_WRITE);
                    } else {
                        log.info("连接失败");
                        System.exit(1);
                    }
                } else if (key.isWritable()) {
                    log.info("请输入数据：");
                    String input = scanner.nextLine();
                    writeBuffer_client.clear();
                    writeBuffer_client.put(input.getBytes());
                    writeBuffer_client.flip();

                    channel.write(writeBuffer_client);

                    channel.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    readBuffer_client.clear();
                    int length = channel.read(readBuffer_client);
                    String response = new String(readBuffer_client.array(), 0, length);

                    log.info("服务端响应：" + response);
                    channel.register(selector, SelectionKey.OP_WRITE);
                }
            }
        }
    }
```



## AIO

> BIO、NIO、AIO的区别
>
> ```text
> BIO:
> 发起请求 --->  阻塞等待  ----> 处理完成
> NIO：
> Selector主动轮询通道事件 ----> 处理事件 
> AIO：
> 发起请求 ----> 通知回调
> ```
>
> AIO提供了连接、读、写全异步操作。
>
> 关于netty为什么不选择aio？
>
> 我的理解是
>
> - 如果不使用线程池，大并发下，AIO会为每一个事件操作创建一个线程操作。
> - 目前AsynchronousChannelProvider提供的系统实现，linux下仍然使用epoll，效率不会比nio更强。

- server端

```java
    //测试类
		@Test
    public void server() throws IOException, InterruptedException {
      	//新建线程池 用来处理操作系统io事件通知
      	//非「业务处理线程」
        ExecutorService poll = new ThreadPoolExecutor(
                10, 10,
                5, TimeUnit.MINUTES,
                new ArrayBlockingQueue<>(5),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.CallerRunsPolicy());

      	
        AsynchronousChannelGroup asynchronousChannelGroup = AsynchronousChannelGroup.withThreadPool(poll);
        AsynchronousServerSocketChannel socketChannel = AsynchronousServerSocketChannel.open(asynchronousChannelGroup);
				//为AsynchronousServerSocketChannel绑定8080监听
        socketChannel.bind(new InetSocketAddress("0.0.0.0", 8080));

      	//为通道注册连接socket连接handler
        socketChannel.accept(null, new ServerSocketChannelHandler(socketChannel));

      	//阻塞主线程
        new CountDownLatch(1).await();
    }


/**
 * AsynchronousServerSocketChannel上的连接处理器
 */
@Slf4j
@AllArgsConstructor
class ServerSocketChannelHandler implements CompletionHandler<AsynchronousSocketChannel, Void> {

    private AsynchronousServerSocketChannel serverSocketChannel;

  	//attachment可以理解为请求上下文
    @Override
    public void completed(AsynchronousSocketChannel socketChannel, Void attachment) {
        log.info("channel completed:{}|{}", socketChannel, attachment);
      	//使用当前连接处理器，继续监听。每次都要重新监听。一个连接一个监听。
        serverSocketChannel.accept(attachment, this);
				
      	//创建用于数据读取的byteBuffer
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
	
      	//新建一个StringBuffer作为请求上下文使用。当然也可以自定义
      	//为新连接上来的socket通道注册通道「读取处理器」
        socketChannel.read(byteBuffer, new StringBuffer(), new SocketChannelReadHandler(socketChannel, byteBuffer));
    }

    @Override
    public void failed(Throwable exc, Void attachment) {
        log.error("completed failed:{}", exc.getMessage(), exc);
    }
}

/**
 *AsynchronousSocketChannel上的读取处理器
 */
@Slf4j
@AllArgsConstructor
class SocketChannelReadHandler implements CompletionHandler<Integer, StringBuffer> {

    private AsynchronousSocketChannel socketChannel;
    private ByteBuffer byteBuffer;

    @Override
    public void completed(Integer result, StringBuffer attachment) {
      	//防止客户端主动关闭连接
        if (result == -1) {
            try {
                this.socketChannel.close();
            } catch (IOException e) {
                log.error("channel read failed:{}", e.getMessage(), e);
            }
        }
      	
      	//这里AIO自动帮我们做了拆包。
      	//也就是当client发来的数据大于1024的读取buffer时，会多次调用。
      	//需要做的就是加到请求上下文attachment。
      	
      	//而。这里默认请求都小于1024
      
        log.info("读取数据：{}|{}", result, attachment);
        byteBuffer.flip();
        String request = new String(byteBuffer.array(), 0, byteBuffer.limit());

        attachment.append(request);
        log.info("数据：{}", request);


        byteBuffer.clear();
        log.info("客户端发来的数据：{}", attachment);

      	//回复
        ByteBuffer writeBuffer = ByteBuffer.wrap(("已收到数据：" + attachment).getBytes());
        socketChannel.write(writeBuffer);

				//为通道注册下一次读取监听
        log.info("===准备下一次监听===");
        attachment = new StringBuffer();
        socketChannel.read(byteBuffer, attachment, this);

    }

    @Override
    public void failed(Throwable exc, StringBuffer attachment) {
        log.error("client已经关闭连接.{}", exc.getMessage(), exc);
    }


}
```



- client端

> client选择bio、nio、aio都无所谓，这里只提供aio的简单demo。

```java
		@Test
    public void client() throws IOException, InterruptedException {
      	//创建AsynchronousSocketChannel
        AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open();
				
      	//连接服务端
      	//并为通道绑定连接成功的处理器
        socketChannel.connect(new InetSocketAddress("localhost", 8080), null, new CompletionHandler<Void, Void>() {
            @Override
            public void completed(Void result, Void attachment) {
                log.info("连接到服务器成功");

                try {
                  	//写
                    socketChannel.write(ByteBuffer.wrap("hello".getBytes())).get();

                  	//读buffer
                    ByteBuffer readBuffer = ByteBuffer.allocate(1024);

                    //粗暴的阻塞等待返回
                    socketChannel.read(readBuffer).get();
										
                    log.info("服务端返回：{}", new String(readBuffer.array(), 0, readBuffer.limit()));
                    socketChannel.close();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (ExecutionException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                log.error("数据发送异常：{}", exc.getMessage(), exc);
            }
        });
				//阻塞主线程
        new CountDownLatch(1).await();
    }
```

## Netty

### socket demo

> 添加pom依赖
>
> ```xml
>         <!-- https://mvnrepository.com/artifact/io.netty/netty-all -->
>         <dependency>
>             <groupId>io.netty</groupId>
>             <artifactId>netty-all</artifactId>
>             <version>5.0.0.Alpha2</version>
>         </dependency>
> 
> ```

- server端

```java
@Slf4j
public class NettyServer {
		
  	//测试类
    @Test
    public void server() throws InterruptedException {
        openServer();
    }
		
    public void openServer() throws InterruptedException {
      	
      	//bossGroup 处理连接的「单线程」
        @Cleanup("shutdownGracefully") NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
      	//workerGroup 处理通道事件的「线程池」
        @Cleanup("shutdownGracefully") NioEventLoopGroup workerGroup = new NioEventLoopGroup(10);

      	//创建启动类
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(bossGroup, workerGroup)//绑定两个「线程池」
                .channel(NioServerSocketChannel.class)//指定NIO通道模式
                .childHandler(new ChannelInitializer<SocketChannel>() {//初始化器
                  	//channel注册后，执行这里的初始化方法
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                      	//传入自定义的服务端操作。
                        ch.pipeline().addLast(new SimpleNettyServerHandler());
                    }
                });
      	//启动服务，绑定端口，开始接受连接。
      	//同步。
        ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
        log.info("server started.");
      	//监听channel的关闭，等待服务器 socket 关闭。
      	//同步
        channelFuture.channel().closeFuture().sync();
    }
}

//仅仅模拟了读数据和返回的hello程序
@Slf4j
class SimpleNettyServerHandler extends ChannelHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        log.info("simple handler channel read:{}|{}", ctx, msg);
				//// msg中存储的是ByteBuf类型的数据，把数据读取到byte[]中
        ByteBuf byteBuf = (ByteBuf) msg;

        int limit = byteBuf.readableBytes();
        byte[] bytes = new byte[limit];
        byteBuf.readBytes(bytes);

        String request = new String(bytes);

        log.info("request:{}", request);

        byteBuf.release();// 释放资源，这行很关键

      	//模拟回复
        String response = "hello " + request;

        ByteBuf writeBuf = ctx.alloc().buffer(4 * response.length());

        writeBuf.writeBytes(response.getBytes());

        ctx.write(writeBuf);
        ctx.flush();

    }
}
```

- server端

```java
@Slf4j
public class NettyClient {

    @Test
    public void client() throws InterruptedException {
        openClient();
    }

    private void openClient() throws InterruptedException {
        @Cleanup("shutdownGracefully") NioEventLoopGroup workerGroup = new NioEventLoopGroup(5);

        Bootstrap bootstrap = new Bootstrap();
        bootstrap
                .group(workerGroup)
                .channel(NioSocketChannel.class) //指定nio模式
                .option(ChannelOption.SO_KEEPALIVE, true)//设置连接保持
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new SimpleNettyClientHandler());
                    }
                });
        ChannelFuture channelFuture = bootstrap.connect("localhost", 8080).sync();
        channelFuture.channel().closeFuture().sync();

    }

}

@Slf4j
class SimpleNettyClientHandler extends ChannelHandlerAdapter {
  	//读
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        log.info("simple client handler read:{}|{}", ctx, msg);
        ByteBuf byteBuf = (ByteBuf) msg;

        byte[] alloc = new byte[byteBuf.readableBytes()];
        byteBuf.readBytes(alloc);

        String response = new String(alloc);
        log.info("response:{}", response);

        byteBuf.release();
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
				//通道连接完成，可用，发送数据。
        log.info("simple client handler active:{}", ctx);
        String request = "netty";
        ByteBuf writeBuf = ctx.alloc().buffer(request.length() * 4);
        writeBuf.writeBytes(request.getBytes());
        ctx.write(writeBuf);
        ctx.flush();
    }
}
```

### 编码解码器

> 问题1：使用「网络调试助手」测试上面的server。长字符串、中文，就会出现拆包、中文乱码的问题。
>
> ![image-20200820143936703](https://gitee.com/wangigor/typora-images/raw/master/netty-网络调试助手-长中文测试.png)
>
> ![image-20200820144104058](https://gitee.com/wangigor/typora-images/raw/master/ netty-长中文测试-console.png)
>
> 问题2：之前的server的消息读取，「每次」都会有ByteBuf转String的操作。
>
> 这两个问题。都是基于请求对象为String的。这就是一个例子，你懂叭。

> netty提供了String的编码解码器，也提供了String的沾包拆包的解码器。
>
> - ```java
>   StringEncoder //String编码器
>   StringDecoder //String解码器
>   ```
>   
> - ```java
>   LineBasedFrameDecoder //基于换行
>   DelimiterBasedFrameDecoder //基于指定字符串
>   FixedLengthFrameDecoder //基于固定字符串长度
>   ```

改造一下server端。

```java
		public void openServer() throws InterruptedException {
        @Cleanup("shutdownGracefully") NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        @Cleanup("shutdownGracefully") NioEventLoopGroup workerGroup = new NioEventLoopGroup(10);

        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new CustomChannelInitializer());//自定义初始化器
        ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
        log.info("server started.");
        channelFuture.channel().closeFuture().sync();
    }
class CustomChannelInitializer extends ChannelInitializer {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline()
          			//换行符解码器
                .addLast(new LineBasedFrameDecoder(2048)) //这里是最大长度限制，超长报错。
//固定字符解码器   .addLast(new DelimiterBasedFrameDecoder(1024, Unpooled.copiedBuffer("EOF".getBytes())))
//定长解码器      .addLast(new FixedLengthFrameDecoder(1024))
          			//GBK编码格式解码器
                .addLast(new StringDecoder(Charset.forName("GBK")))
                .addLast(new StringNettyServerHandler());
    }
}

@Slf4j
class StringNettyServerHandler extends ChannelHandlerAdapter{
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      
      	//这里传进来的msg就是string
        log.info("string handler channel read:{}|{}", ctx, msg);

        String request = (String) msg;
        log.info("request:{}", msg);


        String response = "hello " + request;

        ByteBuf writeBuf = ctx.alloc().buffer(4 * response.length());

        writeBuf.writeBytes(response.getBytes());

        ctx.write(writeBuf);
        ctx.flush();
    }
}

```

测试结果

![image-20200820152209403](https://gitee.com/wangigor/typora-images/raw/master/netty-网络调试助手-长中文-优化.png)

![image-20200820152303681](https://gitee.com/wangigor/typora-images/raw/master/netty-长中文-优化-console.png)

> 问题3：消息发送（回复）手动string转ByteBuf。
>
> ```java
> //看看这些冗长的烂代码吧
> String response = "hello " + request;
> ByteBuf writeBuf = ctx.alloc().buffer(4 * response.length());
> writeBuf.writeBytes(response.getBytes());
> ctx.write(writeBuf);
> ctx.flush();
> ```
>
> ```java
> //优化
> //自定义的通道初始化器 增加StringEncoder字符串编码器
> .addLast(new StringEncoder(Charset.forName("GBK")))
>   
> //上面的代码直接发送String  
> String response = "hello " + request;
> ctx.writeAndFlush(response);
> ```

### 自定义编码解码器，处理粘包拆包和数据收发问题

> 自定义一个简单协议
>
> ```txt
> +-------------------------------------+
> |  协议开始标记  |  报文体长度  |  报文体  |
> +-------------------------------------+
> ```
>

- 自定义协议DTO

```java
@Data
@ToString
@RequiredArgsConstructor
public class SimpleProtocol {
    /**
     * 协议开始标志
     */
    public static transient final int SIGN_START = 0xff;

    /**
     * 请求体长度
     */
    @NonNull
    private int bodyLength;

    /**
     * 请求体
     */
    @NonNull
    private String body;

    public SimpleProtocol(String bodyString) {
        this.body = bodyString;
        this.bodyLength = bodyString.getBytes().length;
    }
}
```

- 自定义协议解码器/编码器


```java
/**
 * 解码器
 * <p>
 * 数据格式
 * +-------------------------------------+
 * |  协议开始标记  |  报文体长度  |  报文体 |
 * +-------------------------------------+
 * <p>
 * 开始标记sign_start和报文体长度bodyLength ，是int类型，各占4个字节
 *
 * @author wangke
 */
@Slf4j
public class SimpleDecoder extends ByteToMessageDecoder {

    /**
     * sign_start + bodyLength
     */
    public final int BASE_LENGTH = 4 + 4;
    /**
     * 长度限制
     */
    public final int LIMIT = 4096;

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
				//数据包可读长度
        int readableBytes = in.readableBytes();
				
      	//解析的「当前包」长度起码要大于基础长度「协议头+body长度」两个int，共战8个byte
        if (readableBytes >= BASE_LENGTH) {
						
          	//过滤总长度超长的情况
            if (readableBytes >= LIMIT) {
                in.skipBytes(readableBytes);
            }
          	
						//记录「当前包」头位置
            int begin_read;

            while (true) {
								//记录「当前包」头位置
                begin_read = in.readerIndex();
              	//标记「当前包」
                in.markReaderIndex();
								
              	//读一个int 判断协议头
              	//可能有前一个包的后续数据
                if (in.readInt() == SimpleProtocol.SIGN_START) {
                    break;
                }
								//未读到协议头，因为之前跳过了一个int，这里要重置index
                in.resetReaderIndex();
                in.readByte();
              	//如果这时候，长度不到limit了，就等待后续数据到达
                if (in.readableBytes() < SimpleProtocol.LIMIT) {
                    return;
                }
            }
						
          	//读到这里的时候，是前面读到了「一个完整包的协议头」
          	//继续读取一个int，获取body长度
            int bodyLength = in.readInt();
						//数据长度不够，等待后续数据到达，这里暂不处理。
            if (in.readableBytes() < bodyLength) {
                in.readerIndex(begin_read);
                return;
            }
						
          	//创建读取缓冲区 读取数据
          	//当然也可以使用ByteBuf msgContent = in.readBytes(readableBytes);
            byte[] bytes = new byte[bodyLength];
            in.readBytes(bytes);

          	//组装SimpleProtocol
          	//向后pipeline chain调用
            out.add(
                    new SimpleProtocol(bodyLength, new String(bytes, "GBK"))
            );
        }
    }
}
```

```java
/**
 * 编码器
 *
 * @author wangke
 */
public class SimpleEncoder extends MessageToByteEncoder<SimpleProtocol> {

    @Override
    protected void encode(ChannelHandlerContext ctx, SimpleProtocol msg, ByteBuf out) throws Exception {
      	//写协议头 写一个int
        out.writeInt(SimpleProtocol.SIGN_START);
      	//写body长度 一个int
        out.writeInt(msg.getBodyLength());
      	//写body bytes
        out.writeBytes(msg.getBody().getBytes("GBK"));
    }
}
```

- 服务端

```java
@Slf4j
public class SimpleNettyServer {

    @Test
    public void server() throws InterruptedException {

        @Cleanup("shutdownGracefully") NioEventLoopGroup bossGroup = new NioEventLoopGroup(1);
        @Cleanup("shutdownGracefully") NioEventLoopGroup workerGroup = new NioEventLoopGroup(10);

        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap
                .group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new SimpleChannelInitializer());
        ChannelFuture channelFuture = serverBootstrap.bind(8080).sync();
        log.info("server started.");
        channelFuture.channel().closeFuture().sync();
    }
}

class SimpleChannelInitializer extends ChannelInitializer {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline()
                .addLast(new SimpleDecoder())
                .addLast(new SimpleEncoder())
                .addLast(new SimpleServerHandler());
    }
}
```

```java
//业务处理
@Slf4j
public class SimpleServerHandler extends ChannelHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
      	//msg就是SimpleProtocol
        SimpleProtocol simpleProtocol = (SimpleProtocol) msg;

        log.info("request:" + simpleProtocol.toString());

        String response = "hello,i got ur message:" + simpleProtocol.getBody();
				//组装SimpleProtocol「响应数据」。『返回』。
        ctx.writeAndFlush(new SimpleProtocol(response));
    }
}
```

- 客户端

```java
@Slf4j
public class SimpleNettyClient {
    @Test
    public void client() throws InterruptedException {
        @Cleanup("shutdownGracefully") NioEventLoopGroup workerGroup = new NioEventLoopGroup(5);

        Bootstrap bootstrap = new Bootstrap();
        bootstrap
                .group(workerGroup)
                .channel(NioSocketChannel.class)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ch.pipeline()
                                .addLast(new SimpleEncoder())
                                .addLast(new SimpleDecoder())
                                .addLast(new SimpleNettyClientHandler());
                    }
                });
        ChannelFuture channelFuture = bootstrap.connect("localhost", 8080).sync();
        channelFuture.channel().closeFuture().sync();

    }

}

@Slf4j
class SimpleNettyClientHandler extends ChannelHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        log.info("连接完成：{}", ctx);
				//连接完成之后 发送一个简单字符串
        ctx.writeAndFlush(new SimpleProtocol("custom protocol"));

    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        log.info("接收到服务端回复：{}", msg);
    }
}
```

以上，直接运行server和client测试类，就可以完成一个简单的echo的完整调用。

***

#### 粘包拆包测试

> 这里使用「网络调试助手」发送十六进制，做完整包、粘包、拆包测试。
>
> body数据使用"custom protocol"
>
> 数据说明：
>
> - 协议头0xff要转为int，占4个byte：00 00 00 ff
> - body长度就是15，转为int：00 00 00 0f
> - 发送数据「custom protocol」转bytes，按十六进制显示 ：63 75 73 74 6f 6d 20 70 72 6f 74 6f 63 6f 6c
>
> ```java
>     //发送数据「custom protocol」转bytes，按十六进制显示
> 		@Test
>     public void testBytes() {
>         byte[] bytes = "custom protocol".getBytes();
>         for (byte a : bytes) {
>             log.info(Integer.toHexString(a & 0xff));
>         }
>     }
> ```
> 启动server。
>

- 完整包测试

![image-20200824113548560](https://gitee.com/wangigor/typora-images/raw/master/netty-自定义协议-完整包测试.png)

服务端console获取到的数据为

![image-20200824113738422](https://gitee.com/wangigor/typora-images/raw/master/netty-自定义协议-完整包测试-server-console.png)

解析数据

>  00 00 00 ff 00 00 00 26 68 65 6c 6c 6f 2c 69 20 67 6f 74 20 75 72 20 6d 65 73 73 61 67 65 3a 63 75 73 74 6f 6d 20 70 72 6f 74 6f 63 6f 6c
>
> - [x] 协议头：00 00 00 ff
> - [x] 数据长度：00 00 00 26 「38」
> - [x] 数据体：68 65 6c 6c 6f 2c 69 20 67 6f 74 20 75 72 20 6d 65 73 73 61 67 65 3a 63 75 73 74 6f 6d 20 70 72 6f 74 6f 63 6f 6c
>
> ```java
>     @Test
>     public void testResponse() throws UnsupportedEncodingException {
>         byte[] bytes = {0x68,0x65,0x6c,0x6c,0x6f,0x2c ,0x69 ,0x20 ,0x67 ,0x6f ,0x74 ,0x20 ,0x75 ,0x72 ,0x20 ,0x6d ,0x65 ,0x73 ,0x73 ,0x61 ,0x67 ,0x65 ,0x3a ,0x63 ,0x75 ,0x73 ,0x74 ,0x6f ,0x6d ,0x20 ,0x70 ,0x72 ,0x6f ,0x74 ,0x6f ,0x63 ,0x6f ,0x6c};
>         log.info(new String(bytes,"GBK"));
>     }
> //结果
> //hello,i got ur message:custom protocol
> ```
>

- 拆包测试

> 一个包发送两次
>
> 第一次：
>
> ![image-20200824141255016](https://gitee.com/wangigor/typora-images/raw/master/netty-自定义协议-拆包测试-1.png)
>
> 此时通道处于等待后续数据状态。
>
> 第二次：
>
> ![image-20200824141815236](https://gitee.com/wangigor/typora-images/raw/master/netty-自定义协议-拆包测试-2_.png)
>
> 服务端接收正常
>
> ![image-20200824141912767](https://gitee.com/wangigor/typora-images/raw/master/netty-自定义协议-拆包测试-console.png)
>
> 返回正常。

- 粘包测试

> 两个包一起发。
>
> ![image-20200824142105325](https://gitee.com/wangigor/typora-images/raw/master/netty-自定义协议-粘包测试.png)
>
> ![image-20200824142139936](https://gitee.com/wangigor/typora-images/raw/master/netty-自定义协议-粘包测试-console.png)
>
> 「两次」请求，「两次」响应。正常。



### 群发

> 消息群发是通过channelGroup来完成的。
>
> 把连接上来的客户端channel，放在channelGroup中。接收完一个channel发送的数据后，由channelGroup进行消息发送。

```java
//服务端群发
@Slf4j
public class GroupServerHandler extends ChannelHandlerAdapter {

    /**
     * 用于存储客户端Channel信息
     * 也可以自建分组
     */
    public static ChannelGroup channels = new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);

  	//新建连接
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        SocketChannel channel = (SocketChannel) ctx.channel();
        log.info("新的连接：{}:{}", channel.localAddress().getHostName(), channel.localAddress().getPort());
        //存到group中
      	channels.add(channel);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        SimpleProtocol simpleProtocol = (SimpleProtocol) msg;

        log.info("request:" + simpleProtocol.toString());

        String response = "new message:" + simpleProtocol.getBody();
				//使用ChannelGroup推送消息
        channels.writeAndFlush(new SimpleProtocol(response));
    }

  	//断开连接
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        SocketChannel channel = (SocketChannel) ctx.channel();
        log.info("断开连接：{}:{}", channel.localAddress().getHostName(), channel.localAddress().getPort());
        //从group中移除
      	channels.remove(channel);
    }
}
```

### UDP

todo





## Selector

> ![img](https://gitee.com/wangigor/typora-images/raw/master/selector.png)
>
>  SelectableChannel的多路复用器multiplexor。

### Selector创建

```java
//入口
Selector selector = Selector.open();
```

```java
		//通过调用系统默认的SelectorProvider，来创建selector
		public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }
```

#### provider

```java
    public static SelectorProvider provider() {
      	//懒汉式单例
        synchronized (lock) {
            if (provider != null)
                return provider;
          	//权限相关，暂时不管
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
                      			//如果系统属性有java.nio.channels.spi.SelectorProvider对应的class属性，初始化「newInstance」之。
                            if (loadProviderFromProperty())
                                return provider;
                      			//ServiceLoader.load(SelectorProvider.class,ClassLoader.getSystemClassLoader());
                      			//通过SPI从jar包的META-INF/services/路径下查找SelectorProvider指定的impl类。
                            if (loadProviderAsService())
                                return provider;
                      			//否则使用系统默认的provider
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }
```

#### DefaultSelectorProvider

> 不同平台的jdk有不同的实现

| 系统          | Provider                    |
| ------------- | --------------------------- |
| windows       | **WindowsSelectorProvider** |
| linux-SunOS   | **DevPollSelectorProvider** |
| linux-2.6以后 | **EPollSelectorProvider**   |
| MACos         | **KQueueSelectorProvider**  |

以mac为例。

```java
public class KQueueSelectorProvider extends SelectorProviderImpl {
    public KQueueSelectorProvider() {
    }

    public AbstractSelector openSelector() throws IOException {
        return new KQueueSelectorImpl(this);
    }
}
```

```java
    KQueueSelectorImpl(SelectorProvider var1) {
        super(var1);
      	//native调用pipe()函数返回两个文件描述符fd
        long var2 = IOUtil.makePipe(false);
      	//高32位fd0只读数据
        this.fd0 = (int)(var2 >>> 32);
      	//低32位fd1只写数据
        this.fd1 = (int)var2;

        try {
          	//初始化
            this.kqueueWrapper = new KQueueArrayWrapper();
            this.kqueueWrapper.initInterrupt(this.fd0, this.fd1);
            this.fdMap = new HashMap();//fd红黑树
            this.totalChannels = 1;
        } catch (Throwable var8) {
            try {
                FileDispatcherImpl.closeIntFD(this.fd0);
            } catch (IOException var7) {
                var8.addSuppressed(var7);
            }

            try {
                FileDispatcherImpl.closeIntFD(this.fd1);
            } catch (IOException var6) {
                var8.addSuppressed(var6);
            }

            throw var8;
        }
    }
```



### 通道注册register

```java
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
```

把channel注册到selector上，并标注应该注意的通道事件。

```java
    public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        synchronized (regLock) {
          	//通道检查
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
          	//先查找已经监听的channel的SelectionKey
          	//如果有，设置新的感兴趣操作「事件」
            SelectionKey k = findKey(sel);
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
          	//新注册
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                  	//创建新的SelectionKeyImpl并设置感兴趣事件
                    k = ((AbstractSelector)sel).register(this, ops, att);
                  	//加入selector的SelectionKey[]中
                    addKey(k);
                }
            }
            return k;
        }
    }
```

### select()

```java
    public int select(long var1) throws IOException {
        if (var1 < 0L) {
            throw new IllegalArgumentException("Negative timeout");
        } else {
            return this.lockAndDoSelect(var1 == 0L ? -1L : var1);
        }
    }

    public int select() throws IOException {
        return this.select(0L);
    }

    public int selectNow() throws IOException {
        return this.lockAndDoSelect(0L);
    }

```

> 三个重载方法，都是调用lockAndDoSelect

```java
    //调用子类实现。为了适应各种平台的差异。
		protected abstract int doSelect(long var1) throws IOException;

    private int lockAndDoSelect(long var1) throws IOException {
        synchronized(this) {
            if (!this.isOpen()) {
                throw new ClosedSelectorException();
            } else {
                int var10000;
                synchronized(this.publicKeys) {
                    synchronized(this.publicSelectedKeys) {
                        var10000 = this.doSelect(var1);
                    }
                }

                return var10000;
            }
        }
    }
```

macos的实现是

```java
    protected int doSelect(long var1) throws IOException {
        boolean var3 = false;
        if (this.closed) {
            throw new ClosedSelectorException();
        } else {
            this.processDeregisterQueue();

            int var7;
            try {
                this.begin();
                var7 = this.kqueueWrapper.poll(var1);
            } finally {
                this.end();
            }

            this.processDeregisterQueue();
            return this.updateSelectedKeys(var7);
        }
    }
```

sunos的实现是

```java
    protected int doSelect(long var1) throws IOException {
        if (this.channelArray == null) {
            throw new ClosedSelectorException();
        } else {
            this.processDeregisterQueue();

            try {
                this.begin();
                this.pollWrapper.poll(this.totalChannels, 0, var1);
            } finally {
                this.end();
            }

            this.processDeregisterQueue();
            int var3 = this.updateSelectedKeys();
            if (this.pollWrapper.getReventOps(0) != 0) {
                this.pollWrapper.putReventOps(0, 0);
                synchronized(this.interruptLock) {
                    IOUtil.drain(this.fd0);
                    this.interruptTriggered = false;
                }
            }

            return var3;
        }
    }
```

都是使用了poll.



## Netty源码

### NioEventLoopGroup

> ![image-20200908112212593](https://gitee.com/wangigor/typora-images/raw/master/NioEventLoopGroup.png)
>
> 他是一个「**线程池**」。

#### 构造

```java
//这里提供了5个重载的构造方法。
public class NioEventLoopGroup extends MultithreadEventLoopGroup {

		//默认使用处理器核的两倍线程数。
    public NioEventLoopGroup() {
        this(0);
    }

		//默认执行器null
    public NioEventLoopGroup(int nEventLoops) {
        this(nEventLoops, (Executor) null);
    }

		//使用默认提供的Selector
    public NioEventLoopGroup(int nEventLoops, Executor executor) {
        this(nEventLoops, executor, SelectorProvider.provider());
    }

		//自定义线程池工厂
    public NioEventLoopGroup(int nEventLoops, ExecutorServiceFactory executorServiceFactory) {
        this(nEventLoops, executorServiceFactory, SelectorProvider.provider());
    }

		//自定义执行器-重载调用父类
    public NioEventLoopGroup(int nEventLoops, Executor executor, final SelectorProvider selectorProvider) {
        super(nEventLoops, executor, selectorProvider);
    }
  	//自定义线程池工厂-重载调用父类
    public NioEventLoopGroup(
            int nEventLoops, ExecutorServiceFactory executorServiceFactory, final SelectorProvider selectorProvider) {
        super(nEventLoops, executorServiceFactory, selectorProvider);
    }
}
```

父类是MultithreadEventLoopGroup「多线程时间轮询组」

```java
public abstract class MultithreadEventLoopGroup extends MultithreadEventExecutorGroup implements EventLoopGroup {

    private static final int DEFAULT_EVENT_LOOP_THREADS;
		//默认使用处理器核的两倍。
  	//也可以通过io.netty.eventLoopThreads
    static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));
    }

  	//重写的两个父类构造器
    protected MultithreadEventLoopGroup(int nEventLoops, Executor executor, Object... args) {
        super(nEventLoops == 0 ? DEFAULT_EVENT_LOOP_THREADS : nEventLoops, executor, args);
    }
    protected MultithreadEventLoopGroup(int nEventLoops,
                                        ExecutorServiceFactory executorServiceFactory,
                                        Object... args) {
        super(nEventLoops == 0 ? DEFAULT_EVENT_LOOP_THREADS : nEventLoops, executorServiceFactory, args);
    }
}
```

父类MultithreadEventExecutorGroup「多线程事件处理器组」

```java
    //构造方法重载
		protected MultithreadEventExecutorGroup(int nEventExecutors,
                                            ExecutorServiceFactory executorServiceFactory,
                                            Object... args) {
      	//使用线程池工厂，初始化一个「线程池」
        this(nEventExecutors, executorServiceFactory != null
                                ? executorServiceFactory.newExecutorService(nEventExecutors)
                                : null,
             true, args);
    }
		//构造方法重载
    protected MultithreadEventExecutorGroup(int nEventExecutors, Executor executor, Object... args) {
        this(nEventExecutors, executor, false, args);
    }

		//构造
    private MultithreadEventExecutorGroup(int nEventExecutors,
                                          Executor executor,
                                          boolean shutdownExecutor,
                                          Object... args) {
      	//指定线程数 校验
        if (nEventExecutors <= 0) {
            throw new IllegalArgumentException(
                    String.format("nEventExecutors: %d (expected: > 0)", nEventExecutors));
        }
				
      	//初始化线程池，默认的NioEventLoopGroup会走进这里
        if (executor == null) {
          	//默认初始化一个并行度为nEventExecutors的ForkJoinPool
          	//指定了前缀为nioEventLoopGroup—executorId.getAndIncrement()
          	//工作线程异常处理器使用log.error打印
          	//异步模式为FIFO「先进先出」
            executor = newDefaultExecutorService(nEventExecutors);
            shutdownExecutor = true;
        }
				//初始化线程数组
        children = new EventExecutor[nEventExecutors];
      
      	//定义线程选择策略
      	//这里对并发度是否为2次幂的判断很巧妙
      	//return (val & -val) == val; 因为一个数是2的n次幂，那么一定只有一位是1，后续全是0；
      	//按位选择器更快，尽量并发度为2次幂
        if (isPowerOfTwo(children.length)) {
          	//采用位运算选择器 &
            chooser = new PowerOfTwoEventExecutorChooser();
        } else {
          	//采用取模选择器 %
            chooser = new GenericEventExecutorChooser();
        }

      	//初始化线程数组
        for (int i = 0; i < nEventExecutors; i ++) {
            boolean success = false;
            try {
              	//创建NioEventLoop todo
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
              	//创建失败，释放资源
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        final boolean shutdownExecutor0 = shutdownExecutor;
        final Executor executor0 = executor;
      	//设置对关闭的监听器。为了监听每一个NioEventLoop的终止。
        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                    if (shutdownExecutor0) {
                        // This cast is correct because shutdownExecutor0 is only try if
                        // executor0 is of type ExecutorService.
                        ((ExecutorService) executor0).shutdown();
                    }
                }
            }
        };
				//设置线程终止监听器。
        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }
				
      	//线程组的只读设置
        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```



### NioEventLoop

> 一个「工作线程」要监听处理channel的事件。
>
> ![image-20200908170624068](https://gitee.com/wangigor/typora-images/raw/master/NioEventLoop.png)
>
> 维护了一个线程，线程启动会调用润方法，执行io任务。
>
> NioEventLoopGroup创建NioEventLoop是通过
>
> ```java
>  @Override
>  protected EventLoop newChild(Executor executor, Object... args) throws Exception {
>    	//传入 NioEventLoopGroup，线程池，SelectorProvider
>      return new NioEventLoop(this, executor, (SelectorProvider) args[0]);
>  }
> ```
> 父类SingleThreadEventExecutor
>
> ```java
>     //放置了任务列表和一个「本地」线程
> 		private final Queue<Runnable> taskQueue;
>     private volatile Thread thread;
> ```
>
> 


#### 构造

```java
    //NioEventLoop extends SingleThreadEventLoop
		NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider) {
        //父构造器
      	super(parent, executor, false);
        if (selectorProvider == null) {
            throw new NullPointerException("selectorProvider");
        }
      	//设置selectorProvider并获取selector
        provider = selectorProvider;
        selector = openSelector();
    }
		//SingleThreadEventLoop extends SingleThreadEventExecutor
    protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor, boolean addTaskWakesUp) {
        super(parent, executor, addTaskWakesUp);
    }
		//SingleThreadEventExecutor extends AbstractScheduledEventExecutor
    protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor, boolean addTaskWakesUp) {
        super(parent);

        if (executor == null) {
            throw new NullPointerException("executor");
        }

        this.addTaskWakesUp = addTaskWakesUp;
        this.executor = executor;//线程池
        taskQueue = newTaskQueue();//任务队列 new LinkedBlockingQueue<Runnable>();
    }
		//AbstractScheduledEventExecutor extends AbstractEventExecutor
    protected AbstractScheduledEventExecutor(EventExecutorGroup parent) {
        super(parent);
    }
		//AbstractEventExecutor 
		private final EventExecutorGroup parent;
    protected AbstractEventExecutor(EventExecutorGroup parent) {
        this.parent = parent;
    }
```

> 这时，**线程还没有创建**。

#### 添加任务addTask

```java
    protected void addTask(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (isShutdown()) {
            reject();
        }
      	//就是LinkedBlockingQueue的添加操作
        taskQueue.add(task);
    }
```

#### **启动startExecution**

> 下面看一下任务执行和NioEventLoop启动。

```java
    //PausableChannelEventExecutor 
		//在前面初始化的时候，对eventLoop进行的包裹
		@Override
    public void execute(Runnable command) {
        if (!isAcceptingNewTasks()) {
            throw new RejectedExecutionException();
        }
      	//执行任务
        unwrap().execute(command);
    }
```

```java
		//SingleThreadEventExecutor
		@Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        boolean inEventLoop = inEventLoop();
      	//线程池提交的任务都是进行的addTask操作
        if (inEventLoop) {
            addTask(task);
        } else {
          	//启动
            startExecution();
            addTask(task);
          	//拒绝
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }
				
      	//添加任务唤醒，默认是false
      	//且task实现了NonWakeupRunnable接口，也不唤醒
        if (!addTaskWakesUp && wakesUpForTask(task)) {
          	//唤醒
            wakeup(inEventLoop);
        }
    }
```

> 启动的入口在registry，使用用户线程提交了一个regisrty0的任务，触发startExecution。

```java
    private void startExecution() {
      	//SingleThreadEventExecutor的state状态，通过未启动判断和cas进行修改。为启动started。
        if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
              	//添加了一个清理canneled任务的周期任务。
                schedule(new ScheduledFutureTask<Void>(
                        this, Executors.<Void>callable(new PurgeTask(), null),
                        ScheduledFutureTask.deadlineNanos(SCHEDULE_PURGE_INTERVAL), -SCHEDULE_PURGE_INTERVAL));
                scheduleExecution();
            }
        }
    }
    protected final void scheduleExecution() {
        updateThread(null); //把当前NioEventLoop的Thread属性置为null
        executor.execute(asRunnable);//使用线程池执行
      	//这里的executor是并发度为nEventLoops的ForkJoinPool「netty自己的forkjoinpool」
    }
    private final Runnable asRunnable = new Runnable() {
        @Override
        public void run() {
          	//设置为forkjoin线程nioEventLoopGroup-0-0
            updateThread(Thread.currentThread());

            // lastExecutionTime must be set on the first run
            // in order for shutdown to work correctly for the
            // rare case that the eventloop did not execute
            // a single task during its lifetime.
            if (firstRun) {
                firstRun = false;
                updateLastExecutionTime();
            }

            try {
              	//启动
                SingleThreadEventExecutor.this.run();
            } catch (Throwable t) {
                logger.warn("Unexpected exception from an event executor: ", t);
                cleanupAndTerminate(false);
            }
        }
    };
```
>


```java
    //NioEventLoop
		//可以看到这是一个周期性的递归调用select方法，当前执行完又回到scheduleExecution()
		protected void run() {
      	//这里wakenUp是防止多次执行selector.wakeup()的开销
        boolean oldWakenUp = wakenUp.getAndSet(false);
        try {
          	//如果有等待任务，就执行非阻塞select，分出更多的时间执行等待任务。
            if (hasTasks()) {
                selectNow(); //非阻塞的select，没有io事件就返回o
            } else {
                select(oldWakenUp);//阻塞等待
              	//阻塞时间根据周期队列scheduledTaskQueue的执行时间而定。
              	//如果队列为空，默认等待1s

                if (wakenUp.get()) {//这里是看是否被外部唤醒，如果被外部唤醒了就唤醒select上的等待线程
                  									//比如 添加任务execute(task)就会唤醒。
                    selector.wakeup();
                }
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;	//这里是io比例。也就是处理io操作「processSelectedKeys()」和执行任务「runAllTasks()所占的比例」
          	//默认是50
            if (ioRatio == 100) {
                processSelectedKeys();
                runAllTasks();
            } else {
                final long ioStartTime = System.nanoTime();

                processSelectedKeys();

                final long ioTime = System.nanoTime() - ioStartTime;//记录io执行耗时，计算任务执行耗时。
                runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
            }

            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    cleanupAndTerminate(true);
                    return;
                }
            }
        } catch (Throwable t) {
            logger.warn("Unexpected exception in the selector loop.", t);

            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                // Ignore.
            }
        }
				//进入下一个周期操作。
        scheduleExecution();
    }
```

##### 执行任务 runAllTasks

> 只看runAllTasks()就行，另外一个带timeoutNanos参数的是要计算在当前时间内完成执行，超时就不执行「新poll的」task了。

```java
//从taskQueue队列中取出任务逐个执行
protected boolean runAllTasks() {
  	//scheduledTaskQueue队列中的任务先合并到taskQueue中。
    fetchFromScheduledTaskQueue();
  
  	//先尝试拉一个。要是一个都没有就不进行了。
    Runnable task = pollTask();
    if (task == null) {
        return false;
    }
		
  	//再执行，再拉。
    for (;;) {
        try {
            task.run();
        } catch (Throwable t) {
            logger.warn("A task raised an exception.", t);
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            return true;
        }
    }
}
```

##### 处理io processSelectedKeys

```java
    private void processSelectedKeys() {
        if (selectedKeys != null) {
          	//有selectedKeys，就直接flip可读之后，逐条执行。
          	//没有连接的时候，这里是size=0，而不是null。
          	//有连接进来，创建完channel之后，就会有新的事件。
            processSelectedKeysOptimized(selectedKeys.flip());
        } else {
          	//是null，就现获取selectedKeys
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }
```

> 当有连接进来时。

```java
private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
    for (int i = 0;; i ++) {
        final SelectionKey k = selectedKeys[i];
        if (k == null) {
            break;
        }
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys[i] = null;//相当于help gc
				
      	//这里就是新建的NioServerSocketChannel实例
        final Object a = k.attachment();

      	//进入处理逻辑
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            for (;;) {
                if (selectedKeys[i] == null) {
                    break;
                }
                selectedKeys[i] = null;
                i++;
            }

            selectAgain();
            // Need to flip the optimized selectedKeys to get the right reference to the array
            // and reset the index to -1 which will then set to 0 on the for loop
            // to start over again.
            //
            // See https://github.com/netty/netty/issues/1523
            selectedKeys = this.selectedKeys.flip();
            i = -1;
        }
    }
}
```
> debug一下
>
> ![image-20200916145342606](https://gitee.com/wangigor/typora-images/raw/master/netty-processSelectedKey-debug.png)

```java
    private static void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            // key不可用，直接关闭通道。
            unsafe.close(unsafe.voidPromise());
            return;
        }
				
        try {
          	//查看当前通道事件
          	//    public static final int OP_READ = 1 << 0; 读 1
    				//		public static final int OP_WRITE = 1 << 2; 写 4
    				//		public static final int OP_CONNECT = 1 << 3; 连接 8
    				//		public static final int OP_ACCEPT = 1 << 4; 接受 16
            int readyOps = k.readyOps();
            //读 或者 接收
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
                if (!ch.isOpen()) {
                    return;
                }
            }
          	//写
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                ch.unsafe().forceFlush();
            }
          	//连接
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```

### ServerBootstrap、Bootstrap

> ServerBootstrap和Bootstrap作为服务端和客户端的启动类，都继承自AbstractBootstrap。
>
> 都进行了group、channel、childHandler/handler、option、connect/bind、sync、channelFuture.channel().closeFuture().sync()的操作。
>
> **建造者模式。**

AbstractBootstrap

```java
    //对于客户端，这个是消息发送接收的线程池
		//对于服务端，这个是处理客户端连接请求的线程池
		private volatile EventLoopGroup group;
		//channel工厂，通过传入的channel类型反射获取工厂
    private volatile ChannelFactory<? extends C> channelFactory;
		//socket地址
    private volatile SocketAddress localAddress;
		//channel的配置信息
    private final Map<ChannelOption<?>, Object> options = new LinkedHashMap<ChannelOption<?>, Object>();
    private final Map<AttributeKey<?>, Object> attrs = new LinkedHashMap<AttributeKey<?>, Object>();
		//io事件处理器
    private volatile ChannelHandler handler;
```

ServerBootstrap

私有成员

```java
    //都是对于workerGroup。
		//因为服务端比客户端多一套loopGroup用于处理「业务」，父类的处理「连接请求」
		//所以也就能看到在demo的使用上，handler服务端的设置的是childHandler。
		private final Map<ChannelOption<?>, Object> childOptions = new LinkedHashMap<ChannelOption<?>, Object>();
    private final Map<AttributeKey<?>, Object> childAttrs = new LinkedHashMap<AttributeKey<?>, Object>();
    private volatile EventLoopGroup childGroup;
    private volatile ChannelHandler childHandler;
```

Bootstrap

私有成员

```java
		//名称解析器组合socket地址
		private static final NameResolverGroup<?> DEFAULT_RESOLVER = DefaultNameResolverGroup.INSTANCE;
    private volatile NameResolverGroup<SocketAddress> resolver = (NameResolverGroup<SocketAddress>) DEFAULT_RESOLVER;
    private volatile SocketAddress remoteAddress;
```

- group设置NioEventLoopGroup

> ServerBootstrap有差异，重载了父类的group

**super.group**

```java
public B group(EventLoopGroup group) {
  	//非空检验
    if (group == null) {
        throw new NullPointerException("group");
    }
    if (this.group != null) {
        throw new IllegalStateException("group set already");
    }
  	//填上属性
    this.group = group;
    return (B) this;
}
```

ServerBootstrap重载方法

```java
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
  	//处理客户端连接请求的放在父类「acceptor」
    super.group(parentGroup);
    if (childGroup == null) {
        throw new NullPointerException("childGroup");
    }
    if (this.childGroup != null) {
        throw new IllegalStateException("childGroup set already");
    }
  	//处理业务的放在子类
    this.childGroup = childGroup;
    return this;
}
```

- channel

> channel方法一样，都是通过传进来的channel类型，反射得到对应的channel工厂。

```java
//获得channel工厂。
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new ReflectiveChannelFactory<C>(channelClass));
}

//放置属性
public B channelFactory(ChannelFactory<? extends C> channelFactory) {
    if (channelFactory == null) {
        throw new NullPointerException("channelFactory");
    }
    if (this.channelFactory != null) {
        throw new IllegalStateException("channelFactory set already");
    }

    this.channelFactory = channelFactory;
    return (B) this;
}
```

ReflectiveChannelFactory这个工厂很简单

```java
public class ReflectiveChannelFactory<T extends Channel> implements ChannelFactory<T> {
		
  	//就是保存了class，通过newInstance创建实例
    private final Class<? extends T> clazz;

    public ReflectiveChannelFactory(Class<? extends T> clazz) {
        if (clazz == null) {
            throw new NullPointerException("clazz");
        }
        this.clazz = clazz;
    }

    @Override
    public T newChannel() {
        try {
          	//创建channel实例
            return clazz.newInstance();
        } catch (Throwable t) {
            throw new ChannelException("Unable to create Channel from class " + clazz, t);
        }
    }

    @Override
    public String toString() {
        return StringUtil.simpleClassName(clazz) + ".class";
    }
}
```

- handler

> channelHandler用于处理我们的业务逻辑，基于Pipeline的自定义handler机制
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/channelHandler.jpg)

```java
//通道初始化的模板类
//这个类下面会进行详细展开
@Sharable
public abstract class ChannelInitializer<C extends Channel> extends ChannelHandlerAdapter {

  	//供子类实现的初始化通道方法
  	//添加channelHandler
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    @SuppressWarnings("unchecked")
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        ChannelPipeline pipeline = ctx.pipeline();
        boolean success = false;
        try {
            initChannel((C) ctx.channel());
            pipeline.remove(this);
            ctx.fireChannelRegistered();
            success = true;
        } catch (Throwable t) {
            logger.warn("Failed to initialize a channel. Closing: " + ctx.channel(), t);
        } finally {
            if (pipeline.context(this) != null) {
                pipeline.remove(this);
            }
            if (!success) {
                ctx.close();
            }
        }
    }
}
```

> 通道pipeline初始化的时候，会初始化固定的**head**和**tail**节点。
>
> ```java
> DefaultChannelPipeline(AbstractChannel channel) {
>     if (channel == null) {
>         throw new NullPointerException("channel");
>     }
>     this.channel = channel;
> 
>     tail = new TailContext(this);
>     head = new HeadContext(this);
> 
>     head.next = tail;
>     tail.prev = head;
> }
> ```
>
> 中间可以初始化自己的ChannelHandler，组成**双向链表**。

> TailContext提供了兜底的异常捕获、信息处理等方案。用户线程是从tail开始操作。

```java
    static final class TailContext extends AbstractChannelHandlerContext implements ChannelHandler {
        private static final int SKIP_FLAGS = skipFlags0(TailContext.class);
        private static final String TAIL_NAME = generateName0(TailContext.class);

        TailContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, TAIL_NAME, SKIP_FLAGS);
        }

        @Override
        public ChannelHandler handler() {
            return this;
        }

        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelUnregistered(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelInactive(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception { }

        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception { }

      
      	//提供默认异常处理
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            logger.warn(
                    "An exceptionCaught() event was fired, and it reached at the tail of the pipeline. " +
                            "It usually means the last handler in the pipeline did not handle the exception.", cause);
        }
				
      	//默认的通道未读信息释放
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            try {
                logger.debug(
                        "Discarded inbound message {} that reached at the tail of the pipeline. " +
                                "Please check your pipeline configuration.", msg);
            } finally {
                ReferenceCountUtil.release(msg);
            }
        }

        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception { }

        @Skip
        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception { }

        @Skip
        @Override
        public void handlerRemoved(ChannelHandlerContext ctx) throws Exception { }

      
      	//交给pipeline操作
      
        @Skip
        @Override
        public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
                throws Exception {
            ctx.bind(localAddress, promise);
        }

        @Skip
        @Override
        public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress,
                            SocketAddress localAddress, ChannelPromise promise) throws Exception {
            ctx.connect(remoteAddress, localAddress, promise);
        }

        @Skip
        @Override
        public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            ctx.disconnect(promise);
        }

        @Skip
        @Override
        public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            ctx.close(promise);
        }

        @Skip
        @Override
        public void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            ctx.deregister(promise);
        }

        @Skip
        @Override
        public void read(ChannelHandlerContext ctx) throws Exception {
            ctx.read();
        }

        @Skip
        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            ctx.write(msg, promise);
        }

        @Skip
        @Override
        public void flush(ChannelHandlerContext ctx) throws Exception {
            ctx.flush();
        }
    }
```



> head节点，提供了netty底层操作事件触发、unsafe操作等。head更「远离」用户线程

```java
    static final class HeadContext extends AbstractChannelHandlerContext implements ChannelHandler {
        private static final int SKIP_FLAGS = skipFlags0(HeadContext.class);
        private static final String HEAD_NAME = generateName0(HeadContext.class);

        private final Unsafe unsafe;

        HeadContext(DefaultChannelPipeline pipeline) {
            super(pipeline, null, HEAD_NAME, SKIP_FLAGS);
            unsafe = pipeline.channel().unsafe();
        }

        @Override
        public ChannelHandler handler() {
            return this;
        }
      
      	//底层操作相关

        @Override
        public void bind(
                ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
                throws Exception {
            unsafe.bind(localAddress, promise);
        }

        @Override
        public void connect(
                ChannelHandlerContext ctx,
                SocketAddress remoteAddress, SocketAddress localAddress,
                ChannelPromise promise) throws Exception {
            unsafe.connect(remoteAddress, localAddress, promise);
        }

        @Override
        public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            unsafe.disconnect(promise);
        }

        @Override
        public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
            unsafe.close(promise);
        }

        @Override
        public void deregister(ChannelHandlerContext ctx, final ChannelPromise promise) throws Exception {
            assert !((PausableEventExecutor) ctx.channel().eventLoop()).isAcceptingNewTasks();

            // submit deregistration task
            ctx.channel().eventLoop().unwrap().execute(new OneTimeTask() {
                @Override
                public void run() {
                    unsafe.deregister(promise);
                }
            });
        }

        @Override
        public void read(ChannelHandlerContext ctx) {
            unsafe.beginRead();
        }

        @Override
        public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
            unsafe.write(msg, promise);
        }

        @Override
        public void flush(ChannelHandlerContext ctx) throws Exception {
            unsafe.flush();
        }

        @Skip
        @Override
        public void handlerAdded(ChannelHandlerContext ctx) throws Exception { }

        @Skip
        @Override
        public void handlerRemoved(ChannelHandlerContext ctx) throws Exception { }

      
      	//事件触发相关
        @Skip
        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
            ctx.fireExceptionCaught(cause);
        }

        @Skip
        @Override
        public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelRegistered();
        }

        @Skip
        @Override
        public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelUnregistered();
        }

        @Skip
        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelActive();
        }

        @Skip
        @Override
        public void channelInactive(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelInactive();
        }

        @Skip
        @Override
        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
            ctx.fireChannelRead(msg);
        }

        @Skip
        @Override
        public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelReadComplete();
        }

        @Skip
        @Override
        public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
            ctx.fireUserEventTriggered(evt);
        }

        @Skip
        @Override
        public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelWritabilityChanged();
        }
    }
```

> 那下面看一下用户线程如何进行添加自定的handler。
>
> 要从demo的匿名内部类ChannelInitializer添加开始看。
>
> ```java
> .handler(new ChannelInitializer<SocketChannel>() {
> 	@Override
> 	protected void initChannel(SocketChannel ch) throws Exception {
> 		ch.pipeline().addLast(new SimpleNettyClientHandler());
> 	}
> });
> ```
> 这里只是新建匿名内部类放在handler中。
>
> 在channel创建的时候，会创建一个新的pipeline与channel对应。
>
> ```java
> protected AbstractChannel(Channel parent) {
> 		this.parent = parent;
> 		id = DefaultChannelId.newInstance();
> 		unsafe = newUnsafe();
> 		pipeline = new DefaultChannelPipeline(this);//一个channel对应一个pipeline
> }
> ```
> channel初始化的时候，会把handler逐一添加到pipeline中。
>
> ```java
> ChannelPipeline p = channel.pipeline();
> if (handler() != null) {
> p.addLast(handler());
> }
> ```
>
>
> ```java
> @Override
> public ChannelPipeline addLast(ChannelHandler... handlers) {
> 	//每个handler添加都会进这里。
> 	//也可以添加多个。
>  return addLast((ChannelHandlerInvoker) null, handlers);
> }
> 
> @Override
> public ChannelPipeline addLast(EventExecutorGroup group, ChannelHandler... handlers) {
>  if (handlers == null) {
>      throw new NullPointerException("handlers");
>  }
> 		//简单循环。逐条添加。
>  for (ChannelHandler h: handlers) {
>      if (h == null) {
>          break;
>      }
>      addLast(group, generateName(h), h);
>  }
> 
>  return this;
> }
> ```
>
> 具体添加
>
> ```java
> @Override
> public ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
> 	//锁pipeline
>  synchronized (this) {
>    	//名称校验 默认是StringUtil.simpleClassName(className) + "#0"
>    	//会放入nameCaches的WeakHashMap中
>    	//重名会报 new IllegalArgumentException("Duplicate handler name: " + name);异常
>      name = filterName(name, handler);
>    	//把每一个handler用包装成context
>    	//group一般是空
>      addLast0(name, new DefaultChannelHandlerContext(this, findInvoker(group), name, handler));
>  }
>  return this;
> }
> ```
>
> addLast0
>
> ```java
> private void addLast0(final String name, AbstractChannelHandlerContext newCtx) {
>  //检查重复项
>  //当前handler的added状态是否为true 且 没有被@Sharable标注
>  checkMultiplicity(newCtx);
> 
>  //插入tail之前
>  //双向链表插入
>  AbstractChannelHandlerContext prev = tail.prev;
>  newCtx.prev = prev;
>  newCtx.next = tail;
>  prev.next = newCtx;
>  tail.prev = newCtx;
> 
>  name2ctx.put(name, newCtx);
> 
>  //执行添加完成的回调
>  callHandlerAdded(newCtx);
> }
> ```
> ```java
>     private void callHandlerAdded(final AbstractChannelHandlerContext ctx) {
>       
>         if ((ctx.skipFlags & AbstractChannelHandlerContext.MASK_HANDLER_ADDED) != 0) {
>             return;
>         }
> 				
>       	//这里还是查看当前线程是否是线程池线程。
>       	//否则放入线程池操作
>         if (ctx.channel().isRegistered() && !ctx.executor().inEventLoop()) {
>             ctx.executor().execute(new Runnable() {
>                 @Override
>                 public void run() {
>                     callHandlerAdded0(ctx);
>                 }
>             });
>             return;
>         }
>       	//就是添加完成的通知
>         callHandlerAdded0(ctx);
>     }
> ```
>
> 在channel初始化的register阶段，会调用channelHandler的channelRegistered方法。
>
> ```java
> //ChannelInitializer重写了这个方法
> @Override
> @SuppressWarnings("unchecked")
> public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
>     ChannelPipeline pipeline = ctx.pipeline();
>     boolean success = false;
>     try {
>       	//把自己实现的子类initChannel添加进来的自定义handler，添加到channel中。
>         initChannel((C) ctx.channel());
>       	//删除自己 
>         pipeline.remove(this);
>       	//触发每一个handler的通道注册成功操作
>         ctx.fireChannelRegistered();
>         success = true;
>     } catch (Throwable t) {
>         logger.warn("Failed to initialize a channel. Closing: " + ctx.channel(), t);
>     } finally {
>         if (pipeline.context(this) != null) {
>             pipeline.remove(this);
>         }
>         if (!success) {
>             ctx.close();
>         }
>     }
> }
> ```

handler删除可以做一些事情，比如**权限认证**。

比如

```java
//在channel建立之初，first添加一个鉴权的handler。
//客户端建立连接完，第一步先发送password，进行校验。
//校验成功，就删除这个handler，当前channel安全。
//不成功，关闭。
public class AuthHandler extends ChannelHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object password) throws Exception {
        if (pass((ByteBuf) password)) {
            ctx.pipeline().remove(this);
        } else {
            ctx.close();
        }
    }

    private boolean pass(ByteBuf password) {
        //业务逻辑
        return true;
    }
}
```


- option

> 通道可选「TCP参数」属性
>
> ```java
>     public static final ChannelOption<ByteBufAllocator> ALLOCATOR = valueOf("ALLOCATOR");
>     public static final ChannelOption<RecvByteBufAllocator> RCVBUF_ALLOCATOR = valueOf("RCVBUF_ALLOCATOR");
>     public static final ChannelOption<MessageSizeEstimator> MESSAGE_SIZE_ESTIMATOR = valueOf("MESSAGE_SIZE_ESTIMATOR");
> 
>     public static final ChannelOption<Integer> CONNECT_TIMEOUT_MILLIS = valueOf("CONNECT_TIMEOUT_MILLIS");
>     public static final ChannelOption<Integer> MAX_MESSAGES_PER_READ = valueOf("MAX_MESSAGES_PER_READ");
>     public static final ChannelOption<Integer> WRITE_SPIN_COUNT = valueOf("WRITE_SPIN_COUNT");
>     public static final ChannelOption<Integer> WRITE_BUFFER_HIGH_WATER_MARK = valueOf("WRITE_BUFFER_HIGH_WATER_MARK");
>     public static final ChannelOption<Integer> WRITE_BUFFER_LOW_WATER_MARK = valueOf("WRITE_BUFFER_LOW_WATER_MARK");
> 
>     public static final ChannelOption<Boolean> ALLOW_HALF_CLOSURE = valueOf("ALLOW_HALF_CLOSURE");
>     public static final ChannelOption<Boolean> AUTO_READ = valueOf("AUTO_READ");
> 
>     public static final ChannelOption<Boolean> SO_BROADCAST = valueOf("SO_BROADCAST");
>     public static final ChannelOption<Boolean> SO_KEEPALIVE = valueOf("SO_KEEPALIVE");
>     public static final ChannelOption<Integer> SO_SNDBUF = valueOf("SO_SNDBUF");
>     public static final ChannelOption<Integer> SO_RCVBUF = valueOf("SO_RCVBUF");
>     public static final ChannelOption<Boolean> SO_REUSEADDR = valueOf("SO_REUSEADDR");
>     public static final ChannelOption<Integer> SO_LINGER = valueOf("SO_LINGER");
>     public static final ChannelOption<Integer> SO_BACKLOG = valueOf("SO_BACKLOG");
>     public static final ChannelOption<Integer> SO_TIMEOUT = valueOf("SO_TIMEOUT");
> 
>     public static final ChannelOption<Integer> IP_TOS = valueOf("IP_TOS");
>     public static final ChannelOption<InetAddress> IP_MULTICAST_ADDR = valueOf("IP_MULTICAST_ADDR");
>     public static final ChannelOption<NetworkInterface> IP_MULTICAST_IF = valueOf("IP_MULTICAST_IF");
>     public static final ChannelOption<Integer> IP_MULTICAST_TTL = valueOf("IP_MULTICAST_TTL");
>     public static final ChannelOption<Boolean> IP_MULTICAST_LOOP_DISABLED = valueOf("IP_MULTICAST_LOOP_DISABLED");
> 
>     public static final ChannelOption<Boolean> TCP_NODELAY = valueOf("TCP_NODELAY");
> ```
>
> 

```java
    public <T> B option(ChannelOption<T> option, T value) {
        if (option == null) {
            throw new NullPointerException("option");
        }
        if (value == null) { //删除属性
            synchronized (options) {
                options.remove(option);
            }
        } else {	//添加属性
            synchronized (options) {
                options.put(option, value);
            }
        }
        return (B) this;
    }
```



- bind

> 
>
> 「**服务端**」在bind步骤之前，我们已经做好了，设置线程组、指定通道类型、自定义通道初始化器、指定通道参数等操作。

```java
    //绑定本地端口
		public ChannelFuture bind(int inetPort) {
        return bind(new InetSocketAddress(inetPort));
    }
		//重载方法
    public ChannelFuture bind(SocketAddress localAddress) {
      	//参数校验
        validate();
        if (localAddress == null) {
            throw new NullPointerException("localAddress");
        }
        return doBind(localAddress);
    }

    private ChannelFuture doBind(final SocketAddress localAddress) {
      	
      	//初始化和注册
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
        if (regFuture.cause() != null) {
            return regFuture;
        }
				//初始化和注册成功
        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise); //进行doBind0操作
            return promise;
        } else {
          	//否则对上一步的promise添加成功监听器
          	//监听到成功后，再doBind0
          	//doBind0才是NioEventLoop线程的启动
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.executor = channel.eventLoop();
                    }
                    doBind0(regFuture, channel, localAddress, promise);
                }
            });
            return promise;
        }
    }
```

```java
    //初始化和注册
		final ChannelFuture initAndRegister() {
      	//使用通道工厂，初始化一个channel「NioServerSocketChannel」
      	//newInstance  
        final Channel channel = channelFactory().newChannel();
        try {
          	//初始化通道
            init(channel);
        } catch (Throwable t) {
          	//失败，强制退出
            channel.unsafe().closeForcibly();
          	//并手动调用GlobalEventExecutor全局事件处理器告知失败。
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
        }
				
      	//注册到bossGroup中
        ChannelFuture regFuture = group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```

> 服务端的通道初始化

```java
    void init(Channel channel) throws Exception {
      	//设置option
        final Map<ChannelOption<?>, Object> options = options();
        synchronized (options) {
            channel.config().setOptions(options);
        }
			
      	//设置attr
        final Map<AttributeKey<?>, Object> attrs = attrs();
        synchronized (attrs) {
            for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
                @SuppressWarnings("unchecked")
                AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
                channel.attr(key).set(e.getValue());
            }
        }
				
      	//设置服务端bossworker的通道初始化器
      	//demo中没有
        ChannelPipeline p = channel.pipeline();
        if (handler() != null) {
            p.addLast(handler());
        }

      	//把跟workerGroup相关的child「属性」组成成ServerBootstrapAcceptor  
      	//这个ServerBootstrapAcceptor是为了接受通道连接/转换用的，bossGroup通过这个handler,把注册上来的channel注册到workerGroup上。
      	//放在在pipeline中
        final EventLoopGroup currentChildGroup = childGroup;
        final ChannelHandler currentChildHandler = childHandler;
        final Entry<ChannelOption<?>, Object>[] currentChildOptions;
        final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
        synchronized (childOptions) {
            currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
        }
        synchronized (childAttrs) {
            currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
        }

        p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(Channel ch) throws Exception {
                ch.pipeline().addLast(new ServerBootstrapAcceptor(
                        currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
```

> 通道注册
>
> 通道注册，netty使用了一个隔离接口Unsafe「invoker、localAddress、remoteAddress、closeForcibly、register、deregister、voidPromise」，对业务系统比如read、write，进行隔离。

```java
//调用NioEventLoopGroup「boss」的register方法
ChannelFuture regFuture = group().register(channel);
```

```java
		//MultithreadEventLoopGroup
		@Override
    public ChannelFuture register(Channel channel) {
      	//next()就是通过loopGroup的chooser「二进制选择器」,获取到一个NioEventLoop「线程」
        return next().register(channel);
    }
```

```java
    //super SingleThreadEventLoop
		@Override
    public ChannelFuture register(Channel channel) {
      	//生成了一个默认的DefaultChannelPromise用于返回ChannelFuture
        return register(channel, new DefaultChannelPromise(channel, this));
    }
    @Override
    public ChannelFuture register(final Channel channel, final ChannelPromise promise) {
      	//参数检查
        if (channel == null) {
            throw new NullPointerException("channel");
        }
        if (promise == null) {
            throw new NullPointerException("promise");
        }
				
      	//调用了通道隔离的register方法
      	//这里的channel是server指定的NioServerSocketChannel，他在初始化的时候，会初始化一个NioMessageUnsafe座位unsafe，进行channel操作的隔离。
        channel.unsafe().register(this, promise);
        return promise;
    }
```

```java
				//AbstractChannel
				@Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
          	//参数检查
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (promise == null) {
                throw new NullPointerException("promise");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }

            //对eventLoop进行包装
            if (AbstractChannel.this.eventLoop == null) {
                AbstractChannel.this.eventLoop = new PausableChannelEventLoop(eventLoop);//包装后的eventLoop
            } else {
                AbstractChannel.this.eventLoop.unwrapped = eventLoop; //NioEventLoop
            }
						
          	//判断Thread.currentThread当前线程是否是NioEventLoop的线程池线程
            if (eventLoop.inEventLoop()) {
              	//如果是，就直接当前线程执行register0操作
                register0(promise);
            } else {
              	//否则提交线程池操作
              	//因为我们是从业务线程进行的bind操作，所以肯定走这里。
                try {
                    eventLoop.execute(new OneTimeTask() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
```

```java
private void register0(ChannelPromise promise) {
    try {
        //检查这个promise，如果设置为不可取消失败了，或者不是打开的，直接返回
        if (!promise.setUncancellable() || !ensureOpen(promise)) {
            return;
        }
      	//如果channel没有注册过，就是true，反之false
        boolean firstRegistration = neverRegistered;
        doRegister(); //注册
        neverRegistered = false; //设置注册成功
        registered = true;//设置已注册
        eventLoop.acceptNewTasks();//设置接收新任务标识isAcceptingNewTasks为true
        safeSetSuccess(promise); //promise设置为成功
        pipeline.fireChannelRegistered();//触发通道注册完成事件
        // Only fire a channelActive if the channel has never been registered. This prevents firing
        // multiple channel actives if the channel is deregistered and re-registered.
        if (firstRegistration && isActive()) {
            pipeline.fireChannelActive(); //触发通道可用事件
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

```java
    protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            try {
              	//channel注册到selector上。
              	//attachment对象就是NioServerSocketChannel实例，当事件发生时，需要对事件进行处理，就会拿到这个NioServerSocketChannel
                selectionKey = javaChannel().register(((NioEventLoop) eventLoop().unwrap()).selector, 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    //由于尚未调用Select.select（..）操作，因此强制选择器立即选择，因为“取消的” SelectionKey可能仍被缓存并且未被删除。
                    ((NioEventLoop) eventLoop().unwrap()).selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
    }
```

> 最后，看一下doBind0。

```java
    private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {

        // 在触发channelRegistered（）之前调用此方法。 使用户处理程序有机会在其channelRegistered（）实现中设置管道。
      	// 之前出发了registry0，启动了NioEventLoop线程
      	// 向通道所在的NioEventLoop提交任务「触发NioEventLoop开始工作」
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                  	//通道绑定端口，并绑定promise对象
                  	//添加通道关闭监听
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                  	//这一步上在上游注册失败时，对当前promise置为失败。
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }
```

- connect
- sync
- channelFuture.channel().closeFuture().sync()





### 通道事件传播

#### 出入库context判断

todo



#### 接收连接

> 对于server的接收连接操作，server的bossGroup负责accept接收连接，然后注册到「child」workerGroup上。
>
> read事件和accept事件是对于NioEventLoop来说是同一个处理流程。
>
> 那我们就从NioEventLoop的processSelectedKey方法处理读事件「unsafe.read()」开始。
>
> ![image-20200916162038069](https://gitee.com/wangigor/typora-images/raw/master/netty-accept-unsafe-read-debug.png)

> 注意。这里的channel是**NioServerSocketChannel**。

```java
        // unsafe NioMessageUnsafe
				@Override
        public void read() {
          
          	//配置检查
            assert eventLoop().inEventLoop();
            final ChannelConfig config = config();
            if (!config.isAutoRead() && !isReadPending()) {
                // ChannelConfig.setAutoRead(false) was called in the meantime
                removeReadOp();
                return;
            }
						
          	//每次读取循环要读取的最大消息数
            final int maxMessagesPerRead = config.getMaxMessagesPerRead();
          	//NioServerSocketChannel的pipeline
          	// Head -> ServerBootstrapAcceptor -> Tail
            final ChannelPipeline pipeline = pipeline();
            boolean closed = false;
            Throwable exception = null;
            try {
                try {
                  	//循环读取，读到读不出来
                    for (;;) {
                      	//readBuf new ArrayList<Object>()
                        int localRead = doReadMessages(readBuf);
                        if (localRead == 0) {
                            break;
                        }
                        if (localRead < 0) {
                            closed = true;
                            break;
                        }

                        // stop reading and remove op
                        if (!config.isAutoRead()) {
                            break;
                        }

                        if (readBuf.size() >= maxMessagesPerRead) {
                            break;
                        }
                    }
                } catch (Throwable t) {
                    exception = t;
                }
              
                setReadPending(false);
                int size = readBuf.size();
              	//触发NioServerSocketChannel的pipeline的通道读取事件
              	//这里入参是组装的NioSocketChannel
              	//pipeline 先经过Head节点，主要处理节点为ServerBootstrapAcceptor
                for (int i = 0; i < size; i ++) {
                    pipeline.fireChannelRead(readBuf.get(i));
                }
							
                readBuf.clear();
                pipeline.fireChannelReadComplete();
              	//触发读取完成事件

              	//异常处理。
                if (exception != null) {
                    if (exception instanceof IOException && !(exception instanceof PortUnreachableException)) {
                        // ServerChannel should not be closed even on IOException because it can often continue
                        // accepting incoming connections. (e.g. too many open files)
                        closed = !(AbstractNioMessageChannel.this instanceof ServerChannel);
                    }
										//触发异常捕获事件
                    pipeline.fireExceptionCaught(exception);
                }

                if (closed) {
                    if (isOpen()) {
                        close(voidPromise());
                    }
                }
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!config.isAutoRead() && !isReadPending()) {
                    removeReadOp();
                }
            }
        }
    }
```



##### doReadMessages

```java
		//NioServerSocketChannel
		protected int doReadMessages(List<Object> buf) throws Exception {
      	//javaChannel() 获取通道，ServerSocketChannelImpl
      	//accept() serverSocketChannel.accept();
        SocketChannel ch = javaChannel().accept();

        try {
            if (ch != null) {
              	//组装成NioSocketChannel放入readBuf集合里。
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {
					//略            
        }

        return 0;
    }
```

##### pipeline.fireChannelRead

```java
    //向头结点触发通道读取事件
		//msg 不是读取信息，而是通道
		public ChannelPipeline fireChannelRead(Object msg) {
        head.fireChannelRead(msg);
        return this;
    }
```

```java
    public ChannelHandlerContext fireChannelRead(Object msg) {
      	//按照next一级一级往下找，查找类型为Inbound的context
      	//在服务端的NioServerSocketChannel上，只有一个ServerBootstrapAcceptor
        AbstractChannelHandlerContext next = findContextInbound();
        ReferenceCountUtil.touch(msg, next);//这一步是为了「引用计数」用的，暂时不考虑
        invokedNextChannelRead = true;
      	//下一个节点，执行『读』操作
        next.invoker().invokeChannelRead(next, msg);
        return this;
    }
```

```java
    //就是从context剥离出handler，然后执行。
		//因为前面进行了PausableChannelEventExecutor包裹
		//PausableChannelEventExecutor
		public void invokeChannelRead(ChannelHandlerContext ctx, Object msg) {
        unwrapInvoker().invokeChannelRead(ctx, msg);
    }
		//DefaultChannelHandlerInvoker
		public void invokeChannelRead(final ChannelHandlerContext ctx, final Object msg) {
				//略

        if (executor.inEventLoop()) {
            invokeChannelReadNow(ctx, msg);//线程池线程，直接执行
        } else {
            safeExecuteInbound(new OneTimeTask() {
                @Override
                public void run() {
                    invokeChannelReadNow(ctx, msg);
                }
            }, msg);
        }
    }
		//DefaultChannelHandlerInvoker
		public static void invokeChannelReadNow(final ChannelHandlerContext ctx, final Object msg) {
        try {
            ((AbstractChannelHandlerContext) ctx).invokedThisChannelRead = true;
          	//ServerBootstrapAcceptor 读
            ctx.handler().channelRead(ctx, msg);
        } catch (Throwable t) {
            notifyHandlerException(ctx, t);
        }
    }
```

##### ServerBootstrapAcceptor

> 这是作为从boss到worker的「传递类」
>
> 包含了服务端启动时的worker相关的group、handler、options、attrs.
>
> 继承了ChannelHandlerAdapter，只重写了读、关闭、异常三个事件处理方法，是Inbound类型

```java
private static class ServerBootstrapAcceptor extends ChannelHandlerAdapter {

    private final EventLoopGroup childGroup;
    private final ChannelHandler childHandler;
    private final Entry<ChannelOption<?>, Object>[] childOptions;
    private final Entry<AttributeKey<?>, Object>[] childAttrs;

    ServerBootstrapAcceptor(
            EventLoopGroup childGroup, ChannelHandler childHandler,
            Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
        this.childGroup = childGroup;
        this.childHandler = childHandler;
        this.childOptions = childOptions;
        this.childAttrs = childAttrs;
    }

    //连接处理
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
      	//为新连接NioSocketLoop添加业务处理handler
        final Channel child = (Channel) msg;
        child.pipeline().addLast(childHandler);
				//设置options和attrs
      	//略

        try {
          	//把新连接NioSocketLoop注册到workerGroup
          	//并添加一个close的监听
          	//就到了workerGroup中选一个loop与他绑定，然后触发对当前通道的通道监听。
            childGroup.register(child).addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (!future.isSuccess()) {
                        forceClose(child, future.cause());
                    }
                }
            });
        } catch (Throwable t) {
            forceClose(child, t);
        }
    }
		//关闭处理
    private static void forceClose(Channel child, Throwable t) {
        child.unsafe().closeForcibly();
        logger.warn("Failed to register an accepted channel: " + child, t);
    }

    //异常捕获处理
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        final ChannelConfig config = ctx.channel().config();
        if (config.isAutoRead()) {
            config.setAutoRead(false);
            ctx.channel().eventLoop().schedule(new Runnable() {
                @Override
                public void run() {
                    config.setAutoRead(true);
                }
            }, 1, TimeUnit.SECONDS);
        }
        ctx.fireExceptionCaught(cause);
    }
}
```

#### 连接

#### 读

#### 写









