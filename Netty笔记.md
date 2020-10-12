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
          	//异步模式为FxieIFO「先进先出」
            executor = newDefaultExecutorService(nEventExecutors);
            shutdownExecutor = true;
        }
				//初始化线程数
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

> connect是客户端连接服务端发起的操作。
>
> ```java
> bootstrap.connect("localhost", 8080).sync();
> ```
>
> ```java
> 		//Bootstrap
>     public ChannelFuture connect(String inetHost, int inetPort) {
>         return connect(InetSocketAddress.createUnresolved(inetHost, inetPort));
>     }
>     public ChannelFuture connect(SocketAddress remoteAddress) {
>       	//都是些非空判断
>         if (remoteAddress == null) {
>             throw new NullPointerException("remoteAddress");
>         }
>         validate();
>       
>         return doResolveAndConnect(remoteAddress, localAddress());
>     }
> ```

```java
    private ChannelFuture doResolveAndConnect(SocketAddress remoteAddress, final SocketAddress localAddress) {
      	//新建channel，并注册到NioEventLoopGroup上，与一个NioEventLoop对应。
        final ChannelFuture regFuture = initAndRegister();
        if (regFuture.cause() != null) {
            return regFuture;
        }

        final Channel channel = regFuture.channel();//通道NioSocketChannel
        final EventLoop eventLoop = channel.eventLoop();//NioEventLoop的包装 PausableChannelEventLoop
      	//SocketAddress「localhost:8080」的解析器
        final NameResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);

        if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {
            // 如果解析器解析不了 或者 已经解析过，直接进行连接后续操作。
            return doConnect(remoteAddress, localAddress, regFuture, channel.newPromise());
        }
				
      	//socketAddress解析
        final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);
        final Throwable resolveFailureCause = resolveFuture.cause();
      	//解析异常直接失败。
        if (resolveFailureCause != null) {
            channel.close();
            return channel.newFailedFuture(resolveFailureCause);
        }
				//解析完，进行连接后续操作。
      	//这里不是一个阻塞的等待。
        if (resolveFuture.isDone()) {
          	//result: InetSocketAddress「localhost/127.0.0.1:8080」
            return doConnect(resolveFuture.getNow(), localAddress, regFuture, channel.newPromise());
        }
				
      	//添加「解析完成」的监听器。用于后续连接。
        final ChannelPromise connectPromise = channel.newPromise();
        resolveFuture.addListener(new FutureListener<SocketAddress>() {
            @Override
            public void operationComplete(Future<SocketAddress> future) throws Exception {
                if (future.cause() != null) {
                    channel.close();
                    connectPromise.setFailure(future.cause());
                } else {
                    doConnect(future.getNow(), localAddress, regFuture, connectPromise);
                }
            }
        });

        return connectPromise;
    }
```

```java
    //doConnect先做了「通道初始化及注册」future的容错。
		//如果没有完成就添加监听等待。
		private static ChannelFuture doConnect(
            final SocketAddress remoteAddress, final SocketAddress localAddress,
            final ChannelFuture regFuture, final ChannelPromise connectPromise) {
        if (regFuture.isDone()) {
            doConnect0(remoteAddress, localAddress, regFuture, connectPromise);
        } else {
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    doConnect0(remoteAddress, localAddress, regFuture, connectPromise);
                }
            });
        }

        return connectPromise;
    }
```

```java
    private static void doConnect0(
            final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelFuture regFuture,
            final ChannelPromise connectPromise) {

        final Channel channel = connectPromise.channel();
      	//使用channel的eventLoop「NioEventLoop」执行连接任务
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                  	//连接
                    if (localAddress == null) {
                        channel.connect(remoteAddress, connectPromise);
                    } else {
                        channel.connect(remoteAddress, localAddress, connectPromise);
                    }
                  	//添加失败监听
                    connectPromise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                  	//通道创建及注册失败。
                    connectPromise.setFailure(regFuture.cause());
                }
            }
        });
    }
```

> 上面都是「数据准备」，下面开始连接。

```java
//AbstractChannel
@Override
public ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
  	//调用pipeline进行连接
    return pipeline.connect(remoteAddress, promise);
}
```

```java
		//DefaultChannelPipeline
		@Override
    public ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
      	//从tail节点开始执行「tail靠近用户」
        return tail.connect(remoteAddress, promise);
    }
```

```java
    //重载
		@Override
    public ChannelFuture connect(SocketAddress remoteAddress, ChannelPromise promise) {
        return connect(remoteAddress, null, promise);
    }
    @Override
    public ChannelFuture connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
      	//寻找outBound类型的HandlerContext
      	//没有么，就到Head了。
        AbstractChannelHandlerContext next = findContextOutbound();
      	//使用head执行连接操作
        next.invoker().invokeConnect(next, remoteAddress, localAddress, promise);
        return promise;
    }
```

```java
    @Override
    public void invokeConnect(
           ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
        unwrapInvoker().invokeConnect(ctx, remoteAddress, localAddress, promise);
    }
```



```java
//DefaultChannelHandlerInvoker
public void invokeConnect(
        final ChannelHandlerContext ctx,
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
  	//非空判断及通道关闭的校验
    if (remoteAddress == null) {
        throw new NullPointerException("remoteAddress");
    }
    if (!validatePromise(ctx, promise, false)) {
        // promise cancelled
        return;
    }

  	//NioEventLoop执行
    if (executor.inEventLoop()) {
        invokeConnectNow(ctx, remoteAddress, localAddress, promise);
    } else {
        safeExecuteOutbound(new OneTimeTask() {
            @Override
            public void run() {
                invokeConnectNow(ctx, remoteAddress, localAddress, promise);
            }
        }, promise);
    }
}
```

```java
//ChannelHandlerInvokerUtil
public static void invokeConnectNow(
        final ChannelHandlerContext ctx,
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    try {
      	//适配及异常捕获
        ctx.handler().connect(ctx, remoteAddress, localAddress, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
```

```java
				//DefaultChannelPipeline
				public void connect(
                ChannelHandlerContext ctx,
                SocketAddress remoteAddress, SocketAddress localAddress,
                ChannelPromise promise) throws Exception {
          	//使用与用户线程隔离的Unsafe进行通道操作
            unsafe.connect(remoteAddress, localAddress, promise);
        }
```

```java
        public final void connect(
                final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
            if (!promise.setUncancellable() || !ensureOpen(promise)) {
                return;
            }

            try {
                if (connectPromise != null) {
                    throw new IllegalStateException("connection attempt already made");
                }

                boolean wasActive = isActive();
              	//连接
                if (doConnect(remoteAddress, localAddress)) {
                  	//连接成功处理
                    fulfillConnectPromise(promise, wasActive);
                } else {
                  	//连接超时处理
                    connectPromise = promise;
                    requestedRemoteAddress = remoteAddress;

                    // Schedule connect timeout.
                    int connectTimeoutMillis = config().getConnectTimeoutMillis();
                    if (connectTimeoutMillis > 0) {
                        connectTimeoutFuture = eventLoop().schedule(new OneTimeTask() {
                            @Override
                            public void run() {
                                ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                                ConnectTimeoutException cause =
                                        new ConnectTimeoutException("connection timed out: " + remoteAddress);
                                if (connectPromise != null && connectPromise.tryFailure(cause)) {
                                    close(voidPromise());
                                }
                            }
                        }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
                    }

                    promise.addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            if (future.isCancelled()) {
                                if (connectTimeoutFuture != null) {
                                    connectTimeoutFuture.cancel(false);
                                }
                                connectPromise = null;
                                close(voidPromise());
                            }
                        }
                    });
                }
            } catch (Throwable t) {
                promise.tryFailure(annotateConnectException(t, remoteAddress));
                closeIfClosed();
            }
        }
```

```java
    protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
      	//通道绑定
        if (localAddress != null) {
            javaChannel().socket().bind(localAddress);
        }

        boolean success = false;
        try {
          	//连接
            boolean connected = javaChannel().connect(remoteAddress);
            if (!connected) {
              	//设置对连接事件感兴趣
                selectionKey().interestOps(SelectionKey.OP_CONNECT);
            }
            success = true;
            return connected;
        } finally {
            if (!success) {
                doClose();
            }
        }
    }
```

> connect就看到这里。

- sync

> Netty 默认使用DefaultPromise

```java
    public Promise<V> sync() throws InterruptedException {
        await();//阻塞等待
        rethrowIfFailed();//失败抛出异常
        return this;
    }
```



- channelFuture.channel().closeFuture().sync()

> 对Netty的close future进行同步等待。
>
> 否则，在bind().sync()或者connect().sync()操作的等待完成之后，程序会退出。



### 通道事件传播

#### context判断Inbound和Outbound

> Netty 4以及之前使用的的是ChannelHandler的两个子类ChannelOutboundHandlerAdapter和ChannelInboundHandlerAdapter来判断的。
>
> 现在这俩接口统一合并在了ChannelHandlerAdapter里。

> 在ChannelHandlerContext的统一抽象父类里，提供了统一的判断
>
> ```java
> private AbstractChannelHandlerContext findContextInbound() {
> AbstractChannelHandlerContext ctx = this;
> do {
>   ctx = ctx.next;
> } while ((ctx.skipFlags & MASKGROUP_INBOUND) == MASKGROUP_INBOUND);
> return ctx;
> }
> 
> private AbstractChannelHandlerContext findContextOutbound() {
> AbstractChannelHandlerContext ctx = this;
> do {
>   ctx = ctx.prev;
> } while ((ctx.skipFlags & MASKGROUP_OUTBOUND) == MASKGROUP_OUTBOUND);
> return ctx;
> }
> ```
> 也就是通过skipFlags来计算的。
>
> ```java
> static final int MASK_HANDLER_ADDED = 1;
> static final int MASK_HANDLER_REMOVED = 1 << 1;
> 
> private static final int MASK_EXCEPTION_CAUGHT = 1 << 2;
> private static final int MASK_CHANNEL_REGISTERED = 1 << 3;
> private static final int MASK_CHANNEL_UNREGISTERED = 1 << 4;
> private static final int MASK_CHANNEL_ACTIVE = 1 << 5;
> private static final int MASK_CHANNEL_INACTIVE = 1 << 6;
> private static final int MASK_CHANNEL_READ = 1 << 7;
> private static final int MASK_CHANNEL_READ_COMPLETE = 1 << 8;
> private static final int MASK_CHANNEL_WRITABILITY_CHANGED = 1 << 9;
> private static final int MASK_USER_EVENT_TRIGGERED = 1 << 10;
> 
> private static final int MASK_BIND = 1 << 11;
> private static final int MASK_CONNECT = 1 << 12;
> private static final int MASK_DISCONNECT = 1 << 13;
> private static final int MASK_CLOSE = 1 << 14;
> private static final int MASK_DEREGISTER = 1 << 15;
> private static final int MASK_READ = 1 << 16;
> private static final int MASK_WRITE = 1 << 17;
> private static final int MASK_FLUSH = 1 << 18;
> 
> private static final int MASKGROUP_INBOUND = MASK_EXCEPTION_CAUGHT |
>      MASK_CHANNEL_REGISTERED |
>      MASK_CHANNEL_UNREGISTERED |
>      MASK_CHANNEL_ACTIVE |
>      MASK_CHANNEL_INACTIVE |
>      MASK_CHANNEL_READ |
>      MASK_CHANNEL_READ_COMPLETE |
>      MASK_CHANNEL_WRITABILITY_CHANGED |
>      MASK_USER_EVENT_TRIGGERED;
> 
> private static final int MASKGROUP_OUTBOUND = MASK_BIND |
>      MASK_CONNECT |
>      MASK_DISCONNECT |
>      MASK_CLOSE |
>      MASK_DEREGISTER |
>      MASK_READ |
>      MASK_WRITE |
>      MASK_FLUSH;
> ```
> 就是通过handler重写了那些方法来判断的。
>
> - Inbound : 重写registered、unregistered、active、inactive、read、read_complete、writability_changed、user_event_triggered
> - Outbound : 重写connect、disconnect、close、deregister、read、write、flush

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

> Bootstrap的connect方法。

#### 读/写

> 也是一样，当事件发生时，通过pipeline，一个一个handler进行处理。



### 内存管理

> 在没读源码前，只是知道是使用堆外内存进行channel到handler的数据传输。那堆外内存呢，使用虚引用的继承类，进行回收操作。
>
> 但是堆外内存的开辟比堆内存分配慢十几倍。不知道作为优化狂魔的netty还做了这么多东西。

> ByteBuf随处可见「在还没有自定义编码解码器的时候」，它用于传输对象。
>
> 以读写为例：

- **读**

> NioEventLoop通过select得到的读事件 「NioEventLoop.processSelectedKey()」 -> 读ByteBuf -> 传递给handlerContext

![image-20200917171859874](https://gitee.com/wangigor/typora-images/raw/master/netty-read-byteBuf.png)

使用的是可自动回收的堆外内存。

- **写**

> 用户数据通过handlerContext -> 写ByteBuf的转换 -> 通道传输出去

![image-20200917172657423](https://gitee.com/wangigor/typora-images/raw/master/netty-write-byteBuf.png)

```java
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg;
        }
				
      	//即便是自建的ByteBuf，都要变成「堆外内存」
        return newDirectBuffer(buf);
    }

    if (msg instanceof FileRegion) {
        return msg;
    }

    throw new UnsupportedOperationException(
            "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```



#### jemalloc实现

> [jemalloc2006年论文](https://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf)
>
> [facebook的jemalloc实践](https://www.facebook.com/notes/facebook-engineering/scalable-memory-allocation-using-jemalloc/480222803919)
>
> 内存规格分档：
>
> <img src="https://gitee.com/wangigor/typora-images/raw/master/netty-内存规格分档.png" alt="img" style="zoom: 33%;" />
>
> - **tiny** 0~512B
> - **small** 512B~8K
> - **normal** 8K~16M
> - **huge** 16M以上
>
> 下面是需要进行内存管理的池化内存模型
>
> <img src="https://gitee.com/wangigor/typora-images/raw/master/PoolArea结构.png" alt="image-20200924150614560" style="zoom:150%;" />
>
> 内存在使用完，释放「release」时，不会真的释放而是加入cache中。
>
> ![image-20200927090908875](https://gitee.com/wangigor/typora-images/raw/master/PoolThreadCache结构.png)

##### 分配内存allocte

> ```java
> //入口
> byteBuf = allocHandle.allocate(allocator);
> //allocHandle
> //使用自适应接收ByteBuf分配器AdaptiveRecvByteBufAllocator.HandleImpl
> //对ByteBuf容量进行预测，默认采用1024读取，如果ByteBuf上一次循环读取时，累计读取长度大于ByteBuf的分配空间「缓冲空间」，优雅自适应扩容。
> 
> //allocator
> //内存分配器，可通过-Dio.netty.allocator.type参数指定。
> //默认使用「池化的堆外内存」PooledByteBufAllocator(PlatformDependent.directBufferPreferred());
> ```

```java
//AdaptiveRecvByteBufAllocator
@Override
public ByteBuf allocate(ByteBufAllocator alloc) {
  	//通过ByteBufAllocator进行内存分配
  	//nextReceiveBufferSize是预测接收ByteBuf大小，初始是1024
  	//分为53档[16,512]区间间隔16，[512-1G]翻倍。初始是33档「32」
    return alloc.ioBuffer(nextReceiveBufferSize);
}
```

```java
//AbstractByteBufAllocator「PooledByteBufAllocator」
@Override
public ByteBuf ioBuffer(int initialCapacity) {
  	//可以通过-Dsun.misc.Unsafe参数指定  使用堆内/堆外内存
  	//默认第true「堆外内存」
    if (PlatformDependent.hasUnsafe()) {
        return directBuffer(initialCapacity);
    }
    return heapBuffer(initialCapacity);
}
```

```java
//AbstractByteBufAllocator「PooledByteBufAllocator」
@Override
public ByteBuf directBuffer(int initialCapacity) {
  	//默认最大分配空间为Integer.MAX_VALUE 「 2G -1 」
    return directBuffer(initialCapacity, Integer.MAX_VALUE);
}
```

```java
//AbstractByteBufAllocator「PooledByteBufAllocator」
@Override
public ByteBuf directBuffer(int initialCapacity, int maxCapacity) {
  	//容错校验
    if (initialCapacity == 0 && maxCapacity == 0) {
        return emptyBuf;
    }
    validate(initialCapacity, maxCapacity);
  	//分配
    return newDirectBuffer(initialCapacity, maxCapacity);
}
```

```java
//AbstractByteBufAllocator「PooledByteBufAllocator」
@Override
protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
  
  	//每个线程单独分配
  	//使用了FastThreadLocal<PoolThreadCache> directArena在threadLocal初始化的时候，就已经初始化好了一个PoolThreadCache,对应一个空「empty」PoolArena.
    PoolThreadCache cache = threadCache.get();
    PoolArena<ByteBuffer> directArena = cache.directArena;

    ByteBuf buf;
  	//创建ByteBuf对象并分配内存空间
    if (directArena != null) {
        buf = directArena.allocate(cache, initialCapacity, maxCapacity);
    } else {
        if (PlatformDependent.hasUnsafe()) {
            buf = new UnpooledUnsafeDirectByteBuf(this, initialCapacity, maxCapacity);
        } else {
            buf = new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }
    }

  	//加入内存泄漏检测
    return toLeakAwareBuffer(buf);
}
```

```java
//PoolArena.DirectArena
PooledByteBuf<T> allocate(PoolThreadCache cache, int reqCapacity, int maxCapacity) {
  	//创建ByteBuf对象
  	//这里使用了Recycler对象池，下面会讲到，这里就不展开了。
    PooledByteBuf<T> buf = newByteBuf(maxCapacity);
  	//为ByteBuf分配空间
    allocate(cache, buf, reqCapacity);
    return buf;
}
```

```java
private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
  	//对申请内存大小进行规整，规整到2次幂或者16的倍数
    final int normCapacity = normalizeCapacity(reqCapacity);
  	//8K一下的分配，tiny或者small
    if (isTinyOrSmall(normCapacity)) { // capacity < pageSize
        int tableIdx;
        PoolSubpage<T>[] table;
      
      	//tiny和small的分配步骤一样。
      	//优先从cache分配，否则从subpage分配，否则分配新chunk
        if (isTiny(normCapacity)) { // < 512
          	//cache
            if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                return;
            }
          	//匹配subpage
            tableIdx = tinyIdx(normCapacity);
            table = tinySubpagePools;
        } else {
          	//cache
            if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                return;
            }
          	//匹配subpage
            tableIdx = smallIdx(normCapacity);
            table = smallSubpagePools;
        }
				//subpage分配
        synchronized (this) {
            final PoolSubpage<T> head = table[tableIdx];
            final PoolSubpage<T> s = head.next;
            if (s != head) {
                assert s.doNotDestroy && s.elemSize == normCapacity;
                long handle = s.allocate();
                assert handle >= 0;
                s.chunk.initBufWithSubpage(buf, handle, reqCapacity);
                return;
            }
        }
    //8K~16M的分配 normal
    } else if (normCapacity <= chunkSize) {
        if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {
            // was able to allocate out of the cache so move on
            return;
        }
    //16M以上的分配，huge不计入cache
    //直接分配一个reqCapacity的chunk，不需要拆分
    } else {
        // Huge allocations are never served via the cache so just call allocateHuge
        allocateHuge(buf, reqCapacity);
        return;
    }
  	//正常分配「重新分配chunk」，也就是cache没有分配成功「或许是首次分配，或许是空间不足」
    allocateNormal(buf, reqCapacity, normCapacity);
}
```

###### normalizeCapacity申请内存规整

```java
int normalizeCapacity(int reqCapacity) {
    if (reqCapacity < 0) {
        throw new IllegalArgumentException("capacity: " + reqCapacity + " (expected: 0+)");
    }
  	//16M以上的分配，不计入池化管理，直接走Huge分配
    if (reqCapacity >= chunkSize) {
        return reqCapacity;
    }

  	//512及以上的内存申请，规整到2的n次幂
    if (!isTiny(reqCapacity)) { // >= 512
        // Doubled

        int normalizedCapacity = reqCapacity;
        normalizedCapacity --;
        normalizedCapacity |= normalizedCapacity >>>  1;
        normalizedCapacity |= normalizedCapacity >>>  2;
        normalizedCapacity |= normalizedCapacity >>>  4;
        normalizedCapacity |= normalizedCapacity >>>  8;
        normalizedCapacity |= normalizedCapacity >>> 16;
        normalizedCapacity ++;

        if (normalizedCapacity < 0) {
            normalizedCapacity >>>= 1;
        }

        return normalizedCapacity;
    }

    // Quantum-spaced
  	//16到512区间内，16的倍数不需要规整，直接返回
    if ((reqCapacity & 15) == 0) {
        return reqCapacity;
    }

  	//非16的倍数，规整到16的倍数。
    return (reqCapacity & ~15) + 16;
}
```

###### cache分配

> 可以先略过，先看normal分配新chunk。释放之后，才会加入cache中。

###### subpage分配

> 可以先略过，先看normal分配新chunk。chunk分配完成才会加入subpage。

> 看完了normal分配之后，这里的问题就很简单了，就是找到申请内存规整后，对应的subpage档位的双向链表，尝试进行分配。

```java
//PoolChunk
//使用了一个重载initBufWithSubpage方法
void initBufWithSubpage(PooledByteBuf<T> buf, long handle, int reqCapacity) {
    initBufWithSubpage(buf, handle, (int) (handle >>> Integer.SIZE), reqCapacity);
}
```

###### normal分配新chunk

> 初次内存分配，会使用ByteBuffer.allocateDirect(chunkSize)进行内存开辟。
>
> ```java
> //入口
> //buf 对象池中的ByteBuf对象
> //reqCapacity 申请大小
> //normCapacity 规整后的申请大小
> allocateNormal(buf, reqCapacity, normCapacity);
> ```

```java
//PoolArena「DirectArena」
private synchronized void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
  	//优先分配顺序
  	//q050 > q025 > q000 > qInit > q075 > q100
  	//优先q050和q025上分配，应该是为了提高内存使用率。
  	//q075和q100分配成功率较低，放在最后。
  	//当然。首次分配的时候，这些都不存在。
    if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
        q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
        q075.allocate(buf, reqCapacity, normCapacity) || q100.allocate(buf, reqCapacity, normCapacity)) {
        return;
    }

    //已存在的chunkList都不能分配成功，申请新chunk「16M」
  	//pageSize 8K
  	//maxOrder 12层 「11」
  	//pageShifts 「13 2^13=8192=8K」
  	//chunkSize 16M
    PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);
  	//chunk分配空间
    long handle = c.allocate(normCapacity);
    assert handle > 0;
  	//「填充」ByteBuf
    c.initBuf(buf, handle, reqCapacity);
  	//从qInit节点开始插入chunkList中
    qInit.add(c);
}
```

***

==申请Chunk==

```java
//PoolArena 「DirectArena」
protected PoolChunk<ByteBuffer> newChunk(int pageSize, int maxOrder, int pageShifts, int chunkSize) {
    return new PoolChunk<ByteBuffer>(
      			//使用unsafe申请一个chunkSize大小的内存,返回一个nio的DirectByteBuffer
            this, ByteBuffer.allocateDirect(chunkSize), pageSize, maxOrder, pageShifts, chunkSize);
}
```

<img src="https://gitee.com/wangigor/typora-images/raw/master/netty-newChunk-DirectByteBuffer.png" alt="image-20200927143324525" style="zoom: 50%;" />

记录了内存地址、大小，以及一个内存清理器cleaner。

下面来看一下PoolChunk的构造函数

```java
PoolChunk(PoolArena<T> arena, T memory, int pageSize, int maxOrder, int pageShifts, int chunkSize) {
    unpooled = false;//池化
    this.arena = arena;//父PoolArena
    this.memory = memory;//DirectByteBuffer
    this.pageSize = pageSize;//8192
    this.pageShifts = pageShifts;//13
    this.maxOrder = maxOrder;//11
    this.chunkSize = chunkSize;//16777216
    unusable = (byte) (maxOrder + 1);//失效的二叉节点置为12
    log2ChunkSize = log2(chunkSize);//chunkSize是2的log2ChunkSize次幂
    subpageOverflowMask = ~(pageSize - 1);//~8191
    freeBytes = chunkSize;//剩余空间

    assert maxOrder < 30 : "maxOrder should be < 30, but is: " + maxOrder;
    maxSubpageAllocs = 1 << maxOrder;//总共分配2048「2^11」个「空间节点」,也就是二叉树的最底层为分配空间节点。对应后面2048个subpage

    //生成两个二叉树「数组形式」公有4096个节点
    memoryMap = new byte[maxSubpageAllocs << 1];
    depthMap = new byte[memoryMap.length];
    int memoryMapIndex = 1;
  	//遍历二叉树，把每个节点的阶数放进去「0-11」
  	//注意。这里多分配了一个0阶节点。「0-0，1-0」
    for (int d = 0; d <= maxOrder; ++ d) { 
        int depth = 1 << d;
        for (int p = 0; p < depth; ++ p) {
            memoryMap[memoryMapIndex] = (byte) d;
            depthMap[memoryMapIndex] = (byte) d;//只是用来记录阶数，不会发生修改。
            memoryMapIndex ++;
        }
    }
		//初始化2048个PoolSubpage
    subpages = newSubpageArray(maxSubpageAllocs);
}
```

- chunk分配

  ```java
  //PoolChunk
  long allocate(int normCapacity) {
    	//8k以上的分配方式
      if ((normCapacity & subpageOverflowMask) != 0) { // >= pageSize
          return allocateRun(normCapacity);
      //8k一下的分配方式
      } else {
          return allocateSubpage(normCapacity);
      }
  }
  ```

  > 注意：
  >
  > allocateRun对应了page级别的分配
  >
  > allocateSubpage对应了subpage级别的分配
  >
  > 两个返回值的不同在long的高64位：
  >
  > - subpage：bitmapIdx | 最高位为1
  > - page : 0

  - allocateRun 8K以上分配方式

  ```java
  //PoolChunk
  private long allocateRun(int normCapacity) {
    	//计算节点深度 16M-0阶 8K-11阶
      int d = maxOrder - (log2(normCapacity) - pageShifts);
      int id = allocateNode(d);//找到对应节点
      if (id < 0) {
          return id;
      }
      freeBytes -= runLength(id);//占用当前节点所在层级的空间大小
      return id;
  }
  ```

  - allocateSubpage 8K以下分配方式

  ```java
  //PoolChunk
  private long allocateSubpage(int normCapacity) {
      int d = maxOrder; 
    	//8K以下，分配第11层的节点
      int id = allocateNode(d);
    	//未分配成功
      if (id < 0) {
          return id;
      }
  		
      final PoolSubpage<T>[] subpages = this.subpages;
      final int pageSize = this.pageSize;//8192
    
      freeBytes -= pageSize;//计算空闲空间
    
  		//计算所在的Subpage 比如2048=0，2049=1
    	//memoryMapIdx ^ maxSubpageAllocs「2048 后11位为0」
      int subpageIdx = subpageIdx(id);
      PoolSubpage<T> subpage = subpages[subpageIdx];
      if (subpage == null) {//Subpage初始化
        	//runOffset(id) 当前空间对于DirectBuffer的起始偏移量
        	//初始化PageSubpage,占用空间，分档放入subpage[]双向链表中。
          subpage = new PoolSubpage<T>(this, id, runOffset(id), pageSize, normCapacity);
          subpages[subpageIdx] = subpage;
      } else {
          subpage.init(normCapacity);//占用内存 
      }
    	//subpage分配
      return subpage.allocate();
  }
  ```

  ```java
  //PoolSubpage
  PoolSubpage(PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
      this.chunk = chunk;//父chunk
      this.memoryMapIdx = memoryMapIdx;
      this.runOffset = runOffset;//起始偏移量
      this.pageSize = pageSize;//当前page大小
    	//初始化8个bytemap
    	//每一位代表16字节的空间，因为最小分配16「规整」,bytemap需要8192/16=512位来记录这全部的8K
    	//而一个long占64位，所以bitmap可以表示为8个long
      bitmap = new long[pageSize >>> 10]; // pageSize / 16 / 64
    	//subpage初始化
      init(elemSize);
  }
  ```

  ```java
  //PoolSubpage
  void init(int elemSize) {
      doNotDestroy = true;
      this.elemSize = elemSize;//已占用空间
      if (elemSize != 0) {
        	// 把pageSize按照申请大小 进行分割，这里记录分割数量
        	//比如申请1024，这里maxNumElems就是8
        	//比如申请16，这里maxNumElems就是512
          maxNumElems = numAvail = pageSize / elemSize;
          nextAvail = 0;
        	//bitmapLength 是在bitmap中「可用」当前用于记录的long的数量
        	//比如 maxNumElems=8，8/64,使用1个long记录
        	//比如 maxNumElems=512，512/64,使用8个long记录
          bitmapLength = maxNumElems >>> 6;
          if ((maxNumElems & 63) != 0) {//不足1的按1算。
              bitmapLength ++;
          }
  				
        	//这一步好像没有用处，因为bitmap初始就是 {0,0,0,0,0,0,0,0}
          for (int i = 0; i < bitmapLength; i ++) {
              bitmap[i] = 0;
          }
      }
  		//添加到双向链表「PoolSubpage[] tinySubpagePools/smallSubpagePools」中
    	//因为subpagePools进行了分档，tiny每16为一档，共32档；small每档翻倍，共4档。
    	//获取节点，加入每一档的双向链表中，这里就不展开了
      addToPool();
  }
  ```

allocateNode 寻找二叉树节点

```java
private int allocateNode(int d) {
    int id = 1;//从根节点开始
    int initial = - (1 << d); //反码 后11位为1
    byte val = value(id);//memoryMap[id];
    if (val > d) { // 当前节点的值大于当前节点的阶数，说明当前节点已经分配出去了「非空闲」
        return -1;
    }
  	//当前节点空闲 且 在d阶节点之上。就继续循环
    while (val < d || (id & initial) == 0) { // id & initial == 1 << d for all ids at depth d, for < d it is 0
        id <<= 1;//当前节点的左节点『下一阶』
        val = value(id);
      	//左节点非空闲
        if (val > d) {
          	//去兄弟节点查找
            id ^= 1;
            val = value(id);
        }
    }
  	//经过上面的循环，就找到了可以分配的节点
    byte value = value(id);
  	//检查阶数是否正确
    assert value == d && (id & initial) == 1 << d : String.format("val = %d, id & initial = %d, d = %d",
            value, id & initial, d);
  	//设置当前阶数的值为已占用「12」
    setValue(id, unusable); // mark as unusable
  	//级联更新父节点
    updateParentsAlloc(id);
    return id;
}
```

runOffset 空间偏移量

> 当前节点「比如2049号节点」对应的DirectBuffer整个空间的起始位置偏移量「2049对应8192，因为2048分配出去8K」

```java
private int runOffset(int id) {
    int shift = id ^ 1 << depth(id);//计算当前节点相对于二叉树最左节点的偏移量 2048=0 2049=1
    return shift * runLength(id);//runLength(id)表示id这一层「2049所在的第11层」每个节点相对于前一个节点的偏移量「8192」
}
```

	- subpage分配

```java
//PoolSubpage
//返回subpage对应的bitmap索引
long allocate() {
    if (elemSize == 0) {
        return toHandle(0);
    }
    if (numAvail == 0 || !doNotDestroy) {
        return -1;
    }
		//找到下一个可用的bitmap「初始化为0，经过这一次，nextAvail被设为了-1,并返回0」
    final int bitmapIdx = getNextAvail();
    int q = bitmapIdx >>> 6;//第q个long
    int r = bitmapIdx & 63;//第q个long的r个位置要开始被「占用」
    assert (bitmap[q] >>> r & 1) == 0;//检查当前位置未被占用
    bitmap[q] |= 1L << r;//占用，标记为1 「一个1，就代表了1个「申请大小」的subpage的分段被占用」

    if (-- numAvail == 0) {//可用数量-1，如果没了，说明全占满了，从subpage的双向链表中移除
        removeFromPool();
    }
		//返回，低32位记录subpage所在chunk的位置「2048」，高32位记录分配内存所在subpage的索引「第几块」
    return toHandle(bitmapIdx);
}
```

```java
private long toHandle(int bitmapIdx) {
  	//0x4000000000000000L 是除了符号位，最高位为1的long
  	//返回结果 高32位存放bitmapIdx，低32位存放memoryMapIdx
    return 0x4000000000000000L | (long) bitmapIdx << 32 | memoryMapIdx;
}
```

那么到这里，申请chunk就完成了。

***

==「填充」byteBuf==

> ```java
> //入口
> //handle上面有介绍
> c.initBuf(buf, handle, reqCapacity);
> ```

```java
void initBuf(PooledByteBuf<T> buf, long handle, int reqCapacity) {
  	//handle按照高、低32位拆分开。
    int memoryMapIdx = (int) handle;
    int bitmapIdx = (int) (handle >>> Integer.SIZE);
  	//page级别的分配「allocateRun」
    if (bitmapIdx == 0) {
        byte val = value(memoryMapIdx);
        assert val == unusable : String.valueOf(val);
      	//runOffset(memoryMapIdx) 相对于DirectByteBuffer的偏移量
      	//runLength(memoryMapIdx) 空间
        buf.init(this, handle, runOffset(memoryMapIdx), reqCapacity, runLength(memoryMapIdx));
    } else {
      	//subpage级别的分配「allocateSubpage」
        initBufWithSubpage(buf, handle, bitmapIdx, reqCapacity);
    }
}
```

```java
private void initBufWithSubpage(PooledByteBuf<T> buf, long handle, int bitmapIdx, int reqCapacity) {
    assert bitmapIdx != 0;

    int memoryMapIdx = (int) handle;
		//获取subpage
    PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
    assert subpage.doNotDestroy;
    assert reqCapacity <= subpage.elemSize;
		
  	//分配
    buf.init(
        this, handle,
      	//runOffset(memoryMapIdx)对于directByteBuff的偏移量
      	//(bitmapIdx & 0x3FFFFFFF) * subpage.elemSize 已占用的偏移量 「起始位置」
      	//subpage.elemSize 分配大小
      	//reqCapacity 申请大小 「规整前实际申请大小」
        runOffset(memoryMapIdx) + (bitmapIdx & 0x3FFFFFFF) * subpage.elemSize, reqCapacity, subpage.elemSize);
}
```

```java
//PooledUnsafeDirectByteBuf
void init(PoolChunk<ByteBuffer> chunk, long handle, int offset, int length, int maxLength) {
    super.init(chunk, handle, offset, length, maxLength);
  	//物理内存位置
    initMemoryAddress();
}
private void initMemoryAddress() {
		memoryAddress = PlatformDependent.directBufferAddress(memory) + offset;
}
```

***

==chunk节点加入chunkList==

> 从qInit开始，根据当前chunk的使用情况，放入对应的chunkList
>
> 顺序：qInit -> q000 -> q025 -> q050 -> q075 -> q100

```java
void add(PoolChunk<T> chunk) {
  	//超过上限，后移
    if (chunk.usage() >= maxUsage) {
        nextList.add(chunk);
        return;
    }

    chunk.parent = this;
    if (head == null) {
        head = chunk;
        chunk.prev = null;
        chunk.next = null;
    } else {
        chunk.prev = null;
        chunk.next = head;
        head.prev = chunk;
        head = chunk;
    }
}
```

***

##### 释放内存release

> 谁最后使用，谁释放
>
> 从pipeline的head节点开始读取「创建」，要么从解码器『或自定义解码器』释放，要么从tail节点默认释放。
>
> ```java
> ReferenceCountUtil.release(msg);
> ```

```java
public static boolean release(Object msg) {
  	//ByteBuf默认实现了引用计数接口
    if (msg instanceof ReferenceCounted) {
        return ((ReferenceCounted) msg).release();
    }
    return false;
}
```

> 这个msg经过了两层『包裹』
>
> - 内存检测 「默认SimpleLeadAwareByteBuf」
> - 一层wrap 「SlicedByteBuf」
>
> 中间这些跳过不看直接看PoolUnsafeDirectByteBuf的释放方法

```java
//AbstractReferenceCountedByteBuf
public final boolean release() {
  	//获取当前byteBuf的引用计数
  	//自旋cas -1
  	//没有引用「可释放」时进行释放。
    for (;;) {
        int refCnt = this.refCnt;
        if (refCnt == 0) {
            throw new IllegalReferenceCountException(0, -1);
        }

        if (refCntUpdater.compareAndSet(this, refCnt, refCnt - 1)) {
            if (refCnt == 1) {
                deallocate();
                return true;
            }
            return false;
        }
    }
}
```

```java
//PooledByteBuf
protected final void deallocate() {
    if (handle >= 0) {//说明已经释放过了
        final long handle = this.handle;
        this.handle = -1;
        memory = null;//memory属性置为null
      	//创建线程是否跟释放线程一致
        boolean sameThread = initThread == Thread.currentThread();
        initThread = null;
      	//释放空间
        chunk.arena.free(chunk, handle, maxLength, sameThread);
      	//对象池「回收」ByteBuf对象
        recycle();
    }
}
```

```java
//PoolArena
void free(PoolChunk<T> chunk, long handle, int normCapacity, boolean sameThreads) {
  	//非池化的chunk直接销毁，比如huge
    if (chunk.unpooled) {
        destroyChunk(chunk);//手动调用Cleaner进行销毁
    } else {
      	//通线程释放 加入线程独有的cache中
        if (sameThreads) {
            PoolThreadCache cache = parent.threadCache.get();
            if (cache.add(this, chunk, handle, normCapacity)) {
                // cached so not free it.
                return;
            }
        }
				//缓存满了就「释放」
        synchronized (this) {
            chunk.parent.free(chunk, handle);
        }
    }
}
```

###### 加入缓存

```java
//PoolThreadCache
boolean add(PoolArena<?> area, PoolChunk chunk, long handle, int normCapacity) {
    MemoryRegionCache<?> cache;
  	//按切分档位放入三类缓存中 tiny[16,512) small[512,8k) normal[8k,16M]
    if (area.isTinyOrSmall(normCapacity)) {
        if (PoolArena.isTiny(normCapacity)) {
            cache = cacheForTiny(area, normCapacity);
        } else {
            cache = cacheForSmall(area, normCapacity);
        }
    } else {
        cache = cacheForNormal(area, normCapacity);
    }
    if (cache == null) {
        return false;
    }
  	//缓存添加
  	//创建一个带有chunk和handle的Entry,在数组中新婚徐添加
    return cache.add(chunk, handle);
}
```

```java
//PoolThreadCache


//tiny
private MemoryRegionCache<?> cacheForTiny(PoolArena<?> area, int normCapacity) {
    int idx = PoolArena.tinyIdx(normCapacity);
    if (area.isDirect()) {
      	//32档获取一个
        return cache(tinySubPageDirectCaches, idx);
    }
    return cache(tinySubPageHeapCaches, idx);
}
//small
private MemoryRegionCache<?> cacheForSmall(PoolArena<?> area, int normCapacity) {
    int idx = PoolArena.smallIdx(normCapacity);
    if (area.isDirect()) {
      	//4档获取一个
        return cache(smallSubPageDirectCaches, idx);
    }
    return cache(smallSubPageHeapCaches, idx);
}
//normal
private MemoryRegionCache<?> cacheForNormal(PoolArena<?> area, int normCapacity) {
    if (area.isDirect()) {
      	//4档获取一个 这里按理，应该从8K~16M有12档，但是默认normal档位之后4档
      	//不明白为啥,可能超过64k不缓存？
        int idx = log2(normCapacity >> numShiftsNormalDirect);
        return cache(normalDirectCaches, idx);
    }
    int idx = log2(normCapacity >> numShiftsNormalHeap);
    return cache(normalHeapCaches, idx);
}
```

###### chunk释放

> 入口
>
> ```java
> chunk.parent.free(chunk, handle);
> ```

```java
void free(PoolChunk<T> chunk, long handle) {
  	//释放
    chunk.free(handle);
  	//释放后小于当前chunklist的最小使用率,就前移比如q050移到q025
    if (chunk.usage() < minUsage) {
        remove(chunk);
        if (prevList == null) {
            assert chunk.usage() == 0;
          	//q000前置为null，最小值1%
          	//也就是使用率如果为0，就真实释放
            arena.destroyChunk(chunk);
        } else {
            prevList.add(chunk);
        }
    }
}
```

```java
void free(long handle) {
    int memoryMapIdx = (int) handle;
    int bitmapIdx = (int) (handle >>> Integer.SIZE);
		//找到对应的bitmap，修改为未占用
    if (bitmapIdx != 0) { // free a subpage
        PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
        assert subpage != null && subpage.doNotDestroy;
        if (subpage.free(bitmapIdx & 0x3FFFFFFF)) {
            return;
        }
    }
  	//更新剩余长度
    freeBytes += runLength(memoryMapIdx);
  	//设置chunk的二叉树节点和父节点
    setValue(memoryMapIdx, depth(memoryMapIdx));
    updateParentsFree(memoryMapIdx);
}
```

#### Recycler对象池

> 为了减少对象频繁创建与销毁，减少GC压力。netty使用了Recycler对象池。
>
> ![image-20201010155517123](https://gitee.com/wangigor/typora-images/raw/master/Recycler对象池结构.png)
>
> - **「A线程」创建的对象被『A线程』回收，放入Stack-A的elements中。**
> - **「A线程」创建的对象被『B线程』回收，放入Stack-A的WeakOrderQueue-B中。**
> - **『A线程』创建对象，优先从Stack-A的elements中获取，为空则先在WeakOrderQueue进行部分回收。**

##### 使用

###### 示例

```java
@Slf4j
public class RecyclerTest {

  	//定义一个Recycler对象池，对象类型为DataObject
  	//重写newObject对象创建方法。
    public static final Recycler<DataObject> recycler = new Recycler<DataObject>() {
        @Override
        protected DataObject newObject(Handle<DataObject> handle) {
            return new DataObject(handle);
        }
    };

  	//测试
    @Test
    public void test() {
        DataObject data0 = recycler.get();

        data0.recycle();//data0回收

        DataObject data1 = recycler.get();
				DataObject data2 = recycler.get();
      
        assert data0 == data1;//相同
      	assert data1 != data2;//不相同
    }

}

//自定义POJOclass
//带有Recycler.Handle处理器
//handle跟pojo是一对一关系
@Data
@RequiredArgsConstructor
class DataObject {
    @NonNull
    private Recycler.Handle<DataObject> handle;


    public void recycle() {
        this.handle.recycle(this);
    }
}
```

###### netty的byteBuf对象池

> 入口
>
> ```java
> PooledByteBuf<T> buf = newByteBuf(maxCapacity);
> ```
>
> ```java
> //PoolArena
> protected PooledByteBuf<ByteBuffer> newByteBuf(int maxCapacity) {
>     if (HAS_UNSAFE) {
>         return PooledUnsafeDirectByteBuf.newInstance(maxCapacity);
>     } else {
>         return PooledDirectByteBuf.newInstance(maxCapacity);
>     }
> }
> ```
>
> ```java
> //PoolUnsafeDirectByteBuf
> //对象池
> private static final Recycler<PooledUnsafeDirectByteBuf> RECYCLER = new Recycler<PooledUnsafeDirectByteBuf>() {
>   @Override
>   protected PooledUnsafeDirectByteBuf newObject(Handle<PooledUnsafeDirectByteBuf> handle) {
>     return new PooledUnsafeDirectByteBuf(handle, 0);
>   }
> };
> 
> static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
>   	//对象池获取
>     PooledUnsafeDirectByteBuf buf = RECYCLER.get();
>     buf.setRefCnt(1);
>     buf.maxCapacity(maxCapacity);
>     return buf;
> }
> ```
>
> ```java
> //PoolByteBuf
> //release时进行回收
> private void recycle() {
>   	//回收
>     recyclerHandle.recycle(this);
> }
> ```

##### 源码

###### 属性和构造



- Recycler

```java
//线程的id生成器
private static final AtomicInteger ID_GENERATOR = new AtomicInteger(Integer.MIN_VALUE);
private static final int OWN_THREAD_ID = ID_GENERATOR.getAndIncrement();
private static final int DEFAULT_MAX_CAPACITY; //默认最大容量 256*10「2^18」 可通过io.netty.recycler.maxCapacity进行设置
private static final int INITIAL_CAPACITY; //初始容量 默认256「2^8」

private final int maxCapacity;
//ThreadLocal thread -> Stack
private final FastThreadLocal<Stack<T>> threadLocal = new FastThreadLocal<Stack<T>>() {
  protected Stack<T> initialValue() {
    return new Stack<T>(Recycler.this, Thread.currentThread(), maxCapacity);
  }
};
```

```java
//构造 指定最大容量
protected Recycler(int maxCapacity) {
    this.maxCapacity = Math.max(0, maxCapacity);
}
```

- Stack

```java
final Recycler<T> parent;//父对象池
final Thread thread;//所属线程
private DefaultHandle<?>[] elements; //存放元素
private final int maxCapacity;//最大容量
private int size;//已存容量

//指向WeakOrderQueue链表
private volatile WeakOrderQueue head;
private WeakOrderQueue cursor, prev;//两个指针
```

```java
//构造
Stack(Recycler<T> parent, Thread thread, int maxCapacity) {
  	//设置属性
    this.parent = parent;
    this.thread = thread;
    this.maxCapacity = maxCapacity;
  	//初始化数组
    elements = new DefaultHandle[Math.min(INITIAL_CAPACITY, maxCapacity)];
}
```

- DefaultHandle

```java
private int lastRecycledId;//上次回收的id 初始0 todo
private int recycleId;//所属线程id own_thread_id 初始0 todo

private Stack<?> stack;
private Object value;//目标对象
```

- WeakOrderQueue

```java
// 存放的数据链 head -> link -> link <- tail
private Link head, tail;
// 指向同一个stack的另一个线程队列
private WeakOrderQueue next;
private final WeakReference<Thread> owner; //归属线程的弱引用
private final int id = ID_GENERATOR.getAndIncrement();//本队列id
```

```java
//stack是对象的创建stack
//thread是对象的回收线程
WeakOrderQueue(Stack<?> stack, Thread thread) {
  	//初始化一个16的数组
    head = tail = new Link();
  	//保存回收线程的弱引用
    owner = new WeakReference<Thread>(thread);
  	//添加到stack的WeakOrderQueue的头结点
    synchronized (stack) {
        next = stack.head;
        stack.head = this;
    }
}
```

- Link

```java
//固定16长度的DefaultHandle数组
private final DefaultHandle<?>[] elements = new DefaultHandle[LINK_CAPACITY];
//读取指针
private int readIndex;
private Link next;//组成链表
```

###### 对象释放

> 先看对象释放，对我们理解对象池结构有帮助。
>
> ```java
> this.handle.recycle(this);
> ```

```java
public void recycle(Object object) {
		//校验 略。
    Thread thread = Thread.currentThread();
  
  	//创建线程和回收线程是同一个线程
  	//把对象添加进Stack的DefaultHandle集合中。
    if (thread == stack.thread) {
        stack.push(this);
        return;
    }
  
    //获取回收线程对应的WeakOrderQueue
  	//也就是object「DefaultHandle」被一个stack「线程A」创建
  	//但是却被线程B或者线程C回收，每一个其他线程回收都对应创建一个WeakOrderQueue，挂在线程A对应的stack的WeakOrderQueue链表下。
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    WeakOrderQueue queue = delayedRecycled.get(stack);
    if (queue == null) {
      	//初始化WeakOrderQueue并放入DELAYED_RECYCLED中
        delayedRecycled.put(stack, queue = new WeakOrderQueue(stack, thread));
    }
  	//添加入队列中
    queue.add(this);
}
```

- 同一线程回收对象

> 直接放入创建stack的DefaultHandle数组中
>
> ```java
> stack.push(this);
> ```

```java
void push(DefaultHandle<?> item) {
  	//说明已经被回收了
    if ((item.recycleId | item.lastRecycledId) != 0) {
        throw new IllegalStateException("recycled already");
    }
  	//指定recycleId和lastRecycledId为OWN_THREAD_ID「创建线程id」自己设计的自增主键
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;
		
    int size = this.size;//已存容量
    if (size >= maxCapacity) {
        return;//达到最大容量 不存
      	//不存的意思是不保存进对象池
      	//而是通过线程执行结束断掉强引用，而被gc回收
    }
  	//未达到最大容量，但是数组满了，就翻倍扩容
    if (size == elements.length) {
        elements = Arrays.copyOf(elements, Math.min(size << 1, maxCapacity));
    }
		//放进数组中
  	//已存容量+1
    elements[size] = item;
    this.size = size + 1;
}
```

- 不同线程回收对象

> 不是创建线程进行回收，则放入对应WeakOrderQueue队列
>
> ```java
> queue.add(this);
> ```

```java
void add(DefaultHandle<?> handle) {
  	//上一次回收id设置为回收线程对应的id
    handle.lastRecycledId = id;
		//从尾结点开始防止
    Link tail = this.tail;
    int writeIndex;
  	//如果尾结点Link的16数组已经放满了，就新建一个Link放到尾结点继续存放
    if ((writeIndex = tail.get()) == LINK_CAPACITY) {
        this.tail = tail = tail.next = new Link();
        writeIndex = tail.get();//这里获取的是Link父AtomicInteger的值，也就是对应的数组下标。
    }
    tail.elements[writeIndex] = handle;
    handle.stack = null;
  
		//设置数组下标
    tail.lazySet(writeIndex + 1);
}
```

> 关于最后这个lazySet要着重说一下。
>
> [Doug Lea提交的注释说明](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6275329)
>
> [Netty作者的解释](https://github.com/netty/netty/issues/8215)
>
> - 这里做的是一个**非原子的volatile属性**的更新操作。
>
> 多线程环境中，先get，再操作，无法保证原子性。那这一步的set和lazySet执行效果一样。
>
> - 这本身作为一个**缓存对象**池，回收对象不需要100%的被缓存住。
>
> 也就是可以接受，高并发环境下，一些并发入池的对象可以失败，而被GC。
>
> - lazySet比set**执行效率高**
>
> 「个人理解」volatile属性的赋值，前后有两个内存屏障「前：写内存屏障；后：读内存屏障」。后置的读内存屏障，会阻止指令重排，且通过总线嗅探将其他cpu寄存器缓存对应的缓存值置为失效，在下一次读volatile属性时，重新去内存读取。
>
> lazySet则是取消了后置的读内存屏障，保留前置写内存屏障。提高了效率。
>
> ```java
> //测试demo
> //两个具有volatile的对象
> volatile AtomicInteger a=new AtomicInteger();
> volatile AtomicInteger b=new AtomicInteger();
> 
> @Test
> public void testLazySet(){
> 		//10万次
>     int count=100000;
> 
>     StopWatch watch=new StopWatch();
> 
>     watch.start("set");
>     IntStream.range(0,count).parallel().forEach(item->{
>       			//set 非原子累加
>             int temp = a.intValue()+1;
>             a.set(temp);
>     });
>     watch.stop();
>     watch.start("lazySet");
>     IntStream.range(0,count).parallel().forEach(item->{
>       	//lazySet 非原子累加
>         int temp = b.intValue()+1;
>         b.lazySet(temp);
>     });
>     watch.stop();
> 
>   	//结果输出
>     System.out.println(a);
>     System.out.println(b);
>     System.out.println(watch.prettyPrint());
> 
> }
> ```
>
> ```log
> 12587
> 14712
> StopWatch '': running time = 65762849 ns
> ---------------------------------------------
> ns         %     Task name
> ---------------------------------------------
> 061659196  094%  set
> 004103653  006%  lazySet
> ```
>
> 可以明显看到，lazySet在性能上有10倍以上的提升。

###### 对象创建

> ```java
> recycler.get()
> ```

```java
public final T get() {
  	//获取threadLocal的对应Stack
    Stack<T> stack = threadLocal.get();
  	//弹出一个缓存对象
    DefaultHandle<T> handle = stack.pop();
  	//如果要没有就通过子类实现的newObject创建
  	//并包裹成DefaultHandle
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle);
    }
  	//返回实例
    return (T) handle.value;
}
```

```java
DefaultHandle<T> pop() {
  	//当前stack大小
    int size = this.size;
  	//如果没有，先进行回收操作
    if (size == 0) {
        if (!scavenge()) {
            return null;
        }
        size = this.size;
    }
  	//从自己的DefaultHandle集合中返回最后一个。
    size --;
    DefaultHandle ret = elements[size];
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    this.size = size;
    return ret;
}
```

```java
boolean scavenge() {
  // continue an existing scavenge, if any
  if (scavengeSome()) {
    return true;
  }

  // reset our scavenge cursor
  prev = null;
  cursor = head;
  return false;
}
boolean scavengeSome() {
  	//对WeakOrderQueue进行清理
  	//找到游标位置，否则从头开始
    WeakOrderQueue cursor = this.cursor;
    if (cursor == null) {
        cursor = head;
        if (cursor == null) {
            return false;//当前stack没有WeakOrderQueue，直接返回
        }
    }

    boolean success = false;
    WeakOrderQueue prev = this.prev;
  	//注意：这个循环不做全部转移。也就是假设当前这个stack有三个其他线程帮他回收了对象，那会有三个WeakOrderQueue
  	//不需要三个WeakOrderQueue全部转移，保证转移成功可以弹出对象即可。
    do {
      	//将当前游标对应的WeakOrderQueue里的DefaultHandle转移到stack中
      	//这里也不是对WeakOrderQueue的全部转移，而是从头结点head Link开始，能转移成功即返回。
        if (cursor.transfer(this)) {
            success = true;
            break;
        }
				//游标下移
        WeakOrderQueue next = cursor.next;
      	//如果对应的回收线程执行完成
        if (cursor.owner.get() == null) {
          	//且WeakOrderQueue队列有数据
            if (cursor.hasFinalData()) {
              	//全部回收至stack数组中
                for (;;) {
                    if (cursor.transfer(this)) {
                        success = true;
                    } else {
                        break;
                    }
                }
            }
            if (prev != null) {
                prev.next = next;
            }
        } else {
            prev = cursor;
        }

        cursor = next;

    } while (cursor != null && !success);

    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```

#### 内存泄漏检测

> 前面讲到，ByteBuf分配完成之后，会通过
>
> ```java
> toLeakAwareBuffer(buf)
> ```
>
> 对buf进行包裹，加入泄漏检测。
>
> **检查ByteBuf没有release的情况。**

```java
protected static ByteBuf toLeakAwareBuffer(ByteBuf buf) {
    ResourceLeak leak;
  	//检测等级
  	//可以通过io.netty.leakDetectionLevel参数设置
  	//默认使用Level.SIMPLE
    switch (ResourceLeakDetector.getLevel()) {
        //简单模式
        case SIMPLE:
            leak = AbstractByteBuf.leakDetector.open(buf);
            if (leak != null) {
                buf = new SimpleLeakAwareByteBuf(buf, leak);
            }
            break;
        //增强模式 跟偏执模式一样
        //但是检测抽样频率和简单模式一样。113个对象创建抽取一次。
        //调用栈记录和偏执模式一样。每次都记录。记录创建。
        case ADVANCED:
        //偏执模式
        case PARANOID:
            leak = AbstractByteBuf.leakDetector.open(buf);
            if (leak != null) {
                buf = new AdvancedLeakAwareByteBuf(buf, leak);
            }
            break;
    }
    return buf;
}
```

##### ResourceLeak追踪资源对象

```java
public interface ResourceLeak {
    //记录资源对象ByteBuf的当前调用堆栈信息
    void record();

    //记录资源对象ByteBuf的当前调用堆栈信息 附加信息
    void record(Object hint);

    //关闭泄漏，不发出告警
    boolean close();
}
```

> ResourceLeakDetector探测器对它进行了默认实现

```java
//继承了ResourceLeak接口
//是虚引用的子类实现 对象被gc时加入到refQueue中
private final class DefaultResourceLeak extends PhantomReference<Object> implements ResourceLeak {

    private static final int MAX_RECORDS = 4;

  	//资源对象创建时的堆栈记录 「增强和偏执型」
    private final String creationRecord;
  	//数组实现的双端队列
    private final Deque<String> lastRecords = new ArrayDeque<String>();
    private final AtomicBoolean freed;//对象是否已被回收
  
  	//链表 存储所有追踪对象
    private DefaultResourceLeak prev;
    private DefaultResourceLeak next;

  	//构造
    DefaultResourceLeak(Object referent) {
      	//加入引用队列refQueue
        super(referent, referent != null? refQueue : null);

      	//对象未被回收
        if (referent != null) {
            Level level = getLevel();
          	//增强和偏执级别 记录创建栈信息
            if (level.ordinal() >= Level.ADVANCED.ordinal()) {
                creationRecord = newRecord(null, 3);
            } else {
                creationRecord = null;
            }
						//初始链表的头结点
            synchronized (head) {
                prev = head;
                next = head.next;
                head.next.prev = this;
                head.next = this;
                active ++;
            }
            freed = new AtomicBoolean();
        } else {
          	//对象被gc回收
            creationRecord = null;
            freed = new AtomicBoolean(true);
        }
    }
		
    public void record() {
        record0(null, 3);
    }

    public void record(Object hint) {
        record0(hint, 3);
    }
		//recordsToSkip忽略的栈顶深度 3
  	//就是忽略record、record0、newRecord，只记录到业务调用处。
    private void record0(Object hint, int recordsToSkip) {
        if (creationRecord != null) {
          	//创建栈信息记录
            String value = newRecord(hint, recordsToSkip);
						
          	//调用栈信息记录 在双端队列尾部插入
            synchronized (lastRecords) {
                int size = lastRecords.size();
                if (size == 0 || !lastRecords.getLast().equals(value)) {
                    lastRecords.add(value);
                }
              	//超过最大记录数，从队列头部移除一个
              	//默认保存4条记录。
                if (size > MAX_RECORDS) {
                    lastRecords.removeFirst();
                }
            }
        }
    }

    public boolean close() {
      	//手动改为已释放
        if (freed.compareAndSet(false, true)) {
            synchronized (head) {
                active --;
                prev.next = next;
                next.prev = prev;
                prev = null;
                next = null;
            }
            return true;
        }
        return false;
    }

    //略
    public String toString() {}
}
```

> 记录调用栈信息
>
> 就是调用栈toString的过程，只是过滤不需要记录的信息。

```java
static String newRecord(Object hint, int recordsToSkip) {
    StringBuilder buf = new StringBuilder(4096);

    // 记录对象
    if (hint != null) {
        buf.append("\tHint: ");
        if (hint instanceof ResourceLeakHint) {
            buf.append(((ResourceLeakHint) hint).toHintString());
        } else {
            buf.append(hint);
        }
        buf.append(NEWLINE);//换行
    }

    // 记录调用栈信息
    StackTraceElement[] array = new Throwable().getStackTrace();
    for (StackTraceElement e: array) {
      	//过滤栈顶recordsToSkip条
        if (recordsToSkip > 0) {
            recordsToSkip --;
        } else {
            String estr = e.toString();

            //忽略不需要记录的调用栈
          	//"io.netty.util.ReferenceCountUtil.touch("
            //"io.netty.buffer.AdvancedLeakAwareByteBuf.touch("
            //"io.netty.buffer.AbstractByteBufAllocator.toLeakAwareBuffer("
            boolean excluded = false;
            for (String exclusion: STACK_TRACE_ELEMENT_EXCLUSIONS) {
                if (estr.startsWith(exclusion)) {
                    excluded = true;
                    break;
                }
            }

            if (!excluded) {
                buf.append('\t');
                buf.append(estr);
                buf.append(NEWLINE);
            }
        }
    }
    return buf.toString();
}
```

##### ResourceLeakDetector资源泄漏探测器

```java
//属性

//活跃对象 量表
private final DefaultResourceLeak head = new DefaultResourceLeak(null);
private final DefaultResourceLeak tail = new DefaultResourceLeak(null);

//引用队列
private final ReferenceQueue<Object> refQueue = new ReferenceQueue<Object>();
//内存泄漏报告
//注意这个ConcurrentMap是netty自己封装的 jdk8使用jdk的，8以下使用自己封的
private final ConcurrentMap<String, Boolean> reportedLeaks = PlatformDependent.newConcurrentHashMap();

private final String resourceType;//对象类型
private final int samplingInterval;//取样周期 默认113「次」一次
private final long maxActive;//本类对象的最大活跃数 默认long的最大值
private long active;//当前对象的活跃数
private final AtomicBoolean loggedTooManyActive = new AtomicBoolean();//是否打印日志

private long leakCheckCnt;//泄漏检查次数
```

###### 创建对象探测 open

```java
public ResourceLeak open(T obj) {
    Level level = ResourceLeakDetector.level;
  	//级别为关闭 不创建
    if (level == Level.DISABLED) {
        return null;
    }
		
  	//简单和增强级别。有一定的抽样频率。113个对象创建。记录一个。
    if (level.ordinal() < Level.PARANOID.ordinal()) {
        if (leakCheckCnt ++ % samplingInterval == 0) {
            reportLeak(level);//检测
            return new DefaultResourceLeak(obj);
        } else {
            return null;
        }
    } else {
    //偏执级别 每「个」都检测
        reportLeak(level);//检测
        return new DefaultResourceLeak(obj);
    }
}
```

###### 泄漏检测 reportLeak

```java
private void reportLeak(Level level) {
    //省略
    for (;;) 
      	//持续从引用队列中取数据
      	//被gc的对象就会出现在这里。
        DefaultResourceLeak ref = (DefaultResourceLeak) refQueue.poll();
        if (ref == null) {
            break;
        }

        ref.clear();
				//close方法是修改freed值的。
  			//close第一次返回true，之后都返回false
  			//release会调用close方法。
        if (!ref.close()) {
            continue;//说明之前close过了。
        }
				
  			//没有是放过的。写入reportedLeaks报告中。
        String records = ref.toString();
        if (reportedLeaks.putIfAbsent(records, Boolean.TRUE) == null) {
            //输出报告日志
        }
    }
}
```