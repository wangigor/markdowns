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
