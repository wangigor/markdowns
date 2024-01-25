# etcd

etcd是一个开源的、分布式的键值存储系统，用于可靠地存储关键数据并保证数据的一致性。它是一个核心组件，常被用于分布式系统中作为元数据和配置数据的存储。etcd的主要特点包括：

1. **分布式和强一致性**：etcd使用Raft算法来处理日志复制，以确保强一致性。这意味着它可以在分布式系统中可靠地存储数据，并确保所有节点之间的数据一致。

2. **键值存储**：etcd提供简单的键值存储接口，可以存储、检索和删除键值对。

3. **高可用性**：etcd设计为高可用的服务，可以在多个节点上运行。在发生故障时，它可以自动选举新的领导者，从而确保服务的连续性。

4. **观察者和监视机制**：etcd允许客户端观察键值的变化。这对于实时更新配置或协调分布式系统中的各种组件非常有用。

5. **多版本并发控制（MVCC）**：etcd支持多版本并发控制，使其可以处理同时对数据进行读写的情况，而不会产生数据冲突。

6. **安全性**：etcd支持SSL/TLS加密，用于节点之间的通信和客户端到节点的通信，确保数据传输的安全性。它还支持访问控制列表（ACLs）来管理对键值的访问权限。

7. **应用场景**：etcd常用于服务发现、配置管理、分布式系统的协调和状态管理等场景。例如，Kubernetes用它来存储所有集群数据，确保数据的一致性和可靠性。

总之，etcd是一个为分布式系统设计的可靠、高效和安全的键值存储解决方案。

## 部署

### 单节点部署

#### 方案一：centos-压缩包部署

为了在CentOS系统上安装单节点的etcd，我们将遵循以下步骤：

1. **更新系统包**
   - 打开终端。
   - 执行命令 `sudo yum update` 以更新所有系统包。

   > 这个命令会更新CentOS系统上所有安装的包。这是一个好的实践，以确保系统安全和软件兼容性。
   
2. **安装etcd**
   
   - 下载最新版的etcd。你可以从[etcd发布页面](https://github.com/etcd-io/etcd/releases)找到最新版本。
   - 假设我们下载的是`etcd-v3.5.0-linux-amd64.tar.gz`，使用以下命令下载并解压：
     ```
     wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
     tar xzvf etcd-v3.5.0-linux-amd64.tar.gz
     ```
   
3. **移动etcd二进制文件到系统路径**
   
   - 将`etcd`和`etcdctl`二进制文件移动到`/usr/local/bin`目录下：
     ```
     sudo mv etcd-v3.5.0-linux-amd64/etcd /usr/local/bin
     sudo mv etcd-v3.5.0-linux-amd64/etcdctl /usr/local/bin
     ```
   
4. **配置etcd服务**
   - 创建一个etcd用户：
     ```
     sudo adduser etcd
     ```
   - 创建etcd的默认数据和配置目录：
     ```shell
     sudo mkdir -p /var/lib/etcd/ # 创建etcd的数据存储目录。
     sudo mkdir -p /etc/etcd # 创建etcd的配置文件目录。
     sudo chown -R etcd:etcd /var/lib/etcd/ #将数据目录的所有权赋予etcd用户。
     ```
   - 创建etcd服务文件`/etc/systemd/system/etcd.service`并添加以下配置：
     ```conf
     # 描述了服务的元信息。
     [Unit]
     Description=etcd key-value store
     Documentation=https://github.com/coreos
     
     # 定义了如何启动服务、服务类型、重启策略等。
     [Service]
     User=etcd
     Type=notify
     ExecStart=/usr/local/bin/etcd --data-dir=/var/lib/etcd/
     Restart=always
     RestartSec=10s
     LimitNOFILE=40000
     
     # 定义了如何安装这个服务，例如它依赖于哪个目标（target）
     [Install]
     WantedBy=multi-user.target
     ```
   
5. **启动和启用etcd服务**
   - 重新加载systemd以读取新的etcd服务文件：
     ```shell
     sudo systemctl daemon-reload
     ```
   - 启动etcd服务：
     ```shell
     sudo systemctl start etcd
     ```
   - 设置etcd服务开机自启：
     ```shell
     sudo systemctl enable etcd
     ```

6. **验证etcd安装**
   - 使用以下命令检查etcd服务状态：
     ```shell
     sudo systemctl status etcd
     ```
   - 或者使用`etcdctl`工具检查集群健康状况：
     ```shell
     etcdctl member list
     ```

现在，你应该有一个运行在CentOS上的单节点etcd实例。

#### 方案二：docker

为CentOS系统安装单节点etcd的另一种方法是通过使用Docker。这种方法简化了安装和配置过程，特别适合那些希望通过容器化部署快速启动etcd的用户。

**使用Docker安装etcd的步骤**

1. **安装Docker**
   
   - 确保你的系统已安装Docker。如果还未安装，可以通过以下命令安装：
     ```bash
     sudo yum install -y docker
     ```
   - 启动Docker服务：
     ```bash
     sudo systemctl start docker
     ```
   - （可选）设置Docker开机自启：
     ```bash
     sudo systemctl enable docker
     ```
   
2. **拉取etcd Docker镜像**
   - 使用以下命令拉取最新的etcd镜像：
     ```bash
     docker pull quay.io/coreos/etcd
     ```

3. **运行etcd容器**
   - 使用Docker运行etcd。以下命令将启动一个etcd实例，监听本地端口2379和2380：
     ```bash
     docker run -d -p 2379:2379 -p 2380:2380 --name etcd quay.io/coreos/etcd /usr/local/bin/etcd
     ```

4. **验证etcd容器运行情况**
   - 检查容器状态：
     ```bash
     docker ps
     ```
   - 如果需要进入etcd容器进行操作，可以使用：
     ```bash
     docker exec -it etcd /bin/sh
     ```

使用Docker安装etcd的优点是快速且易于管理。你可以轻松地启动、停止或重启容器，并且在不同环境之间迁移和复制etcd实例变得非常简单。

**注意事项**

- 确保你的系统允许2379和2380端口的流量。
- 容器化部署可能不适合所有生产环境。根据你的特定需求和安全考虑，选择最适合的部署方式。

#### 方案三：docker-compose

为了使用`docker-compose`在CentOS系统上安装单节点etcd，我们将遵循以下步骤：

1. **安装Docker和Docker Compose**
   
   - 如果尚未安装Docker，请先安装Docker。
   - 安装Docker Compose：
     ```bash
     sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
     sudo chmod +x /usr/local/bin/docker-compose
     ```
   
2. **创建Docker Compose文件**
   - 在一个新目录中创建一个`docker-compose.yml`文件。例如，在`~/etcd`目录中创建此文件。
   - 添加以下内容到`docker-compose.yml`：
     ```yaml
     version: '3'
     services:
       etcd:
         image: quay.io/coreos/etcd
         volumes:
           - /data/etcd:/etcd-data
         environment:
           ETCD_DATA_DIR: /etcd-data
           ETCD_ENABLE_V2: "true"
         command:
           - /usr/local/bin/etcd
           - --data-dir=/etcd-data
           - --name node1
           - --initial-advertise-peer-urls http://0.0.0.0:2380
           - --listen-peer-urls http://0.0.0.0:2380
           - --listen-client-urls http://0.0.0.0:2379
           - --advertise-client-urls http://0.0.0.0:2379
           - --initial-cluster node1=http://0.0.0.0:2380
         ports:
           - "2379:2379"
           - "2380:2380"
     ```

3. **启动etcd服务**
   - 在包含`docker-compose.yml`文件的目录中，运行以下命令来启动etcd服务：
     ```bash
     docker-compose up -d
     ```

4. **验证etcd服务**
   - 检查etcd服务的状态：
     ```bash
     docker-compose ps
     ```
   - 你也可以使用`etcdctl`命令行工具与etcd服务交互。

这种方法利用了Docker Compose的优势，使得部署和管理etcd服务变得更加简单和直观。通过使用`docker-compose.yml`文件，你可以很容易地配置etcd服务的参数，并在需要时轻松地修改和重新部署服务。

> 顺便提供docker-compose.yml的文件解释
>
> ```yaml
> version: '3'
> ```
> - 这一行指定了`docker-compose.yml`文件使用的Docker Compose文件格式版本。这里我们使用版本3，这是目前最常用的版本之一。
>
> ```yaml
> services:
> ```
> - `services`部分定义了所有需要运行的服务。在我们的情况下，我们只有一个服务，即etcd。
>
> ```yaml
>   etcd:
> ```
> - 这定义了一个名为`etcd`的服务。
>
> ```yaml
>     image: quay.io/coreos/etcd
> ```
> - 指定`etcd`服务使用的Docker镜像。这里我们从quay.io的CoreOS仓库中使用最新的etcd镜像。
>
> ```yaml
>     volumes:
>       - /data/etcd:/etcd-data
> ```
> - 这一部分定义了挂载卷。它将宿主机的`/data/etcd`目录映射到容器内的`/etcd-data`。这用于持久化etcd的数据，确保即使容器被删除，数据也不会丢失。
>
> ```yaml
>     environment:
>       ETCD_DATA_DIR: /etcd-data
>       ETCD_ENABLE_V2: "true"
> ```
> - `environment`节定义了在容器内部设置的环境变量。这里我们设置了etcd的数据目录和启用etcd的v2 API。
>
> ```yaml
>     command:
>       - /usr/local/bin/etcd
>       - --data-dir=/etcd-data
>       - --name node1
>       - --initial-advertise-peer-urls http://0.0.0.0:2380
>       - --listen-peer-urls http://0.0.0.0:2380
>       - --listen-client-urls http://0.0.0.0:2379
>       - --advertise-client-urls http://0.0.0.0:2379
>       - --initial-cluster node1=http://0.0.0.0:2380
> ```
> - `command`部分指定容器启动时运行的命令。这里包括设置etcd节点名称、广告和监听URL以及初始集群配置。
>
> ```yaml
>     ports:
>       - "2379:2379"
>       - "2380:2380"
> ```
> - `ports`部分将容器内的端口映射到宿主机的端口。etcd默认使用2379端口用于客户端请求和2380端口用于集群通信。
>

#### macos-Homebrew

1. Update homebrew:

```sh
$ brew update
```

1. Install etcd:

```sh
$ brew install etcd
```

1. Verify install

```sh
$ etcd --version
```

### 多机集群部署

> [官方集群部署文档](https://etcd.io/docs/v3.5/op-guide/clustering/)

部署一个由三个节点组成的etcd集群到三台独立的虚拟机（CentOS Linux 7）上，涉及以下详细步骤：

**步骤1：在所有虚拟机上安装etcd**

对于每台虚拟机（13.133.222.3、13.133.222.4、13.133.222.5），执行以下步骤：

1. **下载etcd**：
   - 使用`wget`来下载etcd的压缩包：
     ```bash
     wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
     ```

2. **解压etcd**：
   - 使用`tar`命令解压下载的文件：
     ```bash
     tar xzvf etcd-v3.5.0-linux-amd64.tar.gz
     ```

3. **移动etcd二进制文件**：
   - 将etcd和etcdctl二进制文件移动到系统路径中，以便可以从任何位置运行：
     ```bash
     sudo mv etcd-v3.5.0-linux-amd64/etcd /usr/local/bin/
     sudo mv etcd-v3.5.0-linux-amd64/etcdctl /usr/local/bin/
     ```

**步骤2：配置etcd集群**

在每台虚拟机上，根据其IP地址配置etcd。

1. **创建etcd数据目录**：
   - 为了存储etcd数据，需要创建一个目录：
     ```bash
     sudo mkdir -p /var/lib/etcd
     ```

2. **创建并配置etcd服务**：
   - 在每台虚拟机上，创建一个服务配置文件（例如`/etc/etcd.conf`），并根据每台机器的IP地址进行调整。以下是在13.133.222.3上的配置示例：
     ```ini
     name: 'etcd1'
     data-dir: '/var/lib/etcd'
     initial-advertise-peer-urls: 'http://13.133.222.3:2380'
     listen-peer-urls: 'http://13.133.222.3:2380'
     listen-client-urls: 'http://13.133.222.3:2379,http://127.0.0.1:2379'
     advertise-client-urls: 'http://13.133.222.3:2379'
     initial-cluster: 'etcd1=http://13.133.222.3:2380,etcd2=http://13.133.222.4:2380,etcd3=http://13.133.222.5:2380'
     initial-cluster-state: 'new'
     initial-cluster-token: 'etcd-cluster-1'
     ```
   - 对于13.133.222.4和13.133.222.5，相应地修改`name`、`initial-advertise-peer-urls`、`listen-peer-urls`、`listen-client-urls`和`advertise-client-urls`的值。

**步骤3：启动etcd服务**

在每台虚拟机上，使用以下命令启动etcd服务：

```bash
etcd --config-file /etc/etcd.conf >> /var/log/etcd.log 2>&1 &
```

**步骤4：验证集群状态**

在任意一台虚拟机上，运行以下命令来检查集群成员和状态：

```bash
etcdctl member list
```

**注意事项**

- 确保各个虚拟机之间的网络通信是畅通的，特别是在etcd使用的2379和2380端口上。

- 对于生产环境，建议使用更复杂的安全措施，比如TLS/SSL加密和身份验证。

- 对于故障转移和数据持久性，建议进一步研究和实施备份策略。

  > 要实现故障转移和数据持久性，我们需要在搭建的etcd集群上实施备份和恢复策略。
  >
  > 1. **定期备份**
  >    - etcd数据可以通过创建快照来备份。etcd提供了一个命令行工具`etcdctl`，可以用来创建快照。
  >    - 示例命令：`ETCDCTL_API=3 etcdctl snapshot save backup.db`
  >    - 这个命令会在本地创建一个`backup.db`文件，这是etcd数据的快照。
  >    - 快照应该定期进行，可以通过计划任务（如cron）来自动化这个过程。
  >
  > 2. **自动化备份**
  >    - 可以编写一个脚本来自动执行备份命令，并将快照保存到安全的位置，例如远程服务器或云存储。
  >    - 脚本示例（假设使用bash脚本）:
  >      ```bash
  >      #!/bin/bash
  >      BACKUP_DIR="/path/to/backup"
  >      ETCDCTL_API=3 etcdctl snapshot save $BACKUP_DIR/etcd-$(date +%Y-%m-%d_%H:%M:%S).db
  >      ```
  >    - 使用cron来定期运行这个脚本。
  >
  > 3. **故障恢复**
  >    - 如果etcd集群发生故障，可以使用快照来恢复数据。
  >    - 首先停止etcd服务，然后使用以下命令来恢复数据：
  >      - `etcdctl snapshot restore backup.db`
  >    - 恢复后的数据可以用来重新启动etcd服务。
  >
  > 4. **数据持久性**
  >    - 确保etcd的数据目录（默认为`/var/lib/etcd`）在持久化存储上，如硬盘或SSD。
  >    - 在虚拟化环境中，可以使用像是VMware的vSAN或者云提供商的持久化存储解决方案。
  >
  > 5. **监控和警报**
  >    - 对etcd集群进行监控，确保在发生异常时能及时发出警报。
  >    - 可以使用像Prometheus和Grafana这样的工具来监控etcd的性能和健康状态。
  >
  > 6. **测试故障转移和恢复**
  >    - 定期进行故障转移和数据恢复的演练，确保在真正的故障发生时，恢复过程可以顺利进行。
  >
  > 通过实施这些策略，可以有效地提高etcd集群的可靠性和容错能力，确保关键数据的安全。

这些步骤将在您的虚拟机上创建一个基本的etcd集群，适用于模拟生产环境的高可用部署。

> etcd启动告警 member xxxxxxxxxxx has already been bootstrapped
>
> - 方法一： 该节点已经被引导，可能是因为之前这个节点已经加入过集群了，但配置失败，在启动参数那里设置的是**new**，所以需要将 --initial-cluster-state=**new** 修改成  --initial-cluster-state=existing。 改别的设置文件均不可行，etcd.service才是启动项。 
>
> - 方法二：把所有etcd节点的 data-dir 文件 rm  /var/lib/etcd/member/*  -rf 然后重启。

### 创建systemd服务

为了创建一个`systemd`服务文件用于管理您的`etcd`服务，您需要创建一个名为`etcd.service`的文件，通常位于`/etc/systemd/system/`目录下。这个文件将包含`etcd`服务的配置，使其能够在系统启动时自动启动，并且能够通过`systemctl`命令进行管理。

以下是一个基本的`etcd.service`文件的示例：

```ini
[Unit]
Description=etcd key-value store
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd --config-file /etc/etcd.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

这个文件包含了三个部分：

1. `[Unit]`：提供了服务的描述和文档链接。
2. `[Service]`：指定了启动服务的命令、重启策略和重启延迟。
3. `[Install]`：指定了服务应该在哪个系统级别下启动。

请确保`ExecStart`指向您的`etcd`二进制文件的正确路径，并且指定了正确的配置文件路径。

完成后，您可以使用以下`systemctl`命令来管理`etcd`服务：

- 启动服务：`sudo systemctl start etcd`
- 停止服务：`sudo systemctl stop etcd`
- 使服务在启动时自动启动：`sudo systemctl enable etcd`
- 禁用自动启动：`sudo systemctl disable etcd`
- 查看服务状态：`sudo systemctl status etcd`

请确保在使用这些命令之前，您已经将`etcd.service`文件放置在了正确的位置，并且具有合适的权限。

### 设置TLS加密



1. **生成CA证书和私钥**：CA（证书颁发机构）用于签发其他TLS证书，确保集群通信的安全性。

2. **生成服务器证书和私钥**：对于etcd集群中的每个节点，需要生成其自己的TLS证书和私钥。这些证书由CA签发。

3. **生成客户端证书和私钥**：如果需要客户端与etcd集群安全通信，还需要生成客户端证书和私钥。

4. **配置etcd使用TLS**：修改etcd配置，使其使用生成的TLS证书和私钥，以确保节点之间的安全通信。

5. **配置客户端使用TLS**：如果etcd客户端需要安全通信，还需要配置它们使用TLS。



- **生成CA证书和私钥**

首先，使用以下命令创建CA证书和私钥：

```bash
openssl genrsa -out ca.key 2048
openssl req -new -x509 -key ca.key -out ca.crt -days 3650 -subj "/CN=etcd-ca"
```

- **生成服务器证书和私钥**

然后，为每个etcd节点生成证书和私钥。以下是针对单个节点的示例：

```bash
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=etcd-server" -config server.cnf
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 3650 -extensions v3_req -extfile server.cnf
```

这里的`server.cnf`是一个配置文件，定义了证书的使用范围和约束。它应该包含etcd服务器的IP地址和/或主机名。

- **生成客户端证书和私钥**

对于需要与etcd安全通信的客户端，也要生成对应的证书和私钥：

```bash
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj "/CN=etcd-client" -config client.cnf
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 3650 -extensions v3_req -extfile client.cnf
```

- **配置etcd使用TLS**

接下来，需要在etcd的配置中指定TLS证书和私钥的路径。在etcd的配置文件中添加或修改以下字段：

```yaml
peer-client-cert-auth: true
peer-trusted-ca-file: "/path/to/ca.crt"
peer-cert-file: "/path/to/server.crt"
peer-key-file: "/path/to/server.key"

client-cert-auth: true
trusted-ca-file: "/path/to/ca.crt"
cert-file: "/path/to/server.crt"
key-file: "/path/to/server.key"
```

- **配置客户端使用TLS**

```shell
etcdctl --cert="/path/to/client.crt" --key="/path/to/client.key" --cacert="/path/to/ca.crt" member list
```

