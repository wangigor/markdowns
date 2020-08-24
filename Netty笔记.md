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













