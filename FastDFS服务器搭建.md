# FastDFS服务器搭建

> 开源项目地址：https://github.com/happyfish100

FastDFS 是一个开源的高性能分布式文件系统（DFS）。 它的主要功能包括：文件存储，文件同步和文件访问，以及高容量和负载平衡。主要解决了海量数据存储问题，特别适合以中小文件（建议范围：4KB < file_size <500MB）为载体的在线服务。

## FastDFS架构

- Tracker Server（跟踪服务器）

  跟踪服务器，主要做**==调度==**工作，起到均衡的作用；负责管理所有的 storage server和 group，每个 storage 在启动后会连接 Tracker，告知自己所属 group 等信息，并保持周期性==**心跳**==。通过Trackerserver在文件上传时可以根据一些策略找到Storageserver提供文件上传服务。

- Storage Server（存储服务器）

存储服务器，主要提供==**容量**==和==**备份**==服务；以 group 为单位，每个 group 内可以有多台 storage server，数据互为备份。<u>Storage server没有实现自己的文件系统而是利用操作系统 的文件系统来管理文件</u>。

![img](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1583299482858&di=7be84bd438677f119ed2ce93af7669b4&imgtype=jpg&src=http%3A%2F%2Fimg1.imgtn.bdimg.com%2Fit%2Fu%3D4156961061%2C1162815830%26fm%3D214%26gp%3D0.jpg)

## Tracker集群

​		没有集群。FastDFS集群中的Tracker server可以有多台，Trackerserver之间是相互==**平等**==关系同时提供服务，Trackerserver不存在单点故障。客户端请求Trackerserver采用轮询方式，如果请求的tracker无法提供服务则换另一个tracker。

## Storage集群

为了支持大容量，存储节点（服务器）采用了分卷（或分组）的组织方式。存储系统由一个或多个卷组成，卷与卷之间的文件是相互独立的，所有卷的文件容量累加就是整个存储系统中的文件容量。一个卷由一台或多台存储服务器组成，卷内的Storage server之间是平等关系，不同卷的Storageserver之间不会相互通信，==同卷内的Storageserver之间会相互连接进行文件同步，从而保证同组内每个storage上的文件完全一致的。==一个卷的存储容量为该组内存储服务器容量最小的那个，由此可见组内存储服务器的软硬件配置最好是一致的。卷中的多台存储服务器起到了冗余备份和负载均衡的作用

在卷中增加服务器时，同步已有的文件由系统自动完成，同步完成后，系统自动将新增服务器切换到线上提供服务。当存储空间不足或即将耗尽时，可以动态添加卷。只需要增加一台或多台服务器，并将它们配置为一个新的卷，这样就扩大了存储系统的容量。

采用分组存储方式的好处是灵活、可控性较强。比如上传文件时，可以由客户端直接指定上传到的组也可以由tracker进行调度选择。一个分组的存储服务器访问压力较大时，可以在该组增加存储服务器来扩充服务能力（纵向扩容）。当系统容量不足时，可以增加组来扩充存储容量（横向扩容）。

## 文件上传

![img](https://img-blog.csdn.net/20180516113258902?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0thbVJvc2VMZWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 文件索引信息

![img](https://img-blog.csdn.net/20180516113315796?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0thbVJvc2VMZWU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

- 组名（group1）：文件上传后所在的storage组名称，在文件上传成功后有storage服务器返回，需要客户端自行保存。

- 虚拟磁盘路径（M00）：storage配置的虚拟路径，与磁盘选项store_path*对应。如果配置了store_path0则是M00，如果配置了store_path1则是M01，以此类推。

- 数据两级目录（02/44）：storage服务器在每个虚拟磁盘路径下创建的两级目录，用于存储数据文件。

- 文件名：与文件上传时不同。是由存储服务器根据特定信息生成，文件名包含：源存储服务器IP地址、文件创建时间戳、文件大小、随机数和文件拓展名等信息。
  

## 文件同步

写文件时，客户端将文件写至group内一个storage server即认为写文件成功，storage server写完文件后，会由后台线程将文件同步至同group内其他的storage server。

每个storage写文件后，同时会写一份binlog，binlog里不包含文件数据，只包含文件名等元信息，这份binlog用于后台同步，storage会记录向group内其他storage同步的进度，以便重启后能接上次的进度继续同步；进度以时间戳的方式进行记录，所以最好能保证集群内所有server的时钟保持同步。

storage的同步进度会作为元数据的一部分汇报到tracker上，tracke在选择读storage的时候会以同步进度作为参考。

## 安装

以centos7为例

### 安装GCC和libevent

```shell
yum -y install gcc-c++
yum -y install libevent
```

### 安装libfastcommon

libfastcommon是FastDFS官方提供的，libfastcommon包含了FastDFS运行所需要的一些基础库。

> 地址：https://github.com/happyfish100/libfastcommon/releases 选择版本
>
> 当时最新版：
>
> fastdfs-6.06.tar.gz
>
> libfastcommon-1.0.43.tar.gz

```shell
[root@iZbp1bsithym5g2chjgu63Z up]# cd /usr/local/src/
[root@iZbp1bsithym5g2chjgu63Z src]# tar -zxvf libfastcommon-1.0.43.tar.gz
[root@iZbp1bsithym5g2chjgu63Z src]# cd libfastcommon-1.0.43
[root@iZbp1bsithym5g2chjgu63Z libfastcommon-1.0.43]# ./make.sh
[root@iZbp1bsithym5g2chjgu63Z libfastcommon-1.0.43]# ./make.sh install
```

### 安装fastDFS

> https://github.com/happyfish100/fastdfs/releases 
>
> 跟libfastcommon一样的去/usr/local/src下，解压，安装。

```shell
[root@iZbp1bsithym5g2chjgu63Z bin]# cd /usr/local/src/
[root@iZbp1bsithym5g2chjgu63Z src]# tar -zxvf fastdfs-6.06.tar.gz 
[root@iZbp1bsithym5g2chjgu63Z src]# cd fastdfs-6.06
[root@iZbp1bsithym5g2chjgu63Z fastdfs-6.06]# ./make.sh
[root@iZbp1bsithym5g2chjgu63Z fastdfs-6.06]# ./make.sh install
```

![](/Users/wangke/Documents/markdown/file/WX20200304-114958@2x.png)

安装了一些默认的文件和目录：

 - 服务脚本：

   > /etc/init.d/fdfs_storaged
   > /etc/init.d/fdfs_trackerd

- 配置文件样例：

  > /etc/fdfs/client.conf.sample
  > /etc/fdfs/storage.conf.sample
  > /etc/fdfs/tracker.conf.sample

- 命令工具在 /usr/bin/ 目录下:

  > fdfs_appender_test
  > fdfs_appender_test1
  > fdfs_append_file
  > fdfs_crc32
  > fdfs_delete_file
  > fdfs_download_file
  > fdfs_file_info
  > fdfs_monitor
  > fdfs_storaged
  > fdfs_test
  > fdfs_test1
  > fdfs_trackerd
  > fdfs_upload_appender
  > fdfs_upload_file
  > stop.sh
  > restart.sh

### 配置tracker

[FastDFS配置文件详解](http://bbs.chinaunix.net/forum.php?mod=viewthread&tid=1941456&extra=page%3D1%26filter%3Ddigest%26digest%3D1)

- 进入配置文件地址/etc/fdfs，修改案例。

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z fastdfs-6.06]# cd /etc/fdfs/
  [root@iZbp1bsithym5g2chjgu63Z fdfs]# cp tracker.conf.sample tracker.conf
  
  ```

> #配置文件是否不生效，false 为生效
> disabled=false
>
> #提供服务的端口
>
> port=22122
>
> #Tracker 数据和日志目录地址(根目录必须存在,子目录会自动创建)
>
> base_path=/home/up/fastdfs/tracker
>
> #HTTP 服务端口 默认8080 ，建议修改 防止冲突
>
> http.server_port=9080
> 

- 创建基础目录

```shell
[root@iZbp1bsithym5g2chjgu63Z fdfs]# mkdir -p /home/up/fastdfs/tracker
```

- 开启防火墙22122端口

- 启动

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z sbin]# /etc/init.d/fdfs_trackerd start
  [root@iZbp1bsithym5g2chjgu63Z sbin]# service fdfs_trackerd start
  [root@iZbp1bsithym5g2chjgu63Z sbin]# systemctl start fdfs_trackerd
  ```
  
- 启动检查

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z sbin]# systemctl status fdfs_trackerd
  ● fdfs_trackerd.service - LSB: FastDFS tracker server
     Loaded: loaded (/etc/rc.d/init.d/fdfs_trackerd; bad; vendor preset: disabled)
     Active: active (running) since Wed 2020-03-04 14:48:33 CST; 25s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 6240 ExecStart=/etc/rc.d/init.d/fdfs_trackerd start (code=exited, status=0/SUCCESS)
      Tasks: 7
     Memory: 4.4M
     CGroup: /system.slice/fdfs_trackerd.service
             └─6245 /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf
  
  Mar 04 14:48:33 iZbp1bsithym5g2chjgu63Z systemd[1]: Starting LSB: FastDFS tracker server...
  Mar 04 14:48:33 iZbp1bsithym5g2chjgu63Z fdfs_trackerd[6240]: Starting FastDFS tracker server:
  Mar 04 14:48:33 iZbp1bsithym5g2chjgu63Z systemd[1]: Started LSB: FastDFS tracker server.
  [root@iZbp1bsithym5g2chjgu63Z sbin]# netstat -anp |grep fdfs
  tcp        0      0 0.0.0.0:22122           0.0.0.0:*               LISTEN      6245/fdfs_trackerd  
  [root@iZbp1bsithym5g2chjgu63Z sbin]# 
  ```

- 关闭

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z sbin]# service fdfs_trackerd stop
  [root@iZbp1bsithym5g2chjgu63Z sbin]# systemctl stop fdfs_trackerd #centos7 推荐
  [root@iZbp1bsithym5g2chjgu63Z sbin]# /etc/init.d/fdfs_trackerd stop
  ```

- 设置开机自启

  > \# chkconfig fdfs_trackerd on
  > 或
  > \#systemctl enable fdfs_trackerd.service
  > 或者：
  > \# vim /etc/rc.d/rc.local
  > 加入配置：
  > /etc/init.d/fdfs_trackerd start

```shell
[root@iZbp1bsithym5g2chjgu63Z sbin]# systemctl enable fdfs_trackerd.service
```

### 配置storage

- 进入配置文件地址/etc/fdfs，修改案例。

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z sbin]# cd /etc/fdfs/
  [root@iZbp1bsithym5g2chjgu63Z fdfs]# cp storage.conf.sample storage.conf
  ```

- 修改配置

  > #配置文件是否不生效，false 为生效
  > disabled=false 
  >
  > #指定此 storage server 所在 组(卷)
  >
  > group_name=group1
  >
  > #storage server 服务端口
  >
  > port=23000
  >
  > #心跳间隔时间，单位为秒 (这里是指主动向 tracker server 发送心跳)
  >
  > heart_beat_interval=30
  >
  > #Storage 数据和日志目录地址(根目录必须存在，子目录会自动生成)  (注 :这里不是上传的文件存放的地址,之前版本是的,在某个版本后更改了)
  >
  > base_path=/home/up/fastdfs/storage/base
  >
  > #存放文件时 storage server 支持多个路径。这里配置存放文件的基路径数目，通常只配一个目录。
  >
  > store_path_count=1
  >
  > #逐一配置 store_path_count 个路径，索引号基于 0。
  >
  > #如果不配置 store_path0，那它就和 base_path 对应的路径一样。
  >
  > store_path0=/home/up/fastdfs/storage
  >
  > #FastDFS 存储文件时，采用了两级目录。这里配置存放文件的目录个数。 
  >
  > #如果本参数只为 N（如： 256），那么 storage server 在初次运行时，会在 store_path 下自动创建 N * N 个存放文件的子目录。
  >
  > subdir_count_per_path=256
  >
  > #tracker_server 的列表 ，会主动连接 tracker_server
  >
  > #有多个 tracker server 时，每个 tracker server 写一行
  >
  > #tracker_server=192.168.0.200:22122
  > tracker_server=127.0.0.1:22122
  >
  > #允许系统同步的时间段 (默认是全天) 。一般用于避免高峰同步产生一些问题而设定。
  >
  > sync_start_time=00:00
  > sync_end_time=23:59
  >
  > #访问端口 默认80  建议修改 防止冲突
  >
  > http.server_port=9888

- 创建目录

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z fdfs]# mkdir -p /home/up/fastdfs/storage/base
  ```

- 服务器开启storage服务端口23000

- 启动

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z storage]# systemctl status fdfs_storaged
  ```

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z storage]# systemctl status fdfs_storaged
  ● fdfs_storaged.service - LSB: FastDFS storage server
     Loaded: loaded (/etc/rc.d/init.d/fdfs_storaged; bad; vendor preset: disabled)
     Active: active (running) since Wed 2020-03-04 15:20:48 CST; 6s ago
       Docs: man:systemd-sysv-generator(8)
    Process: 9890 ExecStart=/etc/rc.d/init.d/fdfs_storaged start (code=exited, status=0/SUCCESS)
      Tasks: 9
     Memory: 64.8M
     CGroup: /system.slice/fdfs_storaged.service
             └─9895 /usr/bin/fdfs_storaged /etc/fdfs/storage.conf
  
  Mar 04 15:20:48 iZbp1bsithym5g2chjgu63Z systemd[1]: Starting LSB: FastDFS storage server...
  Mar 04 15:20:48 iZbp1bsithym5g2chjgu63Z fdfs_storaged[9890]: Starting FastDFS storage server:
  Mar 04 15:20:48 iZbp1bsithym5g2chjgu63Z systemd[1]: Started LSB: FastDFS storage server.
  [root@iZbp1bsithym5g2chjgu63Z storage]# netstat -anp |grep fdfs
  tcp        0      0 0.0.0.0:22122           0.0.0.0:*               LISTEN      6245/fdfs_trackerd  
  tcp        0      0 0.0.0.0:23000           0.0.0.0:*               LISTEN      9895/fdfs_storaged
  ```

- 设置开机自启

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z storage]# systemctl enable fdfs_trackerd
  ```

  > chkconfig fdfs_storaged on
  > 或
  > systemctl enable fdfs_storaged.service  （推荐）
  >
  > 或者：
  >
  > vim /etc/rc.d/rc.local
  >
  > 加入配置：
  >
  > /etc/init.d/fdfs_storaged  start

- 查看通信情况

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z storage]# fdfs_monitor /etc/fdfs/storage.conf
  [2020-03-04 15:25:52] DEBUG - base_path=/home/up/fastdfs/storage/base, connect_timeout=5, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=1, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0
  
  server_count=1, server_index=0
  
  tracker server is 118.31.45.225:22122
  
  group count: 1
  
  Group 1:
  group name = group1
  disk total space = 40,183 MB
  disk free space = 25,867 MB
  trunk free space = 0 MB
  storage server count = 1
  active server count = 1
  storage server port = 23000
  storage HTTP port = 9888
  store path count = 1
  subdir count per path = 256
  current write server index = 0
  current trunk file id = 0
  
          Storage 1:
                  id = 118.31.45.225
                  ip_addr = 118.31.45.225  ACTIVE
                  http domain = 
                  version = 6.06
                  join time = 2020-03-04 15:17:17
                  up time = 2020-03-04 15:20:48
                  total storage = 40,183 MB
                  free storage = 25,867 MB
                  upload priority = 10
                  store_path_count = 1
                  subdir_count_per_path = 256
                  storage_port = 23000
                  storage_http_port = 9888
                  current_write_path = 0
                  source storage id = 
                  if_trunk_server = 0
                  connection.alloc_count = 256
                  connection.current_count = 0
                  connection.max_count = 0
                  total_upload_count = 0
                  success_upload_count = 0
                  total_append_count = 0
                  success_append_count = 0
                  total_modify_count = 0
                  success_modify_count = 0
                  total_truncate_count = 0
                  success_truncate_count = 0
                  total_set_meta_count = 0
                  success_set_meta_count = 0
                  total_delete_count = 0
                  success_delete_count = 0
                  total_download_count = 0
                  success_download_count = 0
                  total_get_meta_count = 0
                  success_get_meta_count = 0
                  total_create_link_count = 0
                  success_create_link_count = 0
                  total_delete_link_count = 0
                  success_delete_link_count = 0
                  total_upload_bytes = 0
                  success_upload_bytes = 0
                  total_append_bytes = 0
                  success_append_bytes = 0
                  total_modify_bytes = 0
                  success_modify_bytes = 0
                  stotal_download_bytes = 0
                  success_download_bytes = 0
                  total_sync_in_bytes = 0
                  success_sync_in_bytes = 0
                  total_sync_out_bytes = 0
                  success_sync_out_bytes = 0
                  total_file_open_count = 0
                  success_file_open_count = 0
                  total_file_read_count = 0
                  success_file_read_count = 0
                  total_file_write_count = 0
                  success_file_write_count = 0
                  last_heart_beat_time = 2020-03-04 15:25:48
                  last_source_update = 1970-01-01 08:00:00
                  last_sync_update = 1970-01-01 08:00:00
                  last_synced_timestamp = 1970-01-01 08:00:00 
  [root@iZbp1bsithym5g2chjgu63Z storage]# 
  ```

  

## 上传测试

### 服务器端测试

- 修改client的样例配置文件

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z storage]# cd /etc/fdfs/
  [root@iZbp1bsithym5g2chjgu63Z fdfs]# cp client.conf.sample client.conf
  [root@iZbp1bsithym5g2chjgu63Z fdfs]# vim client.conf
  ```

  > #Client 的数据和日志目录
  > base_path=/home/up/fastdfs/client
  > #Tracker端口
  >
  > tracker_server=118.31.45.225:22122

```shell
[root@iZbp1bsithym5g2chjgu63Z fdfs]# mkdir -p /home/up/fastdfs/client
```

- 测试

  ```shell
  [root@iZbp1bsithym5g2chjgu63Z up]# cat > /home/up/test.txt <<EOF 
  > hello fastDFS!
  > EOF
  [root@iZbp1bsithym5g2chjgu63Z up]# /usr/bin/fdfs_upload_file /etc/fdfs/client.conf /home/up/test.txt 
  group1/M00/00/00/dh8t4V5fZXyAJRc_AAAADwjNANM665.txt
  [root@iZbp1bsithym5g2chjgu63Z up]# 
  ```

### java测试类

- spring的容器xml

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:context="http://www.springframework.org/schema/context"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">
  
      <!--配置扫描包-->
      <context:component-scan base-package="com.github.tobato.fastdfs.service,com.github.tobato.fastdfs.domain"/>
      <!--配置连接管理器-->
      <bean id="trackerConnectionManager" class="com.github.tobato.fastdfs.conn.TrackerConnectionManager">
          <constructor-arg name="pool" ref="fdfsConnectionPool">
          </constructor-arg>
          <!--配置fastDFS tracker 服务器 ip:port 地址-->
          <property name="trackerList">
              <list>
                  <value>118.31.45.225:22122</value>
              </list>
          </property>
      </bean>
      <!--配置连接池-->
      <bean id="fdfsConnectionPool" class="com.github.tobato.fastdfs.conn.FdfsConnectionPool">
          <!--注入连接池配置-->
          <constructor-arg name="config">
              <bean class="com.github.tobato.fastdfs.conn.ConnectionPoolConfig"/>
          </constructor-arg>
          <!--注入连接池工厂-->
          <constructor-arg name="factory">
              <bean class="com.github.tobato.fastdfs.conn.PooledConnectionFactory"/>
          </constructor-arg>
      </bean>
      <bean id="fileUtils" class="com.uteam.zen.core.utils.FileUtils"/>
  </beans>
  ```

- java测试类

  ```java
  package com.uteam.zen.base.web;
  
  import com.github.tobato.fastdfs.domain.StorePath;
  import com.github.tobato.fastdfs.service.FastFileStorageClient;
  import com.uteam.zen.core.utils.FileUtils;
  import lombok.extern.slf4j.Slf4j;
  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.test.context.ContextConfiguration;
  import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
  import org.w3c.dom.Document;
  
  import javax.xml.parsers.ParserConfigurationException;
  import java.io.ByteArrayInputStream;
  import java.io.File;
  import java.io.FileInputStream;
  import java.io.IOException;
  
  @RunWith(SpringJUnit4ClassRunner.class)
  @ContextConfiguration(locations = {"classpath:applicationContext.xml"})
  @Slf4j
  public class FdfsTest {
      @Autowired
      FastFileStorageClient fastFileStorageClient;
  
      @Autowired
      private FileUtils fileUtils;
  
      @Test
      public void testFdfsContent() throws IOException, ParserConfigurationException {
          String filePath = "/Users/wangke/Documents/测试.doc";
          File file = new File(filePath);
          Document document = fileUtils.file2Document(new FileInputStream(file), (name, content) -> {
              log.info(name);
              log.info(content.toString());
              ByteArrayInputStream stream = new ByteArrayInputStream(content);
              StorePath storePath = fastFileStorageClient.uploadFile(stream, content.length, name, null);
              String filePath_new = getResAccessUrl(storePath);
              log.info("图片地址：" + filePath_new);
              return filePath_new;
          });
  
      }
  
      // 封装图片完整URL地址
      private String getResAccessUrl(StorePath storePath) {
          String fileUrl = "http://XXXXXXX:XXXX/" + storePath.getFullPath();
          return fileUrl;
      }
  
  }
  
  ```

- 测试

  <img src="/Users/wangke/Documents/markdown/file/WX20200304-163007@2x.png" style="zoom:200%;" />

## 配置nginx

- 传统的方式这里不讲
> FastDFS 通过 Tracker 服务器，将文件放在 Storage 服务器存储， 但是同组存储服务器之间需要进行文件复制， 有同步延迟的问题。
> 假设 Tracker 服务器将文件上传到了 192.168.51.128，上传成功后文件 ID已经返回给客户端。
> 此时 FastDFS 存储集群机制会将这个文件同步到同组存储 192.168.51.129，在文件还没有复制完成的情况下，客户端如果用这个文件 ID 在 192.168.51.129 上取文件,就会出现文件无法访问的错误。
> 而 fastdfs-nginx-module 可以重定向文件链接到源服务器取文件，避免客户端由于**==复制延迟==**导致的文件无法访问错误。

### 下载fastdfs-nginx-module模块

```shell
[root@iZbp1bsithym5g2chjgu63Z fastdfs]# wget -O fastdfs-nginx-module-1.20.tar.gz  https://codeload.github.com/happyfish100/fastdfs-nginx-module/tar.gz/V1.20
[root@iZbp1bsithym5g2chjgu63Z fastdfs]# ls
client  fastdfs-nginx-module-1.20.tar.gz  storage  tracker
[root@iZbp1bsithym5g2chjgu63Z fastdfs]# tar -zxvf fastdfs-nginx-module-1.20.tar.gz 
fastdfs-nginx-module-1.20/
fastdfs-nginx-module-1.20/HISTORY
fastdfs-nginx-module-1.20/INSTALL
fastdfs-nginx-module-1.20/src/
fastdfs-nginx-module-1.20/src/common.c
fastdfs-nginx-module-1.20/src/common.h
fastdfs-nginx-module-1.20/src/config
fastdfs-nginx-module-1.20/src/mod_fastdfs.conf
fastdfs-nginx-module-1.20/src/ngx_http_fastdfs_module.c
[root@iZbp1bsithym5g2chjgu63Z fastdfs]#
```

### 修改src/config文件

```c
ngx_addon_name=ngx_http_fastdfs_module

if test -n "${ngx_module_link}"; then
    ngx_module_type=HTTP
    ngx_module_name=$ngx_addon_name
    ngx_module_incs="/usr/local/include"
    ngx_module_libs="-lfastcommon -lfdfsclient"
    ngx_module_srcs="$ngx_addon_dir/ngx_http_fastdfs_module.c"
    ngx_module_deps=
    CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
    . auto/module
else
    HTTP_MODULES="$HTTP_MODULES ngx_http_fastdfs_module"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_fastdfs_module.c"
    CORE_INCS="$CORE_INCS /usr/local/include"
    CORE_LIBS="$CORE_LIBS -lfastcommon -lfdfsclient"
    CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
fi
```

> 修改内容：
>
> ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"
> CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"

### 配置nginx

```shell
[root@iZbp1bsithym5g2chjgu63Z nginx]# /usr/local/nginx/sbin/nginx -s stop
[root@iZbp1bsithym5g2chjgu63Z nginx]# cd /usr/local/src/nginx-1.16.0
[root@iZbp1bsithym5g2chjgu63Z nginx-1.16.0]# ./configure --add-module=/home/up/fastdfs/fastdfs-nginx-module-1.22/src
Configuration summary
  + using system PCRE library
  + OpenSSL library is not used
  + using system zlib library

  nginx path prefix: "/usr/local/nginx"
  nginx binary file: "/usr/local/nginx/sbin/nginx"
  nginx modules path: "/usr/local/nginx/modules"
  nginx configuration prefix: "/usr/local/nginx/conf"
  nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/nginx/logs/nginx.pid"
  nginx error log file: "/usr/local/nginx/logs/error.log"
  nginx http access log file: "/usr/local/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"

[root@iZbp1bsithym5g2chjgu63Z nginx-1.16.0]# make && make install
[root@iZbp1bsithym5g2chjgu63Z nginx-1.16.0]# /usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.16.0
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-39) (GCC) 
configure arguments: --add-module=/home/up/fastdfs/fastdfs-nginx-module-1.22/src
```

复制 fastdfs-nginx-module 源码中的配置文件 mod_fastdfs.conf 到/etc/fdfs 目录

```shell
[root@iZbp1bsithym5g2chjgu63Z nginx-1.16.0]# cp /home/up/fastdfs/fastdfs-nginx-module-1.22/src/mod_fastdfs.conf /etc/fdfs/
[root@iZbp1bsithym5g2chjgu63Z nginx-1.16.0]# vim /etc/fdfs/mod_fastdfs.conf
```



> #连接超时时间
> connect_timeout=10
>
> #Tracker Server
>
> tracker_server=118.31.45.225:22122
>
> #StorageServer 默认端口
>
> storage_server_port=23000
>
> #如果文件ID的uri中包含/group**，则要设置为true
>
> #url_have_group_name = true
>
> #Storage 配置的store_path0路径，必须和storage.conf中的一致
>
> store_path0=/home/up/fastdfs/storage

复制 FastDFS 的部分配置文件到/etc/fdfs 目录

```shell
[root@iZbp1bsithym5g2chjgu63Z nginx-1.16.0]# cd /usr/local/src/fastdfs-6.06/conf
[root@iZbp1bsithym5g2chjgu63Z conf]# cp anti-steal.jpg http.conf mime.types /etc/fdfs/
```

配置nginx，修改nginx.conf

```conf
location ~/group[0-9]/ {
	ngx_fastdfs_module;
}
```

重启nginx；

## 文件鉴权

fastDFS支持token方式的鉴权；暂时没有用到。

## 发布

目前使用物理机发布，后面会做发布改进：

第二版使用docker。

第三版使用k8s。