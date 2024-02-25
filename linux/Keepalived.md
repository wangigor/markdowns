# Keepalived

Keepalived是一个用于Linux操作系统的路由软件，主要用于提高服务器的高可用性（High Availability, HA）。它通过使用标准的VRRP（虚拟路由冗余协议）实现服务的高可用。Keepalived可以监控服务器上的服务或系统的状态，当检测到主服务器出现问题时，能够自动将请求切换到备用服务器上，确保服务的连续可用性。

Keepalived的主要功能包括：

1. **高可用性（HA）：** Keepalived通过VRRP协议支持IP地址的热备份，能够在服务器间实现自动故障转移。
2. **负载均衡：** 除了高可用性之外，Keepalived还支持LVS（Linux Virtual Server）负载均衡，可以根据配置的规则将客户端请求分发到多个服务器上。
3. **健康检查：** Keepalived可以对后端的服务或服务器进行健康检查，确保流量只会被转发到健康的服务器上。
4. **灵活的配置：** Keepalived允许用户通过配置文件灵活地设置VRRP以及健康检查的各种参数。

Keepalived在网络服务和云计算领域中被广泛使用，特别适合需要高可用性和负载均衡的环境，比如Web服务、数据库服务和云服务等。通过Keepalived，可以有效地避免单点故障，提高服务的稳定性和可靠性。

> **VRRP**（虚拟路由器冗余协议）本质上是一个协议规范，定义了在局域网内实现路由器高可用性的一种机制。它是一个标准化的协议，由IETF在RFC 5798文档中定义。VRRP允许多个路由器共享一个虚拟IP地址，其中一台路由器作为主路由器，其他的则作为备份。这种机制确保了当主路由器出现故障时，可以快速地将流量转移到备份路由器上，从而实现高可用性。



我有三台虚拟机「13.133.222.3/13.133.222.4/13.133.222.5」可以将一台虚拟机作为LVS调度器（负载均衡器），其他两台作为真实服务器（后端服务器）。这里，我们假设13.133.222.3作为LVS调度器，13.133.222.4和13.133.222.5作为后端服务器。以下是一个基于LVS的简单配置示例，我们将采用LVS的NAT模式进行配置。

### 环境准备

- LVS调度器（Load Balancer）：13.133.222.3
- 真实服务器1：13.133.222.4
- 真实服务器2：13.133.222.5
- 虚拟服务IP（VIP）：13.133.222.13（假设）
- 服务端口：80（HTTP服务为例）

### 步骤1：配置LVS调度器

1. **安装ipvsadm工具**（在13.133.222.3上执行）

   ```bash
   sudo yum install ipvsadm -y
   ```

2. **配置LVS规则**（选择NAT模式，以HTTP服务为例）

   ```bash
   # 添加虚拟服务
   sudo ipvsadm -A -t 13.133.222.13:80 -s rr
   
   # 添加真实服务器
   sudo ipvsadm -a -t 13.133.222.13:80 -r 13.133.222.4:80 -m
   sudo ipvsadm -a -t 13.133.222.13:80 -r 13.133.222.5:80 -m
   ```

   这里`-s rr`表示使用轮询（round-robin）算法，`-m`表示NAT模式。

3. **启用IP转发**（确保LVS调度器能够转发包）

   ```bash
   echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
   ```

### 步骤2：配置真实服务器

对于13.133.222.4和13.133.222.5上的每台服务器，确保它们能够处理转发到它们的请求。在NAT模式下，服务器需要将返回流量路由到LVS调度器。这通常意味着调整默认网关或特定路由，使得回应客户端的流量通过LVS调度器。

1. **配置默认网关**（或特定路由，以13.133.222.3为网关）

   ```bash
   sudo ip route add default via 13.133.222.3
   ```

2. **配置和启动Web服务**（确保两台后端服务器都运行了HTTP服务）

   ```bash
   # 以安装和启动Apache HTTP服务器为例
   sudo yum install httpd -y
   echo "Hello from $(hostname)" | sudo tee /var/www/html/index.html
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

### 步骤3：验证配置

1. **查看LVS规则**

   ```bash
   sudo ipvsadm -Ln
   ```

2. **从客户端测试**

   访问虚拟服务IP（13.133.222.13）上的HTTP服务，可以使用浏览器或命令行工具如`curl`：

   ```bash
   curl http://13.133.222.13
   ```

   多次执行这个命令应该可以看到请求被轮流分发到两台真实服务器。

![image-20240225175231438](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240225175231438.png)