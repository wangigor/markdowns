# centos-stream-9的Kubernetes 1.26部署

## 环境准备

三台可以访问外网的服务器，我本机使用vagrant创建了三台Centos7的虚拟机

| hostname              | ip           |
| --------------------- | ------------ |
| kube-node01「master」 | 13.133.222.3 |
| kube-node02           | 13.133.222.4 |
| kube-node03           | 13.133.222.5 |

**系统版本**：centos-stream-9

**kubernetes版本**：v1.28.8

- **设置主机名**

  ```shell
  hostnamectl set-hostname <hostname> --static
  ```

- **设置host**

  ```shell
  cat >> /etc/hosts << EOF
  13.133.222.3 kube-node01
  13.133.222.4 kube-node02
  13.133.222.5 kube-node03
  EOF
  ```

- **设置时间同步**
  
  ```shell
  yum install -y chrony -y 
  systemctl enable --now chronyd 
  chronyc sources
  ```
  
- **yum换源**
  
  先备份
  
  ```shell
  cd /etc/yum.repos.d/
  mkdir -pv  bak
  mv -v * bak/
  ls -lhrt
  ```
  
  添加清华大学源`vim base.repo`
  
  ```shell
  [baseos]
  name=CentOS Stream $releasever - BaseOS
  #mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=BaseOS&infra=$infra
  baseurl=https://mirrors.ustc.edu.cn/centos-stream/9-stream/BaseOS/$basearch/os/
  gpgcheck=1
  enabled=1
  gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
  [appstream]
  name=CentOS Stream $releasever - AppStream
  #mirrorlist=http://mirrorlist.centos.org/?release=$stream&arch=$basearch&repo=AppStream&infra=$infra
  baseurl=https://mirrors.ustc.edu.cn/centos-stream/9-stream/AppStream/$basearch/os/
  gpgcheck=1
  enabled=1
  gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
  ```
  
  清理并生成缓存
  
  ```shell
  yum clean all
  yum makecache
  ```
  
- **安装基础工具**
  
  ```shell
  yum install -y wget vim net-tools telnet
  ```
  
- **关闭防火墙**

  ```shell
  #临时关闭
   systemctl stop firewalld
  #永久关闭
   systemctl disable firewalld
  ```

- **关闭selinux**

  ```shell
  #临时关闭
  $ setenforce 0
  #永久关闭
  $ sed -i '/selinux/s/enforcing/disabled/' /etc/selinux/config
  ```

- **关闭swap**

  ```shell
  #临时关闭
  $ swapoff -a
  #永久关闭
  $ sed -ri 's/.*swap.*/#&/' /etc/fstab
  ```

- **将桥接的 IPv4 流量传递到 iptables 的链**

  ```shell
  #启用必要的模块
  sudo modprobe overlay
  sudo modprobe br_netfilter
  
  cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
  overlay
  br_netfilter
  EOF
  
  cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  net.bridge.bridge-nf-call-ip6tables = 1
  EOF
  
  #配置生效
  sudo sysctl --system
  ```

- **安装containerd**

  ```shell
  sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
  
  sudo dnf update
  sudo dnf install -y containerd
  
  sudo mkdir -p /etc/containerd
  sudo containerd config default | sudo tee /etc/containerd/config.toml
  
  # 修改containerd配置
  sudo vi /etc/containerd/config.toml
  # 1、找到[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]并将值更改SystemdCgroup为true
  # 2、找到sandbox_image = "k8s.gcr.io/pause:3.6"并改为sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.6"
  
  # 重启以应用更改
  sudo systemctl restart containerd
  # 加入开机启动
  sudo systemctl enable containerd
  ```

  

## 安装k8s集群及网络插件（flannel）

- **添加kubernetes仓库**

  ```shell
  cat <<EOF > /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  enabled=1
  gpgcheck=0
  repo_gpgcheck=0
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
  EOF
  ```

- **安装Kubernetes modules**

  ```shell
  # 当前最新版本v1.28.8
  sudo dnf update
  sudo dnf install -y kubelet kubeadm kubectl
  # 加入开机启动
  sudo systemctl enable kubelet
  ```

- **部署kubernetes集群**

  - 在k8s-master节点上初始化集群

    ```shell
    # pod-network-cidr填写地址与后面安装的flannel插件默认的网段保持一致
    kubeadm init \
      --apiserver-advertise-address=13.133.222.3 \
      --image-repository registry.aliyuncs.com/google_containers \
      --pod-network-cidr=13.133.0.0/16 \
      --control-plane-endpoint=kube-node01
    ```

    > 如果这一步执行报错了，可以修改完成之后重置kubeadm
    >
    > `kubeadm reset`

    执行完成之后，注意日志

    ```shell
    Your Kubernetes control-plane has initialized successfully!
    
    To start using your cluster, you need to run the following as a regular user:
    
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    Alternatively, if you are the root user, you can run:
    
      export KUBECONFIG=/etc/kubernetes/admin.conf
    
    You should now deploy a pod network to the cluster.
    Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
      https://kubernetes.io/docs/concepts/cluster-administration/addons/
    
    You can now join any number of control-plane nodes by copying certificate authorities
    and service account keys on each node and then running the following as root:
    
      kubeadm join kube-node01:6443 --token 33ddfd.5wze60a804e242ig \
            --discovery-token-ca-cert-hash sha256:7eaf7378c33777a9a71e8592f42990d931db5ec38253a8b914e38fca610561a9 \
            --control-plane 
    
    Then you can join any number of worker nodes by running the following on each as root:
    
    kubeadm join kube-node01:6443 --token 33ddfd.5wze60a804e242ig \
            --discovery-token-ca-cert-hash sha256:7eaf7378c33777a9a71e8592f42990d931db5ec38253a8b914e38fca610561a9
    ```

    执行文件中的命令

    ```shell
      mkdir -p $HOME/.kube
      sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
      sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    ```shell
    export KUBECONFIG=/etc/kubernetes/admin.conf
    ```

    最后的这一行是其他node节点加入集群的命令，可以存下来。

    如果忘了，可以使用命令`sudo kubeadm token create --print-join-command`重新生成。

    ***

    安装flannel网络插件

    ```shell
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```

    过一段时间之后，master节点就安装好了。

    ```shell
    [root@localhost yum.repos.d]# kubectl get nodes -o wide
    NAME          STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE          KERNEL-VERSION          CONTAINER-RUNTIME
    kube-node01   Ready    control-plane   5m16s   v1.28.2   13.133.222.3   <none>        CentOS Stream 9   5.14.0-124.el9.x86_64   containerd://1.6.28
    ```

  - 其他节点加入集群

    执行加入集群命令

    ```shell
    kubeadm join kube-node01:6443 --token 33ddfd.5wze60a804e242ig \
            --discovery-token-ca-cert-hash sha256:7eaf7378c33777a9a71e8592f42990d931db5ec38253a8b914e38fca610561a9
    ```

    验证集群节点加入完成

    ```shell
    [root@localhost yum.repos.d]# kubectl get nodes -o wide
    NAME          STATUS   ROLES           AGE     VERSION   INTERNAL-IP    EXTERNAL-IP   OS-IMAGE          KERNEL-VERSION          CONTAINER-RUNTIME
    kube-node01   Ready    control-plane   7m10s   v1.28.2   13.133.222.3   <none>        CentOS Stream 9   5.14.0-124.el9.x86_64   containerd://1.6.28
    kube-node02   Ready    <none>          52s     v1.28.2   13.133.222.4   <none>        CentOS Stream 9   5.14.0-124.el9.x86_64   containerd://1.6.28
    kube-node03   Ready    <none>          50s     v1.28.2   13.133.222.5   <none>        CentOS Stream 9   5.14.0-124.el9.x86_64   containerd://1.6.28
    ```

    

