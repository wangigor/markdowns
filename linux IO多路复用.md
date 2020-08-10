# IO多路复用

> IO多路复用是一种同步IO模型，实现一个线程可以监视多个文件句柄；
> 一旦某个文件句柄就绪，就能够通知应用程序进行相应的读写操作；
> 没有文件句柄就绪时会阻塞应用程序，交出cpu。
> 多路是指网络连接，复用指的是同一个线程。

## 背景知识

### 网卡和内存

![网络分层](https://gitee.com/wangigor/typora-images/raw/master/网络分层.png)

>  **==网卡== Network Adapter**也被成为网络适配器，是一个硬件设备，有全球唯一的**MAC（media access control）**地址，mac地址在网卡生产时就被烧制在ROM中。
>
> 网卡收到的数据是**光信号或电信号**，然后将其**还原成数字信息**（0和1组成）。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/1.png)
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/网卡数字信息结构.png)
>
> 根据FCS(帧校验序列，Frame Check Sequence)**校验数据**。判断数据传输在传输过程中是否因噪音等影响导致信号失真，从而导致数据丢失，需要**丢弃无效的数据包。**
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/Mac头部.png)
>
> 然后**检查数据包中的MAC头部中的接收方MAC地址**：如果不是发给自己的，**丢弃数据包**；如果数据包是发送给自己的，将**数字信息保存到网卡内部缓冲区。**
>
> 以上过程**不需要cpu参与**，网卡自己就搞定了。cpu这个时候还不知道有数据包到达。

***

==**网卡驱动**==是CPU控制和使用网卡的程序。

> 网卡处理完数字信号后，接下来的数据接收需要cpu参与。
>
> **网卡通过中断将数据包到达的事件通知给cpu**。接着，cpu调用网卡驱动来干活。
>
> - **从网卡缓冲区读取接收到的数据**。
> - 根据MAC头部的以太类型字段**判断协议种类**，并**调用处理该协议的软件**（**协议栈**）
>
> 通常，**以太类型是IP协议，调用TCP/IP协议栈处理。**

***

**==协议栈==**

> 因各层协议看上去像堆叠状态，也就取名”协议栈”。 像TCP、UDP、IP等协议都是规范，而**协议栈则是实现各类协议的网络控制软件**。例如：Windows、Linux各自对协议进行了实现，因此不同系统之间能够通讯，与JVM跨平台原理一致。

**IP模块**

> 当MAC头部以太类型是IP协议时，**网卡驱动把数据包交给TCP/IP协议栈来处理**。
>
> - IP模块检查IP头部以判断数据是不是发给自己
>
> ![IP头部](https://gitee.com/wangigor/typora-images/raw/master/IP头部.png)
>
> - 判断数据包是否分片，如果分片则缓存起来**等待分片全部到达，还原数据包**。
> - 根据IP头部的协议号字段，**将包转给TCP模块或者UDP模块**。

**TCP模块**

> ![TCP报文头部结构](https://gitee.com/wangigor/typora-images/raw/master/TCP报文头部结构.png)
>
> TCP模块会根据**标志位**进行不同处理
>
> - 如果SYN=1,标识这是请求连接的包。
>   - 检查接收方端口号，**判断是否有与该端口号相同且处于等待状态的socket套接字**。
>   - 如果没有，返回错误通知的包。
>   - 如果有，则为这个**套接字复制一个副本**，将**对方IP、端口等必要信息写入套接字**，**分配用于==发送缓冲区==和==接收缓冲区==的内存空间**。
>   - 返回给客户端，客户端会再次确认，这是三次握手的一部分。
> - 如果是正常数据包，**TCP模块需要检查该包对应的套接字**。
>   - 提取出数据，存放到接收缓冲区。
>   - 如果应用程序调用socket的read（），数据就转交给应用程序。
>   - 如果应用程序不来获取数据，数据则一直保存在缓冲区。

不同协议层会逐层处理，剥洋葱一样剔除协议头部，将数据转交到上层。而发送数据时，TCP、IP等也会一层层的为数据包加上头部。

最后，数据就到应用层，**应用层通过socket来操作数据**，下面说说socket。

**socket**

> socket就是套接字。
>
> - 对应用层来说，可通过 socket 与内核中的网络协议栈通信。应用不能直接使用协议栈，更不能控制网卡驱动。所以 **socket 提供了网络编程的系统调用接口**。
> - 从Linux文件系统来说，**socket 是一个打开的文件**。
> - 从Linux内核来说，socket 是一个通信的端点(Endpoint)。
>
> 套接字中记录了通信双方的ip和端口、发送缓冲区、接收缓冲区、等待队列等

**文件描述符**

> 文件描述符（File Description）简称fd，是一个正整数，**起到一个文件索引的作用**，保存了一个指向文件的指针。
>
> 每创建或打开文件都会返回一个fd(一个数字)，其实主流操作系统将TCP/UDP连接也当做fd管理，**每新建一个连接就会返回一个fd。**

### 中断

> **中断**是cpu唤醒，知晓有网络数据到来的发起者。
>
> 计算机执行程序时，会有优先级的需求。比如，当计算机收到断电信号时（电容可以保存少许电量，供CPU运行很短的一小段时间），它应立即去保存数据，保存数据的程序具有较高的优先级。
>
> 一般而言，**由硬件产生的信号需要cpu立马做出回应（不然数据可能就丢失），所以它的优先级很高。**cpu理应中断掉正在执行的程序，去做出响应；当cpu完成对硬件的响应后，再重新执行用户程序。中断的过程如下图，和函数调用差不多。只不过函数调用是事先定好位置，而中断的位置由“信号”决定。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/中断.png)
>
> 当网卡把数据写入到内存后，**网卡向cpu发出一个中断信号，操作系统便能得知有新数据到来**，再通过**网卡中断程序**去处理数据。

### 线程阻塞和cpu

> ```java
> //创建socket
> int s = socket(AF_INET, SOCK_STREAM, 0);   
> //绑定
> bind(s, ...)
> //监听
> listen(s, ...)
> //接受客户端连接
> int c = accept(s, ...)
> //接收客户端数据
> recv(c, ...);
> //将数据打印出来
> printf(...)
> ```
>
> 这是一段最基础的网络编程代码，先新建socket对象，依次调用bind、listen、accept，最后调用recv接收数据。recv是个阻塞方法，当程序运行到recv时，它会一直等待，直到接收到数据才往下执行。
>
> **ecv、select和epoll都是阻塞方法。**

#### 阻塞原理

##### 工作队列

> 操作系统为了支持多任务，实现了线程调度的功能，会把**线程分为“运行”和“等待”等几种状态**。运行状态是线程获得cpu使用权，正在执行代码的状态；等待状**态是阻塞状态，比如上述程序运行到recv时，程序会从运行状态变为等待状态，接收到数据后又变回运行状态**。操作系统会分时执行各个运行状态的线程，由于速度很快，看上去就像是同时执行多个任务。
>
> 下图中的计算机中运行着A、B、C三个线程，其中线程A执行着上述基础网络程序，一开始，这3个线程都被操作系统的工作队列所引用，处于运行状态，会分时执行。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/工作队列.png)

##### 等待队列

> 当程序执行到recv时，操作系统会将线程A从工作队列移动到该socket的等待队列中（如下图）。由于工作队列只剩下了线程B和C，依据进程调度，cpu会轮流执行这两个线程的程序，不会执行线程A的程序。**所以线程A被阻塞，不会往下执行代码，也不会占用cpu资源**。
>
> ![img](https://gitee.com/wangigor/typora-images/raw/master/等待队列.png)
>
> 操作系统添加等待队列只是添加了对这个“等待中”线程的引用，以便在接收到数据时获取进程对象、将其唤醒，而非直接将线程管理纳入自己之下。



##### 唤醒线程

> 当socket接收到数据后，操作系统将该socket等待队列上的进程重新放回到工作队列，该进程变成运行状态，继续执行代码。也由于socket的接收缓冲区已经有了数据，recv可以返回接收到的数据。

## 

## 多路复用

### 演进过程

> 没有IO多路复用机制时，有BIO、NIO两种实现方式，但有一些问题

#### 同步阻塞（BIO）

```java
// 伪代码描述
while(1) {
  // accept阻塞
  client_fd = accept(listen_fd)
  fds.append(client_fd)
  for (fd in fds) {
    // recv阻塞（会影响上面的accept）
    if (recv(fd)) {
      // logic
    }
  }  
}
```

服务端采用单线程，当accept一个请求后，在recv或send调用阻塞时，将无法accept其他请求（必须等上一个请求处recv或send完），==无法处理并发==。

#### 多线程

```java
// 伪代码描述
while(1) {
  // accept阻塞
  client_fd = accept(listen_fd)
  // 开启线程read数据（fd增多导致线程数增多）
  new Thread func() {
    // recv阻塞（多线程不影响上面的accept）
    if (recv(fd)) {
      // logic
    }
  }  
}
```

服务器端采用多线程，当accept一个请求后，开启线程进行recv，可以完成并发处理，但随着请求数增加需要增加系统线程，==大量的线程占用很大的内存空间，并且线程切换会带来很大的开销，10000个线程真正发生读写事件的线程数不会超过20%，每次accept都开一个线程也是一种资源浪费==。

#### 同步非阻塞NIO

```java
setNonblocking(listen_fd)
// 伪代码描述
while(1) {
  // accept非阻塞（cpu一直忙轮询）
  client_fd = accept(listen_fd)
  if (client_fd != null) {
    // 有人连接
    fds.append(client_fd)
  } else {
    // 无人连接
  }  
  //当然这里可以线程池内执行 nio的worker。
  for (fd in fds) {
    // recv非阻塞
    setNonblocking(client_fd)
    // recv 为非阻塞命令
    if (len = recv(fd) && len > 0) {
      // 有读写数据
      // logic
    } else {
       无读写数据
    }
  }  
}
```

服务器端当accept一个请求后，加入fds集合，每次轮询一遍fds集合recv(非阻塞)数据，没有数据则立即返回错误，==每次轮询所有fd（包括没有发生读写事件的fd）会很浪费cpu==

#### IO多路复用

```java
fds = [listen_fd]
// 伪代码描述
while(1) {
  // 通过内核获取有读写事件发生的fd，只要有一个则返回，无则阻塞
  // 整个过程只在调用select、poll、epoll这些调用的时候才会阻塞，accept/recv是不会阻塞
  for (fd in select(fds)) {
    if (fd == listen_fd) {
        client_fd = accept(listen_fd)
        fds.append(client_fd)
    } elseif (len = recv(fd) && len != -1) { 
      // logic
    }
  }  
}
```

服务器端采用单线程通过select/epoll等系统调用获取fd列表，遍历有事件的fd进行accept/recv/send，使其==能支持更多的并发连接请求==。



### select流程

函数接口

```c
#include <sys/select.h>
#include <sys/time.h>

#define FD_SETSIZE 1024
#define NFDBITS (8 * sizeof(unsigned long))
#define __FDSET_LONGS (FD_SETSIZE/NFDBITS)

// 数据结构 (bitmap)
typedef struct {
    unsigned long fds_bits[__FDSET_LONGS];
} fd_set;

// API
int select(
    int max_fd, 
    fd_set *readset, 
    fd_set *writeset, 
    fd_set *exceptset, 
    struct timeval *timeout
)                              // 返回值就绪描述符的数目

FD_ZERO(int fd, fd_set* fds)   // 清空集合
FD_SET(int fd, fd_set* fds)    // 将给定的描述符加入集合
FD_ISSET(int fd, fd_set* fds)  // 判断指定描述符是否在集合中 
FD_CLR(int fd, fd_set* fds)    // 将给定的描述符从文件中删除 
```

使用示例

```c
int main() {
  /*
   * 这里进行一些初始化的设置，
   * 包括socket建立，地址的设置等,
   */

  fd_set read_fs, write_fs;
  struct timeval timeout;
  int max = 0;  // 用于记录最大的fd，在轮询中时刻更新即可

  // 初始化比特位
  FD_ZERO(&read_fs);
  FD_ZERO(&write_fs);

  int nfds = 0; // 记录就绪的事件，可以减少遍历的次数
  while (1) {
    // 阻塞获取
    // 每次需要把fd从用户态拷贝到内核态
    nfds = select(max + 1, &read_fd, &write_fd, NULL, &timeout);
    // 每次需要遍历所有fd，判断有无读写事件发生
    for (int i = 0; i <= max && nfds; ++i) {
      if (i == listenfd) {
         --nfds;
         // 这里处理accept事件
         FD_SET(i, &read_fd);//将客户端socket加入到集合中
      }
      if (FD_ISSET(i, &read_fd)) {
        --nfds;
        // 这里处理read事件
      }
      if (FD_ISSET(i, &write_fd)) {
         --nfds;
        // 这里处理write事件
      }
    }
  }
```

缺点

- 单个进程所打开的FD是有限制的，通过FD_SETSIZE设置，默认1024
- 每次调用select，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
- 对socket扫描时是线性扫描，采用轮询的方法，效率较低（高并发时）

### poll流程

> poll与select相比，只是没有fd的限制，其它基本一样

函数接口

```c
#include <poll.h>
// 数据结构
struct pollfd {
    int fd;                         // 需要监视的文件描述符
    short events;                   // 需要内核监视的事件
    short revents;                  // 实际发生的事件
};

// API
int poll(struct pollfd fds[], nfds_t nfds, int timeout);
```



使用示例

```c
// 先宏定义长度
#define MAX_POLLFD_LEN 4096  

int main() {
  /*
   * 在这里进行一些初始化的操作，
   * 比如初始化数据和socket等。
   */

  int nfds = 0;
  pollfd fds[MAX_POLLFD_LEN];
  memset(fds, 0, sizeof(fds));
  fds[0].fd = listenfd;
  fds[0].events = POLLRDNORM;
  int max  = 0;  // 队列的实际长度，是一个随时更新的，也可以自定义其他的
  int timeout = 0;

  int current_size = max;
  while (1) {
    // 阻塞获取
    // 每次需要把fd从用户态拷贝到内核态
    nfds = poll(fds, max+1, timeout);
    if (fds[0].revents & POLLRDNORM) {
        // 这里处理accept事件
        connfd = accept(listenfd);
        //将新的描述符添加到读描述符集合中
    }
    // 每次需要遍历所有fd，判断有无读写事件发生
    for (int i = 1; i < max; ++i) {     
      if (fds[i].revents & POLLRDNORM) { 
         sockfd = fds[i].fd
         if ((n = read(sockfd, buf, MAXLINE)) <= 0) {
            // 这里处理read事件
            if (n == 0) {
                close(sockfd);
                fds[i].fd = -1;
            }
         } else {
             // 这里处理write事件     
         }
         if (--nfds <= 0) {
            break;       
         }   
      }
    }
  }
```

缺点

- 每次调用poll，都需要把fd集合从用户态拷贝到内核态，这个开销在fd很多时会很大
- 对socket扫描时是线性扫描，采用轮询的方法，效率较低（高并发时）

### epoll流程

函数接口

```c
// 数据结构
// 每一个epoll对象都有一个独立的eventpoll结构体
// 用于存放通过epoll_ctl方法向epoll对象中添加进来的事件
// epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可
struct eventpoll {
    /*红黑树的根节点，这颗树中存储着所有添加到epoll中的需要监控的事件*/
    struct rb_root  rbr;
    /*双链表中则存放着将要通过epoll_wait返回给用户的满足条件的事件*/
    struct list_head rdlist;
};

// API

int epoll_create(int size); // 内核中间加一个 ep 对象，把所有需要监听的 socket 都放到 ep 对象中
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); // epoll_ctl 负责把 socket 增加、删除到内核红黑树
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);// epoll_wait 负责检测可读队列，没有可读 socket 则阻塞进程
```

使用示例

```c
int main(int argc, char* argv[])
{
   /*
   * 在这里进行一些初始化的操作，
   * 比如初始化数据和socket等。
   */

    // 内核中创建ep对象
    epfd=epoll_create(256);
    // 需要监听的socket放到ep中
    epoll_ctl(epfd,EPOLL_CTL_ADD,listenfd,&ev);
 
    while(1) {
      // 阻塞获取
      nfds = epoll_wait(epfd,events,20,0);
      for(i=0;i<nfds;++i) {
          if(events[i].data.fd==listenfd) {
              // 这里处理accept事件
              connfd = accept(listenfd);
              // 接收新连接写到内核对象中
              epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev);
          } else if (events[i].events&EPOLLIN) {
              // 这里处理read事件
              read(sockfd, BUF, MAXLINE);
              //读完后准备写
              epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
          } else if(events[i].events&EPOLLOUT) {
              // 这里处理write事件
              write(sockfd, BUF, n);
              //写完后准备读
              epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);
          }
      }
    }
    return 0;
}
```





- epoll有EPOLLLT和EPOLLET两种触发模式，LT是默认的模式，ET是“高速”模式。
- LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作
- ET模式下，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无论fd中是否还有数据可读。所以在ET模式下，read一个fd的时候一定要把它的buffer读完，或者遇到EAGAIN错误

### 比较

|            | select             | poll             | epoll                                             |
| :--------- | :----------------- | :--------------- | ------------------------------------------------- |
| 数据结构   | bitmap             | 数组             | 红黑树                                            |
| 最大连接数 | 1024               | 无上限           | 无上限                                            |
| fd拷贝     | 每次调用select拷贝 | 每次调用poll拷贝 | fd首次调用epoll_ctl拷贝，每次调用epoll_wait不拷贝 |
| 工作效率   | 轮询：O(n)         | 轮询：O(n)       | 回调：O(1)                                        |