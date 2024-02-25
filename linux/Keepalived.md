# Keepalived

Keepalived是一个用于Linux操作系统的路由软件，主要用于提高服务器的高可用性（High Availability, HA）。它通过使用标准的VRRP（虚拟路由冗余协议）实现服务的高可用。Keepalived可以监控服务器上的服务或系统的状态，当检测到主服务器出现问题时，能够自动将请求切换到备用服务器上，确保服务的连续可用性。

Keepalived的主要功能包括：

1. **高可用性（HA）：** Keepalived通过VRRP协议支持IP地址的热备份，能够在服务器间实现自动故障转移。
2. **负载均衡：** 除了高可用性之外，Keepalived还支持LVS（Linux Virtual Server）负载均衡，可以根据配置的规则将客户端请求分发到多个服务器上。
3. **健康检查：** Keepalived可以对后端的服务或服务器进行健康检查，确保流量只会被转发到健康的服务器上。
4. **灵活的配置：** Keepalived允许用户通过配置文件灵活地设置VRRP以及健康检查的各种参数。

Keepalived在网络服务和云计算领域中被广泛使用，特别适合需要高可用性和负载均衡的环境，比如Web服务、数据库服务和云服务等。通过Keepalived，可以有效地避免单点故障，提高服务的稳定性和可靠性。

> **VRRP**（虚拟路由器冗余协议）本质上是一个协议规范，定义了在局域网内实现路由器高可用性的一种机制。它是一个标准化的协议，由IETF在RFC 5798文档中定义。VRRP允许多个路由器共享一个虚拟IP地址，其中一台路由器作为主路由器，其他的则作为备份。这种机制确保了当主路由器出现故障时，可以快速地将流量转移到备份路由器上，从而实现高可用性。



## 使用及验证

为了使用`keepalived`在你的三台虚拟机（13.133.222.3, 13.133.222.4, 13.133.222.5）上实现高可用性配置，我们将通过以下步骤进行：

### 1. 安装Keepalived

首先，在所有三台虚拟机上安装`keepalived`。以CentOS为例，使用yum进行安装：

```sh
sudo yum install keepalived -y
```

### 2. 配置Keepalived

我们将配置这三台虚拟机，其中一台作为主（Master）服务器，其他两台作为备份（Backup）服务器。我们将使用虚拟IP地址（VIP）13.133.222.13作为服务的高可用地址。

#### 主服务器配置（13.133.222.3）

在13.133.222.3上，编辑`/etc/keepalived/keepalived.conf`文件，添加如下配置：

```nginx
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        13.133.222.13/24
    }
}
```

- `state MASTER`表明这台服务器为主服务器。
- `interface eth0`可能需要根据你的网络接口名称进行调整。
- `virtual_router_id`是VRRP实例的ID，需要在所有服务器上保持一致。
- `priority`设置了服务器的优先级，主服务器的优先级应该最高。

#### 备份服务器配置（13.133.222.4和13.133.222.5）

在13.133.222.4和13.133.222.5上，进行类似的配置，只是将`state`改为`BACKUP`，并适当降低`priority`。

**13.133.222.4的配置**：

```nginx
vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        13.133.222.13/24
    }
}
```

**13.133.222.5的配置**：

调整`priority`为80或其他，以区分不同的备份服务器。

### 3. 启动Keepalived

在所有三台虚拟机上启动`keepalived`服务：

```sh
sudo systemctl start keepalived
sudo systemctl enable keepalived
```

### 4. 验证配置

- **检查Keepalived状态**：

  在每台服务器上检查`keepalived`的状态，确保服务正在运行：

  ```sh
  sudo systemctl status keepalived
  ```

- **查看VIP**：

  在每台服务器上使用`ip addr show`命令，检查虚拟IP地址是否已正确绑定到主服务器上。

- **测试高可用性**：

  从主服务器（13.133.222.3）上手动停止`keepalived`服务，看看VIP是否会自动转移到一个备份服务器上：

  ```sh
  sudo systemctl stop keepalived
  ```

  然后在其他服务器上再次使用`ip addr show`命令检查VIP的绑定情况。

### 注意

- 根据你的实际网络接口名称（如`eth0`），你可能需要调整配置文件中的`interface`值。
- 确保所有服务器的防火墙设置允许VRRP通信（使用IP协议编号112）。

通过这个基本的`keepalived`配置，你应该能够在三台虚拟机上实现一个简单的高可用性设置，其中一台作为主服务器，另外两台作为备份，以确保服务的连续可用性。