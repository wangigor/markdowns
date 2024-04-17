# Pod详解

## 完整YAML

YAML格式的Pod定义文件完整内容如下：

```yaml
apiVersion: v1        　　          #必选，版本号，例如v1,版本号必须可以用 kubectl api-versions 查询到 .
kind: Pod       　　　　　　         #必选，Pod
metadata:       　　　　　　         #必选，元数据
  name: string        　　          #必选，Pod名称
  namespace: string     　　        #必选，Pod所属的命名空间,默认为"default"
  labels:       　　　　　　          #自定义标签
    - name: string      　          #自定义标签名字
  annotations:        　　                 #自定义注释列表
    - name: string
spec:         　　　　　　　            #必选，Pod中容器的详细定义
  containers:       　　　　            #必选，Pod中容器列表
  - name: string      　　                #必选，容器名称,需符合RFC 1035规范
    image: string     　　                #必选，容器的镜像名称
    imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像,否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]     　　        #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]      　　             #容器的启动命令参数列表
    workingDir: string                     #容器的工作目录
    volumeMounts:     　　　　        #挂载到容器内部的存储卷配置
    - name: string      　　　        #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string                 #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean                 #是否为只读模式
    ports:        　　　　　　        #需要暴露的端口库号列表
    - name: string      　　　        #端口的名称
      containerPort: int                #容器需要监听的端口号
      hostPort: int     　　             #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string                  #端口协议，支持TCP和UDP，默认TCP
    env:        　　　　　　            #容器运行前需设置的环境变量列表
    - name: string      　　            #环境变量名称
      value: string     　　            #环境变量的值
    resources:        　　                #资源限制和请求的设置
      limits:       　　　　            #资源限制的设置
        cpu: string     　　            #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string                  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:       　　                #资源请求的设置
        cpu: string     　　            #Cpu请求，容器启动的初始可用数量
        memory: string                    #内存请求,容器启动的初始可用数量
    livenessProbe:      　　            #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:       　　　　　　        #对Pod容器内检查方式设置为exec方式
        command: [string]               #exec方式需要制定的命令或脚本
      httpGet:        　　　　        #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:      　　　　　　#对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
    restartPolicy: [Always | Never | OnFailure] #Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject   　　    #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:     　　　　#Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork: false      　　    #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:        　　　　　　    #在该pod上定义共享存储卷列表
    - name: string     　　 　　    #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}      　　　　    #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string      　　    #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:       　　　　　　    #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:      　　　　            #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
          path: string
```

## pod中运行的主程序需要在前台运行

pod运行有一个原则：**==pod中运行的主程序需要一直在前台运行。==**

- 如果创建的docker镜像的启动命令是后台执行程序，例如`nohup ./xxxxx.sh &`，kubectl创建完这个容器的pod之后「运行完该命令」就会认为pod执行结束，将立刻销毁pod。
- 如果配置了ReplicationController，检测到pod已经终止，会根据RC中定义的副本数replicas重新生成新的pod。
- 就会陷入无限循环。

> Kubernetes 的 Pod 必须运行前台命令而非后台命令的原因主要与 Kubernetes 的设计理念和容器的运行机制有关。
>
> 1. **进程监控与管理**：Kubernetes 通过 Pod 中的主进程（PID 1）来监控和管理容器的生命周期。如果该进程终止，Kubernetes 会根据容器的重启策略决定是否重启容器。前台运行的应用使得 Kubernetes 能够直接管理和监控这个进程，确保容器的健康和响应状态。
> 2. **日志管理**：前台运行的应用通常会将日志输出到标准输出（stdout）和标准错误（stderr）。Kubernetes 可以捕获这些输出并提供日志管理功能，如日志轮转、日志访问等。如果应用在后台运行，它可能不会直接输出到 stdout 或 stderr，这会使日志管理变得更加复杂。
> 3. **信号处理**：在前台运行可以让应用更好地响应系统信号（如 SIGTERM），从而优雅地关闭和处理清理任务。Kubernetes 在需要停止容器时会发送 SIGTERM 信号给 PID 1 进程，如果应用在后台运行，可能无法正确接收和处理这些信号。

**pod跟docker的container是一对多的关系。**

以下面这个pod的yaml为例

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-php
  labels:
    name: redis-php
spec:
  containers:
    - name: frontend
      image: kubeguide/guestbook-php-frontend:localredis
      ports:
        - containerPort: 80
    - name: redis
      image: kubeguide/redis-master
      ports:
        - containerPort: 6379
```

一个pod包含两个容器`frontend`、`redis-master`。

![image-20240226163508857](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240226163508857.png)

通过`kubectl describe pod redis-php`可以查看pod的详细信息，pod中的两个容器对应了两个docker的contrainerId。

![image-20240226164342469](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240226164342469.png)

进入redis的docker的container，执行`curl localhost`，可以访问正常的frontend。

![image-20240226180311472](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240226180311472.png)

> 让我们通过一个更简单的方式来解释为什么 Kubernetes 的 Pod 中包含的多个容器可以通过 `localhost` 互通，以及这背后的原理。
>
> 想象一下，如果你有一台电脑，在这台电脑上运行了几个不同的程序，这些程序都想要相互通信。如果它们都在同一台电脑上，一个程序可以简单地通过 `localhost` 来和另一个程序通信，因为它们共享同一台电脑的网络环境。在这种情况下，`localhost` 就像是一个内部电话系统，允许这台电脑上的程序直接相互联系。
>
> 在 Kubernetes 的世界里，一个 Pod 类似于一台虚拟的电脑。Pod 中的容器就像是运行在这台虚拟电脑上的程序。由于这些容器共享 Pod 的网络环境（就像程序共享电脑的网络环境），它们可以使用 `localhost` 来相互通信。这种设计允许容器之间的通信变得非常简单和高效，因为它们可以直接通过内部网络（`localhost`）相互联系，无需复杂的网络配置。
>
> 这个共享网络的概念是通过 Kubernetes 和容器技术内部的一些特定设置实现的，主要是：
>
> - **网络命名空间**：在创建 Pod 时，Kubernetes 为这个 Pod 设置了一个网络命名空间。网络命名空间是一种技术，可以让一组进程（在这个情况下，就是容器）拥有独立的网络环境（如 IP 地址、端口号等）。所有在同一个 Pod 内的容器都会被放入这个相同的网络命名空间中，因此它们看到的网络环境是一样的。
> - **`localhost` 通信**：因为所有容器都在同一个网络命名空间内，当一个容器尝试通过 `localhost` 发送数据时，这个数据实际上是在同一个网络命名空间内部传输的。这就像在同一台电脑上的不同程序之间的通信一样简单。
>
> 这种设计使得在同一个 Pod 中的容器之间的通信非常高效，同时还保持了 Kubernetes 管理容器的灵活性和易用性。

## 静态pod

静态Pod是由kubelet进行管理的仅存在于特定Node上的Pod。

它们不能通过API Server进行管理，无法与ReplicationController、Deployment或者DaemonSet进行关联，并且kubelet无法对它们进行健康检查。静态Pod总是由kubelet创建的，并且总在kubelet所在的Node上运行。

- 不能手动删除pod

  ```php
          # kubectl delete pod static-web-node1
          pod "static-web-node1" deleted
  
          # kubectl get pods
          NAME                 READY     STATUS        RESTARTS   AGE
          static-web-node1   0/1       Pending       0           1s
  ```

- 删除配置文件-删除pod

  ```php
          # rm /etc/kubelet.d/static-web.yaml
          # kubectl get pods
          // 无容器运行
  
  ```
创建静态Pod有两种方式：配置文件方式和HTTP方式。
一个简单的nginx做静态pod用。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-nginx-pod
  labels:
    role: myrole
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```

- **配置文件**==「推荐使用」==

  1. **配置 kubelet 使用静态 Pod**：确保 kubelet 配置了 `--pod-manifest-path` 参数，指向包含静态 Pod 配置文件的目录。如果使用 `kubelet` 配置文件 (`--config` 参数指定的文件)，则应该在配置文件中设置 `staticPodPath`。

     ```
     staticPodPath: "/etc/kubernetes/manifests"
     ```

  2. **放置配置文件**：将 `static-nginx-pod.yaml` 文件放入 kubelet 的静态 Pod 路径（例如 `/etc/kubernetes/manifests`）。

  3. **kubelet 自动创建 Pod**：**kubelet 会自动检测到配置文件的更改，并根据静态 Pod 的定义创建和管理 Pod**。

- **HTTP方式**

  Kubernetes 的 `kubelet` 允许通过 `--manifest-url` 参数从一个 HTTP 服务器加载静态 Pod 的配置。这种方式使得管理静态 Pod 配置更加灵活，尤其是在需要从中央位置批量管理多个节点的静态 Pod 配置时。下面是使用 `--manifest-url` 方式配置静态 Pod 的一个基本示例：

  **1. 创建静态 Pod 的 YAML 文件**

  首先，你需要创建一个包含一个或多个 Pod 定义的 YAML 文件。假设有一个简单的静态 Pod 定义如下（`static-pod.yaml`）：

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: static-web
    labels:
      role: myrole
  spec:
    containers:
      - name: web
        image: nginx
        ports:
          - name: web
            containerPort: 80
            protocol: TCP
  ```

  **2. 将 YAML 文件托管在 HTTP 服务器上**

  将这个 YAML 文件放置在一个 HTTP 服务器上，确保 `kubelet` 能够访问。例如，如果你将文件放置在 `http://example.com/static-pod.yaml`，确保这个 URL 是可访问的。

  **3. 配置 `kubelet` 使用 `--manifest-url`**

  你需要在启动 `kubelet` 时通过 `--manifest-url` 参数指定 YAML 文件的 URL。如果 `kubelet` 是作为 systemd 服务运行，你可以通过编辑 `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` 文件（或者对应的配置文件），添加或修改 `--manifest-url` 参数：

  ```bash
  --manifest-url=http://example.com/static-pod.yaml
  ```

  如果你同时使用 `--config` 参数指定了一个配置文件，你也可以在该配置文件中设置 `manifestURL` 和 `manifestURLHeader`（如果需要的话）字段：

  ```yaml
  apiVersion: kubelet.config.k8s.io/v1beta1
  kind: KubeletConfiguration
  manifestURL: "http://example.com/static-pod.yaml"
  manifestURLHeader:
    X-My-Custom-Header: ["MyValue"]
  ```

  注意：如果你使用配置文件方式设置 `manifestURL`，确保在 `kubelet` 的启动参数中指向了这个配置文件。

  **4. 重新加载并重启 `kubelet` 服务**

  ```bash
  sudo systemctl daemon-reload
  sudo systemctl restart kubelet
  ```

  **注意事项**

  - **安全性**：通过 HTTP 加载静态 Pod 配置可能会引入安全风险，特别是如果不是通过 HTTPS 进行的话。确保你了解这种方式可能带来的安全影响。
  - **访问控制**：如果你的 HTTP 服务器需要认证，你可以在 `kubelet` 配置文件中使用 `manifestURLHeader` 字段提供必要的认证信息（如 Basic Auth、Token 等）。

  通过这种方式，你可以更灵活地管理和更新静态 Pod 的配置，无需直接访问每个节点的文件系统。



## pod中的多个容器共享存储卷volume

同一个Pod中的多个容器能够共享Pod级别的存储卷Volume。Volume可以被定义为各种类型，多个容器各自进行挂载操作，将一个Volume挂载为容器内部需要的目录。

![image-20240227153022647](https://wangigor-typora-images.oss-cn-chengdu.aliyuncs.com/image-20240227153022647.png)

在Pod内包含两个容器：tomcat和busybox，在Pod级别设置Volume“app-logs”，用于tomcat向其中写日志文件，busybox读日志文件。配置文件pod-volume-applogs.yaml的内容如下：

```yml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
    - name: tomcat
      image: tomcat
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: app-logs
          mountPath: /usr/local/tomcat/logs
    - name: busybox
      image: busybox
      command: ["sh", "-c", "tail -f /logs/catalina*.log"]
      volumeMounts:
        - name: app-logs
          mountPath: /logs
  volumes:
    - name: app-logs
      emptyDir: {}
```

- 设置的Volume名为app-logs，类型为emptyDir。
- 挂载到tomcat容器内的/usr/local/tomcat/logs目录，同时挂载到logreader容器内的/logs目录。
- tomcat容器在启动后会向/usr/local/tomcat/logs目录写文件，logreader容器就可以读取其中的文件了。



logreader容器的启动命令为`tail -f /logs/catalina*.log`。

```less
# kubectl logs volume-pod -c busybox
............
29-Jul-2016 12:55:59.626 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deploying web application directory /usr/local/tomcat/webapps/manager
29-Jul-2016 12:55:59.722 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /usr/local/tomcat/webapps/manager has finished in 96 ms
29-Jul-2016 12:55:59.740 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["http-apr-8080"]
29-Jul-2016 12:55:59.794 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["ajp-apr-8009"]
29-Jul-2016 12:56:00.604 INFO [main] org.apache.catalina.startup.Catalina.start Server startup in 4052 ms
```

这个文件为tomcat生成的日志文件/usr/local/tomcat/logs/catalina.<date>.log的内容。登录tomcat容器进行查看。

```less
# kubectl exec -ti volume-pod -c tomcat -- ls /usr/local/tomcat/logs
catalina.2016-07-29.log     localhost_access_log.2016-07-29.txt
host-manager.2016-07-29.log  manager.2016-07-29.log

# kubectl exec -ti volume-pod -c tomcat -- tail /usr/local/tomcat/logs/catalina.2016-07-29.log
............
29-Jul-2016 12:55:59.722 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /usr/local/tomcat/webapps/manager has finished in 96 ms
29-Jul-2016 12:55:59.740 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["http-apr-8080"]
29-Jul-2016 12:55:59.794 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["ajp-apr-8009"]
29-Jul-2016 12:56:00.604 INFO [main] org.apache.catalina.startup.Catalina.start Server startup in 4052 ms
```



## pod配置ConfigMap

ConfigMap 是一种 API 对象，用于**存储非机密性的数据（键值对或文本配置内容）**，使你能够将配置与容器镜像分离，从而保持应用程序的容器化和配置的灵活性。**ConfigMap 允许你为容器化的应用程序提供配置数据，而无需修改应用程序的镜像。**这对于在不同环境中部署相同应用时保持配置的灵活性尤为重要。

- **配置数据的存储和管理**：可以将应用配置如数据库的地址、环境变量或配置文件存储在 ConfigMap 中。
- **环境变量**：将 ConfigMap 中的数据作为环境变量传递给 Pod 中的容器。
- **命令行参数**：使用 ConfigMap 中的数据作为容器启动的命令行参数。
- **配置文件**：在 Pod 的卷中创建配置文件，并使用 ConfigMap 中的数据填充这些文件。

ConfigMap有两种数据结构，一个是**key-value形式的键值对**，一个是**名称-内容的文件类型**。

### 创建ConfigMap

- **yaml方式「推荐」**

  ```yml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: example-configmap
  data:
    # 键-值对
    database_url: "https://example.com/db"
    feature_enabled: "true"
    # 可以包含配置文件内容
    config.json: |
      {
          "key": "value",
          "items": [1, 2, 3]
      }
  ```

  文件内容可以采用不同的方式

  ```yaml
  
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: cm-appconfigfiles
  data:
    key-serverxml: |
      <?xml version='1.0' encoding='utf-8'?>
      <Server port="8005" shutdown="SHUTDOWN">
        <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
        <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
      <Listener className=
      "org.apache.catalina.core.JreMemoryLeakPreventionListener" />
      <Listener className=
      "org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
      <Listener className=
      "org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
      <GlobalNamingResources>
      <Resource name="UserDatabase" auth="Container"
      type="org.apache.catalina.UserDatabase"
      description="User database that can be updated and saved"
      factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
      pathname="conf/tomcat-users.xml" />
      </GlobalNamingResources>
      
      <Service name="Catalina">
      <Connector port="8080" protocol="HTTP/1.1"
      connectionTimeout="20000"
      redirectPort="8443" />
      <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
      <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
      <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
      resourceName="UserDatabase"/>
      </Realm>
      <Host name="localhost"  appBase="webapps"
      unpackWARs="true" autoDeploy="true">
      <Valve className="org.apache.catalina.valves.AccessLogValve"
      directory="logs"
      prefix="localhost_access_log" suffix=".txt"
      pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      
      </Host>
      </Engine>
      </Service>
      </Server>
    key-loggingproperties: "handlers
      =1catalina.org.apache.juli.FileHandler, 2localhost.org.apache.juli.FileHandler,3manager.org.apache.juli.FileHandler, 4host-manager.org.apache.juli.FileHandler,java.util.logging.ConsoleHandler\r\n\r\n.handlers= 1catalina.org.apache.
      juli.FileHandler,java.util.logging.ConsoleHandler\r\n\r\n1catalina.org.apache.juli.FileHandler.level
      = FINE\r\n1catalina.org.apache.juli.FileHandler.directory
      = ${catalina.base}/logs\r\n1catalina.org.apache.juli.FileHandler.prefix
      = catalina.\r\n\r\n2localhost.org.apache.juli.FileHandler.level
      = FINE\r\n2localhost.org.apache.juli.FileHandler.directory
      = ${catalina.base}/logs\r\n2localhost.org.apache.juli.FileHandler.prefix
      = localhost.\r\n\r\n3manager.org.apache.juli.FileHandler.level
      = FINE\r\n3manager.org.apache.juli.FileHandler.directory
      = ${catalina.base}/logs\r\n3manager.org.apache.juli.FileHandler.prefix
      = manager.\r\n\r\n4host-manager.org.apache.juli.FileHandler.level
      = FINE\r\n4host-manager.org.apache.juli.FileHandler.directory
      = ${catalina.base}/logs\r\n4host-manager.org.apache.juli.FileHandler.
      prefix = host-manager.\r\n\r\njava.util.logging.ConsoleHandler.level
      = FINE\r\ njava.util.logging.ConsoleHandler.formatter
      = java.util.logging.SimpleFormatter\r\n\r\n\r\norg.apache.catalina.core.
      ContainerBase.[Catalina].[localhost].level
      = INFO\r\norg.apache.catalina.core.ContainerBase.[Catalina].[localhost].
      handlers
      = 2localhost.org.apache.juli.FileHandler\r\n\r\norg.apache.catalina.core.
      ContainerBase.[Catalina].[localhost].[/manager].level
      = INFO\r\norg.apache.catalina.core.ContainerBase.[Catalina].[localhost].[/manager].handlers
      = 3manager.org.apache.juli.FileHandler\r\n\r\norg.apache.catalina.core.
      ContainerBase.[Catalina].[localhost].[/host-manager].level
      = INFO\r\norg.apache.catalina.core.ContainerBase.[Catalina].[localhost].
      [/host-manager].handlers
      = 4host-manager.org.apache.juli.FileHandler\r\n\r\n"
  ```

  **创建configMap**:`kubectl apply -f appConfigMap.yaml`。

  **查看创建好的configMap详情**:`kubectl describe configmap cm-appconfigfiles`

  **以yaml方式查看configMap配置**:`kubectl get cm cm-appconfigfiles -o yaml`

- kubectl命令行方式创建

  命令行参数：

  - `--from-file`

    把文件设置为参数，并可以指定key「默认文件名」

    ```shell
    kubectl create configmap NAME --from-file=[key=]source --from-file=[key=]source
    ```

    也可以把文件目录中的所有文件都批量设置为参数

    ```shell
    kubectl create configmap NAME --from-file=config-files-dir
    ```

  - `--from-literal`

    按照「字面意思」手动创建key-value的configMap

    ```shell
    kubectl create configmap NAME --from-literal=key1=value1 --from-literal=key2=value2
    ```

    > 对于字面量的key-value，推荐使用yaml

### ConfigMap使用

#### 环境变量

示例ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-appvars
data:
  apploglevel: info
  appdatadir: /var/data
```

示例Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-test-pod
spec:
  containers:
    - name: cm-test
      image: busybox
      command: [ "/bin/sh", "-c", "env | grep APP" ]
      env:
        - name: APPLOGLEVEL             # 定义环境变量的名称
          valueFrom:                     # key“apploglevel”对应的值
            configMapKeyRef:
              name: cm-appvars          # 环境变量的值取自cm-appvars：
              key: apploglevel          # key为apploglevel
        - name: APPDATADIR              # 定义环境变量的名称
          valueFrom:                     # key“appdatadir”对应的值
            configMapKeyRef:
              name: cm-appvars          # 环境变量的值取自cm-appvars
              key: appdatadir           # key为appdatadir
  restartPolicy: Never
```

示例pod只是为了输出pod中的环境变量，执行完成就会退出「restartPolicy: Never不重启」。

查看pod日志，说明容器内部的环境变量使用ConfigMap cm-appvars中的值进行了正确设置。

```shell
# kubectl logs cm-test-pod
  APPDATADIR=/var/data
  APPLOGLEVEL=info
```

***

Kubernetes从1.6版本开始，引入了一个新的字段envFrom，实现了在Pod环境中将ConfigMap（也可用于Secret资源对象）中所有定义的key=value自动生成为环境变量：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-test-pod
spec:
  containers:
    - name: cm-test
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
        - configMapRef:
            name: cm-appvars      # 根据cm-appvars中的所有key=value自动生成环境变量
  restartPolicy: Never
```

**注意**：环境变量的名称受POSIX命名规范`[a-zA-z_][a-zA-Z0-9_]*`约束，不能以数字开头。如果包含非法字符，则系统将跳过该条环境变量的创建，并记录一个Event来提示环境变量无法生成，但并不阻止Pod的启动。

#### configMap中的文件挂载进pod

以`configMap.cm-appconfigfiles`为例，配置了两个文件`key-serverxml`和`key-loggingproperties`。

定义一个pod，把两个文件挂在到pod中的`/configfiles`目录：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-test-app
spec:
  containers:
    - name: cm-test-app
      image: kubeguide/tomcat-app:v1
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: serverxml                    # 引用Volume的名称
          mountPath: /configfiles           # 挂载到容器内的目录
  volumes:
    - name: serverxml                      # 定义Volume的名称
      configMap:
        name: cm-appconfigfiles           # 使用ConfigMap“cm-appconfigfiles”
        items:
          - key: key-serverxml               # key=key-serverxml
            path: server.xml                 # value将server.xml文件名进行挂载
          - key: key-loggingproperties     # key=key-loggingproperties
            path: logging.properties        # value将logging.properties文件名进行挂载
```

**如果在引用ConfigMap时不指定items，则使用volumeMount方式在容器内的目录下为每个item都生成一个文件名为key的文件。**



**注意**：

- **ConfigMap需要在Pod之前创建。**

- **受到Namespace限制。处于相同Namespace中才可以引用。**

- **只支持受kubelet API Server管理的pod使用ConfigMap。本节点上的静态pod无法引用ConfigMap。**

- **pod对ConfigMap进行挂载「volumeMount」操作时，只能挂载为目录，不能挂载成文件。**

  **挂载操作会对pod中的原目录覆盖，如果需要保留其他文件，需要ConfigMap挂载到临时目录，在启动脚本中复制或者链接到目录下。**

## Downward API「在容器内获取Pod信息」

Kubernetes 的 Downward API 允许容器获取关于自身或所在 Pod 的信息，而无需通过 Kubernetes API 直接查询这些信息。这种机制使容器能够访问运行它们的环境的数据，例如 Pod 的名字、命名空间、节点名称、以及其他与 Pod 相关的元数据和系统资源。

### 使用场景

Downward API 主要用于以下场景：

- **将 Pod 信息传递给容器**：例如，将 Pod 的 IP 地址、所在的节点名称或者 Pod 名称作为环境变量传递给应用程序。
- **动态配置调整**：基于 Pod 的资源分配（如 CPU、内存限制）动态调整应用的配置。
- **日志和监控**：在应用日志或监控数据中包含 Pod 信息，以便更好地跟踪和关联应用行为和性能指标。

### 实现方式

Downward API 可以通过环境变量和卷文件两种方式实现。

- 环境变量

  你可以通过 Pod 定义中的 `env` 字段使用 Downward API 将 Pod 的信息作为环境变量传递给容器。例如，将 Pod 名称和命名空间作为环境变量传递给容器：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-demo
spec:
  containers:
    - name: main-container
      image: nginx
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
```

- 卷文件

Downward API 也可以通过卷的方式将信息以文件形式提供给容器。这允许你将 Pod 信息写入到一个卷中，容器可以从中读取信息。例如，创建一个包含 Pod 名称和节点名称的文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-demo-vol
spec:
  containers:
    - name: main-container
      image: nginx
      volumeMounts:
        - name: podinfo
          mountPath: "/etc/podinfo"
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "podName"
            fieldRef:
              fieldPath: metadata.name
          - path: "nodeName"
            fieldRef:
              fieldPath: spec.nodeName
```

这会在 `/etc/podinfo` 目录下创建 `podName` 和 `nodeName` 两个文件，文件内容分别是 Pod 的名称和节点名称。

### 限制和注意事项

- **安全考虑**：使用 Downward API 时，应注意不要无意中将敏感信息暴露给容器。
- **性能影响**：频繁访问 Downward API 卷中的信息可能会对节点的性能造成影响，特别是在大规模集群中。

Downward API 提供了一种灵活的方法，使得容器能够更自然地与其运行环境集成，同时避免了直接依赖 Kubernetes API，从而提高了应用的可移植性和效率。

## 声明周期

Pod 的生命周期是指从 Pod 创建到结束的过程，这个周期中 Pod 会经历多个阶段。了解 Pod 的生命周期对于管理和调试 Kubernetes 应用是非常重要的。Pod 的主要状态包括：

### 1. **Pending**

- **含义**：Pod 已被 Kubernetes 系统接受，但是一个或多个容器镜像还未创建。等待的原因可能包括调度延迟、镜像拉取延迟等。
- **处理**：Kubernetes 会尝试调度 Pod 到合适的节点并拉取所需的容器镜像。

### 2. **Running**

- **含义**：Pod 已经被绑定到了一个节点上，所有的容器都已被创建，至少有一个容器正在运行、正在启动或正在重启。
- **处理**：Pod 正在执行其工作负载。

### 3. **Succeeded**

- **含义**：Pod 中的所有容器都正常运行完成并且已经退出了。这个状态通常适用于短暂的任务。
- **处理**：Pod 完成了其生命周期，不会再被调度到任何节点上运行。

### 4. **Failed**

- **含义**：Pod 中的所有容器已停止，至少有一个容器以失败状态（非零退出状态）终止。这意味着至少有一个容器未能成功完成其任务。
- **处理**：根据重启策略，Pod 可能会被 Kubernetes 尝试重启。

### 5. **Unknown**

- **含义**：Pod 的状态无法由 Kubernetes 确定，通常是因为与 Pod 所在节点的通信出了问题。
- **处理**：系统会尝试重新建立与节点的通信，以获取准确的 Pod 状态。

### Pod 生命周期中的重要事件

- **初始化容器（Init Containers）**：在 Pod 中的应用容器启动之前运行的一种特殊容器。它们用于包含一些应用镜像不包含的实用程序或安装脚本。
- **就绪探针（Readiness Probes）**：用于检查容器是否已准备好开始接受流量。只有当容器通过就绪探针检查时，才会被视为 Ready。
- **存活探针（Liveness Probes）**：用于检查容器是否在运行。如果存活探针检查失败，容器会被重启。
- **重启策略**：控制 Pod 中的容器在退出时是否应该被重启。策略包括 Always、OnFailure 和 Never。

## 重启策略

 重启策略是 Kubernetes 在管理 Pods 时用于决定如何响应容器失败的机制。

这些策略决定了在容器退出时 Kubernetes 应该做什么。重启策略适用于 Pod 中的所有容器。

Kubernetes 提供了三种重启策略：

1. **Always**:
   - **适用场景**: 当你希望容器在退出后总是被重启时使用，不论退出状态码是什么。
   - **行为**: 无论容器退出的状态码是什么，Kubernetes 都会尝试重新启动该容器。这是默认策略，适用于需要持续运行的服务。

2. **OnFailure**:
   - **适用场景**: 当你只希望在容器非正常退出（退出状态码非零）时重启容器。
   - **行为**: 如果容器因为错误而退出（即退出状态码非零），Kubernetes 会重新启动该容器。如果容器正常结束（退出状态码为零），则不会重新启动。

3. **Never**:
   - **适用场景**: 当你不希望容器在退出后被重启时使用。
   - **行为**: 容器不论因何种原因退出都不会被重启。

### 如何配置重启策略

你可以在 Pod 的定义中通过设置 `spec.restartPolicy` 字段来指定重启策略，如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: myimage
  restartPolicy: Always # 或者 OnFailure, Never
```

### 注意事项

- **Job 和 CronJob**: 对于 Kubernetes Job 和 CronJob 资源，默认的重启策略是 `OnFailure`。这是因为这些资源通常用于执行一次性任务或定时任务，任务完成后通常不需要重启容器。
- **重启计数器**: Kubernetes 会跟踪 Pod 中容器的重启次数，并在 `kubectl get pods` 命令的输出中显示。
- **影响**: 选择合适的重启策略对于应用的可靠性和资源利用率至关重要。例如，使用 `Always` 策略可能会导致失败的容器不断重启，消耗系统资源。

> Always 和 OnFailure 重启策略在 Kubernetes 中的主要区别在于它们各自触发容器重启的条件。
>
> Always 重启策略
>
> 	•	触发条件: 不论容器以何种状态码退出（无论是正常退出还是异常退出），Kubernetes 都会尝试重启该容器。
> 	•	使用场景: 最适合那些需要持续运行的应用，例如 web 服务器、API 服务等。这些服务通常需要在遇到任何问题时自动恢复，确保服务的可用性。
>
> OnFailure 重启策略
>
> 	•	触发条件: 仅当容器以非零状态码（即，失败或错误）退出时，Kubernetes 才会尝试重启该容器。如果容器正常结束并以状态码 0 退出，表示任务成功完成，Kubernetes 不会重新启动该容器。
> 	•	使用场景: 适用于执行有明确结束条件的任务，如批处理作业、数据处理任务或其他一次性任务。这些任务在遇到可恢复的错误时需要重启以尝试重新执行，但在正常完成后不需要重启。
>
> 主要区别
>
> 	•	重启触发条件: Always 无条件重启，OnFailure 仅在失败时重启。
> 	•	适用类型的任务: Always 适用于需要持续运行的服务，OnFailure 适用于可能会失败并需要重试的一次性任务。
> 	•	资源消耗和管理: 使用 Always 策略的服务可能在出现不可恢复错误时不断重启，消耗更多资源；而 OnFailure 策略允许在任务成功完成后停止，可能更高效地使用资源。
> 	•	应用场景的适应性: Always 重启策略更适合那些需要24/7运行的应用，而 OnFailure 更适合有明确执行完成条件的任务，需要在任务失败时尝试重启。
>
> 选择正确的重启策略对于确保应用的稳定性和效率至关重要，应基于应用的特定需求和行为来决定。



## 健康检查和服务可用性检查

kubelet会定期执行三类探针来判断容器的健康状况。

- `LivenessProbe`存活探针

  用于判断容器是否存活「Running状态」，如果探针检测到不健康「**这个不健康要根据不同的配置而定**」，则会杀死容器，并根据容器的重启策略做响应处理。

  如果一个容器没有配置存活探针，那么kubelet认为容器的存活探针永远返回「健康」。

- `ReadinessProbe`就绪探针

  用于确定容器是否准备好接受流量。就绪探针不会导致容器重启。如果就绪探针检测失败，该容器的ip地址会从与之关联的所有`service`的`endpoints`中移除。这意味着流量不会被发送到这个容器。当就绪探针再次成功时，容器ip会被重新添加到`endpoints`中，恢复流量接收。

- `StartupProbe`启动探针

  用于检测应用程序容器是否已经启动完成。这对于启动时间较长的应用来说尤其有用，因为它们可能需要更多的时间来加载数据或执行初始化任务，而在这个期间，容器可能还不是“活”的（live）或“就绪”的（ready）状态。如果设置了`startupProbe`，那么直到它成功之前，`livenessProbe`和`readinessProbe`都不会开始检测。这样可以防止应用在启动期间因`livenessProbe`失败而被杀掉。

> Kubernetes中的`startupProbe`和`readinessProbe`虽然都用于检查容器状态，但它们的目的和影响是不同的。它们在使用上确实有重要的区别，这些区别体现在它们的应用场景和对Pod行为的影响上。
>
> - `startupProbe`
>
>   **目的**：`startupProbe`用于确定应用程序是否已成功启动。这对于启动时间较长的应用尤其重要，因为这些应用可能需要更长的时间来加载数据或执行初始化任务。
>
>   **行为**：
>
>   - 如果`startupProbe`失败，Kubernetes将重启容器，直到探针成功。
>   - `startupProbe`成功后，它不再进行检测，此时Kubernetes开始运行`livenessProbe`和`readinessProbe`。
>   - `startupProbe`主要防止容器在还未完全启动时就被`livenessProbe`错误地认为是死亡的。
>
> - `readinessProbe`
>
>   **目的**：`readinessProbe`用来检查容器是否准备好接受流量。当应用程序启动后，这个探针确保只有当应用真正准备好服务请求时，才会开始接收客户端流量。
>
>   **行为**：
>
>   - 如果`readinessProbe`失败，表示容器暂时无法接受流量，Kubernetes会将该容器的IP地址从所有关联的Service的Endpoints中移除。
>   - 当`readinessProbe`成功时，容器的IP地址被添加回Endpoints列表中，容器开始接受流量。
>   - `readinessProbe`可以持续运行于容器生命周期的任何阶段，用于动态管理容器的流量接受状态。
>
> **关键区别**
>
> - **启动阶段**：`startupProbe`只在容器启动阶段重要，用以保证应用已经完全启动。一旦应用启动成功，`startupProbe`就完成了其任务。而`readinessProbe`在容器的整个生命周期中都可能运行，用于控制服务流量。
> - **服务流量**：`readinessProbe`直接控制容器是否可以接受服务流量，即是否加入到Service的Endpoints中。而`startupProbe`的成功或失败不影响服务流量的路由，它仅影响容器的启动和重启行为。
>
> **使用建议**
>
> - **长时间启动的应用**：如果你的应用需要较长时间来启动，使用`startupProbe`来避免应用在完全启动之前被`livenessProbe`误判为死亡。
>
> - **动态服务就绪性**：对于需要频繁调整其就绪状态的应用（例如，依赖于外部资源或在处理大量数据时），应该使用`readinessProbe`来管理其加入或退出`Service`的`Endpoints`。
>
>   `readinessProbe`特别适用于那些状态可能会频繁变化的应用，比如：
>
>   - **依赖于外部资源的服务**：如果服务依赖于外部数据库或其他服务，当这些外部依赖不可用时，可以通过探针检查来临时停止接受流量，直到外部依赖恢复。
>   - **资源密集型操作**：在进行大规模数据处理或更新操作时，可能不希望接收新的请求，可以通过失败的`readinessProbe`来暂时退出服务池。
>   - **应用内部状态**：应用可能基于内部逻辑或状态变化（如缓存重建、数据同步等）暂时不接受新请求。

### 探针的三种检测方式

配置格式。

```yml
spec:         　　　　　　　            #必选，Pod中容器的详细定义
  containers:       　　　　            #必选，Pod中容器列表
  - name: string      　　                #必选，容器名称,需符合RFC 1035规范
    image: string     　　                #必选，容器的镜像名称
    livenessProbe/startupProbe/readinessProbe: #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket
      exec:       　　　　　　        #对Pod容器内检查方式设置为exec方式
        command: [string]               #exec方式需要制定的命令或脚本
      httpGet:        　　　　        #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:      　　　　　　#对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged: false
```

对于启动探针startupProbe、就绪探针readinessProbe、存活探针livenessProbe，可配置的检测方式都是下面这三种

#### ExecAction执行容器内命令检测「在容器内部执行一个命令，如果该命令的返回码为0，则表明容器健康。」

在下面的例子中，通过执行“cat /tmp/health”命令来判断一个容器运行是否正常。在该Pod运行后，将在创建/tmp/health文件10s后删除该文件，而LivenessProbe健康检查的初始探测时间（initialDelaySeconds）为15s，探测结果是Fail，将导致kubelet杀掉该容器并重启它：

```yml
        apiVersion: v1
        kind: Pod
        metadata:
          labels:
            test: liveness
          name: liveness-exec
        spec:
          containers:
          - name: liveness
            image: gcr.io/google_containers/busybox
            args:
            - /bin/sh
            - -c
            - echo ok > /tmp/health; sleep 10; rm -rf /tmp/health; sleep 600
            livenessProbe:
              exec:
              command:
              - cat
              - /tmp/health
              initialDelaySeconds: 15
              timeoutSeconds: 1
```

#### TCPSocketAction IP端口检测「通过容器的IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康。」

在下面的例子中，通过与容器内的localhost:80建立TCP连接进行健康检查。

```yml
        apiVersion: v1
        kind: Pod
        metadata:
          name: pod-with-healthcheck
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            livenessProbe:
              tcpSocket:
                port: 80
              initialDelaySeconds: 30
              timeoutSeconds: 1
```

#### HTTPGetAction 响应状态检测「通过容器的IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康。」

通过容器的IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康。

```yml
        apiVersion: v1
        kind: Pod
        metadata:
          name: pod-with-healthcheck
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            livenessProbe:
              httpGet:
              path: /_status/healthz
              port: 80
              initialDelaySeconds: 30
              timeoutSeconds: 1
```

