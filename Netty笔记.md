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











