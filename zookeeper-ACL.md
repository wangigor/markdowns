# Zookeeper Access Control List

***

## 概述

> ACL (Access Control List)访问控制列表。
>
> ACL 权限控制，使用：**scheme : id :  permission **来标识，主要涵盖 3 个方面：
> 　　权限模式（Scheme）：授权的策略
> 　　授权对象（ID）:授权的对象
> 　　权限（Permission）:授予的权限
>
> - ZooKeeper的权限控制是**基于每个znode节点的，需要对每个节点设置权限**
> - 每个znode支持设置多种权限控制方案和多个权限
> - **子节点不会继承父节点的权限**，客户端无权访问某节点，但可能可以访问它的子节点
>
> ==因为子节点不会继承父节点权限：在dubbo启动创建节点前，需要在zookeeper手动创建/dubbo节点，指定权限。==

### **scheme**

- **world：**默认方式，相当于全部都能访问
- **auth**：代表已经认证通过的用户(cli中可以通过**addauth digest user:pwd 来添加当前上下文中的授权用户**)
- **digest**：即用户名:密码这种方式认证，这也是业务系统中最常用的。用 *username:password* 字符串来产生一个MD5串，然后该串被用来作为ACL ID。认证是通过明文发送*username:password* 来进行的，当用在ACL时，表达式为*username:base64* ，**base64是password的SHA1摘要的编码**。
- **ip**：使用客户端的主机IP作为ACL ID 。这个ACL表达式的格式为*addr/bits* ，此时addr中的有效位与客户端addr中的有效位进行比对。

### id

> id是授权对象实体。根据scheme而定。
>
> **scheme是world：**id只有一个anyone选项。
>
> **scheme是auth：**id就是**明文的用户名:密码**
>
> **scheme是digest：**id就是**用户名:密码的sha1摘要的base64加密**
>
> **scheme是ip：**id是**ip地址或者ip段**(192.168.0.100或者192.168.0.1/24)

### permission

> **CREATE、READ、WRITE、DELETE、ADMIN** 也就是 **增、删、改、查、管理**权限，这5种权限简写为crwda
>
> **这5种权限中，delete是指对子节点的删除权限，其它4种权限指对自身节点的操作权限**

- **CREATE**  c 可以创建子节点
- **DELETE**  d 可以删除子节点（仅下一级节点）
- **READ**    r 可以读取节点数据及显示子节点列表
- **WRITE**   w 可以设置节点数据　
- **ADMIN**   a 可以设置节点访问控制列表权限

## ACL命令

- getAcl    <path>   读取ACL权限
- setAcl   <path> <acl>   设置ACL权限
- addauth   <scheme> <auth>   添加认证用户

## zkCli测试

### word方式

```shell
[zk: localhost:2181(CONNECTED) 14] ls /
[dubbo, zookeeper]
[zk: localhost:2181(CONNECTED) 15] create /test-world # 创建节点默认为world:anyone:cdrwa
Created /test-world
[zk: localhost:2181(CONNECTED) 16] getAcl /test-world
'world,'anyone
: cdrwa
[zk: localhost:2181(CONNECTED) 17] setAcl /test-world world:anyone:cdr # 修改权限
[zk: localhost:2181(CONNECTED) 18] getAcl /test-world
'world,'anyone
: cdr
[zk: localhost:2181(CONNECTED) 19] deleteall /test-world # 删除节点
```

### IP方式

> 非固定ip不能访问

```shell
[zk: localhost:2181(CONNECTED) 20] create /test-ip
Created /test-ip
# 多个ip需要逗号分隔：ip:192.168.8.100:cdwra,ip:127.0.0.1:cdwra
[zk: localhost:2181(CONNECTED) 21] setAcl /test-ip ip:127.0.0.1:cdrwa # only本机
[zk: localhost:2181(CONNECTED) 22] getAcl /test-ip
'ip,'127.0.0.1
: cdrwa
```

### auth方式

> 非用户名密码，不得访问

```shell
[zk: localhost:2181(CONNECTED) 29] create /test-auth
Created /test-auth
#添加授权用户，明文账号密码
#且增加权限到当前连接上下文中
[zk: localhost:2181(CONNECTED) 30] addauth digest test:test
#授予权限
[zk: localhost:2181(CONNECTED) 31] setAcl /test-auth auth:test:cdrwa
[zk: localhost:2181(CONNECTED) 32] getAcl /test-auth
'digest,'test:V28q/NynI4JI3Rk54h0r8O5kMug=
: cdrwa
```

### digest方式

> 其实跟auth方式完全一样。只是密码需要手动进行sha1摘要的base64编码。

可以使用linux命令进行密码加密输出

```shell
oot@e0cfab68c7e7:/apache-zookeeper-3.5.6-bin# echo -n test:test | openssl dgst -binary -sha1 | openssl base64
V28q/NynI4JI3Rk54h0r8O5kMug=
```

也可以使用程序加密

```java
String usernameAndPassword = "test:test";
byte digest[] = MessageDigest.getInstance("SHA1").digest(usernameAndPassword.getBytes());
Base64 base64 = new Base64();
String encodeToString = base64.encodeToString(digest);
log.info(encodeToString) //加密结果
```

只是授权命令有一点点变化

```shell
[zk: localhost:2181(CONNECTED) 0] create /test-digest
Created /test-digest
# 手动授权
[zk: localhost:2181(CONNECTED) 1] setAcl /test-digest digest:test:V28q/NynI4JI3Rk54h0r8O5kMug=:cdrwa
# 无访问权限
[zk: localhost:2181(CONNECTED) 2] getAcl /test-digest
Authentication is not valid : /test-digest
# 权限认证
[zk: localhost:2181(CONNECTED) 3] addauth digest test:test
# 权限跟auth完全相同
[zk: localhost:2181(CONNECTED) 4] getAcl /test-digest
'digest,'test:V28q/NynI4JI3Rk54h0r8O5kMug=
: cdrwa
```

## dubbo配置

yml配置

```properties
dubbo.registry.username=test
dubbo.registry.password=test
```

xml配置

```xml
<dubbo:registry protocol ="zookeeper" address="127.0.0.1:2181"  username="test" password="test"/>
```

都可以。

