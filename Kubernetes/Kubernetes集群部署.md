# Kubernetes集群部署

此篇集群搭建为kubeadm在线搭建方式。

## 环境准备

三台可以访问外网的服务器，我本机使用vagrant创建了三台Centos7的虚拟机

| hostname    | ip             |
| ----------- | -------------- |
| kube-node01 | 192.168.56.100 |
| kube-node02 | 192.168.56.101 |
| kube-node03 | 192.168.56.102 |

**系统版本**：CentOS Linux release 7.9.2009 (Core)

**docker版本**「待装」：1.13.1

**kubernetes版本**「待装」：v1.22.0

## 初始化配置

每台机器都要做。

```shell
# 设置主机名
hostnamectl set-hostname <hostname> --static

# 添加host
cat >> /etc/hosts << EOF
192.168.56.100 kube-node01
192.168.56.101 kube-node02
192.168.56.102 kube-node03
EOF

# 关闭防火墙
iptables -F

# 关闭selinux
setenforce 0 
sed -i 's/SELINUX=/SELINUX=disabled/g' /etc/selinux/config

# yum换源
wget -O /etc/yum.repos.d/CentOS-Base.repo https://repo.huaweicloud.com/repository/conf/CentOS-7-reg.repo
yum install -y epel-release
sed -i "s/#baseurl/baseurl/g" /etc/yum.repos.d/epel.repo
sed -i "s/metalink/#metalink/g" /etc/yum.repos.d/epel.repo
sed -i "s@https\?://download.fedoraproject.org/pub@https://repo.huaweicloud.com@g" /etc/yum.repos.d/epel.repo

# 设置时间同步
yum install -y chrony -y 
systemctl enable --now chronyd 
chronyc sources

# 关闭swap
swapoff -a
vi /etc/fstab # 注释swap

# 修改内核参数
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
fs.may_detach_mounts = 1
EOF
sysctl -p /etc/sysctl.d/k8s.conf

# 加载内核模块
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- br_netfilter
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && \
bash /etc/sysconfig/modules/ipvs.modules && \
lsmod | grep -E "ip_vs|nf_conntrack_ipv4"

# 安装ipvm管理工具
yum -y install -y ipset ipvsadm

# 重启
reboot
```

## 安装containerd

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
yum install -y containerd.io cri-tools

#生成containerd的配置文件
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
```

修改vi /etc/containerd/config.toml配置文件以下内容

```toml
…
[plugins]
…
[plugins."io.containerd.grpc.v1.cri"]
…
#sandbox_image = "k8s.gcr.io/pause:3.2"
sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.5"
…
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
…
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]**
SystemdCgroup = true #对于使用 systemd 作为 init system 的 Linux 的发行版，使用 systemd 作为容器的 cgroup driver 可以确保节点在资源紧张的情况更加稳定
[plugins."io.containerd.grpc.v1.cri".registry]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
endpoint = ["https://pqbap4ya.mirror.aliyuncs.com"]
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
endpoint = ["https://registry.aliyuncs.com/k8sxio"]
…
```

重启

```shell
systemctl enable containerd --now
systemctl restart containerd
```

## 安装docker

```shell
# 安装
yum install -y docker
# 换源
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://zpitma3u.mirror.aliyuncs.com"]
}
EOF
# 设置开机自启
systemctl enable docker
# 重启docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```



## 安装kubelet/kubeadm/kubectl

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum list kubeadm --showduplicates
yum install -y kubelet-1.22.0 kubeadm-1.22.0 kubectl-1.22.0
systemctl enable kubelet --now
```

***

到目前未知三个节点初始化完成，但无法运行，只能保证开机自启动。

## 部署master节点

我选择`kube-node01`作为管理端。

先把默认的配置文件拷贝出来。生成用于初始化的`kubeadm.yaml`文件。

```shell
kubeadm config print init-defaults --component-configs KubeletConfiguration --component-configs KubeProxyConfiguration  > kubeadm.yaml
```

编辑`kubeadm.yaml`

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
 advertiseAddress: 192.168.56.100  # 设置master节点的ip地址
  bindPort: 6443
nodeRegistration:
 criSocket: unix:///run/containerd/containerd.sock # 设置containerd的连接套接字
  imagePullPolicy: IfNotPresent
 name: kube-node01 # 指定master的主机名
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers  # 指定下载master组件镜像的镜像仓库地址
kind: ClusterConfiguration
kubernetesVersion: 1.22.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16  # 指定pod ip的网段
scheduler: {}
```

**拉取镜像**

```shell
kubeadm config images pull --config kubeadm.yaml
```

**安装master节点**

```shell
kubeadm init --config kubeadm.yaml
```

看到下面内容就说明成功了。

```log
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
Then you can join any number of worker nodes by running the following on each as root:
kubeadm join 192.168.56.100:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:cad3fa778559b724dff47bb1ad427bd39d97dd76e934b9467507a2eb990a50c7
```

他的意思是，初始化成功，但是要进行集群部署，需要完成下面三步。

- ```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

- 安装网络

  网络我选择的是`flannel`

  ```shell
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
  ```

- 其他节点加入

  ```shell
  kubeadm join 192.168.56.100:6443 --token abcdef.0123456789abcdef \
      --discovery-token-ca-cert-hash sha256:cad3fa778559b724dff47bb1ad427bd39d97dd76e934b9467507a2eb990a50c7
  ```

  

## worker节点加入集群

工作节点加入集群只需要执行下面命令即可。

```shell
kubeadm join 192.168.56.100:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:cad3fa778559b724dff47bb1ad427bd39d97dd76e934b9467507a2eb990a50c7
```

回到工作节点可以查看节点准备情况

```shell
kubectl get nodes
```

```shell
[root@kube-node01 ~]# kubectl get nodes
NAME          STATUS   ROLES                  AGE   VERSION
kube-node01   Ready    control-plane,master   24h   v1.22.0
kube-node02   Ready    <none>                 24h   v1.22.0
kube-node03   Ready    <none>                 24h   v1.22.0
```

```shell
[root@kube-node01 ~]# kubectl get pods -A
NAMESPACE      NAME                                  READY   STATUS    RESTARTS      AGE
kube-flannel   kube-flannel-ds-5rkbb                 1/1     Running   3 (23h ago)   24h
kube-flannel   kube-flannel-ds-kjdld                 1/1     Running   3 (63m ago)   24h
kube-flannel   kube-flannel-ds-ph95p                 1/1     Running   4 (23h ago)   24h
kube-system    coredns-7f6cbbb7b8-fc7lc              1/1     Running   2 (23h ago)   24h
kube-system    coredns-7f6cbbb7b8-xk7tm              1/1     Running   2 (23h ago)   24h
kube-system    etcd-kube-node01                      1/1     Running   6 (63m ago)   24h
kube-system    kube-apiserver-kube-node01            1/1     Running   7 (63m ago)   24h
kube-system    kube-controller-manager-kube-node01   1/1     Running   7 (63m ago)   24h
kube-system    kube-proxy-2ntxf                      1/1     Running   3 (23h ago)   24h
kube-system    kube-proxy-gp4l6                      1/1     Running   3 (23h ago)   24h
kube-system    kube-proxy-nlgkj                      1/1     Running   3 (63m ago)   24h
kube-system    kube-scheduler-kube-node01            1/1     Running   7 (63m ago)   24h
```



至此，三节点kubernetes集群已经简单部署完成。



## 集群重置

在三个节点上可以执行`kubeadm reset`进行重置。

后续再通过`kubeadm init`就行。