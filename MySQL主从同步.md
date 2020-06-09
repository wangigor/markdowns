# MySQL主从同步

> 为啥要选择主从同步
>
> - 数据热备份。可以作为后备数据库，主库故障，切换从库。
> - 读写分离。

## binlog复制原理

![mysql主从复制原理图](https://gitee.com/wangigor/typora-images/raw/master/mysql主从复制原理图.png)

Binary log：主库master的二进制文件。【binlog】

Relay log：从库slave的中继日志。

- 第一步：主库master在每个事物完成数据更新的时候，将该记录串行的写入binlog日志中。
- 第二步：从库slave开始同步【slave start；】的时候，开启一个I/O线程【打开主库master连接，握手，交换binlog位置和position】。该线程把记录写入从库的relay log中；如果进度已经赶上了master，进入休眠状态【Waiting for master to send event】等待master的新事件。
- 第三步：从库slave的线程，读取中继日志relay log，顺序执行sql事件，保证与主库数据一致。

## 主从配置

### 前置条件

- 本机环境 : mac+docker+docker-compose
- 主从库账号密码：root -> 123456
- Host: localhost
- 端口映射：主库33060  从库33065
- 工作目录：~/docker/mysql
- MySQL版本：5.7

### 创建docker-compose.yml

> 进入~/docker/mysql目录下。

```yaml
version: '3'
services:
  mysql-master00:
    image: mysql:5.7
    container_name: mysql-master00
    environment:
      - "MYSQL_ROOT_PASSWORD=123456"
    volumes:
      - "./master00/data:/var/lib/mysql"
      - "./master00/my.cnf:/etc/mysql/my.cnf"
      - "./master00/log:/var/log/mysql"
    links:
      - mysql-slave00
    ports:
      - "33060:3306"
    restart: always
    hostname: mysql-master00
  mysql-slave00:
    image: mysql:5.7
    container_name: mysql-slave00
    environment:
      - "MYSQL_ROOT_PASSWORD=123456"
    volumes:
      - "./slave00/data:/var/lib/mysql"
      - "./slave00/log:/var/log/mysql"
      - "./slave00/my.cnf:/etc/mysql/my.cnf"
    ports:
      - "33065:3306"
    restart: always
    hostname: mysql-slave00
```

### 编辑主从库配置文件my.cnf

> 注意：5.7版本没有这样的属性【default-character-set=utf8】 。用character-set-server=utf8代替。

- master的my.cnf

```properties
[client]
#default-character-set=utf8 5.7版本没有这样的属性 用character-set-server=utf8代替

[mysql]
#default-character-set=utf8

[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
#跳过名称解析，不然连接会很慢。
skip-name-resolve

#设置server_id,唯一。可以设置为ip，但是不能是字符开头。
server_id=1 
#开启二进制日志功能
log-bin=mysql-bin
#不开启只读
read_only=0
#配置需要主从同步的数据库。
binlog-do-db=test
#binlog-do-db=test1

#过滤
replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema

#【可选】 为每个session分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
#【可选】 主从复制的格式（mixed,statement,row，默认格式是statement。建议是设置为row，主从复制时数据更加能够统一）
binlog_format=row
#【可选】 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
#【可选】 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
# 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
```

- slave的my.cnf

```properties
[client]
#default-character-set=utf8

[mysql]
#default-character-set=utf8

[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
skip-name-resolve

#设置server_id,唯一。可以设置为ip，但是不能是字符开头。
server_id=2
#开启二进制日志功能。为了其他从节点进行复制操作
log_bin=mysql-bin
#设置只读
read_only=1
binlog-do-db=test

replicate-ignore-db=mysql
replicate-ignore-db=sys
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema

#【可选】 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M
#【可选】 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=row
#【可选】 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7
#【可选】 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
# 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
#【可选】 relay_log配置中继日志
relay_log=replicas-mysql-relay-bin
# log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates=1
```

### 启动mysql服务

```shell
docker-compose up -d
```

### master数据准备

```sql
# 建表
CREATE TABLE `tb_test` (
  `name` varchar(255) DEFAULT NULL,
  `id` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
# 插入数据
INSERT INTO tb_test (id,name) value (1,"test01");
INSERT INTO tb_test (id,name) value (2,"test02");
INSERT INTO tb_test (id,name) value (3,"test03");
```

查看主库状态

```sql
show master status;
```

![mysql主从同步-主库状态](https://gitee.com/wangigor/typora-images/raw/master/mysql主从同步-主库状态.png)

记录这里的file和position。

### slave配置

执行命令

```sql
change master to master_host='mysql-master00',master_port=3306,master_user='root',master_password='123456',master_log_file='mysql-bin.000006',master_log_pos=0;
```

- master_host：主库host/ip。【我这里配置了容器的hostname，就直接使用了。】
- master_port：主库端口。【要是走外网就填映射端口。】
- master_user：主库用户名。
- master_password：主库密码
- master_log_file：master status的file参数
- master_log_pos：position参数。【注意，如果填当前的位置，存量数据不会同步。可以直接填0，自动从建库开始。】

### 开启同步

从库执行

```sql
start slave;
```

查看从库同步状态

![](https://gitee.com/wangigor/typora-images/raw/master/mysql从库同步状态.png)

### 同步关闭

```sql
stop slave;
```

## 注意事项

### 从库的只读问题

my.cnf中配置了read-only【只读】。

对super用户不生效。也就是root用户依旧可以修改。

1）使用普通用户【已验证】

```sql
grant select,insert,update,delete on *.* to 'test'@'%' identified by '123456';
flush privileges;
```

2) 设置super用户只读【已验证】

```sql
SET GLOBAL super_read_only=1;
```



